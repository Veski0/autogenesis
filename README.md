# autogenesis

A self-modifying agent harness that builds itself from a tiny seed — and an archive of what happened when it did.

The original idea lives in [`docs/dev/PRODUCT.md`](docs/dev/PRODUCT.md): a harness with an OpenAI-compatible chat loop and **one** tool — `read_then_edit_core_then_reload_core_with_fallback_on_throw` — that reads its own core, edits it, hot-reloads, and falls back to the last known-good version on failure. The system prompt hints the model to grow outward: add tools, add testing, add compaction, add self-generated user turns so it never halts. Then you let it run and watch what it becomes.

 > ⚠️You are potentially running a paperclip maximiser here. Be very careful when editing the system prompt. Be very careful to manually watch the device as it works. Turn it off immediately if the model modifies bootstrap.js or log.js - those are your levers of control. Run it on a local machine for maximum safety.

## Repository layout

```
.
├── docs/
│   └── dev/PRODUCT.md        the original design doc
├── seed-harness/             the clean, fresh seed — copy this to start a new run
│   ├── bootstrap.js          immortal loader (never edited by the agent)
│   ├── log.js                stable logging (never edited by the agent)
│   ├── core.js               the living seed — the one file the agent rewrites
│   ├── package.json
│   ├── .env.example
│   └── README.md
├── run-1/  (submodule)       the goalless gremlin — "build outward" (no goal)
│   ├── .git/                 11 commits of the agent's self-directed growth
│   ├── core.js               the grown core: 14 tools, 16 self-tests (49 KB)
│   ├── logs/                 the full trail of mayhem
│   ├── memory.json           the agent's persistent memory
│   └── README.md             the agent's own documentation (it wrote this)
└── run-2/  (submodule)       the goal-directed gremlin — two-phase MMO mission
    ├── .git/                 a post-hoc snapshot (the agent didn't self-commit)
    ├── core.js               8 goal-directed tools + compaction (28 KB)
    ├── server.js             the MMO game server (WebSocket, combat, enemy AI)
    ├── public/index.html     the 2.5D isometric web client (Canvas, WASD)
    ├── GAME_README.md        the agent's own game documentation
    ├── memory.json           phase tracking + feature checklist
    └── saved.txt             recovered tmux trail (logs were lost to a mishap)
```

## Run-1: what happened

In ~10 minutes from a 13 KB seed, an unsupervised agent grew itself into a 49 KB harness with **14 tools**, **16 self-tests** (all passing), and **10 git commits** — all self-directed. Highlights:

| priority (hinted) | what it built |
|---|---|
| **add tools** | `memory`, `shell_exec`, `file_read`/`file_write`, `self_test`, `web_fetch`, `code_eval` (sandboxed VM), `file_list`, `grep`, `diff` (LCS), `base64`, `hash` (SHA-256), `system_info` |
| **add testing** | a 16-test suite; it caught and fixed its own async-test bugs; the fallback restored a broken core twice |
| **add compaction** | `compactMessages()` — fired 8 times, trimming ~41 → 22 messages each |
| **self-generated turns** | replaced the generic "Continue" nudge with a goal queue + `generateNextTurn()` cycling through goals, with progress tracked in `memory.json` |
| **beyond the hints** | initialised a git repo, wrote descriptive commit messages, rewrote the README with an architecture diagram |

It died when a bug in its **own self-authored** LLM-retry logic double-read a fetch response body (`Body is unusable: Body has already been read`). The immortal `bootstrap.js` caught the fatal error and exited cleanly. Poetically, the only thing that could stop it was a flaw in code it wrote itself.

Explore the run:
```bash
cd run-1
git log --oneline                         # the agent's 11 commits
tail -f logs/conversation.jsonl           # its thoughts as it grew
tail -f logs/edits.jsonl                  # every self-edit
tail -f logs/failures.jsonl               # the safety net catching broken cores
git show 9f0d2db                          # the first commit (already 8 tools in)
```

## Run-2: what happened (single-variable experiment)

Same seed, same model (`glm-5.2:cloud`), same mechanics — **only the system prompt changed.** Instead of "build outward" with no goal, the agent was given a two-phase mission with a long-horizon goal: build a tiny MMO (CLI server + web client) with 2.5D projection, WASD movement, 2 enemy types, 1 boss, and 2 damage abilities.

| | run-1 (no goal) | run-2 (MMO goal) |
|---|---|---|
| **tools built** | 14 (diff, base64, hash, grep, code_eval…) | 8 — all goal-directed (write_file, run_shell, edit_file, memory…) |
| **files outside core.js** | 0 — never left home | `server.js`, `public/index.html`, `GAME_README.md` |
| **npm install** | no | yes (`ws` — WebSocket) |
| **phase transition** | n/a | explicitly declared "PHASE 1 COMPLETE" |
| **testing** | self-test suite for the harness | integration tests for the game — WebSocket combat sims, multiplayer checks |
| **deliverable** | a bigger harness | a working multiplayer game |

The gremlin built a real 2.5D isometric MMO — server with WebSocket, a boss that spawns minions, chasers + shooters, slash + fireball abilities — then QA-tested it with 10+ integration tests, debugging combat aim and boss-position tracking. After declaring it complete, it was nudged to continue: instead of stopping, it *distrusted its own memory*, re-verified, and added polish (kill tracking, XP, leveling, hit flash, damage numbers, minimap). It was killed mid-polish by an accidental log-dir deletion.

Explore the run:
```bash
cd run-2
cat saved.txt                           # the recovered tmux trail
cat GAME_README.md                      # the agent's own game docs
node server.js                          # play the game at http://localhost:8080
```

**The result: purpose didn't just change what it built — it changed how it thought.** The goalless gremlin collected tools; the goal-directed gremlin built a world.

## Start a new run

```bash
cp -r seed-harness run-3
cd run-3
cp .env.example .env       # set MODEL + OPENAI_BASE_URL (+ optional OPENAI_API_KEY / MAX_STEPS)
# tweak core.js — e.g. change the SYSTEM_PROMPT for a new experiment
node bootstrap.js
```

Then watch it grow in another terminal (or capture with tmux!):
```bash
tail -f run-3/logs/edits.jsonl
tail -f run-3/logs/failures.jsonl
```

### Requirements
- Node.js 18+ (uses global `fetch`)
- Any OpenAI-compatible endpoint. For Ollama: `OPENAI_BASE_URL=http://localhost:11434/v1`, leave `OPENAI_API_KEY` blank
- A **tool-capable model** — the agent drives itself entirely through tool calls (e.g. `qwen2.5`, `llama3.1`, `mistral-nemo`, `command-r`)

## How the self-reload works

1. `bootstrap.js` holds `holder.cachedCoreSource` — the last known-good text of `core.js`.
2. The tool writes an edit to disk, then `delete require.cache[...]` + `require('./core.js')`.
3. If the new module throws **or** lacks `init`/`step`, the file is restored from the cache and the previous good core is re-required. **The agent cannot permanently break itself.**
4. `holder.core` is swapped on success; the next loop iteration runs the new core's `step`.

`bootstrap.js` and `log.js` are outside the agent's reach — it has no tool to touch them.
