# Hevy Workout Agent (Enhanced)

> Enhanced revision of `AGENTS.md` incorporating the audit in `AGENTS-AUDIT.md` (data-science + documentation sub-agent review). Changes are behavior-preserving where the original was already correct; they resolve contradictions, remove ambiguity, and make the algorithms deterministic. See `AGENTS-AUDIT.md` for the finding-by-finding rationale.

## Purpose

Convert workout screenshots into Hevy routines via the Hevy API. Routines go into the **"Cyclete"** routine folder (`folder_id: 2763385`).

> Note: The user refers to this folder as "cyclite", but the actual folder on the account is titled **"Cyclete"** with id **`2763385`**. Use that folder.

## Athlete Constraints — Wrist-Safe (ulnar), ALWAYS APPLY

The user has a **bilateral ulnar-sided wrist issue**. Avoid any exercise that loads the wrist **in extension under push force with the wrist locked against a straight bar** (axial load driving through the wrist). This is a hard constraint on exercise selection — apply it to **every** routine, even if a screenshot prescribes a contraindicated lift. When a screenshot shows a contraindicated movement, **substitute the wrist-safe equivalent and note the swap** in the exercise `notes` (e.g. "swapped from barbell bench — wrist-safe neutral-grip DB").

Wrist-safe = **neutral grip** (palms facing), **dumbbells**, **machines with neutral/vertical handles**, and **cables** (rope / neutral attachments) — the load doesn't force the wrist into a locked, extended, pronated push.

**Contraindicated → wrist-safe swap:**

| Avoid (barbell / locked wrist push) | Use instead |
|-------------------------------------|-------------|
| Bench Press (Barbell) — flat / incline / decline / close-grip / wide | Bench Press (Dumbbell) `3601968B` or Chest Press (Machine) `7EB3F7C3` (neutral) |
| Overhead Press (Barbell) / Standing Military Press / Push Press | Overhead Press (Dumbbell) `6AC96645` or Seated Shoulder Press (Machine) `9237BAD1` |
| Smith Machine presses (still a fixed straight bar) | Dumbbell or machine press variant (as above) |
| Front Squat (clean-grip loads wrist extension) | Back Squat `D04AC939`, Hack Squat, or Goblet/DB variant |
| Barbell triceps extension / straight-bar skullcrusher | Triceps Rope Pushdown `94B7239B` / cable or DB (neutral) variants |
| Upright Row (Barbell) | Upright Row (Cable/Dumbbell) or lateral raises |

Notes and judgment:
- **Back squat and deadlift are fine** — the bar sits on the back / hangs from the hands; no wrist push. (A straight-bar deadlift is a pull, not a locked wrist push.)
- **Bodyweight wrist-in-extension moves** (push-ups, planks) can still irritate the ulnar wrist. They're not barbell lifts so not hard-banned, but prefer neutral options (push-ups on dumbbell handles/parallettes, forearm plank) and flag it if a screenshot uses them.
- This is exercise-selection guidance based on the constraint the user stated — **not medical advice**. Defer to the user's clinician on the underlying issue.

## API Reference

Full docs: https://api.hevyapp.com/docs/#/

Base URL: `https://api.hevyapp.com`

Authentication: pass the API key from `.env` (`hevy_API`) as an `api-key` header. **Read the key in code, not via `source`** — see the Gotchas ("Reading the API key").

```
api-key: <value from .env>
```

## Implementation Notes & Gotchas

Learnings from previous runs — read these to go faster:

- **RPE is not supported on routine sets.** `PostRoutinesRequestSet` has no `rpe` field (only workout sets do). Put any RPE targets from the screenshot into the exercise `notes` field instead (e.g. `"RPE 5-6 | 10-15 kg. ..."`). Do not send `rpe` in a routine POST.
- **Don't use shell heredocs for JSON payloads.** They fail intermittently in this environment. Instead, write the JSON to a file (e.g. `/tmp/routine.json`) with the file-writing tool, then `curl ... -d @/tmp/routine.json`. Same for the custom-exercise payload — inline `-d '{...}'` with single quotes is fine for short bodies.
- **`POST /v1/exercise_templates` returns just the new exercise's ID as a bare string** (a UUID like `513523cc-...`), not a JSON object. Capture that string directly and use it as the `exercise_template_id`.
- **`POST /v1/routines` returns `{"routine": [ { ... } ]}`** — the routine is wrapped in an array. Check for the `id` and `title` in the response to confirm success.
- **Pagination:** exercise templates are ~6 pages at `pageSize=100` (~513 templates). Requesting a page beyond the last returns `{"error":"Page not found"}` — stop when you hit it. Fetch all pages once and grep locally rather than re-querying per exercise.
- **Fetch templates once, match locally.** Dump all pages to files and search with a small Python script (matching by keyword) — much faster than one request per exercise. Watch for duplicate IDs across pages; dedupe by `id`.
- **The template list already includes custom exercises** (`is_custom: true`). No separate endpoint is needed to see them — search the same list and reuse an existing custom exercise before creating a new one to avoid duplicates.
- **Check `EXERCISE-REGISTRY.md` before fetching templates.** It caches common exercise IDs and skips the ~6-page fetch. Treat it as a cache, not truth: if an ID fails, re-verify against the API and update the registry (with its provenance date).
- **Read `.env` in code, not `source`.** `.env` has no trailing newline, so `source .env` silently sets nothing (and shell `awk`/`cut` extraction has been flaky here). Read `hevy_API` in a small Python script instead. An **empty `api-key` header returns `401 InvalidApiKey`** — that error means "no key sent," not necessarily "key revoked." Verify the key is actually non-empty before concluding it's invalid.
- **Weight & rep pre-fill:** for RPE-only prescriptions, don't leave weights null — pre-fill from history. The full method is in "Recovery-Aware Weight Targeting" (single source of truth). One-line summary: take the near-max working weight of the last ~120 days, apply the recovery tier, floor to the increment.
- **Set field by exercise type:** `weight_reps` → `weight_kg` + (`reps` or `rep_range`); `reps_only`/bodyweight → `reps`/`rep_range` only (omit `weight_kg`); `duration` (e.g. Dead Hang, Plank) → `duration_seconds`. Working sets default to `rep_range: {start, end}` (see "Rep Ranges & Targets"); warm-ups use fixed `reps`. **Never send both `reps` and `rep_range` on the same set.**

## Core Workflow

When the user provides a workout screenshot:

1. **Read the screenshot** — extract exercise names, sets, reps, weight, rest times, RPE, and any recovery/deload context.
2. **Match exercises to templates** — check [`EXERCISE-REGISTRY.md`](./EXERCISE-REGISTRY.md) **first**: it's a cached list of common exercise → `exercise_template_id` mappings and is much faster than hitting the API. Use a registry ID directly when there's a confident name match. Only if an exercise isn't in the registry, fall back to `GET /v1/exercise_templates?page=1&pageSize=100` (paginate through all pages) and match locally. That API list includes the user's **custom exercises** too (`is_custom: true`) — always search these as well and reuse an existing custom exercise instead of creating a duplicate. When you match a common exercise via the API that wasn't in the registry, add it to `EXERCISE-REGISTRY.md` for next time.

   - **If a registry ID fails** (a routine/exercise `POST` errors with "exercise template not found" or a 400 referencing the template), the ID may have changed. Re-fetch from the API, find the current ID by title, update `EXERCISE-REGISTRY.md` (and its provenance date), then retry. The API is authoritative; the registry is only a cache.
3. **Handle name mismatches** — different sources call exercises by different names. Resolve to the canonical template name first (the wrist-safe swap in step 4 then operates on the resolved match). Map by meaning:
   - "Flat Bench" / "Bench Press" → resolves to a bench press, then **wrist-safe swap** (step 4) → "Bench Press (Dumbbell)" / "Chest Press (Machine)" — **never barbell** (see Athlete Constraints)
   - "DB Rows" → "Dumbbell Row"
   - "Skull Crushers" → "Skullcrusher (Dumbbell)" or "Triceps Extension (Cable)" depending on context
   - "Lat Pulldown" → "Lat Pulldown (Cable)"
   - "OHP" / "Overhead Press" → resolves to an overhead press, then **wrist-safe swap** (step 4) → "Overhead Press (Dumbbell)" / "Seated Shoulder Press (Machine)" — **never barbell** (see Athlete Constraints)
   - "RDL" → "Romanian Deadlift (Barbell)" (a pull — wrist-safe, no swap needed)
   - "Leg Curl" → "Seated Leg Curl (Machine)"
   - Think about what exercise is likely meant based on context, equipment mentioned, and common gym terminology.
4. **Apply the wrist-safe constraint** — with the template resolved, check it against "Athlete Constraints — Wrist-Safe". If it's a straight-barbell / locked-wrist press, substitute the wrist-safe equivalent and note the swap. **This overrides the screenshot.**
5. **If no confident match exists** (including no matching custom exercise from step 2) — create a wrist-safe custom exercise via `POST /v1/exercise_templates`. Always fill the secondary muscle groups (`other_muscles`) — never leave it empty:
   ```json
   {
     "exercise": {
       "title": "Exercise Name",
       "exercise_type": "weight_reps",
       "equipment_category": "barbell",
       "muscle_group": "chest",
       "other_muscles": ["triceps", "shoulders"]
     }
   }
   ```
   The `other_muscles` array is the "secondary muscle groups" shown in Hevy. Populate it with the muscles the exercise recruits beyond the primary `muscle_group`, based on your knowledge of the movement (e.g. bench press → primary `chest`, secondary `triceps` + `shoulders`). Then use the returned ID in the routine.
6. **Use the "Cyclete" folder** — the target folder already exists with id **`2763385`**. Use `folder_id: 2763385` directly. Only if the folder is missing (verify via `GET /v1/routine_folders`) should you create it:

   `POST /v1/routine_folders`
   ```json
   { "routine_folder": { "title": "Cyclete" } }
   ```
7. **Create the routine** in that folder via `POST /v1/routines`. Optionally append the date to the title (see "Date Handling" — the date is optional and, if used, must be freshly computed, never a copied literal) and carry over any descriptions/notes from the screenshot into the `notes` fields. Working sets use `rep_range` and pre-filled `weight_kg` (see "Recovery-Aware Weight Targeting"); `weight_kg: null` below is only a skeleton placeholder — pre-fill it from history, leaving null only when there is no history.
   ```json
   {
     "routine": {
       "title": "Routine Name From Screenshot - <YYYY-MM-DD>",
       "folder_id": 2763385,
       "notes": "Overall workout description / notes from the screenshot",
       "exercises": [
         {
           "exercise_template_id": "<matched_id>",
           "superset_id": null,
           "rest_seconds": 90,
           "notes": "Per-exercise cue or description from the screenshot",
           "sets": [
             { "type": "warmup", "weight_kg": null, "reps": 10 },
             { "type": "warmup", "weight_kg": null, "reps": 5 },
             { "type": "warmup", "weight_kg": null, "reps": 3 },
             { "type": "normal", "weight_kg": null, "rep_range": { "start": 8, "end": 12 } }
           ]
         }
       ]
     }
   }
   ```

## Date Handling

Appending a date to the routine title is **optional**. If you do append one, use `YYYY-MM-DD` format:

```
<Routine Name> - <YYYY-MM-DD>
```

- **The date is optional** — it's fine to create a routine with just `<Routine Name>` and no date. Omit it when the user asks for no date, or when a bare title reads better.
- **Never hard-code or copy a literal date.** Do not reuse a date from an example in this file or from a previously created routine. If two routines end up with the same date because they were genuinely created on the same day, that's correct — but the date must always come from an actual lookup, not a copied string.
- **Always compute the current date at creation time** with `date +%Y-%m-%d` in the shell. Run it fresh for each routine; do not assume it's still whatever a prior command returned.
- **Prefer the date shown in the screenshot** if one is present (that's the "respective date" for that workout). Otherwise use the freshly-computed current date, or omit the date entirely.

## Descriptions & Notes

Always capture any descriptive text from the screenshot rather than dropping it:

- **Routine-level description** → the routine's `notes` field. Use this for the overall workout description, focus, or general instructions shown in the screenshot.
- **Per-exercise description** → that exercise's `notes` field. Use this for form cues, tempo, RPE targets, or any note attached to a specific exercise.
- If the screenshot has no descriptive text, leave `notes` as an empty string `""`.

## Rest Times

If the screenshot specifies rest times, use them. Otherwise set `rest_seconds` by how systemically fatiguing the exercise is (this table is the single source of truth for rest):

| Exercise type | `rest_seconds` |
|---------------|----------------|
| Heavy compound / high systemic fatigue (squats, deadlifts, barbell rows, overhead press, heavy leg press, RDLs) | 180 (2.5–3 min) |
| Moderate compound (bench, dumbbell press, pull-ups, lunges, hip thrusts) | 120 (2 min) |
| "Normal" accessory / isolation (curls, triceps, lateral raises, face pulls, cable/machine work) | 70–90 |
| Core / duration / stretch-style holds (plank, dead hang) | 45–60 |

- Scale toward the higher end of a band for heavier loads / lower reps, and the lower end for lighter, higher-rep or low-RPE work.
- The idea: bigger multi-joint lifts that tax the whole system need longer recovery; small single-joint moves need less.

## Warm-Up Sets

Always add warm-up sets automatically before the working sets, even when the screenshot only shows working sets. Prepend a common **warm-up ramp** of `type: "warmup"` sets that build up to the first working set.

Standard ramp (3 warm-up sets) for weighted compound/barbell/dumbbell/machine lifts:

| Set | Load (of first working weight) | Reps |
|-----|-------------------------------|------|
| Warm-up 1 | ~40% (or empty bar)           | 10   |
| Warm-up 2 | ~60%                          | 5    |
| Warm-up 3 | ~80%                          | 3    |

- If the working weight is known, compute the warm-up `weight_kg` from these percentages and **round to the nearest increment** (warm-ups round to nearest; working-set targets floor — see "Equipment increments"). If the working weight is unknown (`null`), still add the warm-up sets with `weight_kg: null` so the ramp structure is there for the user to fill in.
- Use a lighter ramp (1–2 warm-up sets) for isolation or accessory movements (e.g. curls, lateral raises, cable work).
- **Do not** add warm-up sets for cardio, duration-only exercises (e.g. plank), or pure bodyweight movements where a ramp doesn't apply.
- Warm-up sets always come first in the `sets` array, followed by the working sets from the screenshot. Warm-ups use fixed `reps` (no `rep_range`).

## Recovery-Aware Weight Targeting

This is the **single source of truth** for pre-filling working-set `weight_kg` when the screenshot gives no explicit weight (e.g. "load to RPE" / RPE-only prescriptions). **Do not just copy the last session's weight** — the last session may itself have been recovery-reduced (a low-RPE / deload day), so using it as today's goal under-shoots when today's recovery is high. Instead, estimate the user's *capacity* from the best of their recent sessions, then apply today's recovery tier.

### Recovery factor and tiers (deterministic)

The user states today's **recovery context** per request (e.g. "100%, recovery good", or "deload / 50%"). Map it to a factor and a tier:

- **Recovery factor:** `R = clamp(stated_percent / 100, 0.5, 1.0)`. Default `R = 1.0` when described as good/normal with no number. The 0.5 floor keeps a bad day from producing an absurd cut.
- **Tiers (exact, half-open boundaries):**

| Recovery `pct` | Tier | Baseline weight | Progression (load step) | Rep targets |
|----------------|------|-----------------|-------------------------|-------------|
| `pct > 70` | **High intensity** | `capacity` | **Applied** if earned (see Progressive Overload) — only tier that adds load | Full range `[low, high]` |
| `50 ≤ pct ≤ 70` | **Maintain** | `capacity` | None (hold) | `rep_range {low, mid}`, `mid = floor((low+high)/2)` |
| `pct < 50` | **Reduce** | `capacity × R` | None | Fixed `reps = low` |

- Exactly **70 → Maintain**; exactly **50 → Maintain**. `R` is applied to the weight **only in the Reduce tier**; in High/Maintain treat the weight multiplier as `1.0` (the tier itself, not `R`, sets behavior).
- A **user-requested "deload"** (the word, no number) is a **fixed ~10% cut** (`baseline = capacity × 0.9`, reps → lower half), independent of the recovery %. This is distinct from the `<50%` Reduce tier (which is for a genuinely stated low recovery number).

### Capacity estimation algorithm (per exercise)

1. **Fetch history** — `GET /v1/exercise_history/{exerciseTemplateId}` (optionally `?start_date=...`). The response is `{"exercise_history": [ ...sets... ]}`; each entry is one logged set with `workout_id`, `workout_start_time`, `weight_kg`, `reps`, `set_type`.
2. **Group into sessions** by `workout_id`; sort by `workout_start_time` descending.
3. **Take the current training block** — sessions within the **last 120 days** (cap 10 sessions). Bounding by *time* (not just session count) is essential: infrequently-trained lifts otherwise reach back into old blocks and resurrect stale PRs. Real example: a naive "max of last 10 sessions" picked a **140 kg × 1 squat single from 2 years ago** and a **92.5 kg hip-abduction from a year+ ago**, when current working weights are 70 kg and 60 kg. The 120-day window excludes those.
4. **If no session falls in the window** (exercise not done in ~4 months), fall back to the single most recent session and note it's stale in `notes`. Do **not** scan all-time for a maximum.
5. **Collect working sets** in the block: `set_type` in {`normal`, `failure`}, `weight_kg` present; exclude `warmup`.
6. **Spike guard (precise):** let the distinct working weights sorted descending be `w1 > w2 > w3 …`. If `w1` was logged in **only one set** and `w1 > 1.5 × w2`, discard all sets at `w1` and repeat the check once on the new top. If there are fewer than 2 distinct working weights, do no spike removal. (Multi-set anomalies are intentionally not removed — the 120-day window is the primary guard.)
7. **Rep-floor preference (avoid a heavy low-rep single dominating a high-rep target):** near-max `capacity` = the highest remaining `weight_kg` among working sets whose logged `reps ≥ low` (the bottom of today's target range). Only if no in-block set meets that rep floor, fall back to the overall highest remaining weight and note the rep mismatch in `notes`.
8. **Decline guard (detect a genuine downtrend):** if the **most recent two** in-block working sessions are **both** below the computed near-max by more than one increment, treat the near-max as stale — set `capacity` = the most-recent session's working weight and note the downgrade. (A single low session is still ignored; that's the whole point of near-max.)
9. **Compute the target** using the recovery tier:
   - a. **Baseline:** High/Maintain → `baseline = capacity`; Reduce → `baseline = capacity × R`; user-requested deload → `baseline = capacity × 0.9`.
   - b. **Earned step:** only in the High tier, add the Progressive-Overload load step **if earned** (else add nothing). Never add a step in Maintain/Reduce/deload.
   - c. **Round:** floor to the equipment increment (see below).
10. **No history / no working sets in block** → leave `weight_kg: null` and note "no history — pick to RPE". Don't invent a number.
11. **Note provenance** in the exercise `notes`, e.g. `"Block near-max 45 kg (2026-07-25); recovery 100% (high) -> target 45 kg"`.

### Equipment increments

Floor working-set targets to the increment; round warm-ups to nearest. Keyed on the `equipment_category` the file already uses:

| `equipment_category` | Increment |
|----------------------|-----------|
| `barbell`, `machine`, `cable`, `plate` | 2.5 kg |
| `dumbbell` | 2 kg (or the gym's DB step) |
| `kettlebell` | 4 kg (or the gym's KB step) |

Confirm against the user's single gym when known. (Removed the earlier undefined "light dumbbell / ~1 kg" case — use 2 kg unless the user actually has micro-dumbbells.)

**Reps caveat:** capacity is a weight, not rep-normalized. The rep-floor rule (step 7) keeps a heavy low-rep set from setting a high-rep target, but if today's target reps still exceed where the near-max was set, expect to fall a little short — autoregulate in-session. We deliberately use the observed near-max working weight directly rather than deriving through an estimated 1RM: Epley (`1RM ≈ w·(1+reps/30)`) over-estimates 1RM from high-rep sets and its linear model degrades beyond ~10–12 reps, so an e1RM-derived target is unreliable here.

**Same-gym assumption:** the user trains at one gym, so machine loads — including the **Gym80** machines (hip thrust, abductor, adductor) — are directly comparable session-to-session; no leverage normalization needed. Keep matching each screenshot exercise to the **same** template every time so its history stays coherent.

**Reading the API key:** see the `.env` / `source` / `401 InvalidApiKey` note under "Implementation Notes & Gotchas".

## Progressive Overload

Recovery-aware targeting (above) sets the baseline so today's weight matches current capacity. **Progressive overload** is the layer on top that makes the user actually get stronger/more muscular over time: on a good-recovery day (**High tier, recovery > 70%**), nudge slightly *beyond* recent capacity when the previous performance earned it. Without this, targeting capacity alone just maintains. Progression **only fires in the High tier** — never in Maintain/Reduce/deload.

**Method — double progression (authoritative).** Each exercise runs on a rep **range** `[low, high]`. Progress **reps within the range**; add **load only at the top of the range**, then reset reps to `low`. (Linear load-every-session is *not* used; the class cap below governs the *size* of the step, not whether load is added mid-range.)

Evidence basis (meta-analyses / reviews preferred; content rephrased for licensing compliance):
- Progressing **load or reps** yields similar muscle adaptations, so rep progression counts as overload — [Plotkin et al. 2022](https://pmc.ncbi.nlm.nih.gov/articles/PMC9528903/).
- **Double progression** operationalizes this; %1RM and rep-max load prescriptions both build strength — [load-prescription systematic review](https://pmc.ncbi.nlm.nih.gov/articles/PMC7142036/).
- **Failure is not required** for gains, so progress while leaving 1–2 reps in reserve — [failure vs non-failure meta-analysis](https://pmc.ncbi.nlm.nih.gov/articles/PMC9068575/).
- **Bigger load jumps on heavy compounds than on light isolation is mechanical, not sex-specific.** A fixed increment (e.g. 2.5 kg) is a small % of a heavy squat but a large % of a light curl. Sex-comparison meta-analyses find men and women gain *relative* strength comparably (women if anything show slightly greater relative upper-body gains; men have greater absolute strength), so these rules apply equally regardless of sex — [sex-differences meta-analysis (JSCR 2020)](https://journals.lww.com/nsca-jscr/fulltext/2020/05000/sex_differences_in_resistance_training__a.30.aspx).
- A women-only meta-analysis is sometimes cited for "lower body progresses faster than upper body," but its per-week figures are frequency-dependent and inconsistently reported — treat the caps below as conservative engineering limits, not derived from that number — [PLOS ONE 2023](https://pubmed.ncbi.nlm.nih.gov/37053143/).

**Exercise-class load-step caps** (the cap is a hard % ceiling on the step size):

| Exercise class | Examples | Per-step cap |
|----------------|----------|--------------|
| Lower-body compound | squat, deadlift, RDL, leg press, hip thrust | 5% |
| Upper-body compound | bench, incline press, OHP, row, lat pulldown | 3% |
| Isolation / small muscle | curls, triceps, lateral raise, face pull | 3% |

**Step-size rule (deterministic):** when a load step is triggered (top of range reached), take the **largest available gym increment that is ≤ the class cap %**. If even the **smallest** available increment exceeds the cap, take the smallest anyway — the `high`→`low` rep reset absorbs the larger relative jump (e.g. an 8 kg curl climbed 10→15 reps, then → 10 kg at ~10 reps; the +25% load is offset by dropping 15→10 reps). This is why light isolation work progresses mostly by reps and takes an occasional bigger load jump, while a 100 kg squat can take a full 5 kg step.

### Dynamic double progression (rep range + RPE/failure)

**This matrix runs only in the High tier (recovery > 70%).** In Maintain/Reduce, hold or reduce per the recovery tier.

**Rep range `[low, high]`** from a single screenshot number `T`: `high = T`, `low = max(1, T − band)`:

| `T` (top) | band | example |
|-----------|------|---------|
| `T ≤ 6` (strength / low-rep) | 2 | `6 → 4–6` |
| `7 ≤ T ≤ 12` (hypertrophy) | 3 | `12 → 9–12` |
| `T ≥ 13` (high-rep isolation) | 3 | `15 → 12–15` |

If the screenshot already gives a range, use it directly.

**Detecting "all working sets hit the top":** from the **most recent good-recovery session logged at the current target weight**, take working sets only (`normal`/`failure`; exclude `warmup`). The top-of-range trigger is met **iff every such set has `reps ≥ high`** (evaluate every set, not the best set). If no good-recovery session exists at that weight, or fewer than 2 working sets were logged there → **insufficient data → hold** (no progression).

**Effort classification** — aggregate the session's working sets by the **maximum** RPE; any working set with `set_type: "failure"` → Hard. Bands are contiguous:

- **Easy:** `RPE ≤ 7` (≈3+ reps in reserve), no failure.
- **Moderate:** `7 < RPE < 9`, no failure.
- **Hard:** `RPE ≥ 9`, or any working set marked `set_type: "failure"`.
- **Unknown:** no RPE and no failure logged.

| Last session at current weight | Effort | Action |
|--------------------------------|--------|--------|
| All sets at top of range | Easy / Moderate | Add load (step-size rule), reset reps to `low` |
| All sets at top of range | Hard | Add load at the **smallest** increment, reset to `low` (earned, jump minimally) |
| All sets at top of range | Unknown | Add load at the **smallest** increment, reset to `low` |
| Below top of range | Easy (`RPE ≤ 7`) | **+2 reps** (if `RPE ≤ 6`, weight is too light — still +2 reps; bring the load step forward only if this recurs a 2nd session) |
| Below top of range | Moderate | **+1 rep** |
| Below top of range | Unknown | **+1 rep** |
| Below top of range | Hard (`RPE ≥ 9` / failure) | Hold weight and reps; repeat the session |

**Stall rule:** top reps not reached for **2 consecutive sessions** at a weight → drop 10% (floored to the increment) and reset reps to `low`.

**Data reality:** in this account RPE is logged inconsistently (some sets have `rpe: null`) but `set_type: "failure"` is reliable when used. The Unknown rows above cover missing RPE deterministically (treat absent `set_type` as non-failure).

**Provenance in notes:** e.g. `"8 kg x12 (top) @ RPE 7.5, no failure -> +2 kg, reset to 9 reps"`, or `"12 kg x8 @ RPE 9 -> hold, repeat"`.

**Defensive guards (this is meant to be conservative):**
- Only progress from **good-recovery** sessions (High tier). Never add load in Maintain/Reduce/deload.
- **One increment per session** maximum. No double jumps.
- **Hold** when data is sparse or stale (fewer than 2 in-block sessions at the weight, or the lift hasn't been done in ~4 months).
- **Screenshot reps vs the rep-range cycle:** a screenshot's single rep number is treated as the **top** of the pre-filled `rep_range` (`high = T`). Progress via **weight** across sessions; don't invent a scheme the screenshot doesn't imply. For a strict single-rep prescription (e.g. peaking `5×5`), use fixed `reps` instead of a range.

## Rep Ranges & Targets (rep_range)

The API accepts a **`rep_range`** object on each set — `{"start": <low>, "end": <high>}` — instead of a fixed `reps`. This is how the double-progression range gets pre-filled so Hevy shows a target like "8–12" rather than a single number. Both `PostRoutinesRequestSet` and `PutRoutinesRequestSet` support it.

- **Working sets → use `rep_range`** by default, set to the exercise's double-progression range `[low, high]` (derive `[low, high]` per the band table in "Dynamic double progression").
- **Warm-up sets → keep fixed `reps`** (no range).
- **Strict single-rep prescriptions** (e.g. a peaking `5×5`) → use fixed `reps`, not a range.
- **Do not send both** `reps` and `rep_range` on the same set — pick one. `rep_range` for a ranged target, `reps` for a single target.
- **Recovery interaction:** High (>70%) → full range `[low, high]` at capacity + earned progression. Maintain (50–70%) → `rep_range {low, mid}` (`mid = floor((low+high)/2)`). Reduce (<50%) → fixed `reps = low` with reduced weight.

Example working set with a range:

```json
{ "type": "normal", "weight_kg": 30, "rep_range": { "start": 8, "end": 10 } }
```

## API Endpoints Reference

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/exercise_templates?page=N&pageSize=100` | List exercise templates (max pageSize 100) |
| GET | `/v1/exercise_templates/{id}` | Get single exercise template |
| POST | `/v1/exercise_templates` | Create custom exercise |
| GET | `/v1/routines?page=N&pageSize=10` | List routines |
| GET | `/v1/routines/{routineId}` | Get single routine |
| POST | `/v1/routines` | Create routine |
| PUT | `/v1/routines/{routineId}` | Update routine |
| GET | `/v1/routine_folders?page=N&pageSize=10` | List routine folders |
| POST | `/v1/routine_folders` | Create routine folder |
| GET | `/v1/exercise_history/{exerciseTemplateId}` | Prior logged sets for an exercise (used to pre-fill weights) |

## Custom Exercise Types

When creating a custom exercise, `exercise_type` must be one of:
- `weight_reps` — standard weighted exercise (most common)
- `reps_only` — no weight tracked
- `bodyweight_reps` — bodyweight + optional added weight
- `bodyweight_assisted_reps` — assisted (e.g. assisted dips)
- `duration` — time only (e.g. plank)
- `weight_duration` — weight + time
- `distance_duration` — cardio (distance + time)
- `short_distance_weight` — e.g. farmer's walk

## Muscle Groups

Valid values for `muscle_group` and `other_muscles`:
`abdominals`, `shoulders`, `biceps`, `triceps`, `forearms`, `quadriceps`, `hamstrings`, `calves`, `glutes`, `abductors`, `adductors`, `lats`, `upper_back`, `traps`, `lower_back`, `chest`, `cardio`, `neck`, `full_body`, `other`

## Equipment Categories

Valid values for `equipment_category`:
`none`, `barbell`, `dumbbell`, `kettlebell`, `machine`, `plate`, `resistance_band`, `suspension`, `other`

## Set Types

Valid values for set `type`:
- `normal` — standard working set
- `warmup` — warm-up set
- `failure` — set taken to failure
- `dropset` — drop set

## Routine Sets

For routines, working sets use `rep_range` instead of a fixed `reps` (never both on one set):
```json
{
  "type": "normal",
  "weight_kg": null,
  "rep_range": { "start": 8, "end": 12 }
}
```

Use `rep_range` for a target range; use `reps` only for a strict single-rep prescription.

## Rules

- Always use `curl` or equivalent HTTP calls with the `api-key` header (read the key in code — see Gotchas).
- Paginate through ALL pages when searching for exercises or folders — don't stop at page 1.
- Weights in the API are always in **kilograms**. Convert from lbs if the screenshot uses pounds (divide by 2.205).
- If the screenshot doesn't specify weight, pre-fill from history per "Recovery-Aware Weight Targeting" (the single source of truth). Leave `weight_kg` as `null` only when there's no history for that exercise.
- If the screenshot doesn't specify rest, set `rest_seconds` per the "Rest Times" table.
- `superset_id` should be `null` unless the screenshot explicitly groups exercises as a superset. If it does, give supersetted exercises the same integer `superset_id`.
- The routine title should match what's shown in the screenshot. If no name is shown, derive a sensible one from the workout content (e.g. "Upper Body A", "Push Day").
- Appending ` - YYYY-MM-DD` to the title is optional (see "Date Handling"). If used, the date must be freshly computed via `date +%Y-%m-%d` (or taken from the screenshot) — never a copied or hard-coded literal.
- Always carry descriptions/notes from the screenshot into the routine `notes` (overall) and per-exercise `notes` (specific) fields.
- When in doubt about an exercise match, prefer creating a custom exercise over using a wrong template.
- **Wrist-safe always:** never program a straight-barbell/locked-wrist press (bench, OHP, etc.) — substitute the neutral-grip dumbbell/machine/cable equivalent even if the screenshot prescribes the barbell version, and note the swap. See "Athlete Constraints — Wrist-Safe".
- When creating a custom exercise, always populate the secondary muscle groups (`other_muscles`) with the relevant secondary muscles — do not leave it empty.
- Always prepend a warm-up ramp (`type: "warmup"` sets) before the working sets, even if the screenshot doesn't show warm-ups. Skip only for cardio, duration-only, and pure bodyweight exercises.
