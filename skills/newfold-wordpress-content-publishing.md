---
name: newfold-wordpress-content-publishing
description: Create, update and publish WordPress posts, pages, taxonomy terms and
  media on a Newfold-hosted site through the BLU MCP gateway.
generated: '2026-08-13'
method: generated
source: https://github.com/newfold-labs/wp-module-mcp/blob/main/README.md
api: newfold:blu-mcp
endpoint: https://{site}/wp-json/blu/mcp
operations:
  - blu-call-ability
  - blu-posts-search
  - blu-get-post
  - blu-add-post
  - blu-update-post
  - blu-delete-post
  - blu-list-categories
  - blu-add-category
  - blu-list-tags
  - blu-add-tag
  - blu-pages-search
  - blu-add-page
  - blu-update-page
  - blu-upload-media
  - blu-list-media
  - blu-search-media
  - blu-get-site-info
---

# Publish content on a Newfold WordPress site

Every ability below is invoked through `blu-call-ability`, never as a tool name.
Open a session first — see `newfold-mcp-gateway-session`.

## Orient before writing

`blu-get-site-info` returns site name, URL, description, admin email, WordPress
version, plugins with active state, themes and users **in one call**. Newfold
built it precisely so an agent does not spend five round-trips on orientation.
Start here.

## Find before you create

```json
{"method":"tools/call","params":{"name":"blu-call-ability",
 "arguments":{"ability_name":"blu-posts-search",
              "parameters":{"search":"pricing","per_page":5}}}}
```

`blu-posts-search` and `blu-pages-search` are dedicated abilities with tight
schemas, not wrappers around the full `/wp/v2/posts` collection surface. Use
them rather than the generic REST bridge for discovery.

## Create and update

- Posts: `blu-add-post`, then `blu-update-post` by ID, `blu-get-post` to verify.
- Pages: `blu-add-page`, `blu-update-page`, `blu-get-page`.
- Custom post types: `blu-list-post-types` first, then `blu-cpt-search`,
  `blu-add-cpt`, `blu-update-cpt`.

Fetch the schema with `blu-get-ability-schema` before the first write of each
kind. Field names are the ability's, not necessarily WordPress core's.

## Taxonomy

`blu-list-categories` / `blu-add-category` / `blu-update-category` /
`blu-delete-category`, and the matching `blu-*-tag` set. Create the term before
assigning it — there is no create-on-write behaviour documented.

## Media

`blu-upload-media` adds a file, `blu-list-media` and `blu-search-media` find one,
`blu-get-media-file` returns the actual blob. Image results come back as
`content[0].type: "image"` with base64 `data` and a `mimeType`, not as text.

## Rules that bite

- **No idempotency.** There is no idempotency key and no de-duplication contract.
  A retried `blu-add-post` creates a second post. Search first, then write, and
  never retry a create blindly on a timeout.
- **Read the wrapper, not the HTTP status.** Results arrive as
  `{ statusCode, status, message }` inside `result.structuredContent`. The
  transport returns 200 even when `statusCode` is 400, 404 or 500.
- **Deletes are real.** `blu-delete-media` deletes permanently, and
  `blu-delete-cpt` is documented as permanent. Confirm with the user first.
- **Check `annotations.readonly`** on any ability you have not used before, to
  know whether a call mutates the site.
