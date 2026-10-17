---
name: write-a-new-post
description: Create a new Markdown post in this Hugo blog following the repository's note style.
---

# Write a new post

- Work only on the new post; do not modify unrelated posts
- Read `content/posts/style-guide/index.md` first and follow every rule in it; it is the single and complete source of truth for the house style (naming, headings, lists, punctuation, code blocks, math, diagrams)
- Pick `categories` and `tags` from `taxonomy.md` in this directory; only create new ones when nothing there fits
- Do not read any other post to infer style; the style post overrides everything
- Scaffold the file with `hugo new content/posts/<name>.md` and edit only the generated file; confirm it with `git status --short`
- Do not run any verification: no `hugo` build, no `git diff --check`, no linting; just write the post
