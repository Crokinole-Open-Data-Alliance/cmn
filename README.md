# Crokinole Match Notation (CMN)

| | |
|---|---|
| **Version** | 1.0-draft |
| **Status** | Proposal |
| **Authors** | Crokinole Open Data Alliance (CODA) |

---

## Overview

Crokinole Match Notation (CMN) is a standard JSON format for recording crokinole match results. It is designed to be produced and consumed by any crokinole application, regardless of tech stack or internal data model.

CMN captures **results, not process**. It does not describe live game state, claim status, conflict resolution, or bracket advancement. It records what happened after a match is complete.

---

## Format

A CMN file is a JSON object with the following top-level structure:

```json
{
  "cmn": "1.0",
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "match": { ... },
  "event": { ... },
  "source": { ... }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `cmn` | string | Yes | Spec version. Always `"1.0"` for this version. |
| `id` | string | Yes | Unique identifier for this record. UUID v4 recommended. |
| `match` | object | Yes | The match data. See [Match Object](#match-object). |
| `event` | object | No | Context about the event this match was part of. See [Event Object](#event-object). |
| `source` | object | No | Which application produced this record. See [Source Object](#source-object). |

---

## Match Object

```json
{
  "date": "2026-02-11T19:30:00Z",
  "format": "singles",
  "teams": [
    {
      "players": [
        { "name": "Jeff Decker", "cid": "CRK-00142" }
      ]
    },
    {
      "players": [
        { "name": "Maria Santos", "cid": "CRK-00087" }
      ]
    }
  ],
  "gameFormat": {
    "type": "fixed",
    "count": 4
  },
  "games": [
    { "winner": 0, "hammer": 0, "scores": [20, 5], "twenties": [2, 0] },
    { "winner": 1, "hammer": 1, "scores": [15, 20], "twenties": [0, 1] },
    { "winner": 0, "hammer": 0, "scores": [20, 10], "twenties": [1, 0] },
    { "winner": 0, "hammer": 1, "scores": [15, 5], "twenties": [0, 0] }
  ],
  "winner": 0
}
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `date` | string | Yes | ISO 8601 datetime in UTC. Must end with `Z`. |
| `format` | string | Yes | `"singles"` or `"doubles"`. |
| `teams` | array | Yes | Exactly 2 team objects. Index 0 = team 1, index 1 = team 2. |
| `gameFormat` | object | No | How the match length was determined. See [Game Format](#game-format). |
| `games` | array | Yes | One entry per game played. See [Game Object](#game-object). |
| `winner` | integer or null | Yes | `0` = team 1 won, `1` = team 2 won, `null` = draw. |

### Team Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `players` | array | Yes | 1 player for singles, 2 for doubles. |

### Player Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Display name. |
| `cid` | string | No | Crokinole Player ID, if the player has linked one. |

### Game Format

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | `"fixed"` (play exactly N games) or `"first_to"` (first to N points). |
| `count` | integer | Conditional | Number of games. Required when `type` is `"fixed"`. |
| `target` | integer | Conditional | Points target. Required when `type` is `"first_to"`. |

### Game Object

Each entry in `games` represents one completed game (a single round of play where 2 points are at stake).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `winner` | integer or null | Yes | `0` = team 1 won, `1` = team 2 won, `null` = tie game. |
| `hammer` | integer | No | Which team shot first (had the hammer). `0` = team 1, `1` = team 2. |
| `scores` | array | No | Two-element array `[team1Score, team2Score]`. Raw (pre-cancellation) point totals for the game. |
| `twenties` | array | No | Two-element array `[team1Twenties, team2Twenties]`. Count of 20s (center hole shots) per team. |

---

## Event Object

Optional context about the event this match took place in.

```json
{
  "name": "Tuesday Night League - Week 4",
  "type": "league",
  "round": 2,
  "venue": "Kitchener Crokinole Club",
  "location": { "country": "CA", "region": "Ontario" }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | No | Human-readable event name. |
| `type` | string | No | Free-form event type. Common values: `"casual"`, `"league"`, `"tournament"`, `"ranked"`, `"club"`, `"exhibition"`, `"practice"`. Not restricted to a fixed list. |
| `round` | integer | No | Round/week number within the event, if applicable. |
| `venue` | string | No | Name of the venue where the match was played. |
| `location` | object | No | Geographic location. See [Location Object](#location-object). |

### Location Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `country` | string | No | ISO 3166-1 alpha-2 country code (e.g., `"CA"`, `"US"`, `"ES"`). |
| `region` | string | No | State, province, or region (e.g., `"Ontario"`, `"Catalonia"`). |
| `lat` | number | No | Latitude. |
| `lng` | number | No | Longitude. |

Apps may include additional fields in `event` for their own use. Consumers should ignore fields they don't recognize.

---

## Source Object

Identifies which application produced this record.

```json
{
  "app": "croke",
  "version": "1.5.0",
  "url": "https://croke.app"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `app` | string | Yes | Application identifier. Lowercase, no spaces. |
| `version` | string | No | Application version that generated this record. |
| `url` | string | No | Application URL. |

---

## Scoring Note

When `scores` are included, they represent **raw (pre-cancellation) point totals** — e.g., `[55, 40]` means team 1 scored 55 total and team 2 scored 40 total, before any subtraction. This is the most flexible representation and can be converted to differential form by consumers if needed.

Not all apps track detailed scores. CMN accommodates varying levels of detail:

- **Full scores known:** Include `scores` with raw point totals, and `winner`.
- **Only winner known:** Omit `scores`, include `winner`.
- **Twenties tracked:** Include `twenties`. If not tracked, omit the field.
- **Hammer tracked:** Include `hammer`. If not tracked, omit the field.

Consumers should always be prepared for `scores`, `twenties`, and `hammer` to be absent. The only guaranteed field per game is `winner`.

---

## Batch Format

Multiple matches can be grouped in a batch (e.g., all matches from a league night or tournament):

```json
{
  "cmn": "1.0",
  "batch": true,
  "event": { ... },
  "source": { ... },
  "matches": [
    { "id": "...", "match": { ... } },
    { "id": "...", "match": { ... } }
  ]
}
```

When `batch` is `true`, the top-level `event` and `source` apply to all matches unless overridden at the individual match level. Each entry in `matches` contains an `id` and a `match` object (same structure as the single-match format).

---

## Full Example: Singles League Match

```json
{
  "cmn": "1.0",
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "match": {
    "date": "2026-02-11T19:30:00Z",
    "format": "singles",
    "teams": [
      { "players": [{ "name": "Jeff Decker", "cid": "CRK-00142" }] },
      { "players": [{ "name": "Maria Santos", "cid": "CRK-00087" }] }
    ],
    "gameFormat": { "type": "fixed", "count": 4 },
    "games": [
      { "winner": 0, "scores": [20, 5], "twenties": [2, 0] },
      { "winner": 1, "scores": [15, 20], "twenties": [0, 1] },
      { "winner": 0, "scores": [20, 10], "twenties": [1, 0] },
      { "winner": null, "scores": [10, 10], "twenties": [0, 0] }
    ],
    "winner": 0
  },
  "event": {
    "name": "Kitchener Tuesday League",
    "type": "league",
    "round": 4,
    "venue": "Kitchener Community Centre"
  },
  "source": {
    "app": "croke",
    "version": "1.5.0",
    "url": "https://croke.app"
  }
}
```

## Full Example: Doubles Tournament Match

```json
{
  "cmn": "1.0",
  "id": "f7e6d5c4-b3a2-1098-fedc-ba9876543210",
  "match": {
    "date": "2026-03-08T14:00:00Z",
    "format": "doubles",
    "teams": [
      {
        "players": [
          { "name": "Jeff Decker", "cid": "CRK-00142" },
          { "name": "Alex Kim" }
        ]
      },
      {
        "players": [
          { "name": "Maria Santos", "cid": "CRK-00087" },
          { "name": "Liam Chen", "cid": "CRK-00201" }
        ]
      }
    ],
    "gameFormat": { "type": "first_to", "target": 5 },
    "games": [
      { "winner": 0, "twenties": [1, 0] },
      { "winner": 0, "twenties": [0, 0] },
      { "winner": 1, "twenties": [0, 2] },
      { "winner": 0, "twenties": [1, 0] },
      { "winner": 1, "twenties": [0, 0] },
      { "winner": 1, "twenties": [0, 1] },
      { "winner": 0, "twenties": [2, 0] },
      { "winner": 0, "twenties": [0, 0] }
    ],
    "winner": 0
  },
  "event": {
    "name": "Tavistock Doubles Championship",
    "type": "tournament",
    "venue": "Tavistock Arena"
  },
  "source": {
    "app": "scorekinole",
    "version": "2.1.0",
    "url": "https://scorekinole.web.app"
  }
}
```

## Full Example: Casual Match (Minimal)

The minimum viable CMN record — no event context, no CIDs, no scores, no twenties:

```json
{
  "cmn": "1.0",
  "id": "11223344-5566-7788-99aa-bbccddeeff00",
  "match": {
    "date": "2026-02-15T20:00:00Z",
    "format": "singles",
    "teams": [
      { "players": [{ "name": "Nick" }] },
      { "players": [{ "name": "Xavi" }] }
    ],
    "games": [
      { "winner": 0 },
      { "winner": 0 },
      { "winner": 1 },
      { "winner": 0 }
    ],
    "winner": 0
  }
}
```

---

## Design Principles

1. **Results, not process.** CMN records what happened, not how it was managed. No live state, no conflict resolution, no bracket logic.
2. **Minimal required fields.** Only `date`, `format`, `teams` (with names), `games` (with winners), and `winner` are required. Everything else is optional enrichment.
3. **Extensible.** Apps can add custom fields to any object. Consumers must ignore fields they don't recognize. This allows experimentation without breaking compatibility.
4. **No app lock-in.** A CMN file contains everything needed to understand the match. No external IDs or references are required (CIDs are optional).
5. **Human-readable.** JSON with clear field names. A person should be able to read a CMN file and understand what happened.

---

## Open Questions for Discussion

- [ ] Is the points system (2 for win, 1 for tie, 0 for loss) universal enough to assume, or should it be configurable per-match?
- [ ] Should batch format support mixed events (matches from different events in one file)?
