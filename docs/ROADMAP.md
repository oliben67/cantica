# Cantica — Roadmap

> *A versioned, community-driven vault for AI prompts.*

---

## Name

**Cantica** *(Latin: canticum, pl. cantica — song, hymn, chant)*

Prompts are incantations: words composed with care to summon a specific
performance from an AI. A cantica is a sacred chant — functional, precise,
and crafted to produce a response. Dante structured the *Divine Comedy* as
three cantica: distinct, versioned books within a unified, living work.

The metaphor carries through the entire product:
- You **compose** a prompt
- You **revise** it (commits)
- You **publish** a version (tags)
- Others **fork** and **remix** it
- Communities **collect** related prompts (collections)

Cantica sits alongside **songbook** in a music-themed product family:
songbook orchestrates AI agents; cantica is the library of words they speak.

---

## What Cantica Is

Cantica is a **git-flavored registry for prompts** — part personal vault,
part community hub, part package manager.

Like npm for Node modules or PyPI for Python packages, but for prompts:
- Every prompt is versioned, diffable, and forkable
- Namespaced by user or organization (`osteck/code-reviewer@v2.1`)
- Addressable by URI (`cantica://osteck/code-reviewer@v2.1`)
- Self-hostable locally or consumable from a cloud instance
- Searchable by text, tags, intent (semantic), and model compatibility

Songbook agents can reference prompts directly:
```yaml
agents:
  architect:
    system_prompt: cantica://community/senior-architect@v1.4
```

---

## Core Concepts

### Prompt
The primary object in Cantica. Not just raw text — a rich, structured artifact:

```yaml
name: senior-architect
namespace: community
description: "A senior software architect focused on correctness and scalability."
tags: [engineering, architecture, review, senior]
model_hints:
  - claude-opus-4-7
  - gpt-4o
license: MIT

variables:
  language:
    type: string
    default: "Python"
    description: "Primary programming language for this context"
  focus_area:
    type: string
    required: true

content: |
  You are a senior software architect specializing in {{language}}.
  Your focus is {{focus_area}}. Review for correctness, scalability,
  and maintainability. Flag architectural anti-patterns explicitly.
```

### Version
Every save is a **commit** with a message, author, and timestamp.
Versions are addressed as:
- `name@latest` — most recent commit on default branch
- `name@v1.3` — tagged stable release
- `name@abc123f` — pinned to a specific commit SHA
- `name@experimental` — a named branch

### Namespace
User or organization prefix — mirrors GitHub's model:
- `osteck/architect` — personal prompt
- `my-org/review-team` — organization prompt
- `community/` — curated public namespace (moderated)

### Collection
A named, curated set of prompts — like a GitHub repo with multiple files,
or a songbook's agent roster. Collections can be versioned as a unit.

### Fork
Copy a prompt into your namespace with full lineage tracked.
Upstream changes can be pulled in. PRs (prompt requests?) to upstream.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Interfaces                                                       │
│  ┌──────────────┐   ┌────────────────┐   ┌───────────────────┐  │
│  │  cantica CLI │   │  Web UI        │   │  songbook / IDEs  │  │
│  │  push/pull   │   │  (React SPA)   │   │  cantica:// URIs  │  │
│  │  search/diff │   │  browse/search │   │  SDK / HTTP       │  │
│  └──────┬───────┘   └───────┬────────┘   └─────────┬─────────┘  │
└─────────┼───────────────────┼─────────────────────┼─────────────┘
          │                   │     HTTP/REST        │
┌─────────▼───────────────────▼──────────────────────▼────────────┐
│  Cantica FastAPI Backend                                          │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  API  /v1/                                                │   │
│  │  prompts · versions · namespaces · collections · search   │   │
│  │  forks · stars · comments · tokens (auth)                 │   │
│  └────────────────────────┬──────────────────────────────────┘   │
│                            │                                      │
│  ┌─────────────────────────▼────────────────────────────────┐    │
│  │  Core Services                                            │    │
│  │                                                           │    │
│  │  ┌──────────────────┐   ┌──────────────────────────────┐ │    │
│  │  │  VersionStore    │   │  SearchService               │ │    │
│  │  │  (git-like)      │   │  - full-text (FTS)           │ │    │
│  │  │  - commits       │   │  - tag / category filter     │ │    │
│  │  │  - branches      │   │  - semantic (embeddings)     │ │    │
│  │  │  - tags          │   │  - model-hint filter         │ │    │
│  │  │  - diff          │   └──────────────────────────────┘ │    │
│  │  │  - fork lineage  │                                     │    │
│  │  └──────────────────┘   ┌──────────────────────────────┐ │    │
│  │                         │  CommunityService            │ │    │
│  │  ┌──────────────────┐   │  - stars / bookmarks         │ │    │
│  │  │  TemplateEngine  │   │  - forks & lineage           │ │    │
│  │  │  - {{variables}} │   │  - comments / discussion     │ │    │
│  │  │  - render/fill   │   │  - collections               │ │    │
│  │  │  - schema valid  │   └──────────────────────────────┘ │    │
│  │  └──────────────────┘                                     │    │
│  └───────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  Storage                                                  │   │
│  │  PostgreSQL (+ pgvector)   │   SQLite (local/self-hosted) │   │
│  │  Content-addressable blobs │   Git object store           │   │
│  └───────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
          │                              │
   Self-hosted                     Cantica Cloud
   (Docker Compose,                (cantica.dev — hosted,
    local vault)                    federated, public community)
```

### Federation Model
A local Cantica instance can:
- Serve as a private prompt vault (air-gapped, no cloud required)
- Mirror selected public namespaces from cantica.dev
- Push prompts to cantica.dev for community sharing
- songbook references `cantica://local/...` or `cantica://cantica.dev/...`

This mirrors the git remote model: a local repo with optional upstream remotes.

---

## Data Model

### Prompt Object
```
Prompt
├── id               UUID
├── namespace        string        ("osteck", "community", "my-org")
├── name             string        ("senior-architect")
├── slug             string        ("osteck/senior-architect")
├── description      string
├── tags             string[]
├── model_hints      string[]      (suggested models)
├── license          string        (MIT, Apache-2.0, etc.)
├── visibility       enum          (public, private, unlisted, team)
├── variables        VariableSchema[]
├── star_count       int
├── fork_count       int
├── default_branch   string        ("main")
├── created_at       datetime
└── updated_at       datetime

Version (Commit)
├── sha              string        (content hash, git-style)
├── prompt_id        UUID
├── branch           string
├── parent_sha       string | null
├── message          string        ("Fix variable default for Python context")
├── author           string
├── content          text          (the actual prompt body)
├── variables        VariableSchema[]
├── created_at       datetime
└── tags             Tag[]         (named stable releases: "v1.3", "stable")

Fork
├── id               UUID
├── source_slug      string        ("community/senior-architect")
├── source_sha       string        (SHA at time of fork)
├── fork_slug        string        ("osteck/senior-architect")
└── created_at       datetime
```

---

## Version Control Semantics

Cantica borrows git's object model, simplified for prompts:

| git concept | Cantica equivalent |
|-------------|-------------------|
| blob | prompt content at a version |
| commit | a saved version with message + author |
| branch | a named line of development (default: `main`) |
| tag | a named stable release (`v1.0`, `production`) |
| fork | copy to new namespace, lineage tracked |
| clone | pull all versions of a prompt locally |
| remote | another Cantica instance (local or cantica.dev) |
| push | publish local prompt to remote |
| pull | fetch latest version from remote |
| diff | side-by-side or unified diff of any two versions |
| merge | incorporate upstream changes into a fork |

### Addressing Scheme
```
cantica://[host/]namespace/name[@ref]

cantica://osteck/architect           → latest on main
cantica://osteck/architect@v2.1      → tagged release
cantica://osteck/architect@abc123    → pinned SHA
cantica://osteck/architect@draft     → branch
cantica://cantica.dev/community/arch → cloud instance
cantica://local/my-org/arch          → self-hosted local
```

---

## CLI

```bash
# Setup
cantica init                          # init a local vault
cantica login [--host cantica.dev]    # authenticate
cantica remote add origin cantica.dev # add a remote instance

# Authoring
cantica new osteck/my-prompt          # scaffold a new prompt
cantica edit osteck/my-prompt         # open in $EDITOR
cantica validate osteck/my-prompt     # lint schema + variables

# Versioning
cantica commit -m "Add Python default" # save a new version
cantica tag v1.0                       # tag current version
cantica branch experimental            # create a branch
cantica diff v1.0 v1.1                 # unified diff
cantica log osteck/my-prompt           # full version history
cantica rollback v1.0                  # restore a past version

# Sharing
cantica push osteck/my-prompt          # publish to remote
cantica pull community/architect       # fetch from remote
cantica fork community/architect       # fork into your namespace

# Discovery
cantica search "code review senior"    # full-text search
cantica search --tags engineering,review
cantica search --model claude-opus-4-7
cantica list --namespace community     # browse a namespace
cantica show osteck/architect@v2.1     # display prompt

# Using
cantica render osteck/architect --var language=Go  # fill variables
cantica get osteck/architect@v2.1                  # raw content
```

---

## API Endpoints

```
# Namespaces & prompts
GET    /v1/prompts                              # list/search (+ q, tags, model params)
POST   /v1/prompts                              # create prompt
GET    /v1/prompts/{namespace}/{name}           # get latest
DELETE /v1/prompts/{namespace}/{name}           # archive

# Versioning
GET    /v1/prompts/{namespace}/{name}/versions          # full history
POST   /v1/prompts/{namespace}/{name}/versions          # commit new version
GET    /v1/prompts/{namespace}/{name}/versions/{ref}    # get at SHA/tag/branch
POST   /v1/prompts/{namespace}/{name}/tags              # create tag
POST   /v1/prompts/{namespace}/{name}/diff              # diff two refs

# Branches
GET    /v1/prompts/{namespace}/{name}/branches
POST   /v1/prompts/{namespace}/{name}/branches

# Forks & community
POST   /v1/prompts/{namespace}/{name}/fork      # fork into caller's namespace
POST   /v1/prompts/{namespace}/{name}/star      # star
DELETE /v1/prompts/{namespace}/{name}/star
GET    /v1/prompts/{namespace}/{name}/forks     # list forks
GET    /v1/prompts/{namespace}/{name}/stargazers

# Collections
GET    /v1/collections
POST   /v1/collections
GET    /v1/collections/{namespace}/{name}
POST   /v1/collections/{namespace}/{name}/prompts  # add prompt to collection

# Rendering
POST   /v1/render                               # fill variables, get final text
  body: { slug, ref, variables: {key: value} }

# Search
GET    /v1/search?q=...&tags=...&model=...&semantic=true

# Federation
GET    /v1/federation/status                    # remote connectivity
POST   /v1/federation/push                      # push prompt to remote
POST   /v1/federation/pull                      # pull prompt from remote
GET    /v1/federation/remotes                   # configured remotes

# Auth
POST   /v1/tokens                               # create API token
DELETE /v1/tokens/{id}
GET    /v1/me                                   # current user
```

---

## Phases

---

### Phase 0 — Core Data Model & Storage
**Target: 1 week**

Goals:
- Prompt, Version, Tag, Branch, Namespace Pydantic models
- Content-addressable storage layer (SHA-keyed blobs — simple, no full git required)
- SQLite backend for local mode
- Addressing scheme: `namespace/name@ref` resolver

Deliverables:
- `VersionStore` service (commit, log, diff, tag, branch)
- `TemplateEngine` (variable substitution, schema validation)
- JSON Schema for prompt metadata
- Unit tests covering version resolution

---

### Phase 1 — Core API + Local Mode
**Target: 2 weeks**

Goals:
- FastAPI backend: full CRUD for prompts and versions
- API key auth (no OAuth yet — keep it simple)
- Self-hosted SQLite mode (zero external deps)
- Docker Compose for one-command local deployment

API coverage: prompts CRUD, versions, tags, branches, render, diff

Deliverables:
- Working FastAPI app (`cantica serve`)
- `cantica` CLI: `new`, `commit`, `push`, `pull`, `show`, `diff`, `log`, `render`
- `Dockerfile` + `docker-compose.yaml`
- README: quickstart in < 5 minutes

---

### Phase 2 — Full Versioning
**Target: 1 week**

Goals:
- Branch management (create, switch, list)
- Fork with full lineage tracking
- Merge (fast-forward only to start; conflict model TBD)
- `cantica fork`, `cantica branch`, `cantica merge`

Deliverables:
- Fork endpoint + CLI command
- Lineage graph queryable via API
- Rollback to any past SHA or tag

---

### Phase 3 — Search
**Target: 2 weeks**

Two tiers:

**Tier 1 — Structured search (PostgreSQL FTS / SQLite FTS5):**
- Full-text across name, description, tags, content
- Filter by tags, namespace, model hints, license, visibility

**Tier 2 — Semantic search (opt-in, requires embedding backend):**
- Embed prompt content at commit time (pluggable: OpenAI, Anthropic, local)
- Store in pgvector (Postgres) or a local vector file (SQLite mode)
- Query by intent: "find prompts that do code review"
- Similarity score in results

Deliverables:
- `GET /v1/search` with `q`, `tags`, `model`, `semantic` params
- `cantica search` CLI
- Configurable embedding backend (env var to disable)

---

### Phase 4 — Community & Sharing
**Target: 2 weeks**

Goals:
- User accounts + OAuth2 (GitHub login for cloud; local uses API keys only)
- Public / private / unlisted / team visibility per prompt
- Stars, bookmarks
- Comments/discussions (simple threaded, per prompt)
- Collections (curated named sets of prompts)
- Trending / recently updated discovery feeds

Deliverables:
- Auth: GitHub OAuth + API keys
- Stars, forks, comments endpoints
- Collections CRUD
- Discovery: `GET /v1/prompts?sort=stars|updated|forks`
- Namespace pages (all public prompts by user/org)

---

### Phase 5 — Web UI
**Target: 3 weeks**

React SPA, deployed at cantica.dev (or self-hosted).

Views:
- **Explore** — search/browse all public prompts
  - Filters: tags, model, license, sort order
  - Semantic search toggle
- **Prompt page** — full prompt view
  - Version picker (branch/tag/SHA dropdown)
  - Rendered preview (fill variables inline)
  - Version history timeline (like GitHub commits view)
  - Fork / Star / Copy URI buttons
  - Comments section
- **Diff view** — side-by-side or unified diff between any two refs
- **Editor** — create or edit a prompt
  - YAML front-matter for metadata
  - Prompt body with `{{variable}}` highlighting
  - Variable schema builder
  - Live preview: fill variables and see rendered output
- **Profile** — user's prompts, forks, stars, collections
- **Collection view** — browse a curated set

---

### Phase 6 — songbook Integration
**Target: 1 week**

Connect Cantica to the songbook FastAPI backend and VSCode extension.

Goals:
- songbook resolves `cantica://` URIs when starting a fleet
  - Fetches prompt content at specified ref
  - Caches locally (lock file pinning, like `uv.lock`)
  - Re-fetches on `--update-prompts` flag
- songbook CLI: `songbook prompts update` / `songbook prompts lock`
- VSCode extension: search Cantica from the agent editor inline
- `CanticaClient` Python SDK (thin wrapper around the REST API)

Songbook config example:
```yaml
agents:
  architect:
    system_prompt: cantica://community/senior-architect@v1.4
  reviewer:
    system_prompt: cantica://osteck/pr-reviewer@production
```

Lock file (`songbook.lock`):
```yaml
prompts:
  community/senior-architect:
    ref: v1.4
    sha: abc123def456
    fetched_at: 2026-05-23T09:00:00Z
  osteck/pr-reviewer:
    ref: production
    sha: 789xyz012
    fetched_at: 2026-05-23T09:00:00Z
```

---

### Phase 7 — Cloud Service *(ongoing)*
**Target: ongoing post-MVP**

Goals:
- Deploy cantica.dev: hosted cloud instance of Cantica
- PostgreSQL + pgvector backend (replaces SQLite)
- `community/` namespace: curated, moderated public prompts
- Organization accounts: shared private namespaces for teams
- Federation: self-hosted instances can push/pull from cantica.dev
- Usage analytics: prompt views, forks, usage via API token
- Rate limiting and abuse protection
- Prompt quality signals (model test results, community ratings)

Federation workflow:
```bash
# On a self-hosted Cantica instance:
cantica remote add upstream https://cantica.dev
cantica pull upstream community/senior-architect   # mirror public prompt
cantica push upstream osteck/my-prompt             # share with community
```

---

## Milestone Summary

| Phase | Scope | Duration |
|-------|-------|----------|
| 0 — Data Model | Prompt/Version/Branch/Fork models, VersionStore, TemplateEngine | 1 week |
| 1 — Core API + Local | FastAPI, SQLite, CLI (new/commit/push/pull/diff/log), Docker | 2 weeks |
| 2 — Full Versioning | Branches, forks, rollback, merge (fast-forward) | 1 week |
| 3 — Search | Full-text (FTS5), tag/model filters, semantic (pgvector, opt-in) | 2 weeks |
| 4 — Community | OAuth, stars, forks, comments, collections, discovery | 2 weeks |
| 5 — Web UI | React SPA: explore, prompt page, diff, editor, profiles | 3 weeks |
| 6 — songbook Integration | `cantica://` URI resolution, lock file, VSCode search, SDK | 1 week |
| 7 — Cloud | Hosted cantica.dev, PostgreSQL, federation, orgs | ongoing |

**MVP (Phases 0–1): 3 weeks** — local vault, CLI, self-hosted  
**Community-ready (Phases 0–5): ~12 weeks**  
**songbook-integrated platform (Phases 0–6): ~13 weeks**

---

## Relationship to songbook

```
songbook (fleet orchestrator)
   │
   ├── defines agents with system_prompt: cantica://...
   │
   └── Cantica (prompt registry)
          ├── local instance (self-hosted, private vault)
          │     └── mirrors selected namespaces from cloud
          └── cantica.dev (cloud, community hub)
                └── community/ namespace (curated, public)
```

Cantica is to prompts what PyPI is to Python packages:
a versioned, searchable, community-maintained registry
that any tool (songbook, VSCode, CI pipelines) can pull from.
