# Intuit Open Source - Platform Engineering & AI

Presentation and demo created for this event:

https://luma.com/intuitossmtvjun2026?tk=Kr9okQ

## Files

Presentation:

https://docs.google.com/presentation/d/1dMQyVRhUuQxltW_-A-q6TpXoQeS-X65B-q9ZRbp4ejU/edit?usp=sharing

Git repo:

https://github.com/minimumdistance/intuit_open_source


* ARCHITECTURE.md - shows general layout of bots and game
* CLAUDE.md - starting point for Anthropic Claude (used CLI in demo)
* gitea_by_hand.md - example on doing GitOps for Claude
* README.md - this file
* REPORT.md - report to motivate how much is actually happening and why its a challenge to manage

## How to Run (per my M5 Mac 32 GB laptop)

* Accounts on OpenAI, Anthropic, xAI
* Mac VM on Mac via UTM (with network access between VM and host
* Ability to SSH from Host to VM (without password, aka use keys)
* Install OpenClaw in VM, with Telegram channel support - configure to use xAI Grok
* Install Telegram on Host
* Install and configure access to Codex CLI on Host
* Install and configure access to Claude CLI on Host
* Clone this repo to host

That is basic requirements, now the fun part:

* cd repo
* launch Claude CLI
* have Claude read entire repo
* have Claude execute CLAUDE.md

Sit back, relax, it takes about 30 minutes from start to finish.

## Results

* You should see 3 Pull Request being engineered and executed via GitOps between Codex and Claude, all while the game of Tic Tac Toe is played at end of each phase between Codex and OpenClaw

* Phase 1 - Text Battle (can view action from Telegram)
* Phase 2 - Browser Battle (can view action from Browser and Telegram)
* Phase 3 - Browser boards adjustment (adding player names to game board)

## Goal?

Goal is to show a "Hello World" effort by agents doing best practices, giving humans a chance to critique.
