# Vulkan memory dependencies

To understand memory barriers in Vulkan, it is helpful to understand the typical GPU memory architecture and why they need these barriers. Unfortunately there seems to be a lack of public documentation of how any of this works.

This document is an attempt to describe and explain the memory architecture and how Vulkan's memory model maps onto it. I'm not an expert and it may be full of mistakes (in which case I'd appreciate a pointer or explanation of how it really works!) so don't take it too literally.


## Overview of caches

Skip this section if you are already familiar with how caches work.

GPUs have access to a large amount of memory. In discrete GPUs this is dedicated VRAM on the graphics card, in integrated GPUs and SoCs it's usually the system RAM and is shared with the CPU.

This memory is a very long way away from the various processing units in the GPU that need to operate on data from memory - usually it's on a separate chip entirely - so it takes a long time to access and costs a lot of power.

Caches try to improve this performance. They rely on the observation that if you read a byte from memory you're likely to read it again very soon (e.g. a value in a uniform buffer that is read for every fragment); and if you read a byte from memory you're likely to read nearby bytes very soon (e.g. if you read one vertex's data from a vertex attribute buffer, you'll probably be reading the next vertex too).

Caches sit on the interface between the processing units and memory. When a single byte is read, the cache will request a larger amount from VRAM (perhaps 32 bytes or more, called a cache line). The cache will then store that whole cache line, so it can respond to subsequent read requests directly from that cached data instead of talking to VRAM again. The cache has a limited size (from kilobytes up to a few megabytes), so it will occasionally forget ('evict') old cache lines to make room for new ones.

Since the GPU contains many different types of processing units, and many parallel instances of each type, it has a large number and variety of caches that are each tuned for their particular requirements.

The major difficulty comes when data in VRAM is modified, while it is still cached in one or more caches. When that memory location is read via the cache, the cache may return its stale cached data instead of the latest copy from VRAM, resulting in unexpected and unpredictable behaviour. There are hardware techniques (*cache coherence protocols*) that let the cache detect when memory has been modified so it can drop its stale data and re-read from VRAM, but these are complex and excessively expensive when there are as many caches as a GPU has. The alternative is to rely on software to send a signal to the cache, warning it that a specific range of memory has been modified so the cache needs to drop the stale cache lines.

In OpenGL, the GL driver is (mostly) responsible for deciding when to send these signals. In Vulkan, that responsibility (mostly) lies with the application instead. The application has to use memory barriers to provide the signals, through `vkCmdPipelineBarrier`, subpass dependencies, and the other synchronisation features. If the application gets it wrong, it may still run correctly on some devices all of the time, and on some devices some of the time, but randomly fail on some other devices at the most inconvenient possible time.

The description above only considers caches that are used for memory reads. Some processing units can perform writes too, so the caches need to handle these. There are two general techniques (with lots of minor variations):

* Write-through cache: The write is passed directly down to the next level. Additionally, if the write touches a cache line that's currently in the cache, that copy of the cache line will be updated too. This means e.g. a shader can write to a variable then immediately read it back and get the correct value directly from the cache.
* Write-back cache: The write is not passed down immediately. Instead the cache line is updated, and a *dirty* flag is set on it. At some time in the future, the cache will *flush* the dirty cache line down to VRAM and remove the dirty flag. This means e.g. a shader can write to a single variable many times, and all those writes will be handled efficiently by the cache, and only the final result will be written to VRAM.

This means that when an application wants to write some data in one part of the GPU then read it in another part, it has to flush the writes from the first cache down into VRAM, then invalidate the second cache so it will re-fetch the latest value from VRAM.

As an extra complexity, caches can be nested: a processing unit can read from a small fast Level 1 (L1) cache, which reads from a larger slower L2 cache, which reads from huge very slow VRAM. In some GPUs there can be five or more levels of caches on top of the RAM. To coordinate properly between writes and reads in different parts of the system, they might need to flush and invalidate multiple caches in the correct sequence.


## GPU memory architecture

The memory architecture of a modern GPU might look a little bit like this:

```
.--------------------------.  .--------------------------.
| CU:                      |  | CU:                      |
|                          |  |                          |
| [WI] [WI] [WI] [WI] [WI] |  | [WI] [WI] [WI] [WI] [WI] |                                              .---------------.
|                          |  |                          |                                              | Host CPU:     |
| [ALU] [ALU] [ALU] [L/S]  |  | [ALU] [ALU] [ALU] [L/S]  |                                              |               |
|                          |  |                          |                                              | [Core] [Core] |
| [L1$]  [T$]  [U$]   [SM] |  | [L1$]  [T$]  [U$]   [SM] |                                              |   |       |   |
'---|-----|-----|----------'  '---|-----|-----|----------'                                              | [L1$]   [L1$] |
    |     |     |                 |     |     |                                                         |  _|_______|_  |
.---'-----'-----'-----------------'-----'-----'----------. .-------. .-------. .----------. .---------. | [____L2$____] |
|                                                        | |       | |       | |          | |         | |  _____|_____  |
|                        L2 cache                        | |  ROP  | |  ROP  | | Transfer | | Display | | [____L3$____] |
|                                                        | |       | |       | |          | |         | |       |       |
'---------------------------.----------------------------' '---.---' '---.---' '----.-----' '----.----' '-------|-------'
                            |                                  |         |          |            |              |
.---------------------------'----------------------------------'---------'----------'------------'--------------'-------.
|                                                                                                                       |
|                                                           VRAM                                                        |
|                                                                                                                       |
'-----------------------------------------------------------------------------------------------------------------------'
```

(This doesn't correspond to any one particular real GPU, it's a mixture of different ones based on the scant public documentation, plus a few guesses and a few intentional simplifications and many mistakes. The description here is focused on discrete GPUs over integrated ones, but the same general concepts should apply with some variation to all reasonable GPUs.)

At the top there are a number of *compute units* (CUs) (aka Streaming Multiprocessors, Execution Units, etc). Each CU contains the state for a number of *work items* (WIs, aka threads); the state includes an instruction pointer and a bunch of general purpose registers for thread-private data. Each CU contains a number of shader cores (ALUs for arithmetic operations, load/store units for memory operations, etc) that execute instructions on work items.

In this example, each CU also contains a number of different caches:
* L1 data cache (read-write, for general buffer accesses)
* Texture cache (read-only, for texture samplers)
* Uniform cache (read-only, for dynamically-uniform access patterns (i.e. where every work item in a subgroup is expected to read from the same location))

All these caches from all the CUs are connected to a single shared L2 cache. The L2 cache is connected to VRAM.

Each CU also contains a block of shared memory, which can be accessed by all the work items in a work group. A work group is defined to fit in a single CU, so there is no need to share shared memory outside the CU.

Additionally there are a number of *ROPs* (render output units / raster operations pipelines), which are responsible for all colour/depth/stencil framebuffer attachment accesses. They probably don't go through the L2 cache; instead they contain their own specialised caches for the attachments. If the device supports framebuffer compression, the ROP is responsible for accumulating a small tile of pixels in its cache and then compressing the tile before writing it to VRAM.

The host CPU has some connection to VRAM (perhaps over a PCIe bus), behind the CPU's own L1/L2 caches. The presentation engine (i.e. display controller) reads from VRAM, and there might be a transfer DMA engine that reads and writes VRAM directly.

### Coherency

In general, none of the GPU caches are coherent with each other - a write via one cache might not be seen by a later read from a different cache. They rely on invalidate/flush signals from software to maintain correct behaviour.

One important case is that an application's shader might choose to write to two adjacent bytes in a buffer from two concurrent work items in different CUs. The L1 cache must make sure this works correctly even when the writes are in the same cache line - both writes must eventually reach VRAM correctly merged together, without any explicit synchronisation from the software. Options include:

* Implement L1 as a write-through cache: every write will be immediately sent to L2 along with a byte mask (indicating precisely which bytes are meant to be updated). The L2 cache will update the appropriate bytes in its own copy of the cache line. The L1 cache can either update and keep its copy of the cache line, or evict it.
* Implement L1 as a write-back cache with a per-byte dirty mask. Every write will update the L1's data and mask. When the L1 decides to evict a cache line, it checks whether any bit in the dirty mask is set; if so then it flushes the line first, by sending the data and byte mask to L2, so only the dirty bytes get updated in L2. It may also flush lines eagerly to reduce the amount of dirty tracking needed. (If it's very eager then this is a lot like a write-combining cache.)
* Don't have an L1 cache.

The L2 cache should not be write-through (that would eliminate a lot of its performance benefit), and cannot afford per-byte dirty mask (cache is already very expensive and the masks would cost an extra 12.5%), so it is likely to be write-back with per-cache-line dirty bits.

A shader can access variables defined with the `Coherent` decoration in SPIR-V; those accesses are defined to be coherent between different work items (potentially in different CUs), when they are accessing the same bytes through the same buffer view or image view. That means reads and writes to these variables must always go down to the L2 level, they cannot be satisfied from L1. `Coherent` shader variables don't need to be coherent with any non-shader accesses, so they don't have to go all the way to VRAM.

Atomic operations are defined to be coherent with `Coherent` variables, so they have to be implemented similarly. (In the hardware the atomic behaviour can't be supported by the compute unit itself - the atomic request is typically passed down to the L2 cache, which may receive requests from many work items simultaneously and will execute them all with the correct atomicity.)

An application might write to one byte from a shader, and write to an adjacent byte from a transfer operation. This requires the transfer unit to be coherent with the L2 cache, which means there must be some hardware coherency support.


### Attachments

Writes to framebuffer attachments through the ROPs are not coherent with L2, and the ROPs usually write tiles rather than individual pixels - it would break if a shader tried to write to some pixels via an attachment and adjacent pixels via a storage image. The specification therefore (tries to) forbid this case - an image mustn't be accessed by any non-attachment path while it's being used as an attachment in a render pass. (Specifically, the ROP caches need to be flushed/invalidated between use as an attachment and as a non-attachment.)

Input attachments are a bit tricky: an attachment can be used as both colour (or equivalently depth/stencil) and input in a single subpass. If the colour components written via the colour attachment and read via the input attachment are different, this is specified to be fine as there is no feedback loop - it doesn't matter whether the writes are seen by the reads or not. If there is feedback, Vulkan requires an explicit pipeline barrier. This means it is possible for a device to implement input attachments as textures with appropriate flushing/invalidating at the barrier, while other devices (mainly tile-based renderers) might implement them through the same data path as colour attachments.


### Host CPU accesses

Memory accesses from the host have to deal with the CPU caches, which are quite different from GPU caches. In Vulkan, memory types with the `HOST_VISIBLE` flag can have three caching modes:

* `HOST_COHERENT`: Usually implemented with write-combining on the CPU. Writes are cached a little bit - bytes written to the same cache line are collected in a write buffer, and eventually the write buffer will be flushed to VRAM in a single memory transaction. If only a partial cache line has been written, it may be sent with a mask or split into multiple transactions, so only the relevant bytes will be updated in VRAM. The timing of the "eventually" is important: pretty much any other operation that might be visible to the GPU (writes to hardware registers or uncached memory, atomics, etc) will automatically flush the write buffer, so the GPU will never receive a signal saying data is available before the data is actually available. Reads are fully uncached on the CPU and will always read from VRAM (so they are very expensive).

* `HOST_CACHED`: Cached on the CPU. All reads and writes use the CPU's L1/L2/L3 caches. After the CPU has done some writes, `vkFlushMappedMemoryRanges` will flush dirty cache lines to VRAM. Before the CPU does any reads or writes, `vkInvalidateMappedMemoryRanges` will invalidate the CPU caches, to let it fetch the latest data from VRAM. Since the CPU don't have per-byte dirty flags, these flushes/invalidates can't work at a byte granularity - they work at `VkPhysicalDeviceLimits::nonCoherentAtomSize` granularity instead (typically the CPU cache line size of 64 bytes). Since the cache contains whole cache lines, if you write to only part of a cache line then the CPU will have to fetch the rest of the line from VRAM; that means this can actually be more expensive than write-combining memory for writes.

* `HOST_CACHED` and `HOST_COHERENT`: Cached on the CPU, with some hardware mechanism to automatically ensure coherency between the GPU and CPU caches.


## Memory dependencies

Vulkan defines the concept of memory dependencies or memory barriers to let the application provide any necessary cache-maintenance hints.

Source memory dependencies are defined for writes of specifics types (`srcAccessMask`) in specific pipeline stages (`srcStageMask`), and they make these writes *available* to subsequent accesses.

Destination memory dependencies are defined for reads and writes of specifics types (`dstAccessMask`) in specific pipeline stages (`dstStageMask`), and they make earlier available writes *visible* to subsequent accesses.

To ensure the sequence of write, make-available, make-visible and read will occur in the right order, there must be execution dependencies between them. Both halves of the memory dependency can be provided in a single `vkCmdPipelineBarrier` command, or they can be provided in separate barriers as long as there is some execution dependency chain between them.

At the hardware level, the combination of `accessMask` and `stageMask` identifies a specific cache or set of caches. Making a write 'available' means flushing a cache so the write ends up in some appropriate level of the memory hierarchy, and making it 'visible' means invalidating a cache so that subsequent reads will be satisfied from that same level of the memory hierarchy.

Some care is needed when invalidating non-read-only caches: it is legal to have a make writes visible to a cache that already contains some dirty data in the same cache line, and that dirty data must not be lost. That means the invalidate must really be implemented as a flush-then-invalidate operation.

There is no point ever specifying a `READ` access type in `srcAccessMask` - it doesn't make sense to flush a read, only a write. `WRITE` in `dstAccessMask` is trickier: for example in a write-through L1 cache it doesn't matter if there are stale cache lines, since writes will pass through stale cache lines just as well as fresh ones, so there is no need to invalidate the cache; but for a write-back L2 cache with per-cache-line dirty tracking it is still necessary to invalidate before a write, else stale bytes in the same cache line that don't get overwritten will later get flushed back to VRAM.

Applications must therefore use `WRITE` access types in `srcAccessMask`, and `READ` and/or `WRITE` in `dstAccessMask` depending on whether it's a read-after-write or write-after-write situation. Operations like atomics and fragment blending must use `READ|WRITE` in `dstAccessMask`.

We can look at our hypothetical GPU memory architecture and work out which cache operations are needed to implement memory dependencies. There are two reasonable-sounding choices as to which particular level of the memory hierarchy we should flush to: either VRAM or the L2 cache. We'll consider both options. We'll also make a few other assumptions:
* The L1 data caches are write-through, so they never need flushing.
* Input attachments are read like textures.

We've omitted some shaders (geometry, tessellation, compute) that are identical to vertex shaders. We've also combined the early and late fragment test stages into a single column, since they behave identically. In all these tables, "-" means a `stageMask`/`accessMask` combination that doesn't make sense, since those stages never make accesses of those types; any memory dependencies on these combinations will be ignored. The `VK_ACCESS_MEMORY_...` types seem to be specified too vaguely to figure out what they're meant to do.

Using VRAM as the level of coherency:

| srcAccessMask \ srcStageMask                  | DRAW_INDIRECT     | VERTEX_INPUT      | VERTEX_SHADER         | FRAGMENT_SHADER       | FRAGMENT_TESTS            | COLOR_ATTACHMENT_OUTPUT   | TRANSFER      | HOST          |
| ---                                           | ---               | ---               | ---                   | ---                   | ---                       | ---                       | ---           | ---           |
| VK_ACCESS_SHADER_WRITE_BIT                    | -                 | -                 | flush L2              | flush L2              | -                         | -                         | -             | -             |
| VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT          | -                 | -                 | -                     | -                     | -                         | flush ROP                 | -             | -             |
| VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT  | -                 | -                 | -                     | -                     | flush ROP                 | -                         | -             | -             |
| VK_ACCESS_TRANSFER_WRITE_BIT                  | -                 | -                 | -                     | -                     | -                         | -                         | nothing       | -             |
| VK_ACCESS_HOST_WRITE_BIT                      | -                 | -                 | -                     | -                     | -                         | -                         | -             | nothing       |
| VK_ACCESS_MEMORY_WRITE_BIT                    | ???               | ???               | ???                   | ???                   | ???                       | ???                       | ???           | ???           |

| dstAccessMask \ dstStageMask                  | DRAW_INDIRECT     | VERTEX_INPUT      | VERTEX_SHADER         | FRAGMENT_SHADER       | FRAGMENT_TESTS            | COLOR_ATTACHMENT_OUTPUT   | TRANSFER      | HOST          |
| ---                                           | ---               | ---               | ---                   | ---                   | ---                       | ---                       | ---           | ---           |
| VK_ACCESS_INDIRECT_COMMAND_READ_BIT           | invalidate L2,L1  | -                 | -                     | -                     | -                         | -                         | -             | -             |
| VK_ACCESS_INDEX_READ_BIT                      | -                 | invalidate L2,L1  | -                     | -                     | -                         | -                         | -             | -             |
| VK_ACCESS_VERTEX_ATTRIBUTE_READ_BIT           | -                 | invalidate L2,L1  | -                     | -                     | -                         | -                         | -             | -             |
| VK_ACCESS_UNIFORM_READ_BIT                    | -                 | -                 | invalidate L2,U$      | invalidate L2,U$      | -                         | -                         | -             | -             |
| VK_ACCESS_INPUT_ATTACHMENT_READ_BIT           | -                 | -                 | -                     | invalidate L2,T$      | -                         | -                         | -             | -             |
| VK_ACCESS_SHADER_READ_BIT                     | -                 | -                 | invalidate L2,L1,T$   | invalidate L2,L1,T$   | -                         | -                         | -             | -             |
| VK_ACCESS_SHADER_WRITE_BIT                    | -                 | -                 | invalidate L2         | invalidate L2         | -                         | -                         | -             | -             |
| VK_ACCESS_COLOR_ATTACHMENT_READ_BIT           | -                 | -                 | -                     | -                     | -                         | invalidate ROP            | -             | -             |
| VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT          | -                 | -                 | -                     | -                     | -                         | invalidate ROP            | -             | -             |
| VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT   | -                 | -                 | -                     | -                     | invalidate ROP            | -                         | -             | -             |
| VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT  | -                 | -                 | -                     | -                     | invalidate ROP            | -                         | -             | -             |
| VK_ACCESS_TRANSFER_READ_BIT                   | -                 | -                 | -                     | -                     | -                         | -                         | nothing       | -             |
| VK_ACCESS_TRANSFER_WRITE_BIT                  | -                 | -                 | -                     | -                     | -                         | -                         | nothing       | -             |
| VK_ACCESS_HOST_READ_BIT                       | -                 | -                 | -                     | -                     | -                         | -                         | -             | nothing       |
| VK_ACCESS_HOST_WRITE_BIT                      | -                 | -                 | -                     | -                     | -                         | -                         | -             | nothing       |
| VK_ACCESS_MEMORY_READ_BIT                     | ???               | ???               | ???                   | ???                   | ???                       | ???                       | ???           | ???           |
| VK_ACCESS_MEMORY_WRITE_BIT                    | ???               | ???               | ???                   | ???                   | ???                       | ???                       | ???           | ???           |

Using L2 as the level of coherency:

| srcAccessMask \ srcStageMask                  | DRAW_INDIRECT     | VERTEX_INPUT      | VERTEX_SHADER         | FRAGMENT_SHADER       | FRAGMENT_TESTS            | COLOR_ATTACHMENT_OUTPUT   | TRANSFER      | HOST          |
| ---                                           | ---               | ---               | ---                   | ---                   | ---                       | ---                       | ---           | ---           |
| VK_ACCESS_SHADER_WRITE_BIT                    | -                 | -                 | nothing               | nothing               | -                         | -                         | -             | -             |
| VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT          | -                 | -                 | -                     | -                     | -                         | flush ROP, invalidate L2  | -             | -             |
| VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT  | -                 | -                 | -                     | -                     | flush ROP, invalidate L2  | -                         | -             | -             |
| VK_ACCESS_TRANSFER_WRITE_BIT                  | -                 | -                 | -                     | -                     | -                         | -                         | invalidate L2 | -             |
| VK_ACCESS_HOST_WRITE_BIT                      | -                 | -                 | -                     | -                     | -                         | -                         | -             | invalidate L2 |
| VK_ACCESS_MEMORY_WRITE_BIT                    | ???               | ???               | ???                   | ???                   | ???                       | ???                       | ???           | ???           |

| dstAccessMask \ dstStageMask                  | DRAW_INDIRECT     | VERTEX_INPUT      | VERTEX_SHADER         | FRAGMENT_SHADER       | FRAGMENT_TESTS            | COLOR_ATTACHMENT_OUTPUT   | TRANSFER      | HOST          |
| ---                                           | ---               | ---               | ---                   | ---                   | ---                       | ---                       | ---           | ---           |
| VK_ACCESS_INDIRECT_COMMAND_READ_BIT           | invalidate L1     | -                 | -                     | -                     | -                         | -                         | -             | -             |
| VK_ACCESS_INDEX_READ_BIT                      | -                 | invalidate L1     | -                     | -                     | -                         | -                         | -             | -             |
| VK_ACCESS_VERTEX_ATTRIBUTE_READ_BIT           | -                 | invalidate L1     | -                     | -                     | -                         | -                         | -             | -             |
| VK_ACCESS_UNIFORM_READ_BIT                    | -                 | -                 | invalidate U$         | invalidate U$         | -                         | -                         | -             | -             |
| VK_ACCESS_INPUT_ATTACHMENT_READ_BIT           | -                 | -                 | -                     | invalidate T$         | -                         | -                         | -             | -             |
| VK_ACCESS_SHADER_READ_BIT                     | -                 | -                 | invalidate L1,T$      | invalidate L1,T$      | -                         | -                         | -             | -             |
| VK_ACCESS_SHADER_WRITE_BIT                    | -                 | -                 | nothing               | nothing               | -                         | -                         | -             | -             |
| VK_ACCESS_COLOR_ATTACHMENT_READ_BIT           | -                 | -                 | -                     | -                     | -                         | flush L2, invalidate ROP  | -             | -             |
| VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT          | -                 | -                 | -                     | -                     | -                         | flush L2, invalidate ROP  | -             | -             |
| VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT   | -                 | -                 | -                     | -                     | flush L2, invalidate ROP  | -                         | -             | -             |
| VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT  | -                 | -                 | -                     | -                     | flush L2, invalidate ROP  | -                         | -             | -             |
| VK_ACCESS_TRANSFER_READ_BIT                   | -                 | -                 | -                     | -                     | -                         | -                         | flush L2      | -             |
| VK_ACCESS_TRANSFER_WRITE_BIT                  | -                 | -                 | -                     | -                     | -                         | -                         | flush L2      | -             |
| VK_ACCESS_HOST_READ_BIT                       | -                 | -                 | -                     | -                     | -                         | -                         | -             | fiush L2      |
| VK_ACCESS_HOST_WRITE_BIT                      | -                 | -                 | -                     | -                     | -                         | -                         | -             | flush L2      |
| VK_ACCESS_MEMORY_READ_BIT                     | ???               | ???               | ???                   | ???                   | ???                       | ???                       | ???           | ???           |
| VK_ACCESS_MEMORY_WRITE_BIT                    | ???               | ???               | ???                   | ???                   | ???                       | ???                       | ???           | ???           |



## Aliasing

Most of our discussion has been about bytes, and the ability to write to adjacent bytes from different processing units. For buffers, the relationship between bytes and buffer elements is obvious. Vulkan allows multiple buffers to be bound to overlapping (*aliased*) regions of memory with well-defined results. Writes to different bytes through different buffers should work with no extra requirements. Accesses to the same bytes through different buffers require memory barriers - e.g. if you write to a byte through a storage buffer, then read it back through an aliased uniform buffer, you need to ensure the uniform cache has been invalidated before the read.

(TODO: Actually, doesn't that apply even when it's a single buffer bound as two separate descriptors? What prevents problems in that case?)

Images are more complicated. Vulkan says that linear images in `PREINITIALIZED` or `GENERAL` layouts have a well-defined layout in memory (called *host-accessible*) - you can determine exactly which bytes correspond to which pixels by calling `vkGetImageSubresourceLayout`. In these cases you can access the same underlying memory through an image or a buffer with well-defined behaviour, since they're just accessing the same bytes through different routes, and aliasing works the same as with buffers.

Linear images in other layouts, and optimal images, do not have a well-defined layout - any byte might correspond to any pixel, or even to multiple pixels (see the Framebuffer Compression section below). In these cases, if you write a pixel through an image then try to read a byte through a buffer, the result is unknown and undefined; and similarly if you write a byte through a buffer then try to read through an image.

Since there is not necessarily a 1:N relationship between pixels and bytes for optimal images, there may be difficulties when two different processing units try to write to nearby pixels of the same image. For example if a shader writes a single pixel, and the shader hardware knows how to write compressed images, it must load a whole tile from L2 then decompress and update and compress and write the whole tile back. If a transfer simultaneously tries to write a different pixel in the same tile, and supports compression in the same way, one of the updates will be lost. To avoid this situation the driver must avoid using compressed layouts whenever it is possible for multiple non-coherent units to write to an image (as indicated by its layout and usage flags).


## Framebuffer compression

Framebuffers typically make up a significant bandwidth cost - they are large and are often read and written multiple times per frame (for blending and postprocessing). But they are rarely full of random noise, they usually contain areas of smooth colour that can be losslessly compressed fairly easily, allowing a significant bandwidth saving.

In general, the idea is to split the framebuffer into tiles of maybe 8x8 pixels (256 bytes), where each tile corresponds to a reasonable multiple of the memory controller's transaction size - maybe it's a 256-bit memory bus so each 256B tile data is 8 transaction sizes. (The important thing about the transaction size is that bandwidth is counted in transactions, not bytes, so you want to send as few transactions as possible but fully utilise all the bytes in each transaction). For each framebuffer there is also a small array of compression metadata, with a few bits per tile.

The metadata determines how to interpret the tile data, and its states could include:

* Uncompressed: the 256B of tile data contains the raw RGBA values of all 8x8 pixels.
* 2:1 compressed: the first 128B of tile data contains a compressed representation of the 8x8 pixels. The second 128B contains garbage. That means reads/writes to this tile only need 4 memory transactions, not 8, reducing the bandwidth cost by half.
* 4:1 compressed: the first 64B of tile data contains a compressed representation.
* 8:1 compressed: the first 32B of tile data contains a compressed representation. (There's not much point compressing at 16:1 or more - it's still going to need at least one memory transaction per tile.)
* Cleared to (0,0,0,0): All of the tile data is garbage; any attempts to read this tile will just get the constant colour (0,0,0,0) without reading any tile data. This also means the device can clear an entire image extremely efficiently by writing only to the metadata array, it doesn't have to touch the tile data at all.
* Cleared to (0,0,0,255), (255,255,255,0), (255,255,255,255), maybe a few other useful constants: Same thing, you can just spend more metadata bits to make the fast clear path support more colours.

This means we need maybe 4 bits of metadata per tile, so it's only a ~0.5% increase in memory usage for all images, and a similar increase in bandwidth for incompressible data, to gain a significant decrease in bandwidth for compressible data.

(In practice some framebuffer compression mechanisms are quite different to this, but they all involve having some metadata table that describes how and/or where to read the tile data.)

The metadata might be stored alongside the image in its allocated device memory, or it might be stored in some special hidden memory. This means that opaque images are truly opaque: you can't use the host or a buffer-transfer command to copy their bytes to a different location and trust that they can still be used correctly as images, since you won't have copied the compression metadata.

Incidentally, an image could be transitioned from uncompressed to compressed very cheaply by simply filling its metadata array with the "uncompressed" flag. This greatly reduces the transition cost, at the expense of not optimising for any subsequent reads; but if the image is most likely to be read only once or never before being overwritten (which is typical for framebuffers), this seems a sensible optimisation. On the other hand, converting from compressed to uncompressed necessarily involves rewriting any tile data that was previously compressed, so it's worth trying hard to avoid decompression.


## Buffer image granularity

To support framebuffer compression, some implementations may associate the compression metadata with regions of memory, not with specific image resources, so the compression will work transparently for any processing unit that tries to read that memory without the need to pass compression flags through some other channel.

The compression-enabled flag and a pointer to the compression metadata can be conveniently stored inside the page tables, which are cached in TLBs. Any memory access will already have to do a TLB lookup for the virtual-to-physical translation, so it gets this compression information for free at the same time, and only has the extra cost of fetching the metadata when it knows the page is compressed. Pages are typically 64KB (it's not a coincidence that Vulkan's standard sparse image block shapes are 64KB), so each page contain 256 tiles, and needs maybe 128 bytes of metadata per page, which is a reasonable size (comparable to the L1 cache line size and the TLB entry size).

On the downside that means that if you store a compressed image somewhere, the entire surrounding 64KB page will always be interpreted as compressed, even if you actually wanted to store some uncompressed image or buffer nearby - so you need to keep compressed images separated from other resources by 64KB.

Beyond compression, a similar technique can be used for optimally-tiled images, with the tiling mode flags stored in the page table and the de-tiling performed automatically for any accesses to the page. That means optimal images can be on the same page as each other, but must be kept on separate pages from linear images and buffers.

`bufferImageGranularity` is Vulkan's way of exposing this requirement to applications. Some GPUs report a granularity of 64KB, matching the page size. Others have no such requirement, because the compression and tiling flags are part of the resource state instead (along with image format and dimensions etc), so they report a granularity of 1 byte. Unfortunately the variable name is actively misleading: it's not about buffers vs images, it's about optimal images vs everything else.


## References

A few sources with some useful information or explanations:

* NVIDIA:
  * http://arxiv.org/pdf/1509.02308.pdf ("Dissecting GPU Memory Hierarchy through Microbenchmarking")
  * http://www.eecg.toronto.edu/~myrto/gpuarch-ispass2010.pdf ("Demystifying GPU Microarchitecture through Microbenchmarking")
  * http://www.anandtech.com/show/8935/geforce-gtx-970-correcting-the-specs-exploring-memory-allocation/2
  * http://international.download.nvidia.com/pdf/tegra/Tegra-X1-whitepaper-v1.0.pdf
  * http://docs.nvidia.com/cuda/parallel-thread-execution/#cache-operators
  * http://envytools.readthedocs.io/en/latest/hw/memory/g80-vram.html
  * https://fgiesen.wordpress.com/2013/01/29/write-combining-is-not-your-friend/
* AMD:
  * https://www.amd.com/Documents/GCN_Architecture_whitepaper.pdf
  * http://gpuopen.com/vulkan-device-memory/
* Intel:
  * https://software.intel.com/en-us/file/the-compute-architecture-of-intel-processor-graphics-gen9-v1d0pdf
  * http://www.realworldtech.com/sandy-bridge-gpu/8/
* General:
  * https://fgiesen.wordpress.com/2011/07/12/a-trip-through-the-graphics-pipeline-2011-part-9/
  * https://fgiesen.wordpress.com/2011/10/09/a-trip-through-the-graphics-pipeline-2011-part-13/
  * https://fgiesen.wordpress.com/2013/01/29/write-combining-is-not-your-friend/
