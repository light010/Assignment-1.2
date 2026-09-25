# Multi-Wiki Knowledge Graph — Implementation Plan (reconciled)

> **Status:** Reconciled design, ready to build Phase 0.
> **Owner:** Solo maintainer.
> **Goal:** Generate LLM-authored, densely-cross-linked technical wikis
> (databricks, aws, jenkins, aidlc) — dense intra-wiki links, sparse inter-wiki bridges —
> and ship usable content in the first iteration.
> **Last updated:** 2026-06-23

This plan reconciles two competing designs for the same problem: a lean, LLM-first pipeline
("ship content now") and a rigorous deterministic graph compiler ("lock down integrity
first"). Deep multi-agent review (explorer + two critics + advisor) concluded that **neither
extreme fits a solo maintainer with ~120 pages**: the rigorous plan was correctly-engineered
but mis-sized and deferred the actual goal to its third milestone; the lean plan shipped the
goal immediately but was unsafe about the few things that matter. The synthesis below takes
the lean plan as the **spine** and bolts on exactly **four guardrails**.

---

## 1. The one idea that drives everything: route by page `kind`

The hard tension is "let the LLM author freely (fast)" vs "verify everything (safe)." Don't
choose globally — **put the trust boundary where the stakes are:**

| `kind` | Share of corpus | LLM's role | Review |
|---|---|---|---|
| `page` (explanatory reference) | ~the whole corpus | **Authors the body freely** from raw docs | `git diff` on a branch |
| `runbook` (executable steps) | a handful | Writes *prose only*; **Steps are human-verbatim & `verified_by: human`** | diff + human verify gate |

This dissolves the tension instead of compromising it: ~90% of pages get the fast path; the
few incident-time pages get the rigor. Everything else in this plan follows from it.

---

## 2. Architecture

**Monorepo of per-wiki folders. Three LLM agents. Manual trigger. Branch-based review.**
No orchestrator, no scheduler, no central registry. Each wiki is autonomous (dense internal
links); a thin cross-wiki index exists only to resolve the sparse inter-wiki bridges — this
mirrors the "dense clusters, sparse bridges" topology directly.

```
HAND-AUTHORED / SOURCE                 GENERATED (rebuilt by the pipeline; reviewed via git)
  <wiki>/raw/*          (gitignored)    <wiki>/wiki/*.md            LLM-authored pages
  <wiki>/config.yaml                    <wiki>/manifest.yaml        linkable ids in this wiki
  shared/agents/*.md   (3 prompts)      <wiki>/wiki/*.md  ## Related compiler-owned fenced region
  shared/aliases.yaml  (rename map)     generated/xref-index.json   cross-wiki [[id]] resolution
                                        generated/llms/*            llms.txt + *-full.md bundles
```

The compile agent receives **peer-wiki manifests** as context — that is how it discovers
cross-link opportunities without a central registry.

### 2.1 Directory layout

```
~/wikis/
  .git/
  PLAN.md
  Makefile                     make run WIKI=aws | make check | make new | make migrate
  .githooks/pre-commit         runs `make check` (optional; opt-in)

  shared/
    agents/
      01-ingest.md             prompt: fetch + normalize raw sources (+ redaction enforced upstream)
      02-compile.md            prompt: raw -> polished page; SUGGEST sparse cross-links from peer manifests
      03-lint.md               prompt: style / clarity / link hygiene
    style-guide.md
    terminology.yaml           global vocabulary
    aliases.yaml               flat old-id -> new-id rename map (the whole "registry")
    redaction-rules.yaml       secret regexes + one canary
    templates/{page.md, runbook.md}

  bin/
    run.sh                     run.sh <wiki>: ingest -> compile -> lint on a branch
    check.sh                   the single local gate (see §6)
    redact.sh                  secret scan/redaction at the ingestion boundary
    gen-manifest.sh            (re)generate <wiki>/manifest.yaml from wiki/
    bootstrap.sh               scaffold a new wiki

  aws/                         (same shape for databricks, jenkins, aidlc)
    config.yaml                title, terminology, source roots, model, budget cap
    raw/                       GITIGNORED source material (may contain secrets)
    wiki/                      flat LLM-authored .md pages (frontmatter + body)
    manifest.yaml             GENERATED: list of {id, title, slug} this wiki exposes

  generated/
    xref-index.json            cross-wiki id -> {wiki, path} for inter-wiki [[id]] resolution
    llms/{llms.txt, <wiki>-full.md, all-full.md}
    reports/{unresolved-refs.json, stale.json, redaction-hits.json}
```

### 2.2 Page frontmatter (minimal)

```yaml
---
id: aws:step-functions          # stable [[wiki:id]] token; survives title/slug renames
title: AWS Step Functions
slug: step-functions
kind: page                      # page | runbook
status: draft                   # draft | reviewed | stable   (canonical = reviewed|stable)
source_refs:                    # provenance + the staleness signal
  - path: aws/raw/sfn.md
    sha256: 9f2a…
    tier: official_docs         # official_docs | internal_notes | impl_examples
# --- runbook-only, required when kind: runbook and status in {reviewed, stable} ---
verified_by: human
last_verified: 2026-06-20
---
```

---

## 3. Identity & cross-references

- **Stable `[[wiki:id]]` tokens, not paths.** Authors write `[[aws:step-functions]]`
  (intra- or inter-wiki, same syntax). The compiler resolves token → current relative path.
  Tokens survive renames; raw relative paths rot on the first rename. The `id` is an
  immutable token — rename the `title`/`slug` freely, never the `id`.
- **Renames via a flat `shared/aliases.yaml`** (`old-id: new-id`). This is the entire
  "registry." Resolution flattens transitively and **rejects loops** at write time so chains
  (`A→B→C`) and cycles can't form. Git history is the event log — no separate ledger.
- **No split/merge/tombstone ceremony.** At ~120 pages those happen a handful of times ever;
  do them by hand (new pages + an alias + fix links) and let `check` catch dangling refs.
- **Resolution layers (matches the topology):** each wiki's `manifest.yaml` resolves its own
  ids (dense, autonomous); `generated/xref-index.json` is a thin join used **only** for
  inter-wiki bridges (sparse).
- **Cross-link quota:** intra-wiki dense/unlimited; inter-wiki **sparse — max 3 inline +
  unlimited in a `## Related` table**. The Related table is rendered into a **compiler-owned
  fenced region** (`<!-- related:start -->` … `<!-- related:end -->`) so it's regenerable and
  drift-checked, never hand-edited.

  ```markdown
  <!-- related:start -->
  ## Related
  | Topic | Wiki | Why |
  |---|---|---|
  | CodePipeline | aws | Alternative CI/CD approach |
  <!-- related:end -->
  ```

- A `[[id]]` that doesn't resolve is reported to `generated/reports/unresolved-refs.json`;
  it is a **hard failure on canonical pages**, a lint warning on drafts. The compile never
  crashes on a bad link.

---

## 4. The LLM pipeline (the spine)

`run.sh <wiki>` runs three agents on a fresh branch, then you review the diff and merge:

1. **01-ingest** — pull raw sources, normalize, **redact** (§5 guardrail 4). Secrets never
   reach the model.
2. **02-compile** — raw → polished page body, following the §7 skeleton. Receives peer-wiki
   manifests and may **suggest sparse inter-wiki links restricted to ids that actually exist
   in those manifests** (closed-world — no invented targets). Writes the page body and the
   `<!--related-->` region.
3. **03-lint** — style, clarity, link hygiene, skeleton conformance; emits fixes.

**Per-kind routing (the §1 rule, enforced here):**
- For `kind: page`, the compile agent authors the whole body. This is the user's goal —
  LLM-authored reference content, reviewed by `git diff`.
- For `kind: runbook`, the agent may write Symptoms/Preconditions/Verification/Rollback
  prose, but the **Steps block is human-authored verbatim from raw and never regenerated**
  (`<!-- human: NEVER regenerated -->`). A canonical runbook with `verified_by: none` fails
  the gate.

LLM output is the page body, reviewed and committed — it is *not* quarantined into a tiny
fenced region. Reproducibility comes from **branch review + the gate**, not from pretending
the model is deterministic. Pin `model` + `temperature: 0` in `config.yaml` for stability;
accept that re-runs produce reviewed, non-identical diffs.

---

## 5. The four guardrails (and nothing more)

Exactly four, because these are the ones that pay rent at this scale:

1. **Runbook step safety.** `kind: runbook` Steps are human-verbatim + `verified_by: human`;
   a stale or unverified canonical runbook **fails the gate**. (Narrow — a handful of pages.)
2. **Provenance + staleness.** Every page carries `source_refs` with a `sha256`. When a
   source hash diverges from what produced the page, the page is flagged in
   `generated/reports/stale.json` — a warning for `page`, a **gate failure for canonical
   runbooks**. Answers "is this stale or fabricated?" with one content hash, not a 5-hash
   taxonomy.
3. **Link integrity.** `check` resolves every `[[id]]`; unresolved → hard-fail on canonical
   pages. Duplicate ids and alias loops also hard-fail.
4. **Secret redaction.** `redact.sh` + `redaction-rules.yaml` scrub raw **before** ingest,
   logging, or any API call. A seeded **canary** must appear in no prompt/log/output
   (`canary_leaks = 0`). `raw/` is gitignored (your locked decision), so secrets stay local.

---

## 6. The `check` gate (one local command)

`make check` → `bin/check.sh`, optionally wired as a pre-commit hook. Local only; no CI
server. In order:

1. Frontmatter sanity (required fields present; `id` well-formed).
2. Redaction / canary scan over the working-tree `raw/` (runs even though `raw/` is
   gitignored).
3. Identity: duplicate ids, alias loop/depth, every `[[id]]` resolves (hard-fail canonical).
4. Runbook gate: canonical runbooks have `verified_by: human` and aren't stale.
5. Regenerate `manifest.yaml`, `xref-index.json`, `<!--related-->` regions, and `llms/`
   bundles; **advisory** diff (not a byte-identical commit blocker — just shows drift).
6. Lint conformance (skeleton headings present/in order).

Honest tradeoff: validation runs only when you invoke it. The pre-commit hook makes that
every commit; nothing checks while the laptop sleeps — fine for solo.

---

## 7. Page structure (Karpathy-style, LLM-first)

Every page is **atomic** (one concept), **self-contained**, **front-loaded**. Fixed section
order so retrieval and the LLM always know where the answer is.

**`page` skeleton (`templates/page.md`):**
```
# <title>
> TL;DR. One paragraph; correct-but-shallow if you stop here.
## When to use this        # decision-first
## Key concepts            # 3–6 nouns, defined inline (links are for MORE, never required)
## How it works            # mechanism + minimal runnable example
## Common operations       # task -> exact commands
## Pitfalls & gotchas
## Related                 # compiler-owned <!--related--> region (§3)
## Sources                 # rendered from source_refs
```

**`runbook` skeleton (`templates/runbook.md`):**
```
## Symptoms
## Preconditions
## Steps        # <!-- human: numbered, verified, NEVER regenerated -->
## Verification
## Rollback
```

**Generated LLM bundles** (`generated/llms/`): `llms.txt` (flat index an agent reads first)
and `<wiki>-full.md` / `all-full.md` (pages concatenated **in filename order** with
`<!-- page:aws:step-functions -->` delimiters) — the "paste the whole wiki into context"
artifact. No topological sort; filename order is enough at this scale.

---

## 8. Schedule & cost

- **Manual trigger:** `make run WIKI=aws` when you want. No scheduler (laptop sleeps).
- **Skip logic:** hash `raw/` + `config.yaml`; unchanged → no model call → ~$0.
- **Estimate:** ~$7–12/mo (2 wikis, light use) to ~$15–20/mo (4 wikis). Hard **budget cap in
  `config.yaml`** aborts a run before runaway cost.

---

## 9. Bootstrap & migration

- **New wiki:** `bin/bootstrap.sh databricks "Databricks Platform"` scaffolds
  `config.yaml`, `raw/`, `wiki/`, then: edit config → drop raw docs → `make run` → review
  branch diff → merge.
- **Migrate the 32 AIDLC files:** (1) audit in report mode; (2) pilot 3–5 files end-to-end;
  (3) **mint all ids in one pass** before rewriting any prose links to `[[id]]`; (4) migrate
  the rest in batches of 5–8, one branch each; (5) backfill `source_refs` as
  `tier: internal_notes` where unknown — never fabricate provenance; (6) `check` after each
  batch. The existing `.kiro` 7-agent pipeline keeps running in parallel until cutover.

---

## 10. Phased roadmap

| Phase | Deliverable | Trigger to start next |
|---|---|---|
| **0 (days)** | Monorepo + folders, `[[id]]` tokens, templates, 3 agents on `run.sh`. **Ship one wiki of LLM-authored pages.** | content exists |
| **1** | Cross-link resolution (tokens→paths) + dense/sparse quota + the four guardrails as `check`. Run all 4 wikis. | gate is green on all wikis |
| **2** | Migrate the 32 AIDLC files; generate `llms.txt` + `*-full.md` bundles. | migration parity holds |
| **3 (only if it hurts)** | Redirect-loop hardening, split/merge ops, scheduled refresh, dashboards. | a real pain point appears |

---

## 11. Deliberately dropped — and when to add it back

Recorded so the decisions are auditable. Each was judged team-/scale-infrastructure that a
solo maintainer at ~120 pages would never exercise.

| Dropped | Add back when |
|---|---|
| Append-only registry event log + tombstones | a second author joins (need an audit trail git can't give) |
| Full split/merge/tombstone/alias lifecycle as CLI commands | renames become frequent enough to be painful by hand |
| 6 versioned JSON schemas | a second *consumer* of the artifacts exists |
| `projection_hash` / 5-hash formalism / artifact headers everywhere | LLM caching cost becomes material (1000s of pages) |
| Property-based invariant tests as a hard requirement | the compiler grows beyond a few hundred lines |
| Byte-identical no-dirty-graph commit blocker | generation is provably deterministic and multi-author |
| Topological-sort bundles | a wiki gets large enough that read order matters |
| Central registry / monolithic index | cross-wiki bridges outgrow the thin `xref-index.json` |

---

## 12. Open items (do not block Phase 0)

- Which raw wikis are gitignored vs not — defaulting to **all gitignored** per your decision;
  revisit only if you want reproducible raw for `aidlc`.
- Entity/taxonomy seeding — deferred to Phase 3.
- A local search index (`generated/search-index.json`) — cheap, deterministic, add when a
  consumer exists.

---

## 13. Bottom line

Ship LLM-authored, cross-linked wikis **now** (Phase 0), on a lean Kiro-style spine:
monorepo of autonomous per-wiki folders, three agents, manual run, branch review. Keep
identity rename-safe with **stable `[[id]]` tokens + a flat alias map**, and bolt on exactly
**four guardrails** — runbook step verification, provenance/staleness, link integrity, secret
redaction. Route the rigor by `kind` so the handful of operational runbooks are safe without
slowing the explanatory bulk. Add registries, lifecycles, schemas, and hashing formalism
**only if and when the corpus actually grows into needing them** — not before.
