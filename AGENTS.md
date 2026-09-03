# AGENTS.md

Source for **https://a5huynh.github.io** — a [Zola](https://getzola.org) static site.
This is the repo to edit. Posts, templates, and styles all live here.

- **This repo:** `git@github.com:a5huynh/blog-src.git`
- **Output repo:** `../a5huynh.github.com` → `git@github.com:a5huynh/a5huynh.github.com.git`
  (GitHub Pages user site, served from `master`)

`public/` is gitignored here; the built site is committed to the *output* repo instead.

## Zola version

Builds clean on **Zola 0.23.4**. The site was migrated from ~0.16 straight to 0.23,
which spans two hard breaks — don't assume older Zola docs or examples apply:

- **0.22** replaced the syntect highlighter with [giallo](https://github.com/getzola/giallo).
  Highlighting config moved to `[markdown.highlighting]`, and code blocks now render as
  one `<span class="giallo-l">` per line with line numbers in a `.giallo-ln` span,
  instead of the old `<table><td>` gutter. `sass/_code.scss` styles the new markup —
  read the comments in there before touching it, they encode non-obvious traps (a
  literal newline between line spans, and wrapped lines breaking the gutter).
- **0.23** replaced Tera v1 with Tera v2. **Macros and shortcodes no longer exist** —
  both are now *components* (see below). Markdown content is also templated by default,
  so stray `{{` / `{%` in a post will error; wrap it in `{% raw %}…{% endraw %}`.

The committed output in `../a5huynh.github.com` predates the upgrade, so the next deploy
will show a large diff (~67 files). That's expected: HTML escaping got less aggressive
(`/` and `"` are no longer entity-escaped), `page.summary` lost a trailing newline, and
`blog.css` picked up a UTF-8 BOM from the `grass` Sass compiler (valid CSS, harmless).

`external_links_external = false` is set in `config.toml` specifically to suppress the
new default of adding `rel="external"` to every outbound link.

## Layout

```
config.toml            base_url, taxonomies, markdown opts
content/posts/         posts, one dir per year
sass/                  blog.scss + partials → compiled to blog.css
static/                copied verbatim to site root
templates/             Tera templates
```

### Templates

| File | Renders |
| --- | --- |
| `_layouts/main.html` | base shell: head, header, social icons, footer, MathJax |
| `index.html` | home page; posts grouped by year via `previous_year` loop |
| `page.html` | a single post; extends `index.html`, sets og/meta tags |
| `tags/single.html` | one tag's post list (same year-grouping logic as `index.html`) |
| `tags/list.html` | **empty file** → produces a 0-byte `/tags/index.html` (known bug) |

| `components.html` | all components: `tag_list`, `gallery`, `caption` |
| `404.html`, `robots.txt` | |

The year-grouping loop is duplicated between `index.html` and `tags/single.html` —
change both, or factor into a macro.

`main.html` guards an RSS `<link>` behind `config.generate_rss`, which no longer exists
(renamed `generate_feeds` in Zola 0.19). There is currently **no feed**; enabling one
means setting `generate_feeds` and updating that check.

The post byline uses `page.word_count` (Zola's own field, which excludes code blocks),
not `page.content | wordcount`. The filter counts the highlighter's `<span>` markup, so
it inflated counts by 30-50% on code-heavy posts after the giallo switch.

## Writing a post

Create `content/posts/<year>/YYYY-MM-DD-<slug>.md`. Zola strips the date prefix, so
`2020-07-15-covid-tracker-device.md` → `/posts/2020/covid-tracker-device/`.

```toml
+++
title = "COVID-19 Stats Tracker"

[taxonomies]
tags = ["esp32", "hardware", "iot"]

[extra]
location = "San Francisco"
+++

Intro paragraph shown as the excerpt on the home page.

<!-- more -->

Rest of the post.
```

- `[extra] location` is rendered in the byline next to the date — always set it.
- `<!-- more -->` sets `page.summary`; without it the meta description gets the whole post.
- New year? Add `content/posts/<year>/_index.md` with `transparent = true` so posts roll
  up into `/posts/`.
- Reuse existing tags where possible (`ls ../a5huynh.github.com/tags`) — each new tag
  creates a new taxonomy page.
- Assets go in `static/img/<year>/<slug>/…`, referenced absolutely: `/img/2020/covid-tracker/lcd.jpg`.
- `_drafts/`, `_ideas/`, `_art/` are gitignored scratch dirs (not currently present).

### Components

Components replaced shortcodes and macros in Zola 0.23. All of them live in
`templates/components.html`, are registered globally (no import), and work in both
templates and markdown. Note the JSX-ish call syntax — non-string args need `{…}`:

```jinja2
{# inline #}
{{ <gallery images={["/img/a.jpg", "/img/b.jpg"]} /> }}
{{ <tag_list tags={page.taxonomies.tags} selected={term.name} /> }}

{# with a body, available inside the component as `body` #}
{% <caption text="A caption"> %}
![alt](/img/2020/foo.jpg)
{% </caption> %}
```

Components are hygienic: they only see the parameters passed explicitly.

**Components invoked from markdown must emit flush-left HTML.** Any line indented 4+
spaces is parsed as a markdown code block — this silently turned the whole `gallery`
output into a syntax-highlighted code block during the 0.23 migration.

No post currently uses `caption`, so its rendering is unverified — check it visually.

## Styling

Zola compiles Sass itself (`compile_sass = true`); `sass/blog.scss` is the only entry
point. Add partials as `_name.scss` and `@import` them there.

`sass/_constants.scss` holds the design tokens. Use them rather than hardcoding values:

| Token | Value | Use |
| --- | --- | --- |
| `$orange` | `#ff8000` | the single accent colour |
| `$ink` | `#4d4d4d` | body copy and headings (a dark grey, not black) |
| `$ink-strong` | `#000` | `<strong>` only — deliberately darker than `$ink` |
| `$grey` | `#999` | metadata, captions, de-emphasised text |
| `$paper` | `#fff` | knocked-out text on accent backgrounds |
| `$header-font` | Inconsolata | headings, tags, blockquotes |
| `$content-font` | EB Garamond | body copy |
| `$code-font` | Inconsolata | code — see note below |

Breakpoints are `max-width: 650px` and `651–900px`, declared per-component. Check both
when touching layout.

Two things to preserve:

- **The Google Fonts URL in `_layouts/main.html` must keep requesting `ital` and `700`.**
  It originally asked for weight 400 only, so every `<em>` and `<strong>` on the site was
  a browser-synthesised faux italic/bold. Content uses all four faces, including 4
  bold-italics.
- **Only load fonts the CSS actually references, and vice versa.** `$code-font` used to
  lead with Source Code Pro, which the site never loaded — so code rendered in Source
  Code Pro for visitors who happened to have it installed and Inconsolata for everyone
  else.

### Verifying CSS changes

For refactors that should be visually inert, diff rendered screenshots rather than
trusting inspection. Build with a `file://` base URL (see below), screenshot with
headless Chrome, and compare with ImageMagick (`compare -metric AE a.png b.png null:`).

Strip remote resources from the preview HTML first — the Google Fonts link, the MathJax
CDN script, and `<iframe>`s (one 2019 post embeds a remote demo). They load
asynchronously and race, producing ~30k pixels of noise per code-heavy page and making
any diff meaningless. With those removed the render is deterministic to 0px.

**Check that the screenshot actually contains what you're testing.** Headless Chrome
captures only the window height, and the first code block on the poisson-disk post sits
below ~1600px — so a "code page" shot at 1100x1600 contains no code and will report 0px
for any code change, looking like proof when it's a blind spot. Sample a pixel inside
the region of interest to confirm framing before trusting a 0px result.

## Build & deploy

```sh
zola serve      # http://127.0.0.1:1111, live reload
zola build      # → ./public (gitignored)
zola check      # link check + config validation (5 known-bad links remain, see below)
```

To preview a build in a headless browser without running a server, bake a `file://`
base URL — otherwise the pages load the *live* CSS from a5huynh.github.io and you
verify nothing:

```sh
zola build -u "file:///tmp/blog-preview" -o /tmp/blog-preview --force
```

Images in content use root-absolute paths (`/img/…`), which don't resolve under
`file://`; rewrite them in the throwaway copy if you need to see them.

There is **no CI** — deploy is a manual copy into the sibling output repo. See
`../a5huynh.github.com/AGENTS.md` for the `rsync` invocation and the list of
hand-maintained files there that must not be clobbered.

## Workflow: branch and PR before deploying

**Never commit straight to `master`, and never deploy unmerged work.** `master` moves
only by merging a PR, and the output repo is always built from a merged `master`.

1. Branch: `a5huynh/<type>/<short-name>` (`feature`, `bug`, `chore`…). Post branches
   historically used `posts/<year>/<slug>`.
2. Commit there, push, open a PR with `gh pr create`.
3. Wait for it to be merged — ask, don't merge on someone's behalf.
4. Only then `git checkout master && git pull`, rebuild, and deploy to
   `../a5huynh.github.com`.

So a change ships in two commits in two repos: the source PR, then a separate
"Rebuild" commit in the output repo once it has landed.

GitHub operations go through the `gh` CLI. Confirm before pushing or opening PRs.

### Known-bad links

`zola check` reports 5 failures that are deliberate and should be left alone:
congress.gov and tripit 403 to scripted clients but work in a browser;
`graphics.ucsd.edu/twiki/…` and `rockmyworld.dvanoni.com` are gone with no Wayback
snapshot. The checker is also flaky on timeouts — retry before believing a new
failure (lighthouse3d has reported broken once and returned 200 on every retry).

## License

`content/**` is Copyright Andrew Huynh — not reusable. Templates, sass, and everything
else are MIT.
