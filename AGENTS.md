# Hevy Workout Agent

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

Authentication: pass the API key from `.env` (`hevy_API`) as a header:

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
- **Weight ranges:** the API takes a single `weight_kg` per set. For a range like "10-15 kg", pick a sensible point in the range (mid or low end for low-RPE days) and note the full range in `notes`.
- **Set field by exercise type:** `weight_reps` → `weight_kg` + `reps` (or `rep_range`); `reps_only`/bodyweight → `reps`/`rep_range` only (omit `weight_kg`); `duration` (e.g. Dead Hang, Plank) → `duration_seconds`. Working sets default to `rep_range: {start, end}` (see "Rep Ranges & Targets"); warm-ups use fixed `reps`.
- **Read `.env` in code, not `source`.** `.env` has no trailing newline, so `source .env` silently sets nothing (and shell `awk`/`cut` extraction has been flaky here). Read `hevy_API` in a small Python script instead. An **empty `api-key` header returns `401 InvalidApiKey`** — that error means "no key sent," not necessarily "key revoked." Verify the key is actually non-empty before concluding it's invalid.
- **Pre-fill weights from history** for RPE-only prescriptions: `GET /v1/exercise_history/{id}`, take the **near-max working weight in the last ~120 days** (not the last session — it may have been recovery-reduced), scale by today's recovery factor. See "Recovery-Aware Weight Targeting".

## Core Workflow

When the user provides a workout screenshot:

1. **Read the screenshot** — extract exercise names, sets, reps, weight, and rest times.
2. **Match exercises to templates** — check [`EXERCISE-REGISTRY.md`](./EXERCISE-REGISTRY.md) **first**: it's a cached list of common exercise → `exercise_template_id` mappings and is much faster than hitting the API. Use a registry ID directly when there's a confident name match. Only if an exercise isn't in the registry, fall back to `GET /v1/exercise_templates?page=1&pageSize=100` (paginate through all pages) and match locally. That API list includes the user's **custom exercises** too (`is_custom: true`) — always search these as well and reuse an existing custom exercise instead of creating a duplicate. When you match a common exercise via the API that wasn't in the registry, add it to `EXERCISE-REGISTRY.md` for next time.

   - **If a registry ID fails** (a routine/exercise `POST` errors with "exercise template not found" or a 400 referencing the template), the ID may have changed. Re-fetch from the API, find the current ID by title, update `EXERCISE-REGISTRY.md` (and its provenance date), then retry. The API is authoritative; the registry is only a cache.
2.5. **Apply the wrist-safe constraint** — before finalizing each match, check it against "Athlete Constraints — Wrist-Safe". If the exercise (or a screenshot-prescribed lift) is a straight-barbell/locked-wrist press, substitute the wrist-safe equivalent and note the swap. This overrides the screenshot.
3. **Handle name mismatches** — different sources call exercises by different names. Use your knowledge to map them:
   - "Flat Bench" → "Bench Press (Barbell)"
   - "DB Rows" → "Dumbbell Row"
   - "Skull Crushers" → "Skullcrusher (Dumbbell)" or "Triceps Extension (Cable)" depending on context
   - "Lat Pulldown" → "Lat Pulldown (Cable)"
   - "OHP" → "Overhead Press (Barbell)"
   - "RDL" → "Romanian Deadlift (Barbell)"
   - "Leg Curl" → "Seated Leg Curl (Machine)"
   - Think about what exercise is likely meant based on context, equipment mentioned, and common gym terminology.
4. **If no confident match exists** (including no matching custom exercise from step 2) — create a custom exercise via `POST /v1/exercise_templates`. Always fill the secondary muscle groups (`other_muscles`) — never leave it empty:
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
5. **Use the "Cyclete" folder** — the target folder already exists with id **`2763385`**. Use `folder_id: 2763385` directly. Only if the folder is missing (verify via `GET /v1/routine_folders`) should you create it:
   ```json
   POST /v1/routine_folders
   { "routine_folder": { "title": "Cyclete" } }
   ```
6. **Create the routine** in that folder via `POST /v1/routines`. Optionally append the date to the title (see "Date Handling" below — the date is optional and, if used, must be a freshly computed value, never a copied literal) and carry over any descriptions/notes from the screenshot into the `notes` fields:
   ```json
   {
     "routine": {
       "title": "Routine Name From Screenshot - 2026-07-20",
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
             { "type": "normal", "weight_kg": null, "reps": null }
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

If the screenshot specifies rest times, use them. Otherwise set `rest_seconds` based on how systemically fatiguing the exercise is:

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

- If the working weight is known, compute the warm-up `weight_kg` from these percentages (round to a sensible increment, e.g. nearest 2.5 kg). If the working weight is unknown (`null`), still add the warm-up sets with `weight_kg: null` so the ramp structure is there for the user to fill in.
- Use a lighter ramp (1–2 warm-up sets) for isolation or accessory movements (e.g. curls, lateral raises, cable work).
- **Do not** add warm-up sets for cardio, duration-only exercises (e.g. plank), or pure bodyweight movements where a ramp doesn't apply.
- Warm-up sets always come first in the `sets` array, followed by the working sets from the screenshot.

## Recovery-Aware Weight Targeting

When the screenshot gives no explicit weight (e.g. "load to RPE" / RPE-only prescriptions), pre-fill a target `weight_kg` from history. **Do not just copy the last session's weight** — the last session may itself have been recovery-reduced (a low-RPE / deload day), so using it as today's goal under-shoots when today's recovery is high. Instead, estimate the user's *capacity* from the best of their recent sessions, then scale that by today's recovery.

The user states today's **recovery context** per request (e.g. "100%, recovery good", or "deload / 50%"). Convert it to a factor `R` (1.0 = 100%; default 1.0 if they describe it as good/normal and give no number).

**Recovery gate — >70% unlocks high intensity / progressive overload:**

| Recovery | Tier | Weight | Progression | Rep targets |
|----------|------|--------|-------------|-------------|
| **> 70%** | High intensity | Capacity **+ earned load step** (see Progressive Overload) | **Applied** — this is the only tier that adds load | Full rep range `[low, high]` |
| ~50–70% | Maintain | Capacity (optionally × `R`) | **None** — hold, don't add load | Lower half of the range |
| < 50% / explicit deload | Reduce | Capacity × `R` | None | Low end / fixed lower reps, optionally fewer sets |

Progressive overload (load bumps, pushing to the top of the range) fires **only when recovery > 70%**. At ≤70% you target capacity or below and do not progress load — that day is for maintaining or backing off, not setting PRs.

**Algorithm (per exercise):**

1. **Fetch history** — `GET /v1/exercise_history/{exerciseTemplateId}` (optionally `?start_date=...`). The response is `{"exercise_history": [ ...sets... ]}`; each entry is one logged set with `workout_id`, `workout_start_time`, `weight_kg`, `reps`, `set_type`.
2. **Group into sessions** by `workout_id`; sort by `workout_start_time` descending.
3. **Take the current training block** — sessions within the **last ~120 days** (cap ~10 sessions). Bounding by *time* (not just session count) is essential: infrequently-trained lifts otherwise reach back into old blocks and resurrect stale PRs as today's target. Real example from this account: a naive "max of last 10 sessions" picked a **140 kg × 1 squat single from 2 years ago** and a **92.5 kg hip-abduction from a different block a year+ ago**, when current working weights are 70 kg and 60 kg. The 120-day window excludes those.
4. **If no session falls in the window** (exercise not done in ~4 months), fall back to the single most recent session and note it's stale. Do **not** scan all-time for a maximum.
5. **Near-max capacity** — from the block's **working** sets (`set_type` `normal`/`failure`; exclude `warmup`), take the **highest** `weight_kg`. Drop a lone top spike (a single set >1.5x the next-highest distinct value that appears only once) in favor of the second-highest. This near-max ≈ capacity on a good day and **automatically ignores recovery-reduced sessions** — those are lower, so they never become the max. (This is exactly why last-session copying fails: e.g. Face Pull was 20 kg last session but 45 kg on surrounding sessions → capacity is 45 kg.)
6. **Scale by recovery** — `target = near_max_capacity × R`, rounded to the equipment increment (2.5 kg for barbell / machine / cable; ~1 kg for light dumbbells). Round down between increments. At 100% recovery, target = capacity (aim to match/beat your best); a deload day scales down (e.g. `R = 0.5`).
7. **No history** → leave `weight_kg: null` and note "no history — pick to RPE". Don't invent a number.
8. **Note provenance** in the exercise `notes`, e.g. `"Block near-max 45 kg (2026-07-25); recovery 100% -> target 45 kg"`.

**Reps caveat:** capacity is taken as a weight, not normalized for reps. If today's target reps are much higher than where the near-max was set, expect to fall a little short — autoregulate in-session. (Epley e1RM normalization is an option but overshoots at high reps, so weight-based near-max is the default.)

**Same-gym assumption:** the user trains at one gym, so machine loads — including the **Gym80** machines (hip thrust, abductor, adductor) — are directly comparable session-to-session; no leverage normalization is needed. Keep matching each screenshot exercise to the **same** template every time so its history stays coherent.

**Reading the API key:** read `hevy_API` from `.env` **in code** (e.g. a small Python script), not via `source .env` — the file has no trailing newline, so `source` silently sets nothing. A request sent with an empty `api-key` header returns `401 InvalidApiKey`, which looks like a bad key but really means no key was sent. See the Gotchas.

## Progressive Overload

Recovery-aware targeting (above) sets the baseline so today's weight matches current capacity. **Progressive overload** is the layer on top that makes the user actually get stronger/more muscular over time: on a good-recovery day, nudge slightly *beyond* recent capacity when the previous performance earned it. Without this, targeting capacity alone just maintains.

**Method — double progression.** Work within the screenshot's rep target (treat it as the top of a small range). The rule:

- If the most recent **non-reduced** session hit the **top of the rep range on all working sets** at the capacity weight → add load by the exercise-class increment below (and, if using a range, reset reps to the bottom).
- If it didn't → **hold the weight and add reps** toward the top of the range. Adding reps is progressive overload too.

Evidence basis (meta-analyses / reviews preferred; content rephrased for licensing compliance):
- Progressing **load or reps** yields similar muscle adaptations, so rep progression counts as overload — [Plotkin et al. 2022](https://pmc.ncbi.nlm.nih.gov/articles/PMC9528903/).
- **Double progression** operationalizes this; %1RM and rep-max load prescriptions both build strength — [load-prescription systematic review](https://pmc.ncbi.nlm.nih.gov/articles/PMC7142036/).
- **Failure is not required** for gains, so progress while leaving 1–2 reps in reserve — [failure vs non-failure meta-analysis](https://pmc.ncbi.nlm.nih.gov/articles/PMC9068575/).
- **Bigger load jumps on heavy compounds than on light isolation is mechanical, not sex-specific.** A fixed increment (e.g. 2.5 kg) is a small % of a heavy squat but a large % of a light curl, so heavy compounds tolerate load steps while light lifts progress by reps. The exercise-class caps below encode this. Sex-comparison meta-analyses find men and women gain *relative* strength comparably (women if anything show slightly greater relative upper-body gains; men have greater absolute strength), so these rules apply equally regardless of sex — [sex-differences meta-analysis (JSCR 2020)](https://journals.lww.com/nsca-jscr/fulltext/2020/05000/sex_differences_in_resistance_training__a.30.aspx).
- A women-only meta-analysis is sometimes cited for "lower body progresses faster than upper body," but its per-week figures are frequency-dependent and inconsistently reported, so it is not a reliable basis for a precise rate — treat the caps below as conservative engineering limits, not derived from that number — [PLOS ONE 2023](https://pubmed.ncbi.nlm.nih.gov/37053143/).

**Conservative, exercise-aware load increments** (bias low — smaller than the meta rates, since those are weekly 1RM gains, not guaranteed per-session jumps):

| Exercise class | Examples | Load step when earned | Per-step cap |
|----------------|----------|-----------------------|--------------|
| Lower-body compound | squat, deadlift, RDL, leg press, hip thrust | +2.5–5 kg | ≤ ~5% |
| Upper-body compound | bench, incline press, OHP, row, lat pulldown | +2.5 kg | ≤ ~3% |
| Isolation / small muscle / light dumbbell | curls, triceps, lateral raise, face pull | usually **add reps**; load only +smallest step | ≤ ~2–3% |

**The cap decides load-vs-reps *within the rep range*.** Compute the smallest available increment as a percentage of the current weight. If it is **within** the class cap → add load. If it **exceeds** the cap (common on light loads — e.g. the next dumbbell up from 8 kg is +25%, or +2.5 kg on a 20 kg machine is +12.5%) → **add reps instead** and hold the load — *but only until you reach the top of the rep range.* The top of the range is a hard ceiling that forces the switch to load regardless of the cap (see below). This is why light isolation work progresses mostly by reps, then takes an occasional bigger load jump.

### Dynamic double progression (rep range + RPE/failure)

Each exercise runs on a rep **range** `[low, high]`, not a single number, so it can cycle between adding reps and adding load:

- **Range source:** use the screenshot's range if it gives one (e.g. "8–12"). If it gives a single number `T`, treat `T` as the top and set `low` a few reps below: strength/low-rep `6 → 4–6`, hypertrophy `12 → 9–12`, high-rep isolation `15 → 12–15`.

**The cycle (per working weight):**
1. **Climb reps** within the range across sessions (the "add reps" branch above).
2. When **all working sets reach `high`** → **add one load step and reset reps to `low`.** This is the switch back to weight. It fires **even when the smallest load step exceeds the % cap** — the rep buffer you built from `low`→`high` offsets the larger relative jump (e.g. an 8 kg curl climbed 10→15 reps, then → 10 kg at ~10 reps; the +25% load is absorbed by dropping 15→10 reps).
3. Repeat at the new weight.

So: **the % cap governs progression *within* the range; hitting the top of the range is the hard trigger to add load regardless of the cap.**

**RPE / failure modulation** — read the last comparable good-recovery session at the current weight and classify effort from logged `rpe` and `set_type`:

- **Easy:** RPE ≤ 7 (≈3+ reps in reserve), no `failure` sets.
- **Moderate:** RPE 7.5–8.5, no failure.
- **Hard:** RPE ≥ 9, or any working set marked `set_type: "failure"`.
- **Unknown:** no RPE and no failure logged → skip modulation, decide on reps-vs-range only.

| Last session at current weight | Effort | Action |
|--------------------------------|--------|--------|
| All sets at top of range | Easy / Moderate | Add load (class step, or smallest step if over cap), reset reps to `low` |
| All sets at top of range | Hard | Add load at the **smallest** step, reset to `low` (earned, but jump minimally) |
| Below top of range | Easy (RPE ≤ 7) | Add reps (+1–2); if RPE ≤ 6 the weight is too light — add 2 reps or bring the next load step forward |
| Below top of range | Moderate | Add reps +1 |
| Below top of range | Hard (failure / RPE ≥ 9) | Hold weight and reps; repeat the session |
| Below top, no rep gain for 2 sessions, Hard | — | Plateau: deload ~10% (or drop to `low`) and climb again |

**Data reality:** in this account RPE is logged inconsistently (some sets have `rpe: null`) but `set_type: "failure"` is reliable when used. When RPE is missing, fall back to the failure flag plus reps-vs-range; treat "hit top of range, no failure" as at least Moderate → add load and reset.

**Provenance in notes:** e.g. `"8 kg x12 (top) @ RPE 7.5, no failure -> +2 kg, reset to 10 reps"`, or `"12 kg x8 @ RPE 9 -> hold, repeat"`.

**Defensive guards (this is meant to be conservative):**
- Only progress from **good-recovery** sessions. Never add load on a deload / reduced-recovery day — that day scales *down* via the recovery factor, full stop.
- **One increment per session** maximum. No double jumps.
- **Hold** when data is sparse or stale (fewer than ~2 in-block sessions, or the lift hasn't been done in ~4 months), or when the target reps were missed last time.
- If a weight **stalls for two sessions** (top reps not reached), hold or drop ~10% and rebuild — don't force it.
- **Screenshot reps vs the rep-range cycle:** a screenshot's single rep number is treated as the **top** of the pre-filled `rep_range` (`high = T`), with `low = T − band`. Progress via **weight** across sessions; don't invent a scheme the screenshot doesn't imply. The rep-range climb/reset cycle (Dynamic double progression) governs movement within `[low, high]` and when planning *subsequent* sessions. For a strict single-rep prescription (e.g. peaking `5×5`), use fixed `reps` instead of a range.

**Provenance:** note the decision in the exercise `notes`, e.g. `"Hit 70x6 last block; lower compound +2.5 -> 72.5 kg"` or `"Isolation, +2.5 kg = 10% > cap -> hold 25 kg, add reps toward 15"`.

## Rep Ranges & Targets (`rep_range`)

The API accepts a **`rep_range`** object on each set — `{"start": <low>, "end": <high>}` — instead of a fixed `reps`. This is how the double-progression range gets pre-filled so Hevy shows a target like "8–12" rather than a single number. Both `PostRoutinesRequestSet` and `PutRoutinesRequestSet` support it.

- **Working sets → use `rep_range`** by default, set to the exercise's double-progression range `[low, high]`. Derive it from the screenshot: a single number `T` becomes the **top** (`high = T`, `low = T − band`; band ≈2 low-rep/strength, 2–3 hypertrophy, 3 high-rep). A screenshot that already gives a range maps directly.
- **Warm-up sets → keep fixed `reps`** (no range).
- **Strict single-rep prescriptions** (e.g. a peaking `5×5`) → use fixed `reps`, not a range.
- **Do not send both** `reps` and `rep_range` on the same set — pick one. `rep_range` for a ranged target, `reps` for a single target.
- **Recovery interaction:** at **R > 70%** set the full range `[low, high]` at capacity + earned progression. At ≤70%, target the **lower end** (narrow the range downward or use a fixed lower `reps`) with reduced weight.

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

For routines (as opposed to logged workouts), sets can include a `rep_range` instead of fixed `reps`:
```json
{
  "type": "normal",
  "weight_kg": null,
  "reps": null,
  "rep_range": { "start": 8, "end": 12 }
}
```

Use `rep_range` when the screenshot shows a range (e.g. "8-12 reps"). Use `reps` when it shows a fixed number.

## Rules

- Always use `curl` or equivalent HTTP calls with the `api-key` header.
- Paginate through ALL pages when searching for exercises or folders — don't stop at page 1.
- Weights in the API are always in **kilograms**. Convert from lbs if the screenshot uses pounds (divide by 2.205).
- If the screenshot doesn't specify weight, pre-fill from history when possible (see "Recovery-Aware Weight Targeting"): take the near-max working weight from the last ~120 days via `GET /v1/exercise_history/{id}`, scaled by today's recovery factor — not the last session's weight, which may have been recovery-reduced. Leave `weight_kg` as `null` only when there's no history for that exercise.
- If the screenshot doesn't specify rest, set `rest_seconds` by exercise type (see "Rest Times"): ~180s for heavy compounds, ~120s for moderate compounds, 70–90s for normal accessory/isolation work, 45–60s for core/duration holds.
- `superset_id` should be `null` unless the screenshot explicitly groups exercises as a superset. If it does, give supersetted exercises the same integer `superset_id`.
- The routine title should match what's shown in the screenshot. If no name is shown, derive a sensible one from the workout content (e.g. "Upper Body A", "Push Day").
- Appending ` - YYYY-MM-DD` to the title is optional (see "Date Handling"). If used, the date must be freshly computed via `date +%Y-%m-%d` (or taken from the screenshot) — never a copied or hard-coded literal.
- Always carry descriptions/notes from the screenshot into the routine `notes` (overall) and per-exercise `notes` (specific) fields.
- When in doubt about an exercise match, prefer creating a custom exercise over using a wrong template.
- **Wrist-safe always:** never program a straight-barbell/locked-wrist press (bench, OHP, etc.) — substitute the neutral-grip dumbbell/machine/cable equivalent even if the screenshot prescribes the barbell version, and note the swap. See "Athlete Constraints — Wrist-Safe".
- When creating a custom exercise, always populate the secondary muscle groups (`other_muscles`) with the relevant secondary muscles — do not leave it empty.
- Always prepend a warm-up ramp (`type: "warmup"` sets) before the working sets, even if the screenshot doesn't show warm-ups. Skip only for cardio, duration-only, and pure bodyweight exercises.
