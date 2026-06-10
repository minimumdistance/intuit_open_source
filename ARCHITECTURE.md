# Demo Architecture

Multi-agent tic-tac-toe demo: Claude orchestrates, Codex builds and plays, OpenClaw plays from a VM, Gitea tracks code, and a browser shows the live board.

## Component Diagram

```mermaid
graph TB
    subgraph HOST["Host Mac · 192.168.64.1"]
        direction TB

        subgraph CLAUDE_BOX["Claude (Orchestrator / dev3)"]
            Claude["Claude\nOrchestrates game loop\nReviews & merges PRs\nPolls /state & /isthereawinner\nCreates Gitea tags"]
        end

        subgraph CODEX_BOX["Codex (Builder + Player X / dev1)"]
            Codex["Codex\nBuilds server.py\nRuns game server\nPlays as X via POST /play\nPushes branches & tags"]
        end

        subgraph GITEA_BOX["Gitea · localhost:3000 (Docker)"]
            Gitea["dev3/tictactoe repo\nv1.0.0 → v2.0.0 → v3.0.0\nPR review & merge\nTag watch"]
        end

        subgraph GAME_BOX["Game Server · localhost:8888"]
            GameServer["server.py (Python)\nPOST /play\nGET /board\nGET /state\nGET /isthereawinner\nPOST /reset · POST /config\nGET /stop"]
        end

        Browser["Browser\nhttp://localhost:8888\nLive board · player names"]
    end

    subgraph VM["VM · 192.168.64.6"]
        direction TB
        subgraph OC_BOX["OpenClaw (Player O / dev2)"]
            OCAgent["OpenClaw Agent\nReceives cron trigger\nFetches GET /board\nPlays as O via POST /play\nShows board in Telegram"]
        end
        OCCron["openclaw cron\n(scheduler)"]
    end

    Telegram(["Telegram\n8665412823\nBoard previews\nMove confirmations"])

    %% Claude → Codex
    Claude -->|"MCP tool calls\n(approval-policy: never)"| Codex

    %% Claude → OpenClaw via SSH cron
    Claude -->|"SSH + openclaw cron add"| OCCron
    OCCron -->|"triggers agent turn"| OCAgent

    %% OpenClaw → Telegram
    OCAgent -->|"replies with board\n& move result"| Telegram

    %% Codex → Gitea
    Codex -->|"git push\nhttp://dev1:password@localhost:3000"| Gitea

    %% Claude → Gitea API
    Claude -->|"Gitea API\nwatch tags · create PR\nreview diff · approve\nmerge · tag"| Gitea

    %% Codex builds & runs game server
    Codex -->|"builds & nohup runs\nos.setsid() + allow_reuse_address"| GameServer

    %% Codex plays
    Codex -->|"POST /play\nlocalhost:8888"| GameServer

    %% Claude polls
    Claude -->|"GET /state\nGET /isthereawinner"| GameServer

    %% OpenClaw plays from VM
    OCAgent -->|"GET /board\nPOST /play\n192.168.64.1:8888"| GameServer

    %% Game server → browser
    GameServer -->|"GET /\npolls /state every 1.5s"| Browser

    %% Styling
    classDef orchestrator fill:#1a4a6b,color:#fff,stroke:#0d2d42
    classDef builder fill:#1a5c3a,color:#fff,stroke:#0d3d24
    classDef infra fill:#4a3a00,color:#fff,stroke:#2d2400
    classDef game fill:#4a1a4a,color:#fff,stroke:#2d0d2d
    classDef oc fill:#6b1a1a,color:#fff,stroke:#420d0d
    classDef external fill:#555,color:#fff,stroke:#333

    class Claude orchestrator
    class Codex builder
    class Gitea infra
    class GameServer game
    class OCAgent,OCCron oc
    class Browser,Telegram external
```

## Components

| Component | Role | Gitea user | Location |
|-----------|------|------------|----------|
| Claude | Orchestrator — runs the game loop, reviews PRs, merges, tags | dev3 (admin) | Host Mac |
| Codex | Builder + Player X — writes `server.py`, runs the server, plays via HTTP | dev1 (admin) | Host Mac |
| OpenClaw | Player O — receives turns via SSH cron, fetches board, plays via HTTP | dev2 | VM `192.168.64.6` |
| Gitea | Git server + PR platform | — | Host Mac (Docker, port 3000) |
| Game Server | HTTP API for game state | — | Host Mac (port 8888) |
| Browser | Live board viewer | — | Host Mac |
| Telegram | Async reply channel from OpenClaw | — | External |

## Communication Channels

### Claude → Codex
- **Protocol**: MCP tool calls (`mcp__codex__codex` / `mcp__codex__codex-reply`)
- **Config**: `approval-policy: never`, `sandbox: danger-full-access`
- **Session**: one `threadId` per demo run; all turns use `codex-reply` to continue the same session

### Claude → OpenClaw
- **Protocol**: SSH into VM → `openclaw cron add --at "+1s"` → agent turn fires in ~1s
- **Command format**:
  ```bash
  ssh ronaldpetty@192.168.64.6 'zsh --login -c '\''/opt/homebrew/bin/openclaw cron add \
    --name "<name>" --at "+1s" \
    --message "<instructions>" \
    --session isolated --agent main --to 8665412823 --delete-after-run'\'''
  ```
- **OpenClaw reply**: async via Telegram to `8665412823`

### OpenClaw → Game Server
- **Protocol**: HTTP from VM using host IP `192.168.64.1:8888` (not `localhost`)
- **Each turn**: `GET /board` first (shows ASCII board in Telegram), then `POST /play`

### Codex → Game Server
- **Protocol**: HTTP via `localhost:8888`
- **Each turn**: `POST /play`

### Claude → Game Server
- **Protocol**: HTTP via `localhost:8888`
- **Role**: polls `/state` and `/isthereawinner` after every move to detect end of game

### Codex → Gitea
- **Protocol**: `git push http://dev1:password@localhost:3000/dev3/tictactoe.git`
- **Clone path**: `~/tictactoe-codex/tictactoe`

### OpenClaw → Gitea
- **Protocol**: `git clone http://dev2:password@192.168.64.1:3000/dev3/tictactoe.git`
- **Clone path**: `~/tictactoe-openclaw/tictactoe` on the VM

### Claude → Gitea API
- **Protocol**: REST API using DEV1_TOKEN (open PR) and DEV3_TOKEN (review, approve, merge, tag)
- **Endpoints used**: `POST /pulls`, `GET /pulls/{n}.diff`, `POST /pulls/{n}/reviews`, `POST /pulls/{n}/merge`, `POST /tags`

## Game Server API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/play` | POST | `{"player":"codex","piece":"X","place":4}` — place a piece |
| `/board` | GET | ASCII board for human display |
| `/state` | GET | JSON: `{board, turn, winner, draw, players}` |
| `/isthereawinner` | GET | JSON: `{winner, draw}` |
| `/reset` | POST | Clear board |
| `/config` | POST | `{"players":{"X":"Codex","O":"OpenClaw"}}` — set player names |
| `/stop` | GET | Graceful shutdown |

## Phases

| Version | Branch | Feature | PR |
|---------|--------|---------|----|
| v1.0.0 | `main` | Core game server — all endpoints, CLI play | — |
| v2.0.0 | `feature/web-viewer` | `GET /` browser viewer, auto-polls `/state` every 1.5s | PR #1 |
| v3.0.0 | `feature/named-players` | `POST /config` + `players` in `/state` + named display in browser | PR #2 |

## Key Implementation Notes

- **Server stability**: `server.py` calls `os.setsid()` at startup and uses `allow_reuse_address = True` so the process survives after the launching shell exits.
- **OpenClaw polling**: After sending a cron job, Claude sleeps 8–10s then polls `GET /board` directly to confirm the move registered. If the board is unchanged, the cron job is resent.
- **IP routing**: The game server binds to `0.0.0.0:8888`. Codex reaches it at `localhost:8888`; OpenClaw reaches it at `192.168.64.1:8888` (the host's VM-facing IP).
- **Gitea auth**: All git operations use HTTP basic auth (`username:password` in the URL). API calls use bearer tokens generated via `gitea admin user generate-access-token --raw`.
