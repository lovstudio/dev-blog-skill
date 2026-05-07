# lovstudio:dev-blog

![Version](https://img.shields.io/badge/version-0.3.1-CC785C)

Canonical publishing contract for LovStudio's Supabase-backed website blog
feed. It can write and publish a development blog post directly, and it defines
the automation semantics used by dependent skills such as `deep-research` and
`lovstudio-distill`.

Part of [lovstudio skills](https://github.com/lovstudio/skills) — by [lovstudio.ai](https://lovstudio.ai)

## Install

```bash
npx lovstudio skills add dev-blog -g -y
```

Requires Python 3.8+. No third-party Python packages are needed.

## Usage

Ask Claude Code:

```text
/lovstudio:dev-blog 总结这次开发过程，生成一篇博客并同步到网站
```

The skill will gather context, draft a Chinese article, save a local Markdown
draft, run a dry-run payload check, then publish to Supabase `blog_posts`.

Dependent skills publish generated Markdown through the same contract:

```bash
WEB_ROOT="${LOVSTUDIO_DEV_BLOG_WEB_ROOT:?set LOVSTUDIO_DEV_BLOG_WEB_ROOT}"
cd "$WEB_ROOT" && pnpm run sync:research -- [markdown_path]
cd "$WEB_ROOT" && pnpm run sync:distill -- [markdown_path]
```

Those sync scripts own source-specific parsing, while `lovstudio-dev-blog` owns
the shared `blog_posts` semantics: `source_kind`, `source_path`,
`is_visible`, `show_in_index`, and final publish status reporting.

You can also run the publisher directly:

```bash
WEB_ROOT="${LOVSTUDIO_DEV_BLOG_WEB_ROOT:?set LOVSTUDIO_DEV_BLOG_WEB_ROOT}"
python3 scripts/publish_blog_post.py \
  --input .output/dev-blog/example.md \
  --title "一次开发上下文如何变成可复用博客" \
  --slug "dev-context-to-blog" \
  --excerpt "把开发过程沉淀成网站博客，关键在于先结构化上下文，再用 Supabase 作为发布源。" \
  --tags "dev,lovstudio,blog" \
  --env-file "$WEB_ROOT/.env.local"
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--input` | (required) | Markdown/MDX post body. |
| `--title` | (required) | Blog post title. |
| `--slug` | generated from title | URL slug. |
| `--excerpt` | first paragraph | Blog card summary. |
| `--tags` | `dev,lovstudio` | Comma-separated tags. |
| `--author` | `Mark` | Author name. |
| `--cover` | empty | Optional cover image URL. |
| `--published-at` | now | ISO timestamp. |
| `--source-kind` | `dev-skill` | Stored in `blog_posts.source_kind`. |
| `--source-path` | `dev-blog:<slug>` | Stable source key. |
| `--draft` | false | Publish as hidden draft. |
| `--hide-from-index` | false | Keep visible detail page but omit from `/blog`. |
| `--env-file` | empty | Optional env file containing Supabase credentials. |
| `--dry-run` | false | Print payload without writing. |

## Supabase Target

The script upserts into `blog_posts` by `slug` and sets:

- `is_visible=true`
- `show_in_index=true`
- `source_kind=dev-skill`

It requires `NEXT_PUBLIC_SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` in the
environment or in the file passed through `--env-file`.

## User Configuration

Set `LOVSTUDIO_DEV_BLOG_WEB_ROOT` to the website repo root used for sync scripts
and default `.env.local` lookup.

## License

MIT
