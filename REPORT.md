# Demo Report: Multi-Agent Tic-Tac-Toe

## Overview

Three agents (Claude, Codex, OpenClaw) collaboratively built and played tic-tac-toe across three progressive releases. Each phase added a feature via a real software delivery workflow: spec → build → self-test → commit → tag → PR → review → merge → deploy → play.

---

## Phase I — CLI Game Server (v1.0.0)

**What happened:**
- Claude provisioned Gitea (Docker Compose), created three users with role-based access (dev1=Codex, dev2=OpenClaw, dev3=Claude/operator), and seeded the repo
- Both agents cloned the repo into isolated directories on their respective machines
- Claude gave Codex a written spec; Codex implemented `server.py` from scratch
- Codex self-tested by playing a solo game (both X and O via HTTP) to verify win detection before committing
- Claude watched Gitea for the `v1.0.0` tag and deployed that exact artifact
- Claude orchestrated a full Codex vs OpenClaw game via HTTP, polling `/isthereawinner` after each move

**Software practices:**
| Practice | How it appeared |
|----------|----------------|
| Infrastructure as Code | Gitea environment defined entirely in `docker-compose.yml` |
| Role-based access control | Separate Gitea users per agent; dev3 (operator) holds merge rights |
| Spec-driven development | Claude wrote the full API contract before Codex wrote a line |
| Ad-hoc / smoke testing | Codex self-played a solo game to verify all endpoints before committing |
| Immutable releases | Game played from the tagged artifact `v1.0.0`, not from `main` |
| GitOps | Claude polled the Gitea tags API; deployment triggered by tag presence |
| API-driven integration | All gameplay via HTTP POST/GET — no direct file or process access between agents |
| Multi-agent orchestration | Claude coordinated turn order, retry logic, and end-of-game detection across two agents on different machines |

---

## Phase II — Browser Viewer (v2.0.0)

**What happened:**
- Codex opened a feature branch (`feature/web-viewer`) and added `GET /` serving a live HTML board that auto-polls `/state` every 1.5s
- Codex self-tested locally before pushing
- Claude created a PR via the Gitea API using Codex's token (dev1), then fetched the diff and reviewed it as operator (dev3)
- Claude approved, merged, and tagged `v2.0.0` via the Gitea API
- Codex pulled the tag, started the server, opened the browser
- Claude and OpenClaw played a full game; audience could watch pieces appear in the browser in real time

**Software practices:**
| Practice | How it appeared |
|----------|----------------|
| Feature branch workflow | All changes on an isolated branch; `main` only updated via PR |
| Pull request code review | Claude read the `.diff` and made an explicit approve/reject decision before merging |
| Progressive enhancement | Browser viewer added without breaking any existing CLI endpoints |
| Self-test before PR | Codex verified the page loaded and `/state` polling worked locally before opening the PR |
| GitOps | Deployment triggered by `v2.0.0` tag; Codex checked out that exact tag to run |
| Continuous delivery | Build → test → PR → review → merge → tag → deploy in a single uninterrupted flow |
| Observability | Live browser board gave a real-time view of game state to any audience member |

---

## Phase III — Named Players + Telegram Board Preview (v3.0.0)

**What happened:**
- Codex opened `feature/named-players`, added `POST /config` to register player names, updated `GET /state` to include a `players` field, and updated the browser page to show "Codex (X)'s turn" / "OpenClaw (O)'s turn"
- Before every OpenClaw move, the cron message instructed OpenClaw to `GET /board` and display the ASCII board in Telegram, then make its move, then reply with the updated board
- Same PR flow: Codex pushes → Claude creates PR (DEV1_TOKEN) → Claude reviews diff (DEV3_TOKEN) → approves → merges → tags `v3.0.0`
- Codex called `POST /config` at server startup to register player names before the game began

**Software practices:**
| Practice | How it appeared |
|----------|----------------|
| API-first configuration | Player names injected via `POST /config` at runtime, not hardcoded |
| Feature branch + PR review | Same pattern as Phase II; Claude reviewed the diff independently |
| Separation of concerns | Player name display logic kept in the client (JS); server only stores/exposes the mapping |
| Agent observability | OpenClaw fetched and surfaced board state in Telegram before acting — making each decision step visible |
| Idempotent deployment | `POST /reset` + `POST /config` at startup ensures a clean, reproducible game state regardless of prior runs |
| Versioned artifact deployment | `v3.0.0` tag checked out explicitly; no "latest" or floating refs |

---

## Step Estimates

### Steps to Create (Gitea setup through all three deployments, excluding gameplay)

| Activity | Estimated Steps |
|----------|----------------|
| Clean slate (kill server, tear down Docker, wipe dirs on host + VM) | 3 |
| Gitea: write compose file, start, wait for ready | 3 |
| Create users dev1, dev2, dev3 + generate tokens | 6 |
| Create tictactoe repo via API | 1 |
| Clone repo: Codex (MCP) + OpenClaw (SSH cron) | 2 |
| **Subtotal — environment setup** | **15** |
| Phase I: build server.py, self-test (start, solo game, verify winner, reset, stop) | 8 |
| Phase I: commit, tag v1.0.0, push branch + tag, confirm on Gitea | 5 |
| Phase I: deploy (checkout tag, start server, reset, notify OpenClaw) | 4 |
| **Subtotal — Phase I create** | **17** |
| Phase II: branch, build GET /, self-test, commit, push | 5 |
| Phase II: create PR, fetch diff, review, approve, merge, tag v2.0.0 | 6 |
| Phase II: deploy (checkout tag, start server, reset, open browser) | 4 |
| **Subtotal — Phase II create** | **15** |
| Phase III: branch, add /config + players in /state + browser names, self-test, commit, push | 6 |
| Phase III: create PR, fetch diff, review, approve, merge, tag v3.0.0 | 6 |
| Phase III: deploy (checkout tag, start server, reset, POST /config, open browser) | 5 |
| **Subtotal — Phase III create** | **17** |
| **Total — create** | **~64** |

### Steps to Play (all three games)

| Activity | Estimated Steps |
|----------|----------------|
| Per Codex turn (MCP call to POST /play) | 1 |
| Per OpenClaw turn (SSH cron + poll to confirm move registered) | 2 |
| Per winner check (GET /isthereawinner) | 1 |
| End of game: announce result + stop server (or wait 20s + stop) | 2 |
| Each game (5 Codex turns + 4 OpenClaw turns + winner checks + close) | ~16 |
| Phase I game | 16 |
| Phase II game | 16 |
| Phase III game | 16 |
| **Total — play** | **~48** |

---

## Grand Total

| Category | Steps |
|----------|-------|
| Create (setup + build + deploy across all phases) | ~64 |
| Play (three full games) | ~48 |
| **Grand total** | **~112** |

> A human developer team following the same workflow — Gitea setup, writing the server, opening PRs, reviewing code, deploying tagged releases, and playing three games — would likely take several hours to a full day. The agent pipeline completed the entire sequence autonomously in under 45 minutes.
