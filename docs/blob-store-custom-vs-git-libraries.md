# Blob Store: Custom vs Git Libraries

> Analysis of whether Cantica's blob store should be built on top of a git library (dulwich, pygit2) or remain a custom content-addressable store.

---

## The Candidates

| Library | Approach | Key dependency |
|---------|----------|----------------|
| **dulwich** | Pure-Python git implementation | None (pure Python) |
| **pygit2** | Python bindings for libgit2 | libgit2 (C, system library) |
| **GitPython** | Subprocess wrapper around git CLI | git binary in PATH |
| **Current custom** | SHA256-keyed flat files + SQLite | None |

GitPython is a non-starter: subprocess overhead, git binary required everywhere, slow for high-frequency writes. The real choice is **dulwich or pygit2 vs the current custom store**.

---

## What git actually gives you

Git's object model maps closely to Cantica's roadmap — the roadmap explicitly says so. The correspondence is:

| git concept | Cantica concept | Notes |
|-------------|-----------------|-------|
| blob | prompt content at a version | Exact match |
| commit | Version | Git commits point to *trees*, not blobs — see below |
| branch | Branch | In git, repo-scoped; in Cantica, prompt-scoped |
| tag | Tag | Close match |
| pack file | — | Delta compression across similar blobs |
| remote | Federation peer | Close match |
| wire protocol | `cantica push/pull` | Potentially reusable |

### The structural mismatch problem

Git's commit object does not point to a blob — it points to a **tree** (directory). For Cantica's single-file prompts:

```
commit abc123
  └── tree def456
        └── blob ghi789   ← "prompt.md" or similar
```

Every commit requires writing **three objects** (blob + tree + commit) instead of one. That's 3× the write I/O for the core operation. More critically, the tree object carries no meaning for Cantica — it's structural ceremony.

This also means git repo layout doesn't map cleanly to Cantica's data model:

- **One repo per prompt?** Thousands of repos at scale. Each repo is a directory with its own `.git/` and pack files. GC and repack become per-prompt operations.
- **One repo per namespace?** Better, but branches are repo-global in git — you can't have `osteck/prompt-a@experimental` and `osteck/prompt-b@experimental` as separate branch namespaces within one repo without hacks.
- **One monorepo?** Prompt identity becomes a path in the tree. Conflict-free concurrent commits to different prompts become tricky.

None of these layouts is clean. The current design — SQLite for metadata, flat content-addressed files for blobs — avoids this entirely.

---

## Arguments FOR git libraries

**1. Pack files and delta compression**

This is the strongest argument. Git's pack format stores similar objects as deltas against a base. Prompt versions are almost always small diffs of a common base — ideal input for delta compression. At scale (millions of versions), pack files could reduce disk usage by 80–90% vs storing full content per version.

The current custom store deduplicates *identical* content (same SHA = same file), but stores each distinct version as a full copy with no cross-version compression.

**2. Diff is battle-tested**

libgit2's diff handles encoding, binary detection, hunk merging, and many edge cases. `difflib.unified_diff` is fine for pure-text prompts but is also less configurable.

**3. Federation could reuse the git wire protocol**

Phase 7 federation (`cantica push/pull`) is semantically identical to `git push/pull`. If the storage layer speaks git's wire protocol, federation between Cantica instances becomes "just" git remotes. dulwich and pygit2 both implement the git wire protocol.

**4. dulwich is pure Python, zero system deps**

If going the git-library route, dulwich specifically adds no system dependency — it's a pip install. It implements the full git object model and wire protocol in Python.

**5. Proven at scale**

libgit2 (behind pygit2) powers GitHub, GitLab, and Bitbucket's storage layers. The object model is not an experiment.

---

## Arguments AGAINST git libraries

**1. SQLite is the real source of truth**

Cantica's metadata — namespaces, visibility, star counts, fork lineage, variable schemas, model hints — lives in SQLite. Git has no concept of any of this. Content would live in git objects; everything else in SQLite. That split means:

- Every write is two transactions: git object write + SQLite row update
- Every read may require fetching from git + joining with SQLite
- Consistency between the two systems must be maintained manually — a crash between the git write and the SQLite commit leaves them diverged

The current design is a single SQLite transaction per commit (the content SHA is just a string in the `versions` table; the blob file write is the only out-of-band step, and it's idempotent).

**2. Search requires content in the database anyway**

Phase 3 search (FTS5, pgvector embeddings) indexes the *content* of prompts. Whether content is stored in git objects or flat files, it must be extracted and indexed into SQLite/PostgreSQL. Git objects don't help here — they're just a more complex retrieval path to get to the same bytes.

**3. Commit SHA semantics differ**

Cantica's current commit SHA encodes `content + parent + author + message + timestamp`. Git's commit SHA encodes `tree + parent + author + committer + message + timestamp`. They're not the same hash. Adopting git libraries means adopting git's SHA semantics — including SHA1 (git's default) vs Cantica's SHA256. pygit2 supports SHA256 repos but it's the new `--object-format=sha256` mode, less battle-tested.

**4. Namespace isolation doesn't exist in git**

Git has no concept of user ownership, visibility (public/private/unlisted), or multi-tenancy. A git repo is a flat namespace. All of Cantica's access control, namespace scoping, and visibility must remain in SQLite regardless — git adds nothing there.

**5. Dependency weight**

- **dulwich**: ~20k lines of Python, brings in its own urllib3 usage, adds ~2MB to the package
- **pygit2**: requires `libgit2` C library — complicates Dockerfile (apt-get libgit2-dev), cross-platform packaging, and Lambda/serverless deploys
- **Current custom**: zero deps beyond the stdlib

**6. GC and maintenance overhead**

Git repos accumulate loose objects that must be periodically repacked (`git gc`). For thousands of prompts-as-repos, running GC is a background maintenance job you'd need to build and operate. Custom flat-file store has no GC — blobs are append-only and referenced by SHA; nothing is ever orphaned while the SQLite row exists.

**7. Federation is more than git push/pull**

Git push/pull replicates the object graph. Cantica federation (Phase 7) also needs to synchronize stars, fork metadata, and visibility decisions. The git wire protocol doesn't carry any of that. You'd still need a REST API layer for metadata sync — at which point the wire protocol reuse is only for blob/version content, which is a smaller win than it first appears.

---

## Verdict

**Keep the custom store.** The architectural argument is straightforward:

The fundamental problem is that Cantica is a **database-first** system — SQLite holds the canonical state, and blob files are a content-addressed backing store for that database. Git is a **filesystem-first** system with a rich semantic layer on top. Mixing them creates two sources of truth that must be kept in sync.

The one genuinely compelling argument for git libraries is **pack-file delta compression** at scale. But that's a Phase 7 problem (cloud, millions of versions, storage costs). The right response to that problem is: add optional pack-file compression to the custom store (dulwich can be used just for its pack-file I/O, without adopting its repo model), or migrate to PostgreSQL's TOAST compression + content-addressable blobs.

For the use cases that matter right now (Phase 1–3):

| Concern | Custom store | git library |
|---------|-------------|-------------|
| Correctness | ✅ Simple, auditable | ⚠️ Complex two-system sync |
| Zero deps | ✅ | ❌ dulwich or libgit2 |
| Search indexing | ✅ Content in DB | ✅ Same (extract either way) |
| Federation | ⚠️ Need to build | ⚠️ Partial help only |
| Storage efficiency | ⚠️ Full copies per version | ✅ Delta compression |
| Prompt-scoped branches | ✅ Natural | ⚠️ Mapping is awkward |
| Operational simplicity | ✅ | ❌ GC, repack, two stores |

The storage efficiency gap is real but not urgent until Phase 7. When it matters, the path is to introduce content-delta storage *into* the existing blob store rather than adopting git's repo layout wholesale.
