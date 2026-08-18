# SimCLR Blog Post Specification

## Requested outcome

- Use the final Notion page `3c008262-780b-801a-93ef-e810fe3a2e68` as the article source.
- Download the official ICML 2020 SimCLR paper.
- Identify the nine figures/tables embedded in the Notion article and extract matching, stable assets from the paper PDF.
- Keep the source PDF and extracted assets organized in the repository.
- Append official paper source links to the end of the Notion page.
- Remove the Notion table of contents if it cannot be preserved cleanly in Jekyll.
- Convert the article to `_posts/2026-08-19-simclr.md` with repository-local image paths.
- Use an official Ruby/Bundler installation path if the existing local toolchain cannot build the site.
- Run production build and link checks.
- Stop for user approval before committing the article, pushing, merging, or deploying.

## Non-goals

- Do not publish or deploy the post.
- Do not rewrite article prose beyond conversion-required formatting and the source section requested by the user.
- Do not substitute screenshots from Notion when a matching figure/table can be extracted from the official PDF.
