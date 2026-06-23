# Multi-Wiki Knowledge Graph — Implementation Plan

> **Status:** Design locked, ready to implement V0.
> **Owner:** Solo maintainer.
> **Primary use case:** Operational runbooks (incident-time correctness is the dominant constraint).
> **Last updated:** 2026-06-23

This plan describes a system for maintaining several independent technical wikis
(`databricks`, `aws`, `jenkins`, `aidlc`, …) that are densely linked internally and
sparsely linked to each other — a knowledge graph of tightly-connected clusters with a
few bridges between clusters. It is the product of two rounds of multi-lens review
(architect, analyst, critic, advisor, ops/security, team-lead) and four locked design
decisions (see §1.2).

---

## 1. Executive summary

### 1.1 The two ideas that drive everything

1. **Strict one-way data flow.** Three things are hand-authored and durable; everything
   else is *generated* and disposable. Any place a generated file can also be edited by
   hand is a latent drift bug. The compiler enforces the boundary.

2. **Build the boring core first.** The first milestone (V0) is a deterministic
   Markdown-graph compiler with **no LLM at all**. Identity, links, provenance, and
   artifact freshness must be locked down before any AI assistance, entities, scheduling,
   or dashboards are added. LLM output is a *cached, provenance-stamped suggestion* — never
   canonical truth, and especially never an unverified operational step.

### 1.2 Locked decisions

| Decision | Choice | Consequence |
|---|---|---|
| Primary use case | **Operational runbooks** | Freshness, source-of-truth, and human-verification are elevated to V0/V1 (not deferred). A stale or hallucinated step is an incident risk. |
| Page ID style | **Readable, registry-owned tokens** (`aws:step-functions`) | Greppable and easy to author; treated as an immutable token once minted. Renames happen via alias, never by editing the ID. |
| Generated artifacts in git | **Committed** | Graph changes are visible in branch-review diffs; the `no-dirty-graph` gate works. |
| Raw / CI trust boundary | **Local-only gates, no cloud CI** | The `check` command is the single gate, wired as a git pre-commit hook. Raw never leaves the machine; simplest, safest posture for infra secrets. Tradeoff: validation runs only when invoked. |

### 1.3 The three authored sources of truth

Identity decisions cannot be derived from current content (you cannot look at today's
files and know that page X was *split* from page Y, or that one ID is an *alias* of
another). So there are three durable, hand-authored inputs — not one:

```
content/    page frontmatter + body                     (what the wiki says)
registry/   append-only identity ledger                 (which IDs exist, and their history)
shared/     taxonomy, style guide, terminology           (cross-wiki vocabulary & rules)
```

Everything in `generated/` is a pure function of those three.

---

## 2. Architecture

### 2.1 One-way dependency graph

```
HAND-AUTHORED (source of truth)          GENERATED (build output; never hand-edited)
  content/<wiki>/*.md  (frontmatter+body) ─┐
  registry/*.yaml      (identity ledger) ──┼─►  generated/index.json          id → path, status, degree
  shared/*.yaml|*.md   (vocab/style)     ──┘    generated/backlinks.json      inbound refs per page
                                               generated/graph.json          nodes + scope-tagged edges
                                               generated/coverage.json       gate metrics
                                               generated/redirects.json      flattened alias/redirect map
                                               generated/provenance.json     page → source files + hashes
                                               content/<wiki>/*.md            compiler-owned <!--related--> region
```

The "## Related Pages" footer is the one piece of generated output that must live *inside*
an authored page (Markdown has no native transclusion). It is confined to a
**compiler-owned fenced region** (`<!-- related:start -->` … `<!-- related:end -->`); the
surrounding page body stays authored. The region is treated exactly like `generated/` for
drift purposes.

**Enforced rule:** the compiler regenerates `generated/` **and every `<!--related-->`
region** from scratch on each run, then `check` asserts `git diff --exit-code` over both.
If a human edited a generated file or a related region, or an input changed without
recompiling, the gate fails.

### 2.2 Directory layout

```
~/wikis/
  .git/
  PLAN.md                       this document
  Makefile                      thin wrappers: `make check|build|audit|migrate|new-wiki`
  .githooks/pre-commit          runs `make check`

  bin/                          the compiler & tools (no LLM in V0)
    wikigraph                   single entrypoint CLI; subcommands:
                                  build | check | audit | new | rename | split | merge | migrate
                                  (Makefile `new-wiki` → `wikigraph new`; `migrate` → `wikigraph migrate`)
    redact                      secret redaction (runs at ingestion boundary)

  schemas/                      versioned JSON Schemas (the contracts)
    frontmatter.schema.json
    registry.schema.json
    redirects.schema.json
    provenance.schema.json
    artifact-header.schema.json
    config.schema.json

  shared/
    style-guide.md
    terminology.yaml            global vocabulary
    source-rules.yaml           source-tier priority & "what may leave the machine"
    redaction-rules.yaml        regex denylist + canary secret(s)
    templates/
      page.md                   page scaffold with required frontmatter
      runbook.md                runbook scaffold (steps block is human-only)

  registry/                     AUTHORED — append-only identity ledger
    ids.yaml                    canonical id → {wiki, slug, status, created, tombstoned?}
    aliases.yaml                old-id → canonical-id (rename/merge)
    events.log.yaml             append-only: create/rename/split/merge/tombstone events

  content/                      AUTHORED — the wikis
    databricks/
      config.yaml               wiki-local: title, terminology extensions, source roots
      raw/                      source material (gitignored if it may contain secrets — see §6)
      *.md                      flat wiki pages (frontmatter + body)
    aws/        …
    jenkins/    …
    aidlc/      …               (32 existing files migrated here — see §7)

  generated/                    GENERATED — committed, each file carries an artifact header
    index.json
    backlinks.json
    graph.json
    coverage.json
    redirects.json
    provenance.json
    reports/
      unresolved-refs.json
      changed-graph.json
      stale-sources.json
      redaction-hits.json

  pipeline/                     V2+ ONLY — LLM assistance (absent in V0)
    agents/
      compile.md                prompt: raw → fenced agent regions
      lint.md                   prompt: style/clarity suggestions
    fixtures/                   mocked LLM I/O for reproducible local tests
    cache/                      committed cache of LLM suggestions, keyed by input hash
```

### 2.3 Page frontmatter (canonical schema, V0 minimal)

```yaml
---
id: aws:step-functions          # registry-owned, immutable token. Renames go through aliases.
title: AWS Step Functions       # may change freely
slug: step-functions            # filename-ish; may change freely
status: draft                   # draft | reviewed | stable   (stable = regen-frozen, see §4)
kind: page                      # page | runbook
source_refs:                    # provenance + the real staleness/skip signal
  - path: content/aws/raw/sfn-overview.md
    sha256: 9f2a…
    tier: official_docs         # official_docs | internal_notes | impl_examples | llm_generated
    verified: 2026-06-20
# --- runbook-only, required when kind: runbook and status in {reviewed, stable} ---
verified_by: human              # human | none   (operational steps must be human-verified)
last_verified: 2026-06-20
---
```

Derived fields (`source_count`, `source_priority`, `in_degree`, `out_degree`, backlinks)
are **never** written here — they live in `generated/` and are computed. `check` rejects
any generated field appearing in authored frontmatter unless it is on an explicit
allowlist.

### 2.4 Cross-references

- Authors write `[[aws:step-functions]]` (intra- or inter-wiki, same syntax).
- The compiler resolves `id → current path` via `generated/index.json` (which is built from
  `registry/` + `content/`) and emits both inline relative-Markdown links and an
  auto-generated footer into the page's **compiler-owned fenced region** (§2.1):

  ```markdown
  <!-- related:start -->
  ## Related Pages
  | Topic | Wiki | Why |
  |---|---|---|
  | AWS Step Functions | aws | Cloud orchestration alternative |
  <!-- related:end -->
  ```

- Every resolved edge is tagged `scope: intra | inter` so the "dense clusters, sparse
  bridges" goal is **reported** in `coverage.json` from V0/V1 onward. (Reporting ≠
  enforcement: the coverage *thresholds* in §9 become hard gates only at V3 — see the §8
  sequencing note — but the numbers are visible from the start.)
- Resolution is a **non-fatal compile step that reports** dangling/ambiguous links to
  `generated/reports/unresolved-refs.json`; the **`check` gate fails** on any unresolved
  `[[id]]`. The compile itself never crashes on a bad link.

---

## 3. Identity lifecycle

IDs are **registry-owned**, not path-owned. The registry is append-only; the resolved
index is generated from it. Every operation below is a `bin/wikigraph` subcommand that
writes a `registry/events.log.yaml` entry and updates `ids.yaml`/`aliases.yaml`.

| Operation | Rule |
|---|---|
| **Create** | Mint a readable ID; fail if it collides with any existing ID **or alias**. |
| **Rename (title/slug)** | Edit `title`/`slug` freely. The **ID does not change.** No registry event. |
| **Rename (ID itself)** | Discouraged. If required: new ID + `aliases: old → new`; inbound `[[old]]` keeps resolving. |
| **Split (1 → N)** | Original is tombstoned; N new IDs created; inbound `[[old]]` resolves to a single **designated default successor**, and the split event is recorded so links can be re-pointed to the right successor deliberately. |
| **Merge (N → 1)** | Survivor keeps its ID; losers are tombstoned and aliased to the survivor; backlinks **collapse** onto the survivor. |
| **Delete** | Never a hard delete. Tombstone the ID; aliases to a replacement if one exists, else the link surfaces in `unresolved-refs` as an intentional dead end. |
| **Alias** | `old-id → canonical-id`. Resolution is **flattened on write** (see below). |
| **Collision** | `check` fails if any ID is defined twice, or if an alias target does not exist, or if an ID and an alias share a name. |
| **Cross-wiki canonicalization** | The same real-world concept living in two wikis is *not* merged; instead both pages carry a shared `entity` (V3) and link to each other. Identity stays wiki-scoped. |

**Redirects: flatten-on-write, max depth 1.** When `B → C` is recorded and `A → B` already
exists, `A` is rewritten to `A → C` at write time, and the writer **rejects any operation
that would create a chain or loop**. The `redirect_max_depth <= 1` and `redirect_loops = 0`
gate metrics (§9) are guards that verify the writer held this invariant — they scan the
generated redirect map and fail the build if a buggy write ever slipped a chain through.

---

## 4. LLM is a cached suggestion, not a generator of truth

"Deterministic LLM generation" is a fiction — temperature 0 is not reproducible across
model versions or batching. So the determinism lives in the **compiler**, and the LLM is
treated as an external, cached input:

- LLM output is written **only into fenced regions** and committed as a cache artifact,
  keyed by the `projection_hash` defined once in §5.2:

  ```markdown
  <!-- agent:start key=projection_hash -->
  …generated prose, regenerable…
  <!-- agent:end -->

  <!-- human: hand-written, NEVER regenerated -->
  ```

- A changed prompt, model, template, or source **busts the cache** (all are inputs to
  `projection_hash`); an unchanged `projection_hash` costs **$0** because the committed
  cache artifact is **replayed**, the model is not called. Note the byte-identity that the
  §9 `generated_rebuild_byte_identical` gate checks is guaranteed **only by cache replay** —
  the underlying model is *not* assumed reproducible, so a cache *miss* (intentional input
  change) is expected to produce a new, reviewed diff rather than a byte-identical one.
- `status: stable` pages are **regeneration-frozen** unless a `source_refs` hash diverges.
- **Runbook safety rule (because the use case is operational):** the LLM may write
  *explanation* around a runbook, but **operational steps are human-authored and
  `verified_by: human`**. A canonical runbook (`status` in {reviewed, stable}) with
  `verified_by: none` **fails the gate**. The cost of executing a hallucinated step during
  an incident is unacceptable.
- Every generated claim must cite a `source_ref`. A generated region with an
  uncited factual claim **fails the gate** (`missing-source claim`).
- CI/tests use **mocked LLM fixtures** (`pipeline/fixtures/`), never live calls, so `check`
  is reproducible and free.

---

## 5. Provenance & freshness

### 5.1 Artifact header (every generated file)

```json
{
  "_artifact": {
    "schema_version": "1.0.0",
    "input_hash": "sha256:…",        // hash of all authored inputs that fed this artifact
    "compiler_version": "wikigraph 0.3.1",
    "generator_version": "n/a (deterministic)",   // model id@version when LLM-produced
    "source_commit": "bb75397"
  },
  "...": "..."
}
```

This makes every artifact self-describing: you can tell at a glance whether it is stale
relative to its inputs, which compiler built it, and from which commit.

### 5.2 Hash scopes (defined once, used everywhere)

| Hash | Covers | Used for |
|---|---|---|
| `raw_doc_hash` | a normalized raw source file | staleness detection |
| `normalized_section_hash` | a page section after canonicalization | diff minimization |
| `input_hash` | all authored inputs to a **deterministic** generated artifact (registry + content + shared); the no-LLM analog of `projection_hash` | artifact-header freshness (§5.1) |
| `projection_hash` | the full input set to an **LLM** build step = `input_hash` ⊕ prompt version ⊕ model id\@version ⊕ template version ⊕ cross-link target hashes | LLM cache key / skip logic (§4) |
| `compiler_version` | the `bin/wikigraph` version | cache invalidation on tool change |

**Canonicalization before hashing is mandatory** (normalize line endings, trailing
whitespace, key ordering) — otherwise trivial edits force rebuilds and the gate is noisy.

### 5.3 Staleness (elevated to V1 because of runbooks)

Staleness is a **computed signal, not a manual timestamp.** When any
`source_refs[].sha256` differs from the value recorded at last compile, the page is
auto-flagged stale into `generated/reports/stale-sources.json`. A **stale canonical
runbook fails the gate** — for `kind: page` it is a warning, for `kind: runbook` with
`status: stable` it is an error.

---

## 6. Security & redaction

`content/<wiki>/raw/` for `aws`/`databricks`/`jenkins` is the highest-likelihood secret
carrier (keys, PATs, tokens). **Redaction happens at the ingestion boundary — before
parsing, logging, caching, embedding, prompting, or writing any artifact**, not just
"before the LLM call." Secrets leak into logs and caches even with no LLM present.

- `bin/redact` + `shared/redaction-rules.yaml`: regexes for AWS keys, JWTs, PEM blocks,
  Databricks PATs, Jenkins credentials, generic `*_TOKEN`/`*_SECRET` assignments.
- A **canary secret** is seeded into a test source. The gate `canary_secret_leaks = 0`
  asserts the canary appears in **no** prompt, log, cache, fixture, or generated artifact.
- `bin/wikigraph check` runs a `gitleaks`-style scan over the **working tree**
  `content/**/raw/` — regardless of whether a wiki's `raw/` is gitignored — so the canary
  and redaction checks see source material that is never committed.
- Because gates are **local-only**, raw never leaves the machine. `content/**/raw/` may be
  gitignored entirely (the durable wiki is the compiled page + its provenance hashes, not
  the raw dump) — decide per wiki in `content/<wiki>/config.yaml`. Even when gitignored,
  the scan above still runs against the working tree before any compile.

---

## 7. Migration of the existing 32 AIDLC files

These files predate the `[[id]]` convention, so their references are prose, not IDs.

1. **Report-mode audit (no writes).** `bin/wikigraph audit content/aidlc/` emits a table per
   file: has-frontmatter? has-id? detected prose links, inferable entities, source guess,
   estimated effort. Eyeball it to size the job.
2. **Pilot 3–5 representative files first** (not all 32, not batches yet). Take files of
   varying shape, run the full migration, and confirm the contract holds end-to-end before
   committing to volume.
3. **Mint all IDs in one pass before any link rewriting.** Assigning IDs file-by-file while
   rewriting links creates dangling refs. Freeze the id-map, then rewrite prose references
   to `[[id]]`.
4. **Then migrate the rest in batches of 5–8**, one branch per batch.
5. Backfill `source_refs` as `tier: internal_notes` / `legacy-handwritten` where unknown —
   **never fabricate provenance.**
6. **Gate after each batch:** schema valid; no ID churn except intended mappings; no
   orphaned canonical pages; no unexplained graph diff. **At N=32, "parity" is exact
   accounting, not a percentage:** every pre-migration page must map to exactly one of
   {migrated page, intentional merge target, explicitly-recorded deletion}. Zero canonical
   pages may be lost; any non-canonical deletion must appear in `registry/events.log.yaml`.

The existing `.kiro/` 7-agent pipeline keeps running in parallel during transition — no
breaking change.

---

## 8. Phased roadmap

### V0 — Reliable Markdown graph (no LLM) — build this first
- `content/`, `registry/`, `generated/`, `shared/` scaffolding.
- Minimal frontmatter (`id, title, slug, status, kind, source_refs`); `kind` is needed
  from V0 so the canonical-page and runbook gates are evaluable. `source_refs` required for
  canonical pages (`status` in {reviewed, stable}).
- Readable registry-owned IDs.
- Deterministic compiler emits `index.json`, `backlinks.json`, basic `graph.json` (all pure
  functions of registry + content — no LLM).
- `bin/wikigraph check`: duplicate IDs, dangling links, byte-identical rebuild, redaction
  scan.
- **Exit gate:** `duplicate_ids = 0`, `dead_internal_links = 0`, `generated/` rebuilds
  byte-identically, `canary_secret_leaks = 0`.

### V1 — Compiler + identity lifecycle + migration + freshness
- Versioned schemas for all six contracts (§2.2: frontmatter, registry, redirects,
  provenance, artifact-header, config).
- Full identity lifecycle (§3): rename/split/merge/tombstone/alias + collision detection.
- Hash scopes (§5.2) and artifact headers (§5.1) on every generated file.
- **Staleness detection (§5.3)** — pulled forward from V2 because of runbooks.
- CLI reports: unresolved refs, changed-graph, stale sources, redaction hits.
- Migrate the 32 AIDLC files (§7).
- **Exit gate:** per-batch migration gate passes; staleness report wired; no orphaned
  canonical pages.

### V2 — Assisted authoring (LLM as cached suggestion)
- LLM writes into fenced regions only (§4); committed cache keyed by input hash.
- Citations required; missing-source claim fails; runbook steps stay human-verified.
- Mocked LLM fixtures in `check`; budget caps + token/request reporting; model tiering
  (cheap model for lint/extraction, strong model for compile).
- **Exit gate:** generated content never mutates registry/graph state; canary never appears
  in prompts/logs/caches/fixtures/artifacts.

### V3 — Entities, taxonomy, automation (only if corpus justifies)
- Entity extraction + layered taxonomy (global vocab → wiki-local → page tags → aliases).
- `coverage.json` thresholds enforced (§9).
- Optional: scheduled refresh, auto-PRs, dashboards, multi-agent critic/advisor flows.

> **Sequencing note:** deterministic graph *measurement* (backlinks, `graph.json`,
> `coverage.json`) is cheap and LLM-free, so the artifacts are **produced and reported**
> from V0/V1 — only *entity extraction* (which needs judgment) and *automation* wait for
> V3. The coverage *thresholds* in §9 are enforced as hard gates at V3; before that they are
> visible metrics you watch, not blockers. Measurement ≠ generation; don't defer the cheap
> deterministic part.

---

## 9. Metrics & gates (hard thresholds)

Denominators matter: zero-tolerance metrics apply to **canonical pages only**
(`status` in {reviewed, stable}), so drafts don't block normal work.

```
duplicate_ids                                   = 0
dead_internal_links                             = 0
redirect_loops                                  = 0
redirect_max_depth                              <= 1
pages_without_sources   (canonical only)        = 0
orphan_pages            (canonical only)        = 0     # in_degree==0 && out_degree==0
generated_rebuild_byte_identical                = true
unaccounted_migrated_pages                      = 0     # every pre-migration page maps to migrated|merged|recorded-deletion
lost_canonical_pages                            = 0     # canonical loss is never permitted, at any corpus size
canary_secret_leaks                             = 0
stale_canonical_runbooks                        = 0     # kind:runbook, status:stable, source hash diverged
unverified_canonical_runbooks                   = 0     # kind:runbook, status:stable, verified_by != human
```

### Architecture gate — `check` fails if:
- an authored file depends on a generated artifact
- a generated field appears in authored frontmatter without an allowlist entry
- a generated file lacks an artifact-header
- an ID is path-owned instead of registry-owned (i.e., resolved by path, not registry)
- a redirect resolves at depth > 1
- LLM output affects canonical registry/graph state
- a provenance hash was computed without canonicalization
- a canary secret appears in raw or generated artifacts
- a coverage metric is reported without its denominator
- the test suite has only snapshot tests — **every invariant above also needs a property
  test.** Snapshots catch *change*; invariants catch *rule violation*. Both are required.

---

## 10. The `check` command (local-only gate)

`make check` → `bin/wikigraph check`, also wired as `.githooks/pre-commit`. Runs entirely
on the laptop; no cloud CI. It performs, in order:

1. **Schema validation** of frontmatter, registry, redirects, provenance, config against
   `schemas/`.
2. **Redaction / canary scan** over `content/**/raw/`.
3. **Identity checks**: duplicate IDs, alias targets exist, no ID/alias name clash,
   redirect depth ≤ 1.
4. **Rebuild `generated/` from scratch**, then `git diff --exit-code generated/`
   (no-dirty-graph).
5. **Link resolution**: every `[[id]]` resolves; report dangling to
   `reports/unresolved-refs.json`; fail on any.
6. **Coverage + freshness metrics** against the §9 thresholds.
7. **Invariant + snapshot tests** (incl. mocked-LLM fixtures from V2 on).

Honest tradeoff of local-only: **validation runs only when invoked.** The pre-commit hook
makes that every commit, which is sufficient solo — but nothing checks the repo while the
laptop sleeps.

---

## 11. Cost model (V2+)

Dominant cost = tokens/page × pages × runs. Bounded by:
- **input-hash skip** → unchanged pages cost $0 (the cache artifact is reused, no model
  call);
- **model tiering** — cheap/small model for lint and entity extraction, strong model only
  for compile;
- **incremental compile** — only re-generate pages whose `projection_hash` changed;
- **prompt caching** of the shared system prompt/schema across a batch.

Steady-state cost approaches zero — you pay only when you actually edit a page or bump a
prompt/model version. *(Confirm current model IDs and pricing before hard-coding tiers; the
most capable current models are Opus 4.8 / Sonnet 4.6 / Haiku 4.5.)*

---

## 12. Open items (do not block V0)

- Whether `content/**/raw/` is committed or gitignored per wiki (default: gitignore wikis
  whose raw may contain secrets).
- Entity vocabulary seeding strategy for V3 (manual seed vs extraction-then-review).
- Whether to add a lightweight local search index (`generated/search-index.json`) once the
  graph exists — cheap, deterministic, deferred until there's a consumer.

---

## 13. Karpathy-style LLM-first content structure

The graph (§2–§5) governs *identity and links between* pages. This section governs the
*shape of a page itself* so the corpus is optimal for LLM consumption — the same ethos as
the `llms.txt` convention and nanoGPT's single-file readability: **flat, plain Markdown,
answer-first, greppable, and concatenatable into one paste-able bundle.** It fleshes out
`shared/templates/page.md` and `runbook.md`.

### 13.1 Core principle — a page is one context-window unit

Every page is **atomic** (one concept), **self-contained** (correct with zero prior
context), and **front-loaded** (the answer in the first ~5 lines, depth after). An LLM that
retrieves a single page must be able to act correctly without fetching three others. This is
the content-side reason for the flat, no-deep-nesting rule in §2 — directory depth is
navigation chrome the model does not need. If a page needs an "also covers…" aside, that is
a signal to `split` (§3), not to nest.

### 13.2 The fixed 7-section page skeleton (every `kind: page`)

A predictable section order is the single biggest retrieval lever — the model always knows
where the answer lives, and truncation degrades gracefully because importance descends
(inverted pyramid). Sections are emitted in this exact order:

```markdown
# <title>

> **TL;DR.** One paragraph: what it is, when to use it, the one gotcha.
> A reader who stops here is shallow but still correct.

## When to use this        # decision-first: use when / don't use when / reach for X instead
## Key concepts            # the 3–6 nouns you must know, DEFINED INLINE (not by [[link]])
## How it works            # mechanism, concrete, with a minimal copy-pasteable example
## Common operations       # task → exact commands / recipes
## Pitfalls & gotchas      # the warnings this page exists to deliver
## Related pages           # compiler-owned <!--related--> region (§2.1)
## Sources                 # compiler-rendered from source_refs (§2.3)
```

Two rules that make this LLM-first rather than just a template:
- **Inverted pyramid:** TL;DR → decision → concepts → mechanism → edge cases. Descending
  importance so a truncated read is still useful.
- **Definitions inline, links for *more*:** never force the model to chase `[[term]]` to
  learn what a word means. A link is always for *additional* depth, never a prerequisite.

### 13.3 The runbook skeleton (every `kind: runbook`)

Operational pages use a stricter order, and the **Steps block is the human-authored,
`verified_by: human`, LLM-never region** from §4:

```markdown
## Symptoms          # how you know you are in this situation
## Preconditions     # access, tools, blast radius
## Steps             # <!-- human: numbered, verified, NEVER regenerated -->
   1. exact command — expected output — if this fails → …
## Verification      # how you know it worked
## Rollback          # how to undo
```

### 13.4 LLM-facing bundles — generated, not authored

Two new **generated** artifact families (pure functions of `content/`, deterministic,
artifact-header-stamped, diff-gated exactly like the rest of `generated/`):

```
generated/llms/
  llms.txt                 # root index: every wiki + page, one-line TL;DR + link (llms.txt convention)
  <wiki>.llms.txt          # per-wiki curated index an LLM reads first to decide what to fetch
  <wiki>-full.md           # ALL pages in a wiki, concatenated in dependency order
  all-full.md              # the entire corpus as one paste-able file
```

- **`llms.txt`** is a flat, link-rich Markdown map generated from `index.json` (title, `id`,
  TL;DR line, `scope`). It is what an agent reads first.
- **`*-full.md`** is the "paste the whole thing into context" bundle: every page concatenated
  with stable `<!-- page:aws:step-functions -->` delimiters, ordered by a **topological sort
  over the intra-wiki edges of `graph.json`** so prerequisites precede dependents. This reuses
  the graph already built in V0/V1 — no new analysis.
- Both regenerate from scratch and are covered by the §10 no-dirty-graph gate.

### 13.5 Naming, anchors, chunking

- **Stable greppable anchors:** the fixed headings map to fixed slugs (`#pitfalls-gotchas`),
  so a citation like `aws:step-functions#pitfalls` survives body edits.
- **Chunk-sized sections:** each `##` is self-contained and paragraph-scale, so it embeds /
  retrieves as one clean unit; no section assumes the reader saw the previous one.
- **One concept per file** (atomicity) — enforced socially by the skeleton and structurally
  by the `split` lifecycle op.

### 13.6 Enforcement & sequencing

| Element | Where it lands |
|---|---|
| `page.md` / `runbook.md` skeletons | `shared/templates/` (V0) |
| Section-order lint (required headings present, in order; runbook Steps block is human-only) | new `check` rule (V0/V1) |
| `generated/llms/` bundles (`llms.txt`, `*-full.md`, topological order) | new generated artifacts, diff-gated (V1) |
| Citations / Sources block from `source_refs` | already V0 (§2.3) |

The skeleton lint is cheap and LLM-free, so it ships with the V0/V1 deterministic core; the
bundles depend only on `graph.json`, so they ship in V1. No part of this section needs an
LLM to produce or verify.

---

## 14. Bottom line

Keep the clustered knowledge-graph instinct — it maps correctly to the topology. But before
any AI, entities, or automation, lock down the four foundations: **registry-owned identity,
provenance-based freshness, flatten-on-write redirects, and self-describing generated
artifacts**, all guarded by a single local `check` gate. Because the use case is
operational runbooks, freshness and human-verified steps are first-class from V0/V1, not
afterthoughts. Add LLM assistance, entities, and automation only once that deterministic
core has proven useful.
