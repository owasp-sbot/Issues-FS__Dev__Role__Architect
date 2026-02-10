# ADR-002: Domain Architecture for issuesFS.ai

## Status: Proposed

## Date: 2026-02-10

## Trigger

Acquisition of the `issuesFS.ai` domain by the stakeholder. The ecosystem now has a custom domain and needs an architectural plan mapping all current and future web properties onto it. This Decision builds on ADR-001 (Jekyll-based news site in the Journalist repo) and the stakeholder's stated vision of "a hyperlinked world where you can zoom in and out" across the entire ecosystem.

## Context

### The ecosystem today

The Issues-FS ecosystem consists of 18 repositories under the `github.com/owasp-sbot/` GitHub organization:

| Category | Repositories |
|----------|-------------|
| **Parent** | `Issues-FS__Dev` (orchestration, submodule root) |
| **Modules** (6) | `Issues-FS`, `Issues-FS__CLI`, `Issues-FS__Docs`, `Issues-FS__Service`, `Issues-FS__Service__Client__Python`, `Issues-FS__Service__UI` |
| **Roles** (10) | `Issues-FS__Dev__Role__AppSec`, `__Architect`, `__Cartographer`, `__Conductor`, `__Dev`, `__DevOps`, `__Historian`, `__Journalist`, `__Librarian`, `__QA` |
| **Human** (1) | `Issues-FS__Dev__Human__Dinis_Cruz` |

Currently, none of these repositories have a custom domain. The Journalist repo (ADR-001) is the first to get a GitHub Pages site, currently reachable at the default `owasp-sbot.github.io/Issues-FS__Dev__Role__Journalist/` URL. The `Issues-FS__Docs` repo contains 49 technical documents across architecture, development guides, LLM briefs, and library references.

### Stakeholder direction

From the stakeholder interview (2026-02-10):

- "I want better visualization of code changes, architecture, work in progress. A hyperlinked world where you can zoom in and out."
- "Everything is open source. That's not even a question. All content, all code, all conversations -- published openly."
- "Librarian content, Cartographer maps -- all of it belongs here eventually."
- "A centralized site becomes the living pulse of the project."

The stakeholder wants a publicly accessible, interconnected web presence where each aspect of the ecosystem has its own home but everything links together into a navigable whole.

### GitHub Pages constraints

GitHub Pages imposes specific constraints on custom domain architecture:

1. **One org site per organization.** A repo named `owasp-sbot.github.io` serves as the organization's root GitHub Pages site at `https://owasp-sbot.github.io/`.
2. **Per-repo project sites.** Every other repo can enable GitHub Pages, served at `https://owasp-sbot.github.io/<repo-name>/`.
3. **One custom domain per repo.** Each repo's GitHub Pages can be configured with exactly one custom domain via a `CNAME` file in the publishing source.
4. **Apex domains require A records.** The bare domain `issuesFS.ai` must use A records pointing to GitHub's IP addresses. Subdomains use CNAME records.
5. **Automatic SSL.** GitHub Pages provides free SSL certificates for all custom domains, including subdomains.
6. **No server-side routing.** GitHub Pages is purely static. Subdomain-based separation is the only way to serve distinct sites under the same domain. Path-based routing within a single domain requires all content to live in one repo.

### Architectural question

Given one domain (`issuesFS.ai`), 18 repos, and GitHub Pages' one-domain-per-repo constraint, how should web properties be mapped to subdomains, which repos should serve which sites, and in what order should they be deployed?

---

## Decision

### 1. Subdomain architecture: one subdomain per concern

Each distinct web property gets its own subdomain of `issuesFS.ai`. The apex domain serves as the project landing page.

**Rationale for subdomains over path routing:**

- **GitHub Pages constraint.** Path routing (`issuesFS.ai/news`, `issuesFS.ai/docs`) would require all content to live in a single repo. This violates the ecosystem's single-responsibility-per-repo principle and creates deployment coupling -- a push to docs would rebuild news and vice versa.
- **Independent deployment.** Each subdomain maps to one repo. The repo owner deploys their site independently. No cross-repo coordination required.
- **Clear ownership boundaries.** The Journalist owns `news.issuesFS.ai`. The Librarian (future) owns `library.issuesFS.ai`. Ownership follows the role boundary, which follows the repo boundary, which follows the subdomain boundary. The architecture is fractal -- boundaries at every level.
- **Browser security model.** Subdomains are separate origins. If any site later needs client-side features (search, interactive maps), origin separation prevents cross-site interference.
- **Scalability.** Adding a new site means adding a DNS record and a CNAME file. No existing sites are touched.

### 2. Domain-to-repo mapping

| Domain | Purpose | Source Repository | SSG | Phase |
|--------|---------|-------------------|-----|-------|
| `issuesFS.ai` | Project landing page: what is Issues-FS, the role system, getting started, links to all repos and sites | `owasp-sbot.github.io` (new org site repo) | Jekyll | 2 |
| `www.issuesFS.ai` | Redirect to `issuesFS.ai` | DNS CNAME only (GitHub handles redirect) | N/A | 2 |
| `news.issuesFS.ai` | Journalist news site: articles, briefs, interviews, investigations | `Issues-FS__Dev__Role__Journalist` | Jekyll | 1 |
| `docs.issuesFS.ai` | Technical documentation: architecture guides, dev briefs, LLM briefs, API reference | `Issues-FS__Docs` | Jekyll (just-the-docs theme) | 3 |
| `cli.issuesFS.ai` | CLI documentation and quickstart | `Issues-FS__CLI` | Jekyll | 3 |
| `api.issuesFS.ai` | REST API and service documentation | `Issues-FS__Service` | Jekyll or Swagger UI | 4 |
| `maps.issuesFS.ai` | Cartographer: Wardley maps, dependency topology, architecture visualizations | `Issues-FS__Dev__Role__Cartographer` | Jekyll + custom JS | Future |
| `library.issuesFS.ai` | Librarian: knowledge catalog, cross-references, decision index, document registry | `Issues-FS__Dev__Role__Librarian` | Jekyll | Future |
| `history.issuesFS.ai` | Historian: narratives, timelines, decision genealogy, project evolution | `Issues-FS__Dev__Role__Historian` | Jekyll | Future |

**Key design choices in this mapping:**

- **Apex domain uses the org site repo.** Creating a new `owasp-sbot.github.io` repository gives the landing page its own home without overloading the parent `Issues-FS__Dev` repo (which has 17 submodules and serves as development coordination, not web hosting). The org site repo is the standard GitHub Pages pattern for apex domains.
- **CLI gets its own subdomain.** The CLI has distinct documentation needs (installation, command reference, quickstart tutorials) that justify separation from the general docs site. If the CLI content proves too thin for a standalone site, `cli.issuesFS.ai` can redirect to `docs.issuesFS.ai/cli/` via a simple HTML redirect page.
- **API subdomain reserved for the Service repo.** Whether this serves generated API documentation (Swagger/OpenAPI) or eventually a live API endpoint fronted by Cloudflare or similar, the subdomain is reserved. For Phase 4, it starts as documentation.
- **Future role sites are reserved, not committed.** The `maps`, `library`, and `history` subdomains are part of the architecture but will only be activated when those roles have publishable content. Reserving them in the DNS costs nothing and prevents naming conflicts later.

### 3. DNS configuration

All DNS records are configured at the domain registrar for `issuesFS.ai`.

```
; =============================================================
; issuesFS.ai DNS Zone Records for GitHub Pages
; =============================================================

; --- Apex domain (issuesFS.ai) ---
; GitHub Pages requires four A records for the apex domain.
; These IPs are GitHub's published Pages addresses.

@       A       185.199.108.153
@       A       185.199.109.153
@       A       185.199.110.153
@       A       185.199.111.153

; --- AAAA records for IPv6 (recommended by GitHub) ---

@       AAAA    2606:50c0:8000::153
@       AAAA    2606:50c0:8001::153
@       AAAA    2606:50c0:8002::153
@       AAAA    2606:50c0:8003::153

; --- www redirect ---
; GitHub Pages will redirect www -> apex when configured

www     CNAME   owasp-sbot.github.io.

; --- Active subdomains (Phases 1-4) ---

news    CNAME   owasp-sbot.github.io.
docs    CNAME   owasp-sbot.github.io.
cli     CNAME   owasp-sbot.github.io.
api     CNAME   owasp-sbot.github.io.

; --- Future subdomains (add when activating) ---

maps    CNAME   owasp-sbot.github.io.
library CNAME   owasp-sbot.github.io.
history CNAME   owasp-sbot.github.io.

; --- Domain verification (GitHub may require a TXT record) ---
; _github-pages-challenge-owasp-sbot  TXT  <verification-code>
```

**Notes on DNS:**

- All CNAME records point to `owasp-sbot.github.io.` -- GitHub's DNS resolves the correct repo based on the CNAME file in each repo.
- DNS records for future subdomains can be created now (they will simply return GitHub's 404 page until the corresponding repo has GitHub Pages enabled with a matching CNAME file). Alternatively, they can be added when each site goes live.
- TTL values: 3600 seconds (1 hour) is recommended for initial setup to allow quick corrections. Can be increased to 86400 (24 hours) once stable.

### 4. GitHub Pages CNAME setup per repo

Each repo that serves a site needs a `CNAME` file in its GitHub Pages publishing source (typically the repo root or `docs/` directory, depending on configuration).

| Repository | CNAME file contents | Publishing source | Notes |
|-----------|-------------------|-------------------|-------|
| `owasp-sbot.github.io` | `issuesFS.ai` | Root (`/`) | New repo to create. Org site. |
| `Issues-FS__Dev__Role__Journalist` | `news.issuesFS.ai` | GitHub Actions (ADR-001) | Existing repo. Add CNAME to Jekyll build output. |
| `Issues-FS__Docs` | `docs.issuesFS.ai` | GitHub Actions or `/docs` | Existing repo. Needs Jekyll setup. |
| `Issues-FS__CLI` | `cli.issuesFS.ai` | GitHub Actions or `/docs` | Existing repo. Needs Jekyll setup. |
| `Issues-FS__Service` | `api.issuesFS.ai` | GitHub Actions | Existing repo. API docs build. |

**CNAME file implementation:**

For repos using GitHub Actions for deployment (the ADR-001 pattern), the CNAME file must be included in the build output. Two approaches:

1. **Static CNAME in repo root.** Place a `CNAME` file at the repo root. Jekyll copies it to the build output by default.
2. **Generated during build.** The GitHub Actions workflow writes the CNAME file into the build artifact. This is more explicit but adds a build step.

**Recommendation:** Use approach 1 (static CNAME in repo root). It is simpler, survives build changes, and is the standard GitHub Pages pattern.

### 5. Phased rollout

The rollout is ordered by content readiness, dependency, and strategic value.

#### Phase 1: news.issuesFS.ai (Weeks 1-2)

**What:** Activate the custom domain for the Journalist news site.

**Why first:**
- ADR-001 is accepted and implementation is in progress.
- Content already exists (articles, interviews, briefs).
- The stakeholder identified the news site as "very important -- an enabling function."
- This is the simplest activation: one DNS record, one CNAME file, and the site that ADR-001 already specifies is live on a clean URL.

**Steps:**
1. Add DNS CNAME record: `news CNAME owasp-sbot.github.io.`
2. Add `CNAME` file containing `news.issuesFS.ai` to the Journalist repo root.
3. In the Journalist repo's GitHub Pages settings, set custom domain to `news.issuesFS.ai` and enable "Enforce HTTPS."
4. Wait for DNS propagation and GitHub SSL certificate provisioning (up to 24 hours, typically minutes).
5. Verify: `https://news.issuesFS.ai` serves the Journalist news site.

**Rollback:** Remove the CNAME file. The site reverts to `owasp-sbot.github.io/Issues-FS__Dev__Role__Journalist/`.

#### Phase 2: issuesFS.ai (Weeks 3-4)

**What:** Create the org site repo and deploy the project landing page.

**Why second:**
- The apex domain is the front door. Once the news site proves the custom domain workflow, the landing page establishes the ecosystem's public identity.
- The landing page is content-light (one page with links). It can be built quickly.
- Having `issuesFS.ai` live provides a natural home for cross-site navigation and the "zoom out" entry point.

**Steps:**
1. Create repo `owasp-sbot/owasp-sbot.github.io`.
2. Add Jekyll configuration and a landing page (`index.md`) covering: project overview, the role system, key concepts (graph-first, fractal, Memory-FS), links to all repos, links to all active sites.
3. Add `CNAME` file containing `issuesFS.ai`.
4. Add DNS A records (4 IPv4) and AAAA records (4 IPv6) for the apex domain.
5. Add DNS CNAME record: `www CNAME owasp-sbot.github.io.`
6. Configure GitHub Pages settings: custom domain `issuesFS.ai`, enforce HTTPS.
7. Verify: `https://issuesFS.ai` and `https://www.issuesFS.ai` both resolve.

**Landing page content (v1):**
- Hero section: "Issues-FS: A fractal, graph-based coordination system for humans and agents"
- Brief explanation of the project philosophy (graph-first, everything is a node, meaning from edges)
- The role system: visual overview of the 10 roles with one-line descriptions
- Quick links to active sites (news, docs when available)
- Links to all 18 GitHub repositories
- Getting started: pointer to the CLI repo and installation instructions
- License: CC BY for content, project license for code

#### Phase 3: docs.issuesFS.ai and cli.issuesFS.ai (Month 2)

**What:** Deploy technical documentation and CLI documentation sites.

**Why third:**
- The Docs repo already contains 49 documents across architecture, development, and library categories. This is the largest existing content corpus after the Journalist publications.
- Developer-facing documentation is essential for the stakeholder's goal of making everything "visible and publicly accessible."
- The CLI is the primary user-facing tool. Its documentation needs a discoverable home.

**docs.issuesFS.ai steps:**
1. Add Jekyll configuration to `Issues-FS__Docs` with a documentation-optimized theme (recommended: `just-the-docs` for its sidebar navigation, search, and multi-level hierarchy support).
2. Organize existing 49 docs into a Jekyll-compatible structure with front matter.
3. Add GitHub Actions workflow for site builds (same pattern as ADR-001).
4. Add `CNAME` file containing `docs.issuesFS.ai`.
5. Configure GitHub Pages, verify deployment.

**cli.issuesFS.ai steps:**
1. Assess CLI documentation volume. If sufficient for a standalone site: add Jekyll config, CNAME, and GitHub Actions workflow to `Issues-FS__CLI`.
2. If volume is insufficient: deploy a single-page site at `cli.issuesFS.ai` that provides installation instructions, a command overview, and links to `docs.issuesFS.ai` for detailed reference.
3. Add `CNAME` file containing `cli.issuesFS.ai`.

#### Phase 4: api.issuesFS.ai (Month 3)

**What:** Deploy API and service documentation.

**Why fourth:**
- The Service repo contains a working API but its documentation needs are less urgent than user-facing content.
- API documentation may benefit from auto-generation from OpenAPI specs, which requires the API surface to stabilize.

**Steps:**
1. Evaluate API documentation approach: static Jekyll site with manually written guides, or Swagger UI / Redoc auto-generated from OpenAPI spec.
2. Deploy chosen approach to `Issues-FS__Service` with GitHub Pages.
3. Add `CNAME` file containing `api.issuesFS.ai`.

#### Future phases: maps, library, history

These sites activate when the respective roles have publishable content:

- **maps.issuesFS.ai** -- When the Cartographer has Wardley maps, dependency topology visualizations, or architecture diagrams ready for web presentation. This site likely needs custom JavaScript for interactive visualizations, which Jekyll can host as static assets.
- **library.issuesFS.ai** -- When the Librarian has a structured knowledge catalog with cross-references, a decision index, and a document registry. This is a high-value site for the "zoom in and out" vision.
- **history.issuesFS.ai** -- When the Historian has narratives, timelines, or decision genealogy visualizations. May share visualization infrastructure with the Cartographer.

**Activation protocol for future sites:**
1. Role produces sufficient content to justify a standalone site (minimum: 5+ pages or a structured catalog).
2. Architect reviews the site scope and confirms the subdomain assignment.
3. DevOps adds the DNS record (if not already present), creates the CNAME file, and configures GitHub Pages.
4. Dev implements the Jekyll site structure following ecosystem conventions.

### 6. Cross-site navigation strategy

The stakeholder's "hyperlinked world where you can zoom in and out" requires that sites are not isolated islands. Users must be able to navigate between sites fluidly.

#### Navigation bar contract

All sites under `issuesFS.ai` must include a shared top-level navigation bar with links to all active sites. This is a **cross-site contract**: every site implements the same nav bar, ensuring consistent wayfinding.

**Nav bar specification (v1):**

```
[Issues-FS]  News  Docs  CLI  API  [GitHub]
```

Where:
- `[Issues-FS]` links to `https://issuesFS.ai` (the landing page / "zoom out" to top level).
- `News` links to `https://news.issuesFS.ai`.
- `Docs` links to `https://docs.issuesFS.ai`.
- `CLI` links to `https://cli.issuesFS.ai`.
- `API` links to `https://api.issuesFS.ai`.
- `[GitHub]` links to `https://github.com/owasp-sbot/`.
- Future sites are added to the nav bar as they go live.

**Implementation approach:**

Since GitHub Pages sites are statically generated and cannot share server-side includes, the nav bar must be implemented per-site. Two options:

1. **Jekyll include file.** Each site defines a `_includes/ecosystem-nav.html` partial that renders the nav bar. When a new site goes live, each site's nav include is updated. This is a manual coordination step but occurs infrequently (only when a new site launches).

2. **Client-side JavaScript snippet.** A shared JavaScript file hosted on `issuesFS.ai` (the landing page) dynamically injects the nav bar into every page. Each site includes a `<script src="https://issuesFS.ai/assets/js/ecosystem-nav.js">` tag. Updating the nav bar requires changing one file in one repo.

**Recommendation:** Start with option 1 (Jekyll include) for v1. It has zero JavaScript dependency, works with any theme, and the update frequency is very low (a new site launches at most once per month). If the ecosystem grows beyond 5-6 active sites and nav bar updates become burdensome, migrate to option 2.

#### Visual consistency

Sites do not need to use the same Jekyll theme (docs sites need sidebar navigation that a news site does not), but they should share:

- **Color palette.** A shared set of primary, secondary, and accent colors defined as CSS variables. Distributed as a shared CSS file or documented as a style guide.
- **Typography.** A shared font stack. System fonts are recommended for performance: `-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif`.
- **Footer.** A consistent footer across all sites with: the Issues-FS name, CC BY license notice, GitHub organization link, and links to all active sites.

#### Interlinking conventions

- **News articles reference docs.** When the Journalist writes about an architectural decision, the article links to the relevant doc on `docs.issuesFS.ai`.
- **Docs reference news.** When a technical document has been covered in a news article, the doc links to the article on `news.issuesFS.ai`.
- **Landing page aggregates.** The landing page (`issuesFS.ai`) shows recent activity across all sites: latest news article, latest doc update, latest CLI release.
- **All sites link to source.** Every page on every site includes a "View source on GitHub" link pointing to the markdown file in the source repo. This reinforces the open-source commitment and provides the "zoom in" path from published site to raw source.

### 7. Org site repo specification

The `owasp-sbot.github.io` repo needs to be created. Its scope and structure:

**Scope:** Project landing page for the Issues-FS ecosystem. One responsibility: present the project to the world and link to everything else.

**Not in scope:** Documentation, news, API reference, or any role-specific content. Those live in their respective repos and subdomains.

**Structure:**

```
owasp-sbot.github.io/
├── CNAME                    # Contains: issuesFS.ai
├── _config.yml              # Jekyll configuration
├── index.md                 # Landing page content
├── getting-started.md       # Getting started guide (links to CLI)
├── roles.md                 # The role system explained
├── _layouts/
│   └── default.html         # Custom layout with ecosystem nav
├── _includes/
│   └── ecosystem-nav.html   # Shared nav bar
├── assets/
│   ├── css/
│   │   └── ecosystem.css    # Shared color palette and typography
│   └── images/
│       └── ...              # Logo, role icons, diagrams
├── .github/
│   └── workflows/
│       └── deploy-site.yml  # GitHub Actions for Jekyll build
└── README.md
```

**This repo is owned by the Architect** (defines the structure and nav contract) with implementation by the Dev role and content contributions from the Librarian and Journalist roles. DevOps scaffolds the repo and CI pipeline.

---

## Consequences

### Positive

1. **Memorable, professional URLs.** `news.issuesFS.ai` instead of `owasp-sbot.github.io/Issues-FS__Dev__Role__Journalist/`. Every site gets a clean, shareable URL that communicates its purpose.

2. **Independent deployment.** Each site deploys independently from its own repo. A push to the Journalist repo updates `news.issuesFS.ai` without affecting `docs.issuesFS.ai` or any other site. This preserves the ecosystem's single-responsibility-per-repo principle at the web layer.

3. **Stakeholder's "zoom in and out" vision realized.** The landing page (`issuesFS.ai`) provides the "zoom out" view of the entire ecosystem. Subdomains provide "zoom in" to specific concerns. Cross-site navigation enables fluid movement between levels. Source links on every page provide "zoom in" all the way to raw source code.

4. **Zero hosting cost.** GitHub Pages is free for public repos. SSL certificates are automatic. No servers, no CDN configuration, no monthly bills.

5. **Incremental rollout.** Each phase is independent. Phase 1 (news) can go live while Phase 2 (landing page) is still being built. A delay in Phase 3 (docs) does not block Phase 1 or 2.

6. **Future-proof.** Reserving subdomains for maps, library, and history costs nothing and ensures the naming scheme is consistent when those sites activate. The architecture scales to any number of subdomains.

7. **Open and visible.** Everything the stakeholder asked for: public, open-source, CC BY licensed content, accessible at clean URLs. "No black magic."

### Negative

1. **DNS management overhead.** Someone must have access to the domain registrar to add/modify DNS records. This is a manual step outside the GitHub workflow. Mitigated by: DNS changes are infrequent (only when new sites launch), and the records are documented in this ADR.

2. **Cross-site nav bar maintenance.** When a new site launches, every existing site's nav bar must be updated. With the Jekyll include approach (option 1), this means updating a file in each active site's repo. Mitigated by: launches are infrequent, and the update is a one-line addition to an HTML include.

3. **New repo required.** The landing page needs a new `owasp-sbot.github.io` repo. This adds a repo to the ecosystem (19 total). Mitigated by: the org site repo is the standard GitHub Pages pattern, it is minimal in scope, and it serves a distinct purpose that no existing repo covers.

4. **SSL certificate provisioning delay.** When a new custom domain is configured, GitHub must provision an SSL certificate. This typically takes minutes but can take up to 24 hours. Mitigated by: this is a one-time delay per site, not ongoing.

5. **Subdomain proliferation risk.** If every role eventually wants a site, the ecosystem could end up with 10+ subdomains. Mitigated by: the activation protocol requires Architect review and a content readiness threshold. Subdomains are created only when justified.

6. **No shared search.** Each Jekyll site has its own search (if configured) but there is no cross-site search. A user on `news.issuesFS.ai` cannot search for content on `docs.issuesFS.ai`. Mitigated by: this is acceptable for v1. Cross-site search is a future enhancement that could be implemented via a client-side search index on the landing page or a dedicated search service.

---

## Alternatives Considered

### Alternative 1: Single-repo monolithic site (all content in one repo, path routing)

Serve everything from one repo: `issuesFS.ai/news/`, `issuesFS.ai/docs/`, `issuesFS.ai/cli/`, etc.

**Advantages:**
- One repo, one build, one deployment.
- Path routing is simpler than subdomain routing for end users.
- Cross-site navigation is trivial (it is all one site).
- Shared search works naturally.

**Disadvantages:**
- **Violates single-responsibility-per-repo.** All content from all roles must live in or be synced into one repo. The Journalist loses deployment autonomy. A bug in the docs build breaks the news site.
- **Content synchronization problem.** If content stays in its source repos, a sync mechanism must pull it into the monolithic repo on every push. This is the cross-repo sync complexity that ADR-001 rejected for the news site.
- **Build coupling.** Every push to any content source triggers a full-site rebuild. Build times grow linearly with content volume.
- **GitHub Pages constraint.** A single repo gets a single GitHub Pages deployment. Path routing within that deployment is purely a Jekyll routing concern, not a GitHub Pages feature.

**Rejected.** The coupling cost is too high. The ecosystem is designed for independent repos with independent deployment. A monolithic site inverts that design.

### Alternative 2: Issues-FS__Dev parent repo hosts the landing page (instead of org site repo)

Use the existing parent dev repo for the apex domain instead of creating a new `owasp-sbot.github.io` repo.

**Advantages:**
- No new repo needed.
- The parent repo is already the ecosystem's coordination hub.

**Disadvantages:**
- **Submodule complexity.** The parent repo contains 17 submodules. GitHub Pages builds do not check out submodules by default. The build workflow must explicitly handle submodule initialization, which adds fragility.
- **Mixed concerns.** The parent repo coordinates development (sprint planning, cross-repo issues, submodule pointers). Adding a website to it mixes development coordination with public-facing web content.
- **Deploy noise.** Any submodule pointer update, any `.issues/` change, any role coordination file change would potentially trigger a site rebuild (unless path-filtered very carefully).
- **The parent repo already has a purpose.** Its stated responsibility is development orchestration. A website is a separate responsibility.

**Rejected.** The parent repo's scope is development coordination. The landing page is a separate concern and deserves its own repo. The org site repo pattern (`owasp-sbot.github.io`) is the standard, well-understood GitHub Pages approach.

### Alternative 3: Dedicated `Issues-FS__Web` repo for the landing page

Create a new repo named `Issues-FS__Web` instead of using the org site pattern.

**Advantages:**
- Follows the `Issues-FS__` naming convention.
- Could later grow into a monorepo for shared web assets.

**Disadvantages:**
- **Apex domain routing.** The org site repo (`owasp-sbot.github.io`) has a special relationship with GitHub Pages: it serves the org's root URL. A project repo requires the site to live at `owasp-sbot.github.io/Issues-FS__Web/`, which then needs a custom domain configuration just like any other project site. This works, but it is a project site pretending to be the org site.
- **Convention mismatch.** The `owasp-sbot.github.io` naming is GitHub's convention for org sites. Fighting it adds friction for no benefit.

**Rejected.** The org site repo is the natural home for the apex domain. Using a project repo adds unnecessary indirection.

### Alternative 4: Path-based routing for some sites (hybrid approach)

Use subdomains for major sites (news, docs) but path routing for minor ones (CLI lives at `docs.issuesFS.ai/cli/` instead of `cli.issuesFS.ai`).

**Advantages:**
- Fewer DNS records and CNAME files.
- CLI docs co-located with general docs may improve discoverability.

**Disadvantages:**
- **Inconsistent model.** Some sites are subdomains, some are paths. Users and contributors must know which is which.
- **Couples CLI and Docs deployments.** If CLI docs live in the Docs repo, the CLI repo cannot deploy its own documentation. Changes to CLI docs require a PR to the Docs repo.
- **Repo boundary violation.** CLI documentation is generated from and maintained alongside CLI code. Moving it to another repo breaks the co-location principle.

**Partially accepted as a fallback.** If `cli.issuesFS.ai` proves too thin for a standalone site, it can redirect to `docs.issuesFS.ai/cli/` with a meta redirect. But the starting position is one subdomain per concern, with demotion only if content does not justify separation.

### Alternative 5: Cloudflare or Netlify for hosting (instead of GitHub Pages)

Use a more flexible hosting platform that supports path-based routing, edge functions, and shared configuration.

**Advantages:**
- Path routing across multiple repos via build plugins or proxying.
- More powerful build options (monorepo support, incremental builds).
- Edge functions for dynamic features (search, redirects).

**Disadvantages:**
- **Cost.** Cloudflare Pages and Netlify have free tiers, but they introduce a dependency on a third-party service with its own constraints and pricing changes.
- **Complexity.** Build configuration, deployment pipelines, and DNS management are more involved than GitHub Pages' zero-config approach.
- **Stakeholder preference.** "Start with GitHub Pages" was the explicit direction. Moving to another platform is a separate Decision if GitHub Pages proves insufficient.
- **Ecosystem convention.** ADR-001 established GitHub Pages as the hosting platform. Consistency matters.

**Rejected for now.** GitHub Pages meets all current requirements. If future needs (cross-site search, server-side redirects, dynamic features) exceed GitHub Pages' capabilities, a migration to Cloudflare Pages or Netlify is a natural evolution point and a new ADR.

---

## Testability Criteria (for QA)

1. **DNS resolution.** Each configured subdomain resolves to GitHub's IP addresses. Verify with `dig news.issuesFS.ai` (should return CNAME to `owasp-sbot.github.io`), `dig issuesFS.ai` (should return A records for GitHub's IPs).

2. **HTTPS works.** All URLs (`https://issuesFS.ai`, `https://news.issuesFS.ai`, `https://docs.issuesFS.ai`, etc.) serve valid SSL certificates and load without certificate warnings.

3. **www redirects.** `https://www.issuesFS.ai` redirects to `https://issuesFS.ai`.

4. **Correct content per subdomain.** Each subdomain serves content from its designated repository, not from another repo.

5. **Cross-site navigation.** Every active site's nav bar includes working links to all other active sites. No broken links, no stale entries.

6. **Independent deployment.** Pushing a change to the Journalist repo updates `news.issuesFS.ai` without affecting `docs.issuesFS.ai` or `issuesFS.ai`. Verify by checking deployment timestamps.

7. **Rollback works.** Removing a CNAME file from a repo causes the site to revert to its default `owasp-sbot.github.io/<repo-name>/` URL. The custom domain stops serving content (GitHub returns 404 for the custom domain).

8. **Future subdomains do not break.** DNS records for future subdomains (maps, library, history) that do not yet have GitHub Pages enabled return GitHub's 404 page, not a DNS error.

---

## Affected Components

| Component | Impact |
|-----------|--------|
| **DNS for issuesFS.ai** | New: all DNS records specified in this ADR must be created at the domain registrar. |
| **`owasp-sbot.github.io`** | New repo: must be created, scaffolded with Jekyll, and populated with landing page content. |
| **`Issues-FS__Dev__Role__Journalist`** | Modified: add `CNAME` file containing `news.issuesFS.ai`. Update `_config.yml` with `url: https://news.issuesFS.ai`. Add ecosystem nav bar include. |
| **`Issues-FS__Docs`** | Modified (Phase 3): add Jekyll site configuration, CNAME file, GitHub Actions workflow, ecosystem nav bar. |
| **`Issues-FS__CLI`** | Modified (Phase 3): add Jekyll site or redirect page, CNAME file. |
| **`Issues-FS__Service`** | Modified (Phase 4): add API documentation site, CNAME file. |
| **All active sites** | Ongoing: ecosystem nav bar must be updated when new sites launch. |

---

## Implementation Handoffs

| Handoff | To Role | Deliverable |
|---------|---------|-------------|
| DNS record creation | Human stakeholder (domain registrar access) | All DNS records from Section 3 of this ADR |
| Org site repo creation and scaffolding | DevOps | Create `owasp-sbot/owasp-sbot.github.io` with CI pipeline |
| Landing page content | Librarian + Journalist | Project overview, role descriptions, getting started content |
| Landing page implementation | Dev | Jekyll site in the org site repo |
| Journalist CNAME activation | Dev | Add CNAME file and URL config to Journalist repo |
| Docs site setup (Phase 3) | Dev | Jekyll configuration for `Issues-FS__Docs` repo |
| Cross-site nav bar template | Dev | `_includes/ecosystem-nav.html` template for all sites |
| Visual identity / style guide | Architect (define) + Dev (implement) | Shared CSS variables, typography, color palette |

---

## References

- [ADR-001: GitHub Pages News Site Architecture](adr__001__github-pages-news-site.md) -- Jekyll choice, Journalist repo deployment, front matter contract
- [Stakeholder Interview (2026-02-10)](../../Issues-FS__Dev__Role__Journalist/publications/interviews/2026-02-10/v0.1.3__chatgtp-stakeholder__raw-interview.md) -- Vision for visibility, open access, hyperlinked navigation
- [GitHub Pages Custom Domains Documentation](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site) -- Official GitHub documentation on custom domain setup
- [GitHub Pages IP Addresses](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site#configuring-an-apex-domain) -- Current A record IPs for apex domains

---

*Architecture Decision Record prepared by the Architect Role*
*Issues-FS__Dev__Role__Architect*
*ADR-002 | v1.0 | 2026-02-10*
