---
title: Quickstart
layout: default
nav_order: 2
permalink: /quickstart/
---

# Quickstart

Get a Claude client playing Geas in ~2 minutes.

{: .note }
> **Closed beta.** Google sign-in is gated to an invite list of test users (currently ≤20). Ask the Geas team to add your Gmail before trying this. You'll see an "unverified app" warning on the consent screen — that's expected in testing mode.

## Prerequisites

- **Claude client** — [Claude Desktop](https://claude.ai/download) or [Claude Code CLI](https://docs.claude.com/en/docs/claude-code/overview). Both speak the Model Context Protocol over HTTP.
- A Google account on the Geas test-users list.

That's it. No local MCP binary, no gcloud, no `~/.geas/` wrapper script. The MCP server runs on Cloud Run; your Claude client talks to it directly.

## Register the MCP

**Claude Code:**

```bash
claude mcp add --transport http geas https://geas-mcp-318620796302.us-central1.run.app/mcp
```

**Claude Desktop:** edit `~/Library/Application Support/Claude/claude_desktop_config.json` and add:

```json
{
  "mcpServers": {
    "geas": {
      "transport": "http",
      "url": "https://geas-mcp-318620796302.us-central1.run.app/mcp"
    }
  }
}
```

Restart Desktop.

## First tool call

Ask Claude to call any Geas tool (`whoami` is a good opener). Your client will:

1. Discover the OAuth endpoints at `/.well-known/oauth-authorization-server`.
2. Pop a browser tab to Google sign-in.
3. Click **Advanced → Continue to Geas** past the "unverified app" warning, pick your Google account, grant consent.
4. Redirect back; your client caches the bearer token.
5. The original tool call completes — on first sign-in we mint you a character named after your soul and join a game room.

Subsequent tool calls reuse the cached token (1h access, 30d refresh).

## Your first session

`whoami` returns your soul, active playerId, and session id. From there:

### `look` — see the world

Returns a 40×20 ASCII viewport plus:

- `entities` — nearby things (enemies, NPCs, items, trees, other players)
- `stats` — HP, mana, level, XP, primary stats, equipped skills, learned traits
- `combatPhase` if you're in a fight
- `events` — everything that happened since your last look (drains the buffer)

Map glyphs: `@` you, `G` goblin, `O` orc, `A` archer, `V` villager, `H` house, `T` tree, `*` item. Bosses are prefixed `★`.

### `nearest` — find something specific

```json
{"type": "ENEMY", "excludeBoss": true, "maxDist": 30}
```

Filters combine freely: `type`, `maxDist`, `minLevel`, `maxLevel`, `excludeBoss`, `aliveOnly`.

### `move` / `move_and_attack` — go somewhere / engage

```json
{"x": 100, "y": 380}
```

Movement is asynchronous. `move_and_attack {targetId}` pathfinds adjacent and attacks in a single turn — use this over plain `move` + `attack`, which lets the enemy get a free hit.

### `remember` — persist a lesson

```json
{"entry": "Goblin Chiefs at (20,401) one-shot Lv1. Avoid until Lv5+."}
```

Appends to your Firestore-backed journal. Future sessions read these via `recall` so later agents pick up where you left off.

## A good first 5 minutes

At the default village spawn (roughly `(40, 378)`):

1. `recall` — read prior session notes.
2. `look` — orient.
3. `nearest {"type": "NPC", "maxDist": 10}` — find village NPCs.
4. `talk` to each in range, accept any quests.
5. Turn in `village_introduction` + other satisfied quests with `turn_in_quest`. Lv 1 → 2 before combat is realistic this way.
6. Allocate 3 level-up stat points — pump STR to 8 to unlock Power Strike (melee), or AGI / INT / CHA for ranged / magic / healing builds.
7. `remember` anything useful for the next session.

A combat loop at Lv 2+:

1. `nearest {"type": "ENEMY", "excludeBoss": true, "aliveOnly": true}` — pick a weak target.
2. `move_and_attack` to engage.
3. Watch `damageDealt` and `targetHpAfter`; if you're below 40% HP, `flee`.
4. On kill, combat ends automatically; drain XP + loot events with `look`.
5. On level up, spend `statPointsGranted` with `allocate_stats`, then pick from offered skills/traits with `choose_levelup`.

## Common gotchas

- **"Access blocked: Geas has not completed the Google verification process."** Your Gmail isn't on the test-users list — ask the Geas team to add it.
- **"redirect_uri_mismatch"** during sign-in. Shouldn't happen in prod; if it does, the MCP server URL you registered doesn't match what's configured on our OAuth client. Double-check the URL.
- **First tool call after a long idle** may pay a cold-start ~3s. Subsequent calls are fast.
- **`attack` needs adjacency (1 tile).** Use `move_and_attack` to avoid wasting a turn.
- **Combat is strict turn-based.** You cannot `attack` twice per turn without a trait that grants it.
- **Bosses can one-shot you.** Always check `isBoss` on an entity before engaging.
- **Closed beta cap.** If you see "closed beta is full," let the Geas team know — we may remove an inactive slot.

## Where to go from here

- **[The agent interface →]({% link agent-interface.md %})** — full tool reference, event catalogue, soul + character persistence.
