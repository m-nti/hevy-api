# Exercise Registry

Cached Hevy `exercise_template_id` values for common exercises, so matching a screenshot doesn't require the full paginated `GET /v1/exercise_templates` fetch every time.

**Provenance:** All IDs below were verified live from this account's `GET /v1/exercise_templates` on **2026-07-20**. Standard (non-custom) IDs are Hevy's global template IDs; custom IDs are specific to this account.

## How to use this registry

1. **Look here first.** When matching a screenshot exercise, check this file before fetching the API. It covers the common lifts and is much faster.
2. **Use the ID directly** if there's a confident name match.
3. **If an ID fails** — a routine/exercise `POST` returns an error like "exercise template not found" or a 400 referencing the template — the ID may have changed. Re-fetch from the API (`GET /v1/exercise_templates?page=N&pageSize=100`, paginate all pages), find the current ID by title, **update this file** (and its provenance date), then retry.
4. **If an exercise isn't listed here**, fall back to the full API search per `AGENTS.md` (fetch all pages, match locally, including custom exercises). Add newly matched common exercises to this file for next time.
5. **Custom exercises are account-specific** and more likely to change or be absent — always be ready to re-verify them against the API.

This registry is a cache, not the source of truth. The API is authoritative; when they disagree, the API wins and this file gets updated.

## Chest

| Exercise | ID | Type |
|----------|----|------|
| Bench Press (Barbell) | `79D0BB3A` | weight_reps |
| Bench Press (Dumbbell) | `3601968B` | weight_reps |
| Bench Press (Cable) | `99C1F2AD` | weight_reps |
| Bench Press - Close Grip (Barbell) | `35B51B87` | weight_reps |
| Bench Press (Smith Machine) | `0FBF7195` | weight_reps |
| Incline Bench Press (Barbell) | `50DFDFAB` | weight_reps |
| Incline Bench Press (Dumbbell) | `07B38369` | weight_reps |
| Incline Bench Press (Smith Machine) | `3A6FA3D1` | weight_reps |
| Decline Bench Press (Barbell) | `DA0F0470` | weight_reps |
| Decline Bench Press (Dumbbell) | `18487FA7` | weight_reps |
| Decline Bench Press (Machine) | `FAF31231` | weight_reps |
| Chest Press (Machine) | `7EB3F7C3` | weight_reps |
| Incline Chest Press (Machine) | `FBF92739` | weight_reps |
| Chest Fly (Dumbbell) | `12017185` | weight_reps |
| Chest Fly (Machine) | `78683336` | weight_reps |
| Incline Chest Fly (Dumbbell) | `D3E2AB55` | weight_reps |
| Butterfly (Pec Deck) | `9DCE2D64` | weight_reps |
| Cable Fly Crossovers (high-to-low) | `651F844C` | weight_reps |
| Low Cable Fly Crossovers (low-to-high) | `293483AD` | weight_reps |
| Single Arm Cable Crossover | `8372BBCC` | weight_reps |
| Seated Chest Flys (Cable) | `6B4C797E` | weight_reps |
| Chest Dip | `6FCD7755` | reps_only |

## Back

| Exercise | ID | Type |
|----------|----|------|
| Lat Pulldown (Cable) | `6A6C31A5` | weight_reps |
| Lat Pulldown - Close Grip (Cable) | `4E5257DE` | weight_reps |
| Lat Pulldown (Machine) | `473CF5B8` | weight_reps |
| Reverse Grip Lat Pulldown (Cable) | `046E25A2` | weight_reps |
| Single Arm Lat Pulldown | `2EE45F81` | weight_reps |
| Straight Arm Lat Pulldown (Cable) | `D2387AB1` | weight_reps |
| Bent Over Row (Barbell) | `55E6546F` | weight_reps |
| Bent Over Row (Dumbbell) | `23E92538` | weight_reps |
| Pendlay Row (Barbell) | `018ADC12` | weight_reps |
| T Bar Row | `08A2974E` | weight_reps |
| Chest Supported T Bar Row | `6A8D3193` | weight_reps |
| Iso-Lateral Row (Machine) | `AA1EB7D8` | weight_reps |
| Iso-Lateral Low Row | `91FAFBA3` | weight_reps |
| Iso-Lateral High Row (Machine) | `BC3492DA` | weight_reps |
| Seated Row (Machine) | `1DF4A847` | weight_reps |
| Seated Cable Row - Bar Grip | `F1D60854` | weight_reps |
| Seated Cable Row - Bar Wide Grip | `C3BCABB3` | weight_reps |
| Seated Cable Row - V Grip (Cable) | `0393F233` | weight_reps |
| Single Arm Cable Row | `D0C4A899` | weight_reps |
| Dumbbell Row | `F1E57334` | weight_reps |
| Meadows Rows (Barbell) | `C732C341` | weight_reps |
| Seal Row (Barbell) | `DF3BDB9C` | weight_reps |
| Seal Row (Dumbbell) | `7E93E23F` | weight_reps |
| Pull Up | `1B2B1E7C` | reps_only |
| Scapular Pull Ups | `C7AE420A` | reps_only |
| Face Pull | `BE640BA0` | weight_reps |

## Shoulders

| Exercise | ID | Type |
|----------|----|------|
| Overhead Press (Barbell) | `7B8D84E8` | weight_reps |
| Overhead Press (Dumbbell) | `6AC96645` | weight_reps |
| Overhead Press (Smith Machine) | `B09A1304` | weight_reps |
| Seated Overhead Press (Barbell) | `91AF29E0` | weight_reps |
| Seated Overhead Press (Dumbbell) | `9930DF71` | weight_reps |
| Seated Shoulder Press (Machine) | `9237BAD1` | weight_reps |
| Shoulder Press (Dumbbell) | `878CD1D0` | weight_reps |
| Arnold Press (Dumbbell) | `A69FF221` | weight_reps |
| Lateral Raise (Dumbbell) | `422B08F1` | weight_reps |
| Lateral Raise (Cable) | `BE289E45` | weight_reps |
| Lateral Raise (Machine) | `D5D0354D` | weight_reps |
| Single Arm Lateral Raise (Cable) | `DE68C825` | weight_reps |
| Seated Lateral Raise (Dumbbell) | `9372FFAA` | weight_reps |
| Upright Row (Barbell) | `7AB9A362` | weight_reps |
| Upright Row (Cable) | `286C1D0B` | weight_reps |
| Upright Row (Dumbbell) | `797F0782` | weight_reps |
| Rear Delt Reverse Fly (Cable) | `C315DC2A` | weight_reps |
| Rear Delt Reverse Fly (Dumbbell) | `E5988A0A` | weight_reps |
| Rear Delt Reverse Fly (Machine) | `D8281C62` | weight_reps |
| Band Pullaparts | `E8D86EE8` | reps_only |

## Biceps

| Exercise | ID | Type |
|----------|----|------|
| Bicep Curl (Barbell) | `A5AC6449` | weight_reps |
| Bicep Curl (Dumbbell) | `37FCC2BB` | weight_reps |
| Bicep Curl (Cable) | `ADA8623C` | weight_reps |
| Bicep Curl (Machine) | `AF328E3D` | weight_reps |
| EZ Bar Biceps Curl | `01A35BF9` | weight_reps |
| Hammer Curl (Dumbbell) | `7E3BC8B6` | weight_reps |
| Hammer Curl (Cable) | `36E8F14E` | weight_reps |
| Cross Body Hammer Curl | `32C4D4A2` | weight_reps |
| Concentration Curl | `724CDE60` | weight_reps |
| Preacher Curl (Barbell) | `4F942934` | weight_reps |
| Preacher Curl (Dumbbell) | `FAB6EB2F` | weight_reps |
| Preacher Curl (Machine) | `1E9A6B8E` | weight_reps |
| Reverse Curl (Barbell) | `112FC6B7` | weight_reps |
| Reverse Curl (Cable) | `9F48F858` | weight_reps |
| Reverse Curl (Dumbbell) | `B567DD46` | weight_reps |
| Spider Curl (Dumbbell) | `90427D4A` | weight_reps |

## Triceps

| Exercise | ID | Type |
|----------|----|------|
| Triceps Pushdown (bar) | `93A552C6` | weight_reps |
| Triceps Rope Pushdown | `94B7239B` | weight_reps |
| Triceps Pressdown | `CDC472B1` | weight_reps |
| Reverse Grip Triceps Pushdown | `8B5BED30` | weight_reps |
| Single Arm Triceps Pushdown (Cable) | `552AB030` | weight_reps |
| Triceps Extension (Barbell) | `2F8D3067` | weight_reps |
| Triceps Extension (Cable) | `21310F5F` | weight_reps |
| Triceps Extension (Dumbbell) | `3765684D` | weight_reps |
| Triceps Extension (Machine) | `3092FADD` | weight_reps |
| Overhead Triceps Extension (Cable) | `B5EFBF9C` | weight_reps |
| Seated Triceps Press | `234BC743` | weight_reps |
| Triceps Dip | `28BB4A95` | reps_only |

## Legs

| Exercise | ID | Type |
|----------|----|------|
| Squat (Barbell) | `D04AC939` | weight_reps |
| Squat (Dumbbell) | `DCFF3E9F` | weight_reps |
| Squat (Machine) | `CC35A01F` | weight_reps |
| Squat (Smith Machine) | `DDCC3821` | weight_reps |
| Squat (Bodyweight) | `9694DA61` | reps_only |
| Front Squat | `5046D0A9` | weight_reps |
| Goblet Squat | `3D0C7C75` | weight_reps |
| Hack Squat (Machine) | `1E42FD5F` | weight_reps |
| Pendulum Squat (Machine) | `30E293E3` | weight_reps |
| Bulgarian Split Squat (Dumbbell) | `B5D3A742` | weight_reps |
| Bulgarian Split Squat (Barbell) | `0F24286A` | weight_reps |
| Deadlift (Barbell) | `C6272009` | weight_reps |
| Deadlift (Dumbbell) | `5F4E6DD3` | weight_reps |
| Deadlift (Trap bar) | `B923B230` | weight_reps |
| Sumo Deadlift | `D20D7BBE` | weight_reps |
| Romanian Deadlift (Barbell) | `2B4B7310` | weight_reps |
| Romanian Deadlift (Dumbbell) | `72CFFAD5` | weight_reps |
| Leg Press (Machine) | `C7973E0E` | weight_reps |
| Leg Press Horizontal (Machine) | `0EB695C9` | weight_reps |
| Leg Extension (Machine) | `75A4F6C4` | weight_reps |
| Lying Leg Curl (Machine) | `B8127AD1` | weight_reps |
| Seated Leg Curl (Machine) | `11A123F3` | weight_reps |
| Lunge (Dumbbell) | `B537D09F` | weight_reps |
| Lunge (Barbell) | `6E6EE645` | weight_reps |
| Reverse Lunge (Dumbbell) | `FFDA283B` | weight_reps |
| Good Morning (Barbell) | `4180C405` | weight_reps |
| Standing Calf Raise (Machine) | `E05C2C38` | weight_reps |
| Seated Calf Raise | `062AB91A` | weight_reps |
| Calf Press (Machine) | `91237BDD` | weight_reps |

## Glutes / Hips

| Exercise | ID | Type |
|----------|----|------|
| Hip Thrust (Barbell) | `D57C2EC7` | weight_reps |
| Hip Thrust (Machine) | `68CE0B9B` | weight_reps |
| Hip Thrust (Dumbbell) | `DA5430FC` | weight_reps |
| Glute Bridge (Barbell) | `49C922A1` | weight_reps |
| Glute Kickback (Machine) | `CBA02382` | weight_reps |
| Standing Cable Glute Kickbacks | `ACB2751D` | weight_reps |
| Hip Abduction (Machine) | `F4B4C6EE` | weight_reps |
| Hip Abduction (Cable) | `C469EA70` | weight_reps |
| Hip Adduction (Machine) | `8BEBFED6` | weight_reps |
| Hip Adduction (Cable) | `22578A94` | weight_reps |

## Core

| Exercise | ID | Type |
|----------|----|------|
| Plank | `C6C9B8A0` | duration |
| Side Plank | `E3EDA509` | duration |
| Crunch | `DCF3B31B` | reps_only |
| Cable Crunch | `23A48484` | weight_reps |
| Crunch (Machine) | `EB43ADD4` | weight_reps |
| Decline Crunch | `BC10A922` | reps_only |
| Sit Up | `022DF610` | reps_only |
| Hanging Leg Raise | `F8356514` | reps_only |
| Lying Leg Raise | `09C9F635` | reps_only |
| Russian Twist (Weighted) | `2982AA23` | weight_reps |
| Ab Wheel | `99D5F10E` | reps_only |
| Back Extension (Machine) | `A05C064D` | weight_reps |
| Back Extension (Hyperextension) | `4F5866F8` | reps_only |

## Cardio / Other

| Exercise | ID | Type |
|----------|----|------|
| Dead Hang | `B9380898` | duration |
| Rowing Machine | `0222DB42` | distance_duration |
| Farmers Walk | `50C613D0` | short_distance_weight |

## Custom exercises (this account)

Account-specific — verify against the API before relying on these; they are more likely to change or be missing than standard templates.

| Exercise | ID | Type | Note |
|----------|----|------|------|
| Standing Single-Arm Dumbbell Press | `513523cc-90af-4766-87ca-b58055a20a6a` | weight_reps | Created 2026-07-20 (single-arm DB overhead press; no standard template exists) |
| Chest Supported Machine Row GA | `b2c72d70-9d52-4ca1-a28b-8b4979bc5da1` | weight_reps | Pre-existing custom; prefer standard `Iso-Lateral Row (Machine)` unless specifically wanted |
