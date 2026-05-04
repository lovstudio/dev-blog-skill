---
name: lovstudio:dev-blog
category: Dev Tools
tagline: "Summarize a development session into a practical blog post and publish it to LovStudio's Supabase blog feed."
description: >
  Summarize the current development context, code changes, decisions, and
  lessons into a practical Chinese blog post for yourself and developers facing
  similar issues, then publish it to LovStudio's Supabase `blog_posts` table so
  it appears on the website blog list. Trigger when the user says "生成博客",
  "同步到网站博客", "总结上下文写博文", "开发日志", "generate blog post",
  "sync to website blog", or "summarize context as blog".
license: MIT
compatibility: >
  Requires Python 3.8+. Publishing requires Supabase service-role credentials
  available in environment variables or a local .env file.
metadata:
  author: lovstudio
  version: "0.1.0"
  tags: dev blog supabase writing
---

# Dev Blog

Turn the current development session into a useful Chinese technical blog post
and publish it to LovStudio's website blog feed.

## When to Use

- The user asks to summarize current context and write a blog post.
- The user wants a development log, incident write-up, or lessons learned article.
- The user asks to sync a generated post to the LovStudio website blog list.
- Trigger phrases: "生成博客", "同步到网站博客", "总结上下文写博文", "开发日志", "generate blog post", "sync to website blog".

## Workflow (MANDATORY)

**You MUST follow these steps in order:**

### Step 1: Gather Context

Collect the source material before writing:

- Recent user intent and constraints from the conversation.
- Relevant files, diffs, commands, errors, and verification output.
- The final decision or implementation, including tradeoffs.
- What a future reader should learn from this case.

If the topic, audience, or publish target is unclear, ask one concise question.
Do not ask for fields that can be inferred from the current context.

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

### Step 5: Publish to Supabase

Run a dry run first and inspect the payload:

```bash
python3 ~/.claude/skills/lovstudio-dev-blog/scripts/publish_blog_post.py \
  --input .output/dev-blog/<slug>.md \
  --title "<title>" \
  --slug "<slug>" \
  --excerpt "<excerpt>" \
  --tags "dev,lovstudio" \
  --env-file /Users/mark/lovstudio/coding/web/.env.local \
  --dry-run
```

Then publish:

```bash
python3 ~/.claude/skills/lovstudio-dev-blog/scripts/publish_blog_post.py \
  --input .output/dev-blog/<slug>.md \
  --title "<title>" \
  --slug "<slug>" \
  --excerpt "<excerpt>" \
  --tags "dev,lovstudio" \
  --env-file /Users/mark/lovstudio/coding/web/.env.local
```

The script upserts by `slug`, sets `is_visible=true`, `show_in_index=true`, and
uses `source_kind=dev-skill`. A successful publish returns `/blog/<slug>`.

## CLI Reference

| Argument | Default | Description |
|----------|---------|-------------|
| `--input` | (required) | Markdown/MDX post body. |
| `--title` | (required) | Blog post title. |
| `--slug` | generated from title | URL slug. Use ASCII kebab-case. |
| `--excerpt` | first paragraph | Blog card summary. |
| `--tags` | `dev,lovstudio` | Comma-separated tags. |
| `--author` | `Mark` | Author name. |
| `--cover` | empty | Optional cover image URL. |
| `--published-at` | now | ISO timestamp. |
| `--source-kind` | `dev-skill` | Stored in `blog_posts.source_kind`. |
| `--source-path` | `dev-blog:<slug>` | Stable source key for traceability. |
| `--draft` | false | Set `is_visible=false`. |
| `--hide-from-index` | false | Set `show_in_index=false`. |
| `--env-file` | empty | Local `.env` file to load credentials from. |
| `--dry-run` | false | Print payload without writing to Supabase. |

## Dependencies

No third-party Python dependencies.

Publishing requires:

- `NEXT_PUBLIC_SUPABASE_URL`
- `SUPABASE_SERVICE_ROLE_KEY`

Never print or copy these values into the article or final response.
