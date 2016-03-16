pngs = action-command.png pipeline-barrier.png action-barrier-action.png action-barrier-action-mem.png

all: $(pngs)

$(pngs): %.png: %.tex deps-common.tex
	pdflatex -shell-escape $<
