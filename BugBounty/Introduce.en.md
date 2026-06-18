---
title: Your Article Title Here
dek: One-sentence summary shown in the writing index and OpenGraph description.
date: 2025-01-01
tags: [AppSec, Authentication]
venue: Personal blog
reading_time: 8 min
draft: false
---

# Optional in-body H1 (the frontmatter title is already the page title)

Lead paragraph. Open with the problem or the claim. Keep it tight — the dek
already did the one-line summary, so the first paragraph should add context,
not repeat it.

## A section heading

Body copy. You can use **bold**, *italic*, and `inline code`. Links render
with an accent underline: [example link](https://example.com).

### A sub-heading

Lists work:

- Unordered item with an em-dash marker
- Another item
  - Nested item

1. Ordered items use mono numerals
2. Second item

## Code blocks

Fenced blocks get a left accent border and horizontal scroll:

```ts
async function example(input: string): Promise<string> {
  return input.toUpperCase();
}
```

## Blockquotes

> Blockquotes render with a left rule and muted italic text.
> Use them for callouts or cited material.

## Tables

| Header A | Header B |
| -------- | -------- |
| cell     | cell     |
| cell     | cell     |

## Images

![Alt text](https://example.com/image.png)

---

That is the full set. When you are ready, copy this file to
`BugBounty/YourArticle.en.md` and `BugBounty/YourArticle.id.md` in the
[MyArticles](https://github.com/JustNotSec/MyArticles) repo, fill in the
frontmatter, and rebuild the portfolio.
