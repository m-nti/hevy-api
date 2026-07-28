# hevy-api

Turn workout screenshots into [Hevy](https://www.hevy.com/) routines via the Hevy public API.

Drop in a screenshot of a workout and an agent reads the exercises, sets, reps, weights, rest, and notes, matches each exercise to Hevy's template library (creating custom exercises when needed), and creates the routine in the target folder.

## Setup

1. Copy the example env file and add your Hevy API key:
   ```bash
   cp .env.example .env
   ```
2. Get your key from the Hevy web app (requires Hevy Pro): https://hevy.com/settings?developer
3. Paste it into `.env` as `hevy_API=...`.

> `.env` is gitignored — never commit it.

## Usage

[`EXERCISE-REGISTRY.md`](./EXERCISE-REGISTRY.md) caches common exercise → template-ID mappings so matching is fast and doesn't always hit the API.

The agent workflow, conventions, and API reference live in [`AGENTS.md`](./AGENTS.md). It covers:

- Matching screenshot exercises to Hevy templates (including fuzzy name mapping)
- Creating custom exercises with muscle groups when no match exists
- Warm-up ramps, rest times by exercise type, and carrying over notes/descriptions
- The full Hevy API endpoint and schema reference

## API

Base URL: `https://api.hevyapp.com` — authenticate with the `api-key` header.
Full docs: https://api.hevyapp.com/docs/
