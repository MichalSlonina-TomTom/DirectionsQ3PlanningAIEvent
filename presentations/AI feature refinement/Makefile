# Makefile — build the "Refining features with Claude" deck from Marp Markdown.
#
# Requires marp-cli (`brew install marp-cli`). PDF/PPTX export drives a headless
# Chromium; if marp can't find a browser, point it at one:
#   make pdf CHROME_PATH="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"

SRC      := presentation/slides.md
PDF      := presentation/slides.pdf
HTML     := presentation/slides.html
PPTX     := presentation/slides.pptx
MARP     := marp
MARPFLAGS := --allow-local-files

# Pass CHROME_PATH through to marp if the caller set one.
ifdef CHROME_PATH
export CHROME_PATH
endif

.PHONY: all pdf html pptx watch clean

all: pdf

pdf: $(PDF)

$(PDF): $(SRC)
	$(MARP) $(MARPFLAGS) --pdf $(SRC) -o $(PDF)

html: $(SRC)
	$(MARP) $(MARPFLAGS) --html $(SRC) -o $(HTML)

pptx: $(SRC)
	$(MARP) $(MARPFLAGS) --pptx $(SRC) -o $(PPTX)

# Live preview server with hot reload while editing.
watch:
	$(MARP) $(MARPFLAGS) --watch --server presentation/

clean:
	rm -f $(PDF) $(HTML) $(PPTX)
