# SimCLR Blog Post Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce a locally buildable Jekyll draft of the final SimCLR Notion article with stable PDF-extracted figures/tables and official paper citations.

**Architecture:** Treat the PMLR paper PDF as the canonical visual source, while using the nine Notion images only to identify which paper figures/tables the article references. Store the source PDF separately from web-ready crops, convert Notion-flavored Markdown into Chirpy-compatible Markdown, and validate the result with the same production build and HTML checks used by deployment.

**Tech Stack:** Jekyll/Chirpy, Markdown/YAML front matter, Poppler/Python PDF tooling, Notion MCP, Ruby/Bundler

**Spec:** `docs/superpowers/specs/2026-08-19-simclr-post.md`

## Global Constraints

- Use Notion page `3c008262-780b-801a-93ef-e810fe3a2e68` as the article source.
- Use the official ICML 2020/PMLR paper as the canonical source.
- Preserve article prose except for conversion-required formatting and the requested source section.
- Remove an incompatible Notion table of contents instead of recreating it manually.
- Stop for user approval before committing article changes, pushing, merging, or deploying.

---

### Task 1: Acquire and inspect the canonical paper

**Files:**
- Create: `assets/papers/simclr/chen20j.pdf`
- Create: `tmp/pdfs/simclr/page-*.png` (temporary verification output)

**Interfaces:**
- Consumes: Official PMLR paper URL confirmed through web research.
- Produces: A checksummed source PDF and rendered pages for figure/table identification.

- [ ] **Step 1: Verify the official paper metadata and PDF URL**

  Search the PMLR proceedings record and confirm title, authors, venue, year, volume, pages, and PDF link.

- [ ] **Step 2: Download the canonical PDF**

  Run `curl -L --fail --show-error <official-pdf-url> -o assets/papers/simclr/chen20j.pdf`.

- [ ] **Step 3: Validate and render the PDF**

  Run `pdfinfo assets/papers/simclr/chen20j.pdf`, calculate `shasum -a 256`, and render every page with `pdftoppm -png -r 180` into `tmp/pdfs/simclr/`.

- [ ] **Step 4: Inspect rendered pages**

  Create contact sheets and visually confirm that figures, tables, captions, and page boundaries render correctly.

### Task 2: Extract and verify the nine article visuals

**Files:**
- Create: `assets/img/posts/simclr/*.png`
- Create: `assets/img/posts/simclr/manifest.md`

**Interfaces:**
- Consumes: Rendered paper pages and the nine images in the Notion page.
- Produces: Stable web-ready crops plus a mapping from article order to paper Figure/Table labels and source pages.

- [ ] **Step 1: Fetch the current Notion page and download temporary comparison images**

  Fetch the exact page ID, enumerate all nine signed image URLs in body order, and download them only under `tmp/pdfs/simclr/notion/` for comparison.

- [ ] **Step 2: Match each Notion image to the paper**

  Compare contact sheets and image dimensions to identify the exact Figure/Table label and paper page for each of the nine article images.

- [ ] **Step 3: Crop canonical assets from rendered PDF pages**

  Crop at high resolution from the paper rendering, including the visual and its caption where the article relies on the label, and save descriptive filenames under `assets/img/posts/simclr/`.

- [ ] **Step 4: Verify visual fidelity**

  Inspect every final crop, ensure no clipped borders or unreadable text, and record page/label mappings in `manifest.md`.

### Task 3: Add official sources to Notion and convert the post

**Files:**
- Create: `_posts/2026-08-19-simclr.md`

**Interfaces:**
- Consumes: Final Notion body, official source metadata, and the asset manifest.
- Produces: A Chirpy-compatible post and a Notion page ending with official source links.

- [ ] **Step 1: Append a references section to Notion**

  Add a final `## 참고 자료` section containing the official PMLR proceedings page, official PDF, and arXiv record, then fetch the page again to verify exact insertion.

- [ ] **Step 2: Convert Notion Markdown to Jekyll Markdown**

  Add YAML front matter with the exact title, description, date, categories, and tags; remove `<table_of_contents/>`, `<empty-block/>`, and conversion-only `<br>` tags; replace signed image URLs with repository-local asset paths.

- [ ] **Step 3: Check conversion invariants**

  Verify that the post has nine local image references, zero signed URLs, zero Notion-only tags, the expected section headings, and the appended official references.

### Task 4: Establish a supported Ruby toolchain and verify publication readiness

**Files:**
- Modify only if required: `Gemfile`
- Generated and ignored: `Gemfile.lock`, `.bundle/`, `vendor/`, `_site/`

**Interfaces:**
- Consumes: The converted post and existing GitHub Actions Ruby version.
- Produces: Fresh production build and HTML validation evidence.

- [ ] **Step 1: Confirm official installation guidance**

  Use the official Ruby installation documentation and official Bundler documentation to select a supported local Ruby/Bundler matching the repository's Ruby 3.4 CI environment.

- [ ] **Step 2: Install or activate Ruby/Bundler only if needed**

  Prefer an existing Homebrew Ruby 3.4 installation; otherwise install Ruby through Homebrew and install Bundler with that Ruby's `gem` command.

- [ ] **Step 3: Install project gems**

  Run `bundle install` with the supported toolchain and confirm `bundle exec jekyll --version` and `bundle exec htmlproofer --version`.

- [ ] **Step 4: Run full verification**

  Run `bash tools/test.sh`, scan the generated post HTML for all nine image paths and official references, and run `git diff --check` plus `git status --short`.

- [ ] **Step 5: Prepare approval handoff**

  Present the post path, asset directory, extraction manifest, source links, build results, and uncommitted diff summary. Do not commit, push, merge, or deploy until the user approves.
