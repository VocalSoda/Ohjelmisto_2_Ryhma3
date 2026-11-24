# runway hop game
This is a browser-based client-server game,
based on the original idea of Crossy Road, excepy you're an airplane.

## Concept
You are a tiny electric mail plane. 
The airport is way too busy to taxi you, so you hop across service roads, taxiways 
and runways to reach your departure plane on the far side.

**Rules**
- Grid hops with arrows/WASD. Traffic moves in real time.
- Lane types: pavement (safe), service road (carts, slow), taxiway (tugs, medium), runway (aircraft, fast).
- Hitting something causes a failure.
- Reaching the top row wins the run.

## Architecture
This is quite general, we don't need to follow this exactly:
```
Browser: HTML/CSS/JS + Three.js
  - rendering, input, local movement and collision
        |  JSON over HTTP
Python server: Flask
  - level generation, game records, progress, scoring
        |
SQLite: course airport database + game table
```

Flask serves the static frontend, so everything is same-origin, which means we don't
have to deal with CORS (evil)

**Division of responsibility**

- **Server's problem:** level layout, difficulty, game records, furthest row, score, completion, leaderboard.
- **Browser's problem:** camera, models, animation, key handling, collision response, HUD.

## Data
We'll use the prev database, just write a script like this,
to modify the game table.

```sql
CREATE TABLE game (
  id INTEGER PRIMARY KEY,
  airport_code TEXT NOT NULL,
  player_name TEXT,
  seed INTEGER NOT NULL,
  status TEXT NOT NULL,          -- for example: playing or won or stranded
  max_row INTEGER DEFAULT 0,
  hops INTEGER DEFAULT 0,
  seconds REAL,
  battery_left INTEGER,
  score INTEGER,
  created_at TEXT,
  finished_at TEXT
);
```

Seed! This is important, we need a deterministic run between airports.
What we'll do is, we'll take the ICAO code of the target location, and generate a SHA256.
Then, we'll take like the first maybe 8-10 bytes, and use this as a seed.

This way, we get procedurally generated levels from this seed, which are deterministic.
Do NOT use the built-in python hash function, it's salted per process, meaning it's **not stable**.

On the server, this is gonna be sed to derive the level data, and on the client, this is responsible for positioning stuff.

## API

This is a general idea for the API, no need to followi t exact:
| Method | Path | Body | Returns |
|---|---|---|---|
| GET | `/api/airports` | – | list: icao, name, completed, best_score |
| POST | `/api/games` | `{icao, player_name}` | `{game_id, seed, rows, lanes[]}` |
| POST | `/api/games/{id}/progress` | `{row, hops, battery}` | `{ok: true}` |
| POST | `/api/games/{id}/finish` | `{row, hops, seconds, battery_left}` | `{score, status, best_score}` |
| GET | `/api/games/{id}` | – | current game state (resume, optional) |

Progress SHOULD NOT be pushed per row, maybe every 5 rows is fine.

**example level json might look like this:**
```json
{
  "game_id": 41,
  "airport": {"icao": "EFHK", "name": "Helsinki-Vantaa"},
  "seed": 8437291042,
  "rows": 24,
  "lanes": [
    {"row": 3, "type": "service", "dir": 1, "speed": 1.2,
     "obstacles": [{"x": -5.0, "w": 2.0}, {"x": 2.0, "w": 2.0}]},
    {"row": 4, "type": "runway", "dir": -1, "speed": 3.0,
     "obstacles": [{"x": 0.0, "w": 4.0}]}
  ]
}
```

One cell = one world unit. `x` runs across the lanes and then rows run forward. 
Speeds are cells per second. Collision is 2D on `(x, row)`.

**Score (handled server only):** `max_row * 10 + battery_left * 5 + (200 if won)`.

## Role Division
Backend is kinda flexible as I (tomi) don't want to hog all the programming.

| Person | Role | Owns |
|---|---|---|
| A | Frontend — Graphics (vanya) | Three.js scene, models, game loop, camera, animation |
| B | Frontend — UI (zeynab) | Screens, HUD, input, API client, progress pings |
| C | Backend — Levels (maksym) | DB schema, seed, generator, airports + create-game endpoints |
| D | Backend — Quality & Lead (tomi) | Progress, finish, scoring, validation, leaderboard |


## Tasks
PLEASE NOTE: These are guiding principles, if you have differing ideas and stuff,
please send it in the group chat and I'll take a look at them.

**ALSO NOTE**:
Once you have completed something, replace the `[ ]` of the corresponding section with a `[x]`,
this marks it as done!!

### A - Frontend Graphics

- [ ] Set up Three.js renderer, orthographic camera, resize, hemisphere + directional light
- [ ] Steal models from the website poly.pizza
- [ ] Build lane floors from level JSON: grass, asphalt, runway with white dashes
- [ ] Game loop: move obstacles, wrap at edges, speed × delta time
- [ ] Player hop tween (~150 ms, arc on y) (IMPORTANT!), camera follow with some damping / lerp
- [ ] Collision checking: flash/shake, knockback, win animation (optional?)
- [ ] Blob shadows, lane palette, simple fog
- [ ] Expose `game.hop()`, `game.state`, `onRowChange`, `onFinish` to the UI. No DOM code in here required!

### B - Frontend UI
- [ ] Page shell and CSS: menu, HUD, result screen
- [ ] `api.js`: fetch helpers for all endpoints, JSON error handling
- [ ] Menu: airport cards with code, name, stamp, best score; start button; name field
- [ ] Input: arrows/WASD >> `game.hop()`, prevent page scroll, keep focus
- [ ] HUD: battery bar, row progress, airport code, live updates from `game.state`
- [ ] Progress ping on every new grass median; finish POST; `sendBeacon` on unload
- [ ] Result screen: server score, personal best, retry / back to menu
- [ ] Loading and error states (server down, unknown airport); keyboard-only playable

### C - Backend Levels
Maksym, this is a decent bit, so we can hop on a call and work on this tg; though, none of this is
necessarily complex. It's a lot of skeleton work for the Flask stuff though.

- [ ] Flask skeleton, static file serving, SQLite connection
- [ ] Verify course airport table; create `game` table; seed test data
- [ ] `seed_for(icao)` with SHA-256 and generator version salt
- [ ] Generator: 24 rows, medians every 5–7 rows, obstacle gaps that are always passable, start and goal rows
- [ ] Difficulty from airport data (runway count >> runway lanes, size >> speed); sane defaults if fields are missing
- [ ] `GET /api/airports` and `POST /api/games`; validate ICAO, 404 for unknown, JSON errors
- [ ] Commit `sample_level.json` and keep it in sync with the contract
- [ ] `POST /progress`: validate game, atomic `max_row = max(max_row, row)`, ignore stale pings
- [ ] `POST /finish`: plausibility checks (`0 ≤ row < rows`, `hops ≥ max_row`, sane seconds), status, score, completion, idempotent
- [ ] Best score and completion flag for the airport list (coordinate with C)
- [ ] `GET /api/airports/{icao}/scores` top 5 (optional stretch)
- [ ] Global JSON error format and status codes (400/404/409/500); logging