# HTTTX Bot API: LLM Reference

This repository defines API specifications for hexagonal tic-tac-toe bots and engines.
It does not contain a server implementation. A compliant bot is expected to expose HTTP
and/or websocket endpoints that match the YAML definitions in `definitions/`.

## Purpose

Use this doc as a fast reference for an LLM, agent, or integration layer that needs to:

- discover a bot's supported protocols
- send board positions to a bot
- request a move or a position evaluation
- interpret bot responses consistently

## API Structure

There is one required base API and two optional capability APIs:

- Base API: `v1`
- Stateless API: `v1-alpha`
- Basic Websocket API: `v1-alpha`

Every bot must implement the base API and expose:

- `GET /capabilities.json`

The capability response tells the client which optional APIs are supported and where they live.

## Discovery Flow

### 1. Fetch capabilities

Request:

```http
GET /capabilities.json
```

Response shape:

```json
{
  "meta": {
    "name": "Example Bot",
    "description": "Optional human-readable description",
    "author": "optional-author",
    "version": "1.2.3"
  },
  "stateless": {
    "versions": {
      "v1-alpha": {
        "api_root": "stateless/v1-alpha",
        "move_time_limit": true
      }
    }
  },
  "basic_websocket": {
    "versions": {
      "v1-alpha": {
        "api_root": "bws/v1-alpha",
        "move_time_limit": true,
        "evaluation_time_limit": false,
        "config": {
          "dynamic": false
        },
        "free_setup": false,
        "move_skips": false,
        "dual_sided": false,
        "free_move_order": false,
        "evaluation": true,
        "resettable_state": false,
        "interruptable": false
      }
    }
  }
}
```

Notes:

- Only include capabilities the bot fully supports.
- `meta` is optional informational data.
- If `api_root` is omitted, clients should assume:
  - stateless: `stateless/v1-alpha`
  - websocket: `bws/v1-alpha`

## Common Data Model

### Coordinates

The board uses axial hex coordinates:

- `q`: positive goes right
- `r`: positive goes top-right

Example coordinate:

```json
{ "q": 0, "r": 0 }
```

### Pieces

Valid piece values:

- `"x"`
- `"o"`

### Position evaluation

Bots may attach an evaluation object:

```json
{
  "heuristic": 0.42,
  "win_in": 3
}
```

Interpretation:

- `heuristic > 0` favors crosses (`x`)
- `heuristic < 0` favors circles (`o`)
- `win_in > 0` means a predicted win for `x` in N moves
- `win_in < 0` means a predicted win for `o` in N moves

The spec treats one move as one full turn by a side, meaning placement of two same-side pieces.

## Stateless API

This is the simple request/response HTTP mode.

Default root:

```text
stateless/v1-alpha
```

Main route:

```http
POST /turn
```

### Stateless request

```json
{
  "board": {
    "to_move": "x",
    "cells": [
      { "q": 0, "r": 0, "p": "x" },
      { "q": 1, "r": 0, "p": "o" }
    ]
  },
  "time_limit": 2.5
}
```

Rules:

- `board.to_move` is required and must be `"x"` or `"o"`.
- `board.cells` is required and contains all occupied cells.
- `time_limit` is optional.
- If the bot actually supports and respects `time_limit`, it should advertise
  `stateless.versions.v1-alpha.move_time_limit: true` in `capabilities.json`.

### Stateless response

```json
{
  "move": {
    "pieces": [
      { "q": 0, "r": 1 },
      { "q": 1, "r": 1 }
    ],
    "evaluation": {
      "heuristic": 0.31
    }
  },
  "considerations": [
    {
      "pieces": [
        { "q": -1, "r": 0 },
        { "q": -1, "r": 1 }
      ],
      "evaluation": {
        "heuristic": 0.12
      }
    }
  ]
}
```

Rules:

- `move` is required.
- `considerations` is optional and ordered best to worst.
- A move is intended to place exactly two pieces.
- The stateless YAML describes a move as two-piece placement, but does not strictly enforce
  array length in the schema. Clients and bots should still treat a legal move as two pieces.

## Basic Websocket API

This is the stateful, session-based mode.

Default root:

```text
bws/v1-alpha
```

Main route:

```http
POST /game
```

This route upgrades to a websocket connection.

Each connection represents one bot session. The bot may keep internal state for the session
and clear it after disconnect.

## Websocket Session Model

Typical flow:

1. Client opens websocket to `/game`
2. Client may send `config`
3. Client sends `setup`
4. Client sends `move_request` and/or `eval_request`
5. Bot replies with `move_response` and/or `eval_response`
6. Client may continue sending more requests in the same session

## Websocket Packets

### `heartbeat`

Client-to-bot packet:

```json
{
  "type": "heartbeat",
  "waiting": true
}
```

Meaning:

- `waiting: true` means the client is currently expecting the bot to respond.
- If the bot receives `waiting: true` while it is not processing anything and has no pending
  request context, the spec says that is an error state and the bot should terminate the socket.

### `config`

Optional client-to-bot packet:

```json
{
  "type": "config",
  "depth": 4
}
```

Rules:

- Standard fields may be added in future versions.
- Unknown standard fields should be ignored if unsupported.
- Vendor-specific extension fields should start with `x-`.
- A client should not send `x-...` fields unless it knows the bot supports them.

### `setup`

Client-to-bot packet that defines the starting board:

```json
{
  "type": "setup",
  "board": {
    "cells": [
      { "q": 0, "r": 0, "p": "x" }
    ]
  }
}
```

Rules:

- Normally this is sent once before move/eval requests.
- If `resettable_state` is true, setup may be sent again later.
- If `free_setup` is false, the board must contain exactly one cross at `(0, 0)` and no circles.
- The websocket board schema does not include `to_move`; turn context is inferred from requests.

### `move_request`

Client asks the bot to choose a move:

```json
{
  "type": "move_request",
  "side": "x",
  "previous": [
    {
      "side": "o",
      "pieces": [
        { "q": 1, "r": 0 },
        { "q": 1, "r": 1 }
      ]
    }
  ],
  "move_time_limit": 1.5
}
```

Rules:

- `side` is the side the bot is being asked to play for in this request.
- `previous` contains moves made since the last request or setup.
- Unless special capabilities are enabled, clients should follow normal alternating move order.
- Unless `dual_sided` is true, a session should only request one side consistently.
- If `move_skips` is true, the client may report moves it chose itself on behalf of the bot.
- If `free_move_order` is true, move ordering restrictions are relaxed.
- `move_time_limit` is optional and only meaningful if the capability advertises support.

### `move_response`

Bot reply to `move_request`:

```json
{
  "type": "move_response",
  "move": {
    "pieces": [
      { "q": 2, "r": 0 },
      { "q": 2, "r": 1 }
    ],
    "evaluation": {
      "heuristic": 0.67
    }
  },
  "considerations": [
    {
      "pieces": [
        { "q": 3, "r": 0 },
        { "q": 3, "r": 1 }
      ],
      "evaluation": {
        "heuristic": 0.44
      }
    }
  ]
}
```

Rules:

- Must only be sent after a `move_request`.
- `move` is required.
- `considerations` is optional and ordered best to worst.
- In websocket mode, move options explicitly require exactly two coordinates.
- If `move_skips` is true, the bot must not assume its suggested move was actually applied until
  the client reports subsequent moves.

### `eval_request`

Client asks for an evaluation without making a move:

```json
{
  "type": "eval_request",
  "side": "x"
}
```

Rules:

- Only valid if the bot advertises `evaluation: true`.
- Unless `dual_sided` is true, evaluation side should stay consistent with the rest of the session.

### `eval_response`

Bot reply to `eval_request`:

```json
{
  "type": "eval_response",
  "evaluation": {
    "heuristic": -0.22,
    "win_in": -2
  }
}
```

Rules:

- Must only be sent after an `eval_request`.

## Capability Flags

These flags appear under:

```json
basic_websocket.versions["v1-alpha"]
```

### `move_time_limit`

The bot supports and respects move time limits in websocket move requests.

### `evaluation_time_limit`

The bot supports and respects evaluation time limits. The base capability spec mentions this
flag, but the websocket packet schema in this repo does not define an evaluation time-limit field.

### `config.dynamic`

If true, configuration may be changed after turns have already been played.

### `free_setup`

If true, the client may start from arbitrary board states.

### `move_skips`

If true, the client may choose or skip some bot-side moves itself and report them later.

### `dual_sided`

If true, the same session may request moves for either side.

### `free_move_order`

If true, move requests do not need to follow normal alternating order.

### `evaluation`

If true, the bot supports evaluation-only requests.

### `resettable_state`

If true, the client may send a new setup packet after the session has already started.

### `interruptable`

The base capability spec says the client may send an interrupt packet if this is true.
However, no interrupt packet schema is defined in `definitions/basic_websocket/bws-v1-alpha.yaml`.
Treat this as an incomplete part of the spec unless a separate contract exists.

## Important Spec Gaps and Ambiguities

An implementation or integration should account for these issues:

- No authentication scheme is defined.
- No standard error response format is defined.
- No explicit websocket error packet is defined.
- `interruptable` is advertised, but no interrupt packet schema is included.
- `evaluation_time_limit` is advertised, but no eval time-limit request field is defined.
- The stateless move schema intends a two-piece move but does not enforce array length.
- Some descriptions in the websocket YAML contain small wording issues, but the intended behavior
  is still mostly clear from context.

## Recommended Client Behavior

- Always fetch `capabilities.json` first.
- Respect `api_root` if provided.
- Only use optional features if the relevant capability flag is present and true.
- Treat all legal moves as exactly two piece placements.
- Keep websocket session assumptions conservative unless the bot explicitly advertises more
  permissive behavior.
- Be robust to missing optional fields such as `meta`, `considerations`, and `evaluation`.

## Recommended Bot Behavior

- Expose `GET /capabilities.json` as the stable discovery entrypoint.
- Only advertise features that are fully implemented.
- Ignore unsupported non-extension config fields for forward compatibility.
- Return exactly two coordinates for every move.
- If supporting websocket mode, keep clear session state boundaries per connection.
- If supporting time limits, document how strictly they are enforced.

## File Map

- Base capability spec: `definitions/bot-api-v1.yaml`
- Stateless HTTP spec: `definitions/stateless/stateless-v1-alpha.yaml`
- Basic websocket spec: `definitions/basic_websocket/bws-v1-alpha.yaml`

## One-Sentence Summary

This API is a capability-discovered bot protocol for hexagonal tic-tac-toe with one required
discovery endpoint, one simple stateless move endpoint, and one richer stateful websocket session
protocol with optional evaluation and advanced move-handling features.
