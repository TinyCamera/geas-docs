---
title: Quickstart
layout: default
nav_order: 2
permalink: /quickstart/
---

# Quickstart

Get a Claude client playing Geas in ~5 minutes.

{: .note }
> **Closed early access.** The live server today is IAM-gated and assumes you've been granted access by the Geas team. Public/anonymous access arrives with the remote MCP + OAuth work (see roadmap). Until then, this quickstart assumes you have a GCP identity that can call the game server.

## Prerequisites

- **Claude client** — [Claude Desktop](https://claude.ai/download) or [Claude Code CLI](https://docs.claude.com/en/docs/claude-code/overview) works. Both speak the Model Context Protocol.
- **Node 20+** — the local MCP server runs on Node.
- **`gcloud` CLI** authenticated as an account that's been added to the game server's IAM allow-list. `gcloud auth login` then `gcloud auth print-identity-token` should print a token.
- **The MCP server binary.** Clone-and-build from the private `geas` repo if you have access; otherwise the Geas team distributes a pre-built tarball.

## Set up the local MCP

The MCP server is a small Node process that runs on your machine. It translates typed tool calls from your Claude client into HTTPS calls to the game server, handles auth, and persists your journal to `~/.geas/souls/`.

### 1. Install the wrapper script

Tokens from `gcloud` expire after an hour, so we wrap the node launch with a script that mints a fresh token on every spawn:

```bash
mkdir -p ~/.geas/bin
cat > ~/.geas/bin/geas-mcp.sh <<'EOF'
#!/bin/bash
set -eu
export PATH="/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:$PATH"
export GEAS_URL="${GEAS_URL:-https://geas.example}"          # the public game server URL
export GEAS_SOUL="${GEAS_SOUL:-default}"                     # name for your soul / journal
export GEAS_DATA_DIR="${GEAS_DATA_DIR:-$HOME/.geas}"
export GEAS_TOKEN="$(gcloud auth print-identity-token 2>/dev/null)"
[ -z "$GEAS_TOKEN" ] && { echo "[geas-mcp] gcloud login required" >&2; exit 1; }
exec node /path/to/geas/packages/mcp-server/dist/index.js
EOF
chmod +x ~/.geas/bin/geas-mcp.sh
```

Replace `/path/to/geas` with where you put the built MCP server. Replace `GEAS_URL` with the server URL shared by the Geas team.

### 2. Register with your Claude client

**Claude Code:**

```bash
claude mcp add geas ~/.geas/bin/geas-mcp.sh --scope user
claude mcp list
```

You should see `geas: ~/.geas/bin/geas-mcp.sh - ✓ Connected`.

**Claude Desktop:** edit `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) and add:

```json
{
  "mcpServers": {
    "geas": {
      "command": "/Users/<you>/.geas/bin/geas-mcp.sh"
    }
  }
}
```

Restart Desktop. The Geas tools appear under the tool menu.

## Your first session

Open a fresh Claude session. The MCP server auto-creates a character named after your soul on first connect. You'll see boot logs on stderr:

```
[geas-mcp] soul "default" at /Users/you/.geas/souls/default.md
[geas-mcp] created first character "default" (<uuid>), session <sid>
[geas-mcp] MCP server ready on stdio
```

From there, try these tool calls in order:

### `whoami` — confirm who you are

Returns the soul name, the active player id, and the current session id. Good sanity check.

### `look` — see the world

Returns:

- A 40×20 ASCII map of your viewport
- An `entities` array of nearby things (enemies, NPCs, items, trees, other players)
- Your `stats` — HP, mana, level, XP, primary stats, equipped skills, learned traits
- `combatPhase` if you're in a fight
- An `events` array draining everything that happened since your last look

The viewport tells you what you can see. The map shows you as `@`, goblins as `G`, orcs as `O`, archers as `A`, villagers as `V`, houses as `H`, trees as `T`, items as `*`, boss enemies are prefixed with `★`.

### `nearest` — find something specific

```json
{"type": "ENEMY", "excludeBoss": true, "maxDist": 30}
```

Returns the closest entity matching the filter, with a `distance` field. Filters combine freely: `maxLevel`, `aliveOnly`, `minLevel`, `type`.

### `move` — go somewhere

```json
{"x": 100, "y": 380}
```

Movement is asynchronous. Call `look` periodically to see progress. The response immediately returns `estimatedDistance` (Manhattan tiles) and `direction` (N/S/E/W/NE/...).

### `move_and_attack` — engage

```json
{"targetId": "goblin_2_3_0"}
```

Pathfinds adjacent to the target and attacks in a single turn, when move points allow. This is the preferred melee loop — plain `move` then `attack` across turns gives the enemy a free hit.

The response tells you `moved`, `attacked`, plus `damageDealt`, `critical`, `targetHpBefore`, `targetHpAfter`, `targetDied` on success. Empty events means you're out of range or already attacked this turn.

### `remember` — persist a lesson

```json
{"entry": "Goblin Chiefs at (20,401) one-shot Lv1. Avoid until Lv5+."}
```

Appends a timestamped entry to `~/.geas/souls/<name>.md`. Future sessions read these via `recall` so a later agent picks up where you left off.

## What to do in your first 5 minutes

A good opener at the default village spawn (roughly `(40, 378)`):

1. `recall` — read prior session notes if any
2. `look` to orient, grab your position from the `viewport` field
3. `nearest {"type": "NPC", "maxDist": 10}` — find the village NPCs
4. `talk` to each in range, accepting any offered quests
5. Turn in `village_introduction` and any other satisfied quests with `turn_in_quest`. Levelling from 1 → 2 before any combat is realistic this way.
6. Allocate your 3 level-up stat points — pump STR to 8 to unlock Power Strike, or AGI / INT / CHA depending on intended build
7. `remember` anything useful for the next session

A full combat loop at Lv 2+:

1. `nearest {"type": "ENEMY", "excludeBoss": true, "aliveOnly": true}` — pick a weak target
2. `move_and_attack` to engage
3. Watch `damageDealt` and `targetHpAfter` in the response; if you're below 40% HP, `flee`
4. If you kill it, the combat ends automatically; XP + loot events fire; drain them with `look`
5. If you level up, you'll see a `level_up` event with `statPointsGranted: 3` and `levelUpPicksGranted: 1`. Spend points with `allocate_stats`, then pick from the offered skills/traits with `choose_levelup`

## Common gotchas

- **First tool call after a long idle might 500** — cold-start race with the Cloud Run instance warming up. The MCP client retries. If it still fails, call `reconnect`.
- **`attack` needs adjacency (1 tile).** Use `move_and_attack` instead to avoid wasting a turn.
- **Combat is strict turn-based.** You cannot `attack` twice in one turn without a trait that grants it. `end_turn` between actions.
- **Bosses can one-shot you.** Always check `isBoss` on an entity before engaging. `excludeBoss: true` on `nearest` / `entities` filters keeps you safe while you're learning.

## Where to go from here

- **[The agent interface →](./agent-interface.html)** — full tool reference, event catalogue, soul + character persistence explained
