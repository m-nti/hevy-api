# Hevy Workout Agent

## Purpose

Convert workout screenshots into Hevy routines via the Hevy API. Routines go into the **"Cyclete"** routine folder (`folder_id: 2763385`).

> Note: The user refers to this folder as "cyclite", but the actual folder on the account is titled **"Cyclete"** with id **`2763385`**. Use that folder.

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
- **Set field by exercise type:** `weight_reps` → `weight_kg` + `reps`; `reps_only`/bodyweight → `reps` only (omit `weight_kg`); `duration` (e.g. Dead Hang, Plank) → `duration_seconds`.

## Core Workflow

When the user provides a workout screenshot:

1. **Read the screenshot** — extract exercise names, sets, reps, weight, and rest times.
2. **Match exercises to templates** — check [`EXERCISE-REGISTRY.md`](./EXERCISE-REGISTRY.md) **first**: it's a cached list of common exercise → `exercise_template_id` mappings and is much faster than hitting the API. Use a registry ID directly when there's a confident name match. Only if an exercise isn't in the registry, fall back to `GET /v1/exercise_templates?page=1&pageSize=100` (paginate through all pages) and match locally. That API list includes the user's **custom exercises** too (`is_custom: true`) — always search these as well and reuse an existing custom exercise instead of creating a duplicate. When you match a common exercise via the API that wasn't in the registry, add it to `EXERCISE-REGISTRY.md` for next time.

   - **If a registry ID fails** (a routine/exercise `POST` errors with "exercise template not found" or a 400 referencing the template), the ID may have changed. Re-fetch from the API, find the current ID by title, update `EXERCISE-REGISTRY.md` (and its provenance date), then retry. The API is authoritative; the registry is only a cache.
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
6. **Create the routine** in that folder via `POST /v1/routines`. Append the date to the title (see "Date Handling" below) and carry over any descriptions/notes from the screenshot into the `notes` fields:
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

Always append the date to the routine title when creating it, in `YYYY-MM-DD` format:

```
<Routine Name> - <YYYY-MM-DD>
```

Examples: `Push Day - 2026-07-20`, `Upper Body A - 2026-07-20`.

- Use the date shown in the screenshot if one is present (this is the "respective date" for that workout).
- If the screenshot has no date, use today's date. Get it with `date +%Y-%m-%d` in the shell.

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
- If the screenshot doesn't specify weight, leave `weight_kg` as `null`.
- If the screenshot doesn't specify rest, set `rest_seconds` by exercise type (see "Rest Times"): ~180s for heavy compounds, ~120s for moderate compounds, 70–90s for normal accessory/isolation work, 45–60s for core/duration holds.
- `superset_id` should be `null` unless the screenshot explicitly groups exercises as a superset. If it does, give supersetted exercises the same integer `superset_id`.
- The routine title should match what's shown in the screenshot, with the date appended as ` - YYYY-MM-DD`. If no name is shown, derive a sensible one from the workout content (e.g. "Upper Body A", "Push Day").
- Always append the respective date to the routine title (date from the screenshot, or today's date if absent).
- Always carry descriptions/notes from the screenshot into the routine `notes` (overall) and per-exercise `notes` (specific) fields.
- When in doubt about an exercise match, prefer creating a custom exercise over using a wrong template.
- When creating a custom exercise, always populate the secondary muscle groups (`other_muscles`) with the relevant secondary muscles — do not leave it empty.
- Always prepend a warm-up ramp (`type: "warmup"` sets) before the working sets, even if the screenshot doesn't show warm-ups. Skip only for cardio, duration-only, and pure bodyweight exercises.
