---
layout: post
title: "Welcome to the lab blog"
date: 2026-09-11
---

This is the first post on our lab blog. We'll use this space to share easy-to-follow
guides to our code, research updates, and notes on our models.

## How to add a new post

Create a new file in the `_posts` folder, named with the date and a short title,
for example `2026-10-01-a-new-result.md`. Start the file with a small header:

    ---
    layout: post
    title: "Your post title"
    date: 2026-10-01
    ---

Then write the body in Markdown below the header — paragraphs, headings, links,
images, and lists all work. Commit the file, and the post appears here
automatically, sorted by date. That's the whole workflow.

## Keeping a post unpublished

Add `published: false` to the header and the post stays out of the built site.
Delete the line when it's ready to go live.

    ---
    layout: post
    title: "Not ready yet"
    date: 2026-10-01
    published: false
    ---

Note that the file is still in the repository, so this hides a post from the
website, not from anyone browsing the source.

## Changing the order

Posts appear newest first, by date. To override that, add an `order` number to
the header — posts with one are pinned to the top of the list in ascending
order, and everything else follows by date as usual.

    ---
    layout: post
    title: "Start here"
    date: 2026-09-11
    order: 1
    ---

Thanks for reading.
