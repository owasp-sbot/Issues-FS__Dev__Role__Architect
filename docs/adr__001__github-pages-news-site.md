# ADR-001: GitHub Pages News Site Architecture

## Status: Proposed

## Date: 2026-02-10

## Trigger

Feature request from the Journalist role ([2026-02-09__github-pages-news-site.md](../../Issues-FS__Dev__Role__Journalist/publications/feature-requests/2026-02-09__github-pages-news-site.md)) and implementation handoff ([2026-02-10__news-site-implementation-handoff.md](../../Issues-FS__Dev__Role__Journalist/publications/feature-requests/2026-02-10__news-site-implementation-handoff.md)). The Journalist has produced content (articles, interviews) that is currently buried in a GitHub file browser. A stakeholder-facing website is needed to make this content discoverable, readable, and subscribable.

## Context

The Journalist role publishes markdown articles, interviews, daily briefs, and investigations into the `publications/` directory of the `Issues-FS__Dev__Role__Journalist` repository. This content is currently only accessible by navigating GitHub's file browser -- there is no table of contents, no chronological view, no category filtering, no RSS feed, and no mobile-friendly reading experience.

The existing content structure:

```
publications/
├── articles/
│   └── 2026-02-09__state-of-the-ecosystem.md
├── feature-requests/
│   ├── 2026-02-09__github-pages-news-site.md
│   └── 2026-02-10__news-site-implementation-handoff.md
└── interviews/
    ├── 2026-02-09__librarian-interview-prep.md
    ├── 2026-02-09__librarian-questionnaire.md
    └── 2026-02-10__stakeholder-interview-chatgpt-voice-brief.md
```

Planned but not yet populated: `daily-briefs/`, `investigations/`, `corrections/`.

The Journalist role repo (`Issues-FS__Dev__Role__Journalist`) is a git submodule of the parent dev repo (`Issues-FS__Dev`). The ecosystem has 10 role repos, some of which may eventually want their own web presence (Librarian knowledge base, Cartographer maps).

Key constraints from the feature request:
- Must use GitHub Pages (public repo, free hosting)
- Zero ongoing infrastructure cost
- Content must remain readable as plain markdown in the repo (the site is an enhancement, not a replacement)
- No manual deployment step -- auto-publish on push

This ADR addresses eight architectural questions raised in the implementation handoff.

---

## Decision

### 1. Static Site Generator: Jekyll

**Choice: Jekyll.**

Jekyll is the recommended static site generator for v1 of the news site.

**Rationale:**

- **Native GitHub Pages support.** GitHub provides `actions/jekyll-build-pages@v1`, a first-party action maintained by GitHub. This means zero custom build configuration -- no installing Go, no managing Hugo binaries, no debugging version mismatches. The build step is a single action call.
- **Build speed is irrelevant at this scale.** Hugo's primary advantage is build speed (milliseconds vs seconds). The Journalist's content volume will remain under 100 pages for the foreseeable future. Jekyll builds this in under 10 seconds. Optimizing for build speed at this scale is premature.
- **Theme ecosystem aligned with constraints.** Jekyll's default theme (`minima`) is mobile-responsive, content-focused, and requires zero configuration -- exactly matching the feature request's "minimal design, no custom CSS for v1" requirement.
- **Collections support.** Jekyll collections map cleanly to the Journalist's content categories (articles, interviews, briefs, etc.) without forcing content into the `_posts/` convention with its rigid `YYYY-MM-DD-title.md` naming requirement.
- **Lower cognitive overhead.** The Journalist's workflow is: write markdown, add front matter, push. Jekyll's Liquid templating and YAML configuration are simpler than Hugo's Go template syntax. Since the Journalist will interact with this system daily (adding front matter, checking rendering), the simpler tool wins.

### 2. Site Location: Option A -- Journalist Role Repo

**Choice: Host the site from `Issues-FS__Dev__Role__Journalist`.**

The GitHub Pages site will be deployed from the Journalist role repo itself.

- **URL:** `https://owasp-sbot.github.io/Issues-FS__Dev__Role__Journalist/`

**Rationale:**

- **Content and site co-located.** The Journalist writes articles into `publications/`. The site reads from `publications/`. No cross-repo sync, no submodule tricks, no build step that pulls content from elsewhere. One repo, one push, one build.
- **Clear ownership boundary.** The Journalist role owns the content and the publication channel. This aligns with the ecosystem's single-responsibility-per-repo principle. The Journalist does not need to coordinate with other roles to publish.
- **Deployment simplicity.** GitHub Pages can be enabled directly on this repo. The GitHub Actions workflow runs in the same repo where content lives. No cross-repo triggers, no deploy keys, no PAT scope complications beyond what already exists.
- **Migration path exists.** If the ecosystem later needs a unified web presence (Librarian knowledge base, Cartographer maps), the site can be migrated to a dedicated repo or the parent dev repo. The content structure and front matter conventions established in v1 will transfer directly. Starting simple does not preclude growing later.

The longer URL (`owasp-sbot.github.io/Issues-FS__Dev__Role__Journalist/`) is a cosmetic cost, not a structural one. A custom domain can be added later if desired.

### 3. Content Mapping: Jekyll Collections from `publications/`

**Choice: Use Jekyll collections, with `collections_dir` pointing to `publications/`.**

The `publications/` directory structure maps to Jekyll collections as follows:

| Source Directory | Jekyll Collection | Site Route | Published |
|-----------------|-------------------|------------|-----------|
| `publications/articles/` | `articles` | `/articles/` | Yes |
| `publications/daily-briefs/` | `daily-briefs` | `/briefs/` | Yes |
| `publications/interviews/` | `interviews` | `/interviews/` | Yes |
| `publications/investigations/` | `investigations` | `/investigations/` | Yes |
| `publications/corrections/` | `corrections` | `/corrections/` | Yes |
| `publications/feature-requests/` | `feature-requests` | N/A | No -- internal documents, excluded from site output |

**Jekyll `_config.yml` sketch:**

```yaml
collections_dir: publications
collections:
  articles:
    output: true
    permalink: /articles/:title/
  daily-briefs:
    output: true
    permalink: /briefs/:title/
  interviews:
    output: true
    permalink: /interviews/:title/
  investigations:
    output: true
    permalink: /investigations/:title/
  corrections:
    output: true
    permalink: /corrections/:title/
  feature-requests:
    output: false
```

**Rationale:**

- **Preserves readability.** The `publications/` directory remains a plain, navigable markdown tree. Anyone browsing the repo sees `publications/articles/2026-02-09__state-of-the-ecosystem.md` and can read it directly. The site is an overlay, not a restructuring.
- **No file relocation.** The Journalist does not need to move files into `_posts/` or rename them to match Jekyll's default `YYYY-MM-DD-title.md` convention. Collections accept any filename.
- **Category routing is automatic.** Each collection maps to its own output directory. No Liquid logic needed to filter by category -- the directory structure is the taxonomy.
- **Feature requests stay internal.** Setting `output: false` on the `feature-requests` collection means those documents exist in the repo but are not rendered on the site. This is correct -- feature requests are inter-role coordination artifacts, not public content.

**Front matter requirement:** All publishable content must include YAML front matter. The standardized schema:

```yaml
---
title: "Article Title"
date: 2026-02-09
category: article | brief | interview | investigation | correction
topics: [topic1, topic2]
summary: "One-sentence summary for the landing page and RSS feed."
author: Journalist
---
```

The `category` field is for display and feed purposes. The collection membership (which directory the file lives in) determines routing. The `date` field is required and determines sort order.

### 4. URL Structure: `/{category}/{slug}/`

**Choice: `/{category}/{slug}/` -- e.g., `/articles/state-of-the-ecosystem/`.**

**Rationale:**

- **Readable and predictable.** A reader can see `/articles/state-of-the-ecosystem/` and understand both what kind of content it is and what it is about. The URL communicates two dimensions of information.
- **Stable over time.** Dates in URLs create the illusion that content expires. An article published on 2026-02-09 is still relevant in March. Keeping dates out of the URL means the URL remains valid and meaningful regardless of when it is shared.
- **SEO-friendly.** Category + descriptive slug is the standard pattern for content sites. Search engines weight URL path segments, so including the category improves discoverability.
- **Short.** Compared to `/{category}/{date}/{slug}/`, this avoids four extra path segments (`/2026/02/09/`) that add length without adding value for the reader.

The slug is derived from the filename, minus the date prefix and extension. For example, `2026-02-09__state-of-the-ecosystem.md` becomes `state-of-the-ecosystem`. The date prefix in the filename serves as a sort key in the repository; the permalink drops it.

Note: Jekyll's `:title` permalink token extracts the filename (minus extension). The double-underscore date prefix (`2026-02-09__`) will need to be handled -- either by configuring a permalink pattern that strips it, or by adding a `slug` field to front matter that overrides the filename-derived title. The Dev role should evaluate the cleanest approach during implementation. Adding `slug: state-of-the-ecosystem` to front matter is the most explicit and reliable method.

### 5. Theme: Jekyll Minima (Default)

**Choice: Jekyll's default `minima` theme, unmodified for v1.**

**Rationale:**

- **Mobile-responsive out of the box.** Minima uses a responsive layout that works on desktop, tablet, and mobile without any CSS customization.
- **Content-first design.** Minima's design is deliberately minimal: clean typography, comfortable line length, consistent spacing. This is exactly what the feature request specifies.
- **Zero configuration.** No theme files to download, no `_layouts/` to populate, no CSS to write. Jekyll uses minima by default.
- **Upgradeable.** If v2 needs a custom look, the theme can be overridden incrementally by adding files to `_layouts/`, `_includes/`, or `assets/css/`. Starting with minima does not lock out future customization.

The only customization for v1 is the landing page layout, which will need a custom `index.html` (or `index.md`) that lists recent publications across all collections. This is a single Liquid template, not a theme fork.

### 6. Feed Architecture: Global Feed + Per-Category Feeds

**Choice: Both a single global RSS feed and per-category feeds.**

**Implementation:**

- **Global feed:** Use the `jekyll-feed` plugin (included in the GitHub Pages gem). Configure it to aggregate all collections. This produces `/feed.xml`.
- **Per-category feeds:** Create a simple Liquid template for each collection that generates an Atom/RSS feed. These produce `/articles/feed.xml`, `/briefs/feed.xml`, `/interviews/feed.xml`, etc.

**Rationale:**

- **Global feed serves the primary use case.** The stakeholder wants to "visit a URL and see the latest articles." An RSS reader subscribed to `/feed.xml` delivers this.
- **Per-category feeds serve the specialist use case.** A reader who only wants daily briefs (or only investigations) can subscribe to that category's feed without noise from other categories.
- **Marginal cost is near zero.** Each per-category feed is a single Liquid template file (~20 lines). The build cost is negligible. There is no reason to withhold this capability.

The `jekyll-feed` plugin is whitelisted on GitHub Pages and requires only a `_config.yml` entry to activate.

### 7. Build Trigger: Path-Filtered

**Choice: Trigger builds only when content or site infrastructure files change.**

**GitHub Actions `on` configuration:**

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'publications/**'
      - '_config.yml'
      - '_layouts/**'
      - '_includes/**'
      - 'assets/**'
      - 'index.md'
      - 'index.html'
      - 'feed*.xml'
      - '_data/**'
```

**Rationale:**

- **Avoids wasted CI minutes.** Changes to `ROLE.md`, `tests/`, `scripts/`, `pyproject.toml`, or the Python package source should not trigger a site rebuild. The Journalist repo contains both Python package code and publication content; the build trigger should discriminate.
- **Covers all site-relevant paths.** The filter includes content (`publications/**`), Jekyll configuration (`_config.yml`, `_layouts/`, `_includes/`, `_data/`), static assets (`assets/**`), and the landing page (`index.md` or `index.html`).
- **Branch scope.** Builds trigger on `main` only. The feature request mentions `dev` as well, but deploying from a single branch avoids confusion about which version of the site is live. If a staging preview is desired later, a separate workflow with `dev` branch trigger can deploy to a different path.

### 8. Future Extensibility: v1 Does Not Over-Engineer

**Choice: v1 is scoped to the Journalist role's news site. Future web presence for other roles is a separate Decision.**

**Rationale:**

- **YAGNI (You Aren't Gonna Need It).** The Librarian knowledge base and Cartographer maps are mentioned as future possibilities, not current requirements. Designing v1 to accommodate hypothetical future needs would add complexity without delivering value now.
- **The boundary is clear.** The Journalist owns the news site. The news site lives in the Journalist repo. If the Librarian later wants a knowledge base, the Architect will create a new Decision evaluating whether to: (a) give the Librarian its own GitHub Pages site in its own repo, (b) consolidate multiple role sites into the parent dev repo, or (c) create a dedicated `Issues-FS__Web` repo. That decision depends on the Librarian's actual requirements, which do not yet exist.
- **v1 conventions transfer.** The front matter schema, URL structure, and Jekyll collection patterns established in v1 are portable. If a future consolidation occurs, the Journalist's content can be migrated by moving the `publications/` directory and adjusting `_config.yml`. No architectural lock-in.
- **What v1 establishes for the future:** A precedent that role web presence uses Jekyll on GitHub Pages with collections, YAML front matter, and path-filtered builds. This is a convention, not a constraint -- future Decisions can override it if the requirements demand a different approach.

---

## Front Matter Contract

All publishable content in `publications/` must include the following YAML front matter block. This is the contract between the Journalist (content author) and the site generator (consumer).

### Required Fields

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `title` | string | Display title of the publication | `"State of the Ecosystem"` |
| `date` | date (ISO 8601) | Publication date, used for sorting | `2026-02-09` |
| `summary` | string | One-sentence summary for landing page excerpts and RSS feed descriptions | `"A comprehensive look at where Issues-FS stands."` |
| `author` | string | Authoring role | `Journalist` |

### Optional Fields

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `topics` | list of strings | Tags for cross-referencing and future filtering | `[ecosystem, cli-bug, roadmap]` |
| `slug` | string | URL slug override (if filename-derived slug is unsuitable) | `state-of-the-ecosystem` |
| `type` | string | Content subtype for display purposes | `feature-article`, `daily-brief`, `q-and-a` |

The `category` is determined by collection membership (which subdirectory the file lives in), not by a front matter field. This avoids the possibility of a file in `articles/` declaring itself an `interview`.

---

## Consequences

### Positive

1. **Stakeholders can read content at a URL.** The primary goal is met: `https://owasp-sbot.github.io/Issues-FS__Dev__Role__Journalist/` serves a browsable, mobile-friendly news site.
2. **Zero manual deployment.** Push to `main`, site updates within minutes. The Journalist's workflow is unchanged except for adding front matter.
3. **Content remains repo-readable.** Files in `publications/` are still valid, readable markdown. The YAML front matter is the only addition, and it is ignored by GitHub's markdown renderer.
4. **RSS enables subscription.** Stakeholders can subscribe to all content or specific categories.
5. **Low implementation cost.** Jekyll with minima theme, collections, and the feed plugin is approximately: one `_config.yml`, one landing page template, one per-category feed template, and one GitHub Actions workflow. Estimated implementation effort: 1-2 days for Dev.
6. **Clear ownership boundary.** The Journalist role owns the content and the site. No cross-role coordination needed for publishing.

### Negative

1. **Longer URL.** `owasp-sbot.github.io/Issues-FS__Dev__Role__Journalist/` is not memorable. Mitigated by optional custom domain later.
2. **Front matter migration.** All existing content needs YAML front matter added. This is a one-time cost (currently ~5 files).
3. **Jekyll's Liquid templating is limited.** If the site later needs complex features (search, dynamic filtering, interactive maps), Jekyll's static generation model becomes a constraint. Mitigated by: v1 scope is deliberately simple, and migration to a more capable generator is possible.
4. **Single-repo scope limits future consolidation.** If three roles later want web presence, having three separate GitHub Pages sites creates fragmentation. Mitigated by: this is an explicit future Decision, and v1 content is portable.
5. **Double-underscore filenames.** The existing `YYYY-MM-DD__title.md` naming convention produces slugs with leading date prefixes unless overridden by `slug` in front matter. The Dev role must handle this during implementation.

---

## Alternatives Considered

### Static Site Generator: Hugo

Hugo builds faster, has a richer shortcode system, and does not require Ruby. However:
- It requires a custom GitHub Action (`peaceiris/actions-hugo` or similar) rather than GitHub's first-party `actions/jekyll-build-pages`.
- Its Go-based template syntax is more complex than Liquid.
- Build speed advantage is meaningless at <100 pages.
- The feature request and Journalist both recommended Jekyll.

**Rejected for v1.** Could be reconsidered if build times become a problem (unlikely) or if Hugo-specific features are needed (no current requirement).

### Site Location: Option B -- Dedicated New Repo (`Issues-FS__News`)

A dedicated repo would provide a cleaner URL and separation of content authoring from site rendering. However:
- It requires cross-repo content synchronization (either git submodules, GitHub Actions pulling from another repo, or a copy-on-push workflow).
- This adds a moving part that can break silently: if the sync fails, the site goes stale without the Journalist knowing.
- It creates a new repo that the DevOps role must scaffold and maintain.
- The separation of concerns argument is weaker than it appears: the Journalist's content IS the site's content. There is no separate "site" concern beyond configuration files.

**Rejected for v1.** Could be reconsidered if the site grows to serve multiple content sources.

### Site Location: Option C -- Parent Dev Repo (`Issues-FS__Dev`)

The parent dev repo is the central coordination point and could host a unified web presence. However:
- It mixes site infrastructure with development coordination artifacts.
- The parent repo contains 17 submodules; GitHub Pages deployment from a repo with submodules introduces complexity (submodule content is not checked out by default in GitHub Actions).
- It pre-optimizes for a multi-role web presence that does not yet exist.
- The Journalist loses direct control over deployment: any push to the parent repo triggers a build, and the parent repo is updated by many roles.

**Rejected for v1.** Remains a candidate for a future unified web presence if multiple roles need sites.

### Content Mapping: Jekyll `_posts/` Convention

Jekyll's default blog-aware mode uses a `_posts/` directory with `YYYY-MM-DD-title.md` filenames. This would require:
- Moving all content from `publications/` subdirectories into `_posts/`.
- Renaming files from `YYYY-MM-DD__title.md` to `YYYY-MM-DD-title.md`.
- Using categories in front matter rather than directory structure for routing.
- Losing the clean `publications/articles/` directory browsing experience in the repo.

**Rejected.** The stated constraint is that content must remain readable as plain markdown in the repo. The `publications/` directory structure is the Journalist's organizational taxonomy. Flattening it into `_posts/` destroys that structure and violates the constraint.

### URL Structure: `/{category}/{date}/{slug}/`

Includes the full date in the URL path (e.g., `/articles/2026/02/09/state-of-the-ecosystem/`). This is the WordPress/blogging convention. However:
- It makes URLs long and harder to share.
- It implies content has an expiration date tied to its publication.
- The date is available in the page metadata and front matter; duplicating it in the URL adds no information for the reader.
- It creates deeper directory nesting in the generated site.

**Rejected.** Date is metadata, not a routing concern.

### URL Structure: `/{date}/{slug}/`

Uses only the date without category (e.g., `/2026/02/09/state-of-the-ecosystem/`). This loses category context:
- A reader cannot tell from the URL whether they are reading an article, a brief, or an interview.
- Category pages lose their clean namespace.

**Rejected.** Category is the primary organizational axis and belongs in the URL.

### Feed Architecture: Global Feed Only

A single `/feed.xml` serving all content types. However:
- Daily briefs may be high-frequency. Stakeholders who want only feature articles would receive noise.
- Per-category feeds are trivially cheap to implement (one Liquid template each).
- There is no downside to offering both.

**Rejected as insufficient.** Global-only denies readers the ability to filter by interest.

### Build Trigger: All Pushes

Building on every push to `main`, regardless of which files changed. However:
- The Journalist repo contains Python package code, tests, and scripts alongside publications.
- Non-content changes (test fixes, ROLE.md updates, script changes) would trigger unnecessary builds.
- Path-filtered triggers are a standard GitHub Actions feature with no downside.

**Rejected as wasteful.**

---

## Testability Criteria (for QA)

1. **Build succeeds.** Pushing a new markdown file with valid front matter to `publications/articles/` on `main` triggers a GitHub Actions build that completes without error.
2. **Content appears on site.** The new article is visible on the landing page and its category page within 5 minutes of push.
3. **URL structure is correct.** An article at `publications/articles/2026-02-09__state-of-the-ecosystem.md` with `slug: state-of-the-ecosystem` in front matter is accessible at `/articles/state-of-the-ecosystem/`.
4. **RSS feeds validate.** Both `/feed.xml` and `/articles/feed.xml` pass W3C Feed Validation.
5. **Mobile rendering.** The site is readable on a 375px-wide viewport (iPhone SE) without horizontal scrolling.
6. **Path filter works.** Pushing a change to `tests/` or `ROLE.md` does NOT trigger a site build.
7. **Feature requests excluded.** Files in `publications/feature-requests/` do NOT appear on the site or in any feed.
8. **Front matter validation.** A file missing required front matter fields (`title`, `date`, `summary`, `author`) either fails the build with a clear error or is excluded from the site with a warning in the build log.
9. **Content readable in repo.** All files in `publications/` render correctly when viewed via GitHub's markdown renderer (front matter is hidden, content is displayed).

---

## Affected Components

| Component | Impact |
|-----------|--------|
| `Issues-FS__Dev__Role__Journalist` | New files: `_config.yml`, `index.md`, `_layouts/` (optional overrides), `.github/workflows/deploy-site.yml`, per-category feed templates. Modified files: all existing publications gain YAML front matter. |
| GitHub Pages | Enable GitHub Pages on the Journalist repo, source: GitHub Actions. |
| No other repos affected. | This Decision is scoped to the Journalist role repo. No changes to the parent dev repo, other role repos, or core modules. |

---

## Implementation Handoff

This ADR is handed off to the **Dev role** for implementation. The Dev deliverables are:

1. `_config.yml` with collections, permalink structure, theme, and feed plugin configuration.
2. `index.md` (or `index.html`) landing page template listing recent publications across all collections.
3. Per-category feed templates (`articles/feed.xml`, `briefs/feed.xml`, etc.).
4. `.github/workflows/deploy-site.yml` GitHub Actions workflow with path-filtered trigger.
5. YAML front matter added to all existing publications in `publications/`.
6. A `slug` field in front matter for any file whose filename-derived slug would be incorrect.

The Dev role should also evaluate the cleanest approach for stripping the `YYYY-MM-DD__` prefix from filenames when generating permalinks, and document the chosen method.

---

*Architecture Decision Record prepared by the Architect Role*
*Issues-FS__Dev__Role__Architect*
*ADR-001 | v1.0 | 2026-02-10*
