---
name: write-a-new-post
description: Create a new Markdown post in this Hugo blog following the repository's note style.
---

# Write a new post

- Work only on the new post; do not modify unrelated posts
- Follow the conventions of existing posts in `content/posts/` (read a few similar posts first); do not rely on memory for repository conventions
- Scaffold the file with `hugo new content/posts/<name>.md` and edit only the generated file; confirm it with `git status --short`
- Titles use the form `Topic: specifics`, carry no numbering prefix (`1.`, `01`, `Chapter 1`); version numbers inside tech names (`Python 3`) are fine
- Write concise Chinese technical notes: explanations go into nested unordered lists (one fact/step/conclusion per item, 2-space nesting, 4-space for sub-details), not long prose paragraphs
- `##` for main sections, `###` for subsections; no ordered lists, no standalone "notes"/summary sections at the end
- Code blocks belong to a list item, keep the item's outer indentation (2 spaces at level 1, 4 at level 2), and are always fenced with a language tag (`go`, `java`, `yaml`, `bash`, `sql`)
- Backtick-mark APIs, commands, config fields, types, filenames, and key terms
- Reuse categories/tags from similar posts, ordered broad → narrow; create a new category only when nothing fits
- Set `math: true` only when the post actually contains LaTeX math
- Do not end sentences with `。` or `.`; dots inside code, commands, domains, versions, and filenames stay
- Verify with `git diff --check`; if hugo is installed, run `hugo` to confirm the build passes
