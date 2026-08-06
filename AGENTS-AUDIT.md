# AGENTS.md Audit

Audit of `AGENTS.md`, run via `skill-router` + two specialist sub-agents. Findings drive the fixes applied in [`AGENTS-enhanced.md`](./AGENTS-enhanced.md). `AGENTS.md` itself was left unchanged.

## Method

- **skill-router** (`~/.kiro/skills/skill-router/scripts/route_skill.py`) selected the relevant skills:
  - `md-review` (score 38) — markdown pre-publication quality gate.
  - `statistical-analyst` / `senior-data-scientist` / `math-reasoning` — quantitative/algorithm rigor.
  - `spec-audit` / `requirements-engineering` — measurable thresholds / determinism.
- **Sub-agent A — `cs-data-scientist`:** read-only audit of the quantitative algorithms (recovery targeting, progressive overload, double progression, rep ranges).
- **Sub-agent B — `cs-doc-writer`:** read-only pre-publication doc audit (`md-review`), focused on contradictions, duplication/drift, ordering, and JSON-example validity.

Both audits were read-only; the lead (single writer) synthesized and applied fixes to the enhanced copy to avoid file collisions.

## Priority legend

- **P1** — correctness / safety (wrong or unsafe output). All applied.
- **P2** — ambiguity / non-determinism in core math. All applied.
- **P3** — unhandled edge cases. All applied.
- **P4** — polish / consistency / drift-prevention. Applied where low-risk.

---

## P1 — Correctness & safety

| # | Finding | Fix applied in enhanced |
|---|---------|-------------------------|
| P1-A | **Recovery formula contradicts the tier table.** Step 6 said `target = capacity × R` (and "at 100%, target = capacity", i.e. no step), while the tier table said `>70% = capacity + earned step`. Undecidable for any 71–100%. | Replaced with a tier-keyed pipeline: baseline (High/Maintain = `capacity`; Reduce = `capacity × R`), earned step **only** in High, then floor. `R` applies to weight **only** in Reduce. |
| P1-B | **"% cap decides load-vs-reps within the range" contradicts double progression.** For a heavy compound the smallest increment is always within cap, so the cap rule said "add load mid-range" (linear) while the dynamic model said "add reps until top" (double progression) — two schemes on one set. | Double progression is now authoritative: reps within range, load only at the top. The class cap governs the **step size**, not whether load is added mid-range. |
| P1-C (doc #1) | **Safety: name-map examples resolved to the banned barbell lifts**, and the wrist-safe swap (step 2.5) was ordered *before* the step that produces the barbell name (step 3). | Reordered workflow (name resolution → wrist-safe swap; renumbered 1–7, no more "2.5"). Annotated "Flat Bench"/"OHP" examples to route through the wrist-safe swap, never barbell. |
| P1-D (doc #2) | **`Routine Sets` JSON example sent both `reps` and `rep_range`**, violating the file's own rule. | Example now uses `rep_range` only. |
| P1-E (doc #3) | **Core Workflow working-set example used `reps: null`** (not `rep_range`), contradicting "working sets default to rep_range". | Skeleton working set now uses `rep_range`, with a note that `weight_kg: null` is a placeholder to pre-fill. |

## P2 — Ambiguity / non-determinism

| # | Finding | Fix applied |
|---|---------|-------------|
| P2-C | Recovery tiers fuzzy (`~50–70%`, "optionally × R", no %→R map, open boundaries). | `R = clamp(pct/100, 0.5, 1.0)`, default 1.0. Half-open tiers: `>70` High, `50–70` Maintain, `<50` Reduce; **70→Maintain, 50→Maintain**. |
| P2-D | "hit top of range on all working sets" — detection from per-set history undefined (best-set vs every-set; which session; insufficient data). | Defined: most recent good-recovery session **at the current target weight**, **every** working set `reps ≥ high`; `<2` working sets or none → hold. |
| P2-E | Load-step "+2.5–5 kg" had no selection rule; caps had `~`/ranges; the 5 kg branch was near-dead. | Caps are exact **5% / 3% / 3%**. Step = **largest gym increment ≤ cap %**; if the smallest exceeds cap, take smallest anyway (top-of-range override). |

## P3 — Unhandled edge cases

| # | Finding | Fix applied |
|---|---------|-------------|
| P3-F | Near-max ignores reps → a heavy low-rep single can set a high-rep target. | Rep-floor preference: pick the highest weight among sets with `reps ≥ low`; fall back to overall highest with a noted mismatch. |
| P3-G | Downtrend not detected (return from layoff/injury) → near-max pinned to a stale high. | Decline guard: if the most recent **two** in-block sessions are both >1 increment below near-max, use the most-recent weight and note the downgrade. |
| P3-H | Lone-spike predicate ambiguous; missed 2-set anomalies / second spikes / `<2` distinct weights. | Precise predicate: `w1` in only one set AND `w1 > 1.5 × w2` → drop `w1`, repeat once; `<2` distinct weights → no removal. |
| P3-I | RPE bands had gaps (`8.5–9`, `7–7.5`), no per-set aggregation, no Unknown matrix rows. | Contiguous bands (Easy `≤7`, Moderate `7<RPE<9`, Hard `≥9`/failure), aggregate by **max** RPE, deterministic rep steps (Easy +2 / Moderate +1), explicit **Unknown** rows. |
| P3-J | Two rep-band derivations (hypertrophy band `3` vs `≈2–3`), no cutoff on `T`. | One band table keyed on `T`: `T≤6`→2, `7–12`→3, `T≥13`→3; `low = max(1, T−band)`. |

## P4 — Polish / consistency / drift

| # | Finding | Fix applied |
|---|---------|-------------|
| P4-K | Equipment increments fuzzy/incomplete ("~1 kg light dumbbell"; no kettlebell/plate; floor vs nearest unclear). | Increment table by `equipment_category`; working sets **floor**, warm-ups **round to nearest**; dropped the undefined "light dumbbell". |
| P4-L | `reps` vs `rep_range` inconsistent across examples; reduced-recovery rep target phrased 3 ways. | Examples unified to `rep_range`; Maintain → `{low, mid}`, Reduce → fixed `reps = low`. |
| P4-M | Epley caveat imprecise / arguably backwards ("overshoots at high reps"). | Rephrased: Epley over-estimates 1RM from high-rep sets and degrades past ~10–12 reps, so targets use observed near-max directly. |
| P4-N | "explicit deload" mis-binned in `<50%` with undefined `R`; same word, 5× different magnitude vs the plateau "~10%". | Separated: user-requested **deload = fixed ~10%** cut independent of recovery; `<50%` Reduce tier is for a stated low number with `R ≥ 0.5`. |
| P4-O | Dynamic matrix didn't restate the recovery gate; plateau rule duplicated with different wording. | Matrix now labeled "High tier only"; single stall rule (2 sessions → −10%, reset to `low`). |
| Doc #6–#9 | Duplicated blocks (API-key note ×2; pre-fill ×3; rest ×2; >70% gate ×4) — consistent today, drift risk. | Recovery-Aware is the single source of truth; Gotchas/Rules now point to it. API-key note consolidated. Rest table is the single source; Rules points to it. |
| Doc #11 | A ```json fence contained a non-JSON `POST` line. | Moved the method out of the fence. |
| Doc #13 | Example title carried a literal date (`- 2026-07-20`) the file says never to copy. | Placeholder `- <YYYY-MM-DD>`. |
| Doc #12/#14/#15 | Section-title cross-refs shortened; "capacity" used before definition; awkward "2.5" step number. | Reference text aligned; capacity defined in the algorithm; workflow renumbered 1–7. |

## Checked and NOT a problem (per doc audit)

- API field names consistent: `weight_kg`, `rep_range{start,end}`, `superset_id`, `rest_seconds`; `set_type` used only for the *history* response (correct).
- Prerequisites (Athlete Constraints, Gotchas) correctly precede Core Workflow; reference material at the end.
- "Cyclete" vs "cyclite" is intentional and annotated.
- Pagination facts consistent across the file.
- No arithmetic errors in the worked examples (+12.5%, +25%, 3.57%, 10% all check out).

## Flagged — not independently verified

- **Sports-science validity** of the cited meta-analyses and the cap magnitudes — audited for internal consistency only, not literature validity. The doc hedges these as conservative engineering limits.
- **Hevy API behavior** — `rep_range` acceptance on routine sets, `set_type` values, `exercise_history` shape assumed as documented; `rep_range` was exercised successfully in earlier routine POSTs this project, but the "reps + rep_range together" rejection behavior was not tested.
- **Gym increments** — the increment table assumes standard steps; confirm against the user's single gym.
- **Epley numeric behavior (P4-M)** — flagged as a wording correction, not a re-derived physiology claim.

## Tooling notes (md-review)

- `link_checker.py` → PASS (6 refs, internal link resolves; external PMC/journal URLs inventoried, not fetched).
- `md_review_gate.py` → 1 "no frontmatter" + 17 "heading-case" flags: **false positives** for an agent-instruction file (no web frontmatter needed; headings are internally consistent Title Case).
- `readability_scorer.py` → PASS (Flesch 60.7, grade 8.5); "long sentence" hits were tables/lists collapsed by the scorer, not prose.
