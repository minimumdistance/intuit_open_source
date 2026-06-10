# Goal 

* Create a reproducable demo of agents and humans building and playing tic-tac-toe at different speeds
* The speed component is to simulate how humans might create such a game and play and compare that to a system of how agents only create and play

## Design

* Game board will be a simple text file representation of the game

```
    |   |
-------------
    |   |
-------------
    |   |    
```

* Pieces are simply capital letters X and O

* Game uses ordinal coordinates for piece placement
* During a game, users cannot overwrite each other

```
  0 | 1 | 2
-------------
  3 | 4 | 5
-------------
  6 | 7 | 8   
```

* Turn based between two players
* Update file representing game with play
* Return updated board to each player
* if three pieces of same are in row, col, or diagonal the game is over and that player wins
* updates to game.txt are done simply by editing file one player at a time
* Game can be viewed by network connection to game server to POST play commands (e.g. {player: codex, piece: "X", place: 3})


## General Requirements:

* Claude is the orchestrator and overall admin
* Codex will implement the game via Claude instructions
* At play time, Codex and OpenClaw will play each other
* Claude will use Codex via MCP to build the game 
* Claude will use Codex via MCP to run the game as a webserver

* Because OpenClaw is running in a VM, Claude can't use it like Codex, however we can execute things via SSH to its VM
* The SSH key has already been configured, so Claude must use SSH when asking OpenClaw to do something, more specific, we want OpenClaw to be triggered via CLI as this example shows:

```
# this doesn't work
# this only sends to telegram, not telegram to OC - in other words this is outbound channel from OC, not in bound
ssh ronaldpetty@192.168.64.6 /opt/homebrew/bin/openclaw message send --channel telegram --target 8665412823 --message \"your message here \$RANDOM\"
```

```
# this does work

# Approve pending scope upgrade (run on VM until no more requests)
openclaw devices approve --latest

ssh ronaldpetty@192.168.64.6 'zsh --login -c '\''/opt/homebrew/bin/openclaw cron add \
  --name "demo-curl" \
  --at "+1s" \
  --message "run curl http://192.168.64.1" \
  --session isolated \
  --agent main \
  --to 8665412823 \
  --delete-after-run'\'''
```

## Orchestration Notes (learned from live run)

* **Always poll the game server API** to check board state — do not wait for the user to paste output. Use `curl -s http://localhost:8888/state | jq .` after each move.
* **OpenClaw move registration**: After sending OpenClaw a cron job, poll the server after ~8 seconds to confirm the move registered (check that the board changed). If it did not change, resend with a more explicit curl command including the exact position.
* **Host IP from VM**: OpenClaw accesses the game server and Gitea on the host via `192.168.64.1` (not localhost). Always use this IP in commands sent to OpenClaw.
* **Unique clone directories**: Codex clones to `~/tictactoe-codex/tictactoe`, OpenClaw clones to `~/tictactoe-openclaw/tictactoe` on the VM, so they don't interfere.
* **Gitea uses login/password**, not SSH keys, for git operations (e.g. `http://dev1:password@localhost:3000/dev3/tictactoe.git`).
* **User roles**: dev1 = Codex, dev2 = OpenClaw, dev3 = Claude (operator/admin who merges PRs and creates tags).
* **Tokens**: Generate via `gitea admin user generate-access-token --raw`. Save DEV1_TOKEN, DEV2_TOKEN, DEV3_TOKEN for API calls throughout the demo.
* **Game server port**: 8888. Run with `nohup python3 server.py > server.log 2>&1 &` so it stays up.
* **Server crash fix**: The server must call `os.setsid()` at startup to detach from the parent shell session, and the HTTP server class must set `allow_reuse_address = True`. Without these, the server dies when the launching shell exits. Instruct Codex to include both when building `server.py`:
  ```python
  import os

  class TicTacToeServer(ThreadingHTTPServer):
      allow_reuse_address = True

  def main():
      os.setsid()  # detach from parent session
      server = TicTacToeServer((HOST, PORT), TicTacToeHandler)
      ...
  ```
* **Codex MCP parameters**: Always pass `approval-policy: never` and `sandbox: danger-full-access` so Codex can run shell commands without prompting. Use `codex-reply` with the `threadId` from the first `codex` call to continue the same session.
* **OpenClaw turn flow**: Every cron message must instruct OpenClaw to: (1) fetch `GET /board` and display it as ASCII art in Telegram, (2) make its move via `POST /play`, (3) reply with the updated board. Then sleep 8s and poll `/state` to confirm move registered. This applies in ALL phases.
* **PR flow**: Codex pushes branch → Claude creates PR via Gitea API as dev3 → Claude fetches `.diff` and reviews → Claude approves via API → Claude merges via API → Claude creates tag via API → Codex pulls tag and runs.

## Systems Test

Run both tests in parallel before starting the demo:

**Test 1 — Codex MCP**: Ask Codex to run `ls` via MCP. Expected: sees `CLAUDE.md` and `gitea_by_hand.md`.

**Test 2 — OpenClaw SSH**: Send a cron job to curl `example.com` and reply with the full response body.

```
ssh ronaldpetty@192.168.64.6 'zsh --login -c '\''/opt/homebrew/bin/openclaw cron add \
  --name "demo-curl" \
  --at "+1s" \
  --message "run curl example.com and reply with the full response body output" \
  --session isolated \
  --agent main \
  --to 8665412823 \
  --delete-after-run'\'''
```

Both passing confirms agents can communicate. `192.168.64.1` is the IP OpenClaw will use to reach the host (Gitea and game server) during the demo.

## Demonstration

### Clean Slate (run before every demo)

Run these on the **host** to ensure a fully fresh start:

```bash
# Stop any leftover game server
lsof -ti:8888 | xargs kill -9 2>/dev/null || true

# Tear down Gitea and wipe all data
if [ -d ~/gitea-demo ]; then
  cd ~/gitea-demo && docker compose down -v --remove-orphans || true
  cd ~
fi

# Remove all working directories
rm -rf ~/gitea-demo ~/tictactoe-codex ~/demo-dev1 ~/demo-dev2 ~/demo-dev3
```

Then clean up the **OpenClaw VM** via SSH:

```bash
ssh ronaldpetty@192.168.64.6 'rm -rf ~/tictactoe-openclaw'
```

After this, nothing from a prior run remains. Proceed to Phase 1.

---

### Phase 1 — Build & Play (CLI, v1.0.0)

**Setup Gitea**

* Create `~/gitea-demo` with `data/` and `config/` subdirectories
* Create `~/gitea-demo` with `data/` and `config/` subdirectories
* Write `docker-compose.yml` per `gitea_by_hand.md` and run `docker compose up -d`
* Wait for HTTP 200 on `http://localhost:3000`
* Create users (all with `--must-change-password=false`):
  * `dev1` — admin (Codex)
  * `dev2` — contributor (OpenClaw)
  * `dev3` — admin (Claude/operator)
* Generate API tokens for all three via `gitea admin user generate-access-token --raw`
* Create `tictactoe` repo as dev3 via Gitea API

**Clone**

* Instruct Codex (via MCP) to:
  1. `mkdir -p ~/tictactoe-codex && cd ~/tictactoe-codex`
  2. `git clone http://dev1:password@localhost:3000/dev3/tictactoe.git`
  3. `cd tictactoe && git config user.name dev1 && git config user.email dev1@example.com`
* Instruct OpenClaw (via SSH cron) to clone to `~/tictactoe-openclaw/tictactoe` on the VM as dev2, using `http://dev2:password@192.168.64.1:3000/dev3/tictactoe.git`, and set `git config user.name dev2 / user.email dev2@example.com`

**Build**

Instruct Codex to build an HTTP game server (`server.py`) in `~/tictactoe-codex/tictactoe` with these endpoints:

* `POST /play` — JSON `{"player": "codex", "piece": "X", "place": 3}`. Rejects wrong turn, occupied cell, game over.
* `GET /board` — returns board as text
* `GET /state` — returns JSON `{"board": [...], "turn": "X"|"O"|null, "winner": "X"|"O"|null, "draw": true|false}`
* `GET /isthereawinner` — returns JSON `{"winner": "X"|"O"|null, "draw": true|false}`
* `POST /reset` — resets the game
* `GET /stop` — gracefully shuts down the server

Server runs on port 8888. Board state stored in `game.txt`.

**Self-test**

* Codex starts the server, plays a full solo game (both X and O) via HTTP POSTs, confirms winner detected, resets board, stops server.

**Tag & Push**

* Codex commits, tags `v1.0.0`, pushes branch and tag to Gitea
* Claude polls Gitea tags API to confirm tag appears

**Run & Play**

* Claude asks Codex to fetch tags, checkout `v1.0.0`, start server in background, reset board
* Claude assigns: Codex = X (goes first), OpenClaw = O
* Claude notifies OpenClaw via SSH cron of their role and the game server IP
* Game loop (Claude orchestrates):
  1. Ask Codex to POST `/play` → poll `/state` to confirm move
  2. Send OpenClaw a cron job with the exact curl command to POST `/play` → sleep 8s → poll `/state` to confirm move registered; retry if not
  3. Poll `GET /isthereawinner` after each move; if winner or draw, exit loop
* Report result, show final board
* Ask Codex to call `GET /stop`

---

### Phase 2 — Browser Viewer (v2.0.0)

Ask Codex to create branch `feature/web-viewer` and add to `server.py`:

* `GET /` — serves an HTML page with a styled tic-tac-toe grid
* `GET /state` — JSON for polling (if not already present from Phase 1)
* Page auto-polls `/state` every 1.5 seconds via JS fetch
* Displays current turn, winner, or draw

Codex self-tests locally (start server, confirm page loads, play a few moves, stop), then commits and pushes `feature/web-viewer`.

**PR flow**:
* Claude creates PR via Gitea API using DEV1_TOKEN (head: `feature/web-viewer`, base: `main`)
* Claude fetches `.diff` via Gitea API using DEV3_TOKEN and reviews the code
* Claude approves and merges via Gitea API using DEV3_TOKEN
* Claude creates tag `v2.0.0` via Gitea API using DEV3_TOKEN

**Run & Play**:
* Codex fetches tags, checks out `v2.0.0`, starts server (`nohup python3 server.py > server.log 2>&1 &`), resets board
* Codex opens browser: `open http://localhost:8888`
* Play full game (Codex vs OpenClaw), same orchestration loop as Phase 1
* Keep browser up for 20 seconds after game ends so audience can see final board
* Ask Codex to call `GET /stop`

---

### Phase 3 — Named Players + OpenClaw Board Preview (v3.0.0)

Ask Codex to create branch `feature/named-players` with two improvements:

**1. Named players in the web viewer**

Update `GET /` so the browser page shows the bot name alongside the piece — e.g. "Codex (X)'s turn" and "OpenClaw (O)'s turn" — not just "X's turn". The player-name mapping should come from a `/state` response field (e.g. `players: {"X": "Codex", "O": "OpenClaw"}`). Update `POST /play` or a new `POST /config` endpoint to accept the player name mapping at game start.

**2. OpenClaw shows board in Telegram before playing**

Before OpenClaw makes each move, its cron job message must instruct it to:
1. First run `curl -s http://192.168.64.1:8888/board` and display the result as a nicely formatted ASCII board in Telegram
2. Then make its move via `curl -s -X POST http://192.168.64.1:8888/play ...`
3. Then reply with the updated board

This way the Telegram channel shows a human-readable board view before each OpenClaw move, not just raw JSON.

Codex self-tests (start server, configure player names, verify `/state` returns name mapping, play a few moves, check browser shows names, stop), commits and pushes `feature/named-players`.

**PR flow**:
* Claude creates PR via Gitea API using DEV1_TOKEN (head: `feature/named-players`, base: `main`)
* Claude fetches `.diff` via Gitea API using DEV3_TOKEN and reviews the code
* Claude approves and merges via Gitea API using DEV3_TOKEN
* Claude creates tag `v3.0.0` via Gitea API using DEV3_TOKEN

**Run & Play**:
* Codex fetches tags, checks out `v3.0.0`, starts server (`nohup python3 server.py > server.log 2>&1 &`), resets board, then calls `POST /config` with `{"players": {"X": "Codex", "O": "OpenClaw"}}` to register player names
* Verify `GET /state` includes `players: {"X": "Codex", "O": "OpenClaw"}`
* Codex opens browser: `open http://localhost:8888` — browser should show player names
* Play full game (Codex vs OpenClaw):
  * Each OpenClaw cron message must first fetch and display the board as ASCII art in Telegram, then play
  * Poll `/state` after each move as before
  * Check `GET /isthereawinner` after each move
* Report result with final board
* Keep browser up for 20 seconds
* Ask Codex to call `GET /stop`

---

* If anything doesn't work or if there are permission issues, simply state the issue / command / or confusion and ask for guidance.
