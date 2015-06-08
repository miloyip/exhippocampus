../images/%.svg: %.tex
	pdflatex $<
	pdf2svg $(basename $<).pdf $@
	rm $(basename $<).aux $(basename $<).log $(basename $<).pdf

TEXFILES = $(basename $(wildcard *.tex))
all: $(addprefix ../images/, $(addsuffix .svg, $(TEXFILES)))
