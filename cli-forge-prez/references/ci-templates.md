# CI Templates for Slide Build & Deploy

> **When to read:** Step 4 (Write output), when the project has CI and the user wants automated PDF generation and Pages deployment.

---

## Repository structure

```
docs/slides/
├── pitch-projectname.md       # pitch deck (3-4 slides)
├── lightning-projectname.md   # lightning talk (6-8 slides)
├── talk-kubecon-2026.md       # conference talk (10-12 slides)
├── assets/
│   ├── logo.svg
│   └── arch-diagram.png
├── themes/
│   └── dark-tech.css          # custom Marp theme
└── dist/                      # generated outputs (gitignored or CI artifact)
    ├── pitch-projectname.html
    └── pitch-projectname.pdf
```

Add to `.gitignore`:
```
docs/slides/dist/
```

## GitHub Actions

```yaml
name: Build Slides

on:
  push:
    branches: [main]
    paths: ['docs/slides/**']
  pull_request:
    paths: ['docs/slides/**']

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build HTML
        uses: docker://marpteam/marp-cli:latest
        with:
          args: >-
            --input-dir docs/slides/
            --theme docs/slides/themes/dark-tech.css
            --html
            --output docs/slides/dist/

      - name: Build PDF
        uses: docker://marpteam/marp-cli:latest
        with:
          args: >-
            --input-dir docs/slides/
            --theme docs/slides/themes/dark-tech.css
            --pdf
            --allow-local-files
            --output docs/slides/dist/

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: slides
          path: docs/slides/dist/

      - name: Deploy to GitHub Pages
        if: github.ref == 'refs/heads/main'
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: docs/slides/dist/
```

## GitLab CI

```yaml
stages:
  - build
  - deploy

build-slides:
  stage: build
  image: marpteam/marp-cli:latest
  script:
    - marp --input-dir docs/slides/
           --theme docs/slides/themes/dark-tech.css
           --html
           --output public/
    - marp --input-dir docs/slides/
           --theme docs/slides/themes/dark-tech.css
           --pdf
           --allow-local-files
           --output public/
  artifacts:
    paths:
      - public/

pages:
  stage: deploy
  script:
    - echo "Deploying slides to GitLab Pages"
  artifacts:
    paths:
      - public/
  only:
    - main
```

## Docker local build

```bash
# HTML — all decks in docs/slides/
docker run --rm -v "$(pwd):/home/marp/app" \
  marpteam/marp-cli:latest \
  --input-dir docs/slides/ \
  --theme docs/slides/themes/dark-tech.css \
  --html \
  --output docs/slides/dist/

# PDF — requires --allow-local-files for images
docker run --rm -v "$(pwd):/home/marp/app" \
  marpteam/marp-cli:latest \
  --input-dir docs/slides/ \
  --theme docs/slides/themes/dark-tech.css \
  --pdf \
  --allow-local-files \
  --output docs/slides/dist/

# Single deck
docker run --rm -v "$(pwd):/home/marp/app" \
  marpteam/marp-cli:latest \
  docs/slides/pitch-grob.md \
  --theme docs/slides/themes/dark-tech.css \
  --html \
  --output docs/slides/dist/pitch-grob.html
```

## npx local build (no Docker)

```bash
# HTML
npx @marp-team/marp-cli docs/slides/pitch-grob.md \
  --theme docs/slides/themes/dark-tech.css \
  --html \
  -o docs/slides/dist/pitch-grob.html

# Preview with live reload
npx @marp-team/marp-cli docs/slides/pitch-grob.md \
  --theme docs/slides/themes/dark-tech.css \
  --preview
```

## Makefile (drop-in for any project)

```makefile
SLIDES_DIR := docs/slides
THEME      := $(SLIDES_DIR)/themes/dark-tech.css
DIST       := $(SLIDES_DIR)/dist
MARP       := npx @marp-team/marp-cli

.PHONY: slides slides-html slides-pdf slides-preview clean-slides

slides: slides-html slides-pdf

slides-html:
	$(MARP) --input-dir $(SLIDES_DIR) --theme $(THEME) --html -o $(DIST)/

slides-pdf:
	$(MARP) --input-dir $(SLIDES_DIR) --theme $(THEME) --pdf --allow-local-files -o $(DIST)/

slides-preview:
	$(MARP) $(SLIDES_DIR)/$(DECK).md --theme $(THEME) --preview

clean-slides:
	rm -rf $(DIST)
```

Usage: `make slides`, `make slides-preview DECK=pitch-grob`.

## Notes

- **PDF requires a browser** (Chrome/Edge/Firefox). The Marp Docker image includes Chromium. The `--allow-local-files` flag is needed when slides reference local images.
- **HTML output is standalone** — single file, no external dependencies, ~100-150KB. Can be opened directly in any browser, no server needed.
- **`--input-dir`** processes all `.md` files in the directory. Use it for multi-deck builds.
- **PPTX export** is also available via `--pptx` flag (images are embedded, not editable text).
