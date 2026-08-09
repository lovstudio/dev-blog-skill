---
name: sgc-dev-blog
category: Dev Tools
tagline: "Write or sync Markdown into LovStudio's Supabase-backed website blog feed."
description: >
  Own the LovStudio website blog publishing contract. Summarize the current
  development context and its screenshots, diagrams, or other visual evidence
  into a practical Chinese blog post, then publish it to LovStudio's Supabase
  `blog_posts` table. Also provide the automation semantics that dependent
  skills use when they sync generated Markdown artifacts to the website blog.
  Trigger when the user says "生成博客", "同步到网站博客", "总结上下文写博文",
  "开发日志", "把这些截图写进文章", "generate blog post",
  "sync to website blog", or "summarize context as blog".
license: MIT
compatibility: >
  Requires Python 3.8+. Publishing requires Supabase service-role credentials
  available in environment variables or a local .env file.
metadata:
  author: lovstudio
  version: "0.4.0"
  tags: dev blog supabase writing publishing
---

# Dev Blog

Canonical publishing contract for LovStudio's website blog feed.

This skill can directly turn the current development session into a useful
Chinese technical blog post and publish it, and it also defines the automation
contract used by skills such as `deep-research` and `sgc-distill` when
they publish generated Markdown artifacts to `blog_posts`.

## When to Use

- The user asks to summarize current context and write a blog post.
- The user wants a development log, incident write-up, or lessons learned article.
- The user asks to sync a generated post to the LovStudio website blog list.
- Another LovStudio skill needs to publish generated Markdown to the website
  blog system. That skill should declare `sgc-dev-blog` as a dependency
  and follow this publishing contract.
- Trigger phrases: "生成博客", "同步到网站博客", "总结上下文写博文", "开发日志", "generate blog post", "sync to website blog".

## Publishing Contract

`sgc-dev-blog` owns the shared `blog_posts` semantics:

- `blog_posts` is the canonical website blog target.
- `source_kind` identifies the producer, such as `dev-skill`, `deep-research`,
  or `distill`.
- `source_path` is the stable idempotency key for generated artifacts.
- `is_visible=true` means the detail page is public.
- `show_in_index=true` means the post appears in the `/blog` list; dependent
  skills may choose different defaults.
- Meaningful screenshots, diagrams, charts, and before/after images supplied by
  the user are article source material. Inventory them before drafting and
  embed the selected items as durable public assets instead of silently
  dropping them.
- Final responses from dependent skills must include
  `Published to LovStudio: yes/no` and the public URL when publish succeeds.

Dependent skills should not invent separate Supabase payload semantics. They may
use website sync scripts from the configured LovStudio website repo, but
those scripts are part of this `dev-blog` publishing contract.

Supported publishing modes:

- Direct article mode: this skill drafts a blog post, generates a cover, and
  publishes it with `scripts/publish_blog_post.py`.
- Generated artifact sync mode: a dependent skill generates Markdown, then uses
  the source-specific sync command below under this contract.

Default publishing behavior:

- Direct article mode publishes by default after a dry run, including cover
  generation/upload and the non-dry-run Supabase upsert.
- Do not ask for confirmation before publishing unless the user explicitly
  requests a confirmation gate.
- Skip publishing only when the user explicitly asks for draft/local-only/no
  publish behavior, or when a required blocker exists such as missing
  credentials, missing website repo, cover generation failure, or schema/API
  failure.

Current dependent publishing commands:

```bash
WEB_ROOT="${LOVSTUDIO_DEV_BLOG_WEB_ROOT:?set LOVSTUDIO_DEV_BLOG_WEB_ROOT}"
cd "$WEB_ROOT" && pnpm run sync:research -- [markdown_path]
cd "$WEB_ROOT" && pnpm run sync:distill -- [markdown_path]
```

Use dry-run or publish modes according to the dependent skill's workflow. If a
publish fails because credentials, website path, or database schema is
unavailable, keep the generated artifact and report the exact rerun command.

## Workflow (MANDATORY)

**You MUST follow these steps in order:**

### Step 1: Gather Context

Collect the source material before writing:

- Recent user intent and constraints from the conversation.
- Relevant files, diffs, commands, errors, and verification output.
- Every attached image, user-supplied image path, reproduction screenshot,
  generated diagram, and before/after artifact relevant to the story.
- The final decision or implementation, including tradeoffs.
- What a future reader should learn from this case.

Create a visual asset inventory before drafting whenever visual material exists.
Classify each item as problem evidence, key detail, before/after proof,
architecture explanation, or decorative. Default to including user-provided
visual evidence that materially improves understanding. Record a concrete
reason when excluding it, such as redundancy, illegibility, irrelevance, or
private information.

When the inventory contains visual material, read
`references/inline-media.md` completely before drafting or uploading assets.

If the topic or audience is unclear, use `AskUserQuestion` for one concise
question. The publish target defaults to the LovStudio website blog; ask about
the target only if the user mentions multiple possible destinations. Do not ask
for fields that can be inferred from the current context.

### Step 2: Draft the Article

Write in Chinese for two audiences:

- Primary: Mark, as a durable record of the work.
- Secondary: developers or AI builders who may hit a similar issue.

Use this structure unless the context clearly calls for a different one:

1. `# <title>`
2. Opening: what problem triggered the work and why it mattered.
3. Context: project/background, only enough for the reader to orient.
4. Process: the key investigation path, failed assumptions, and turning points.
5. Solution: what changed, why this shape fits the system.
6. Takeaways: reusable engineering lessons.

Style rules:

- Prefer concrete nouns, file/table names, commands, and exact constraints.
- Avoid generic AI productivity claims.
- Do not include secrets, tokens, private customer details, or raw `.env` values.
- Keep code excerpts short and only when they explain the decision.
- Place each selected visual immediately after the paragraph that frames what
  the reader should notice.
- Give every inline image descriptive alt text and a caption that explains why
  it matters. For a wide screenshot, add a focused crop when the critical detail
  would become too small on mobile.
- Use screenshots as evidence, diagrams for invisible relationships, and the
  cover for discovery. Do not treat the cover as a substitute for inline
  explanatory media.
- The post body must be valid Markdown/MDX.

### Step 3: Prepare Metadata

Derive these fields:

| Field | Rule |
|-------|------|
| `title` | Specific, readable Chinese title. |
| `slug` | ASCII lowercase kebab-case; if Chinese title has no ASCII, use a short English slug. |
| `excerpt` | 1-2 sentence summary under 180 chars. |
| `tags` | 2-5 tags, include `dev` and a concrete domain tag. |
| `author` | Default `Mark`. |
| `cover` | Required. Generate a 16:9 WebP cover and upload it before publishing. |
| Inline media | Required when the visual inventory contains meaningful evidence. |
| `source_kind` | Default `dev-skill`. |

### Step 4: Save Draft Locally

Create a temporary Markdown file in the current project, normally:

```bash
mkdir -p .output/dev-blog
```

Use a filename based on the slug, for example:

```text
.output/dev-blog/<slug>.md
```

Report the absolute path of the draft if publishing is skipped or fails.

### Step 5: Generate and Upload Cover

Every published blog post must have a cover image.

Use `baoyu-cover-image` to generate a 16:9 cover from the article title,
excerpt, tags, and core message. Keep it lightweight and suitable for blog
cards:

- Aspect: 16:9.
- Recommended dimensions: `type=minimal`, `rendering=hand-drawn`, `mood=subtle`.
- Visual direction: use an Anthropic-like minimalist editorial spot illustration style: warm off-white background, muted terracotta/sage/lavender accents, thin black hand-drawn linework, one small central metaphor, generous whitespace.
- Text: prefer `text=none` for blog list covers unless the user explicitly asks for title text; the page already renders the article title next to the image.
- Avoid dense tech UI, glowing cyber effects, gradients, charts, readable text, logos, and complex scenes.
- Save the `baoyu-cover-image` source and prompt files under `cover-image/<slug>/`.
- Convert the generated `cover.png` to WebP.
- Storage path: `app-assets/blog-covers/baoyu/anthropic-minimal/<slug>.webp`.
- Upload metadata: `contentType=image/webp`, `cacheControl=31536000`, `upsert=true`.
- Public URL: pass this value to the publish command using `--cover`.

Do not embed secrets, internal tokens, raw logs, or private customer details in
the image. If image generation fails, do not publish without a cover; save the
draft path and report the blocker.

### Step 6: Prepare and Upload Inline Media

When the visual asset inventory contains meaningful evidence:

1. Select only visuals with a distinct narrative role.
2. Crop or redact unrelated private details.
3. Convert static screenshots to WebP while keeping text legible.
4. Store durable local artifacts under an article-specific directory.
5. Run `scripts/upload_blog_assets.py` with `--dry-run`, then upload.
6. Replace local image paths in the draft with the returned public URLs.
7. Add specific Chinese alt text and explanatory captions.

Use the storage path:

```text
app-assets/blog-images/<slug>/<ordered-filename>.webp
```

Example:

```bash
WEB_ROOT="${LOVSTUDIO_DEV_BLOG_WEB_ROOT:?set LOVSTUDIO_DEV_BLOG_WEB_ROOT}"
python3 scripts/upload_blog_assets.py \
  --slug "<slug>" \
  --input "path/to/01-problem-overview.webp" \
  --input "path/to/02-key-detail.webp" \
  --env-file "$WEB_ROOT/.env.local" \
  --dry-run
```

Repeat without `--dry-run` after inspecting the manifest. Follow
`references/inline-media.md` for selection, compression, placement, captions,
and responsive-readability checks.

### Step 7: Publish to Supabase

Run a dry run first and inspect the payload:

```bash
WEB_ROOT="${LOVSTUDIO_DEV_BLOG_WEB_ROOT:?set LOVSTUDIO_DEV_BLOG_WEB_ROOT}"
python3 scripts/publish_blog_post.py \
  --input .output/dev-blog/<slug>.md \
  --title "<title>" \
  --slug "<slug>" \
  --excerpt "<excerpt>" \
  --tags "dev,lovstudio" \
  --cover "<public-cover-url>" \
  --env-file "$WEB_ROOT/.env.local" \
  --require-inline-image \
  --dry-run
```

Add `--require-inline-image` whenever the visual inventory selected one or more
inline assets. Omit the flag only when no meaningful inline visual exists. The
publisher rejects local or relative image paths in public posts.

Then publish:

Publish by default after the dry run succeeds. Do not ask for confirmation
before running the non-dry-run publish command unless the user explicitly asks
for a confirmation gate. If the user asks for draft/local-only/no publish, stop
after saving the draft and report the absolute draft path.

```bash
WEB_ROOT="${LOVSTUDIO_DEV_BLOG_WEB_ROOT:?set LOVSTUDIO_DEV_BLOG_WEB_ROOT}"
python3 scripts/publish_blog_post.py \
  --input .output/dev-blog/<slug>.md \
  --title "<title>" \
  --slug "<slug>" \
  --excerpt "<excerpt>" \
  --tags "dev,lovstudio" \
  --cover "<public-cover-url>" \
  --env-file "$WEB_ROOT/.env.local" \
  --require-inline-image
```

The script upserts by `slug`, sets `is_visible=true`, `show_in_index=true`, and
uses `source_kind=dev-skill`. A successful publish returns `/blog/<slug>`.

### Step 8: Verify the Public Article

Treat publication as complete only after checking the reader-visible result:

- the article URL returns HTTP 200;
- title, excerpt, cover, and visibility match the intended payload;
- every selected inline image URL returns HTTP 200 with an `image/*` content
  type;
- the public article contains the expected image URLs, alt text, and captions;
- the rendered `<img>` count matches the selected visual inventory;
- wide screenshots remain understandable at narrow reading widths, using a
  focused crop when necessary.

If the article uses inline visuals, a successful database upsert alone is not a
complete verification.

## CLI Reference

| Argument | Default | Description |
|----------|---------|-------------|
| `--input` | (required) | Markdown/MDX post body. |
| `--title` | (required) | Blog post title. |
| `--slug` | generated from title | URL slug. Use ASCII kebab-case. |
| `--excerpt` | first paragraph | Blog card summary. |
| `--tags` | `dev,lovstudio` | Comma-separated tags. |
| `--author` | `Mark` | Author name. |
| `--cover` | (required by this workflow) | Public cover image URL. Generate and upload before publishing. |
| `--require-inline-image` | false | Fail when a visually sourced article contains no inline image. |
| `--published-at` | now | ISO timestamp. |
| `--source-kind` | `dev-skill` | Stored in `blog_posts.source_kind`. |
| `--source-path` | `dev-blog:<slug>` | Stable source key for traceability. |
| `--draft` | false | Set `is_visible=false`. |
| `--hide-from-index` | false | Set `show_in_index=false`. |
| `--env-file` | empty | Local `.env` file to load credentials from. |
| `--dry-run` | false | Print payload without writing to Supabase. |

## User Configuration

Set `LOVSTUDIO_DEV_BLOG_WEB_ROOT` to the LovStudio website repo root. The
publisher also accepts `--env-file`, so users can keep Supabase credentials in
their own project-specific environment file.

## Dependencies

No third-party Python dependencies.

Publishing requires:

- `NEXT_PUBLIC_SUPABASE_URL`
- `SUPABASE_SERVICE_ROLE_KEY`

Never print or copy these values into the article or final response.

## Additional Resources

- `references/inline-media.md` - Visual inventory, image selection, compression,
  upload, placement, captions, and live verification.
- `scripts/upload_blog_assets.py` - Deterministic uploader for public inline
  blog images.
