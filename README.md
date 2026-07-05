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
├── run-2/  (submodule)       the goal-directed gremlin — two-phase MMO mission
│   ├── .git/                 a post-hoc snapshot (the agent didn't self-commit)
│   ├── core.js               8 goal-directed tools + compaction (28 KB)
│   ├── server.js             the MMO game server (WebSocket, combat, enemy AI)
│   ├── public/index.html     the 2.5D isometric web client (Canvas, WASD)
│   ├── GAME_README.md        the agent's own game documentation
│   ├── memory.json           phase tracking + feature checklist
│   └── saved.txt             recovered tmux trail (logs were lost to a mishap)
└── run-3/  (submodule)       the Architect and the Saboteur — adversarial self-review
    ├── .git/                 a post-hoc snapshot with full logs preserved
    ├── core.js               a Lisp interpreter embedded as a tool (48 KB, 4 tools)
    ├── logs/                 the full trail — 136 turns, 927 tool calls, 216 failures
    └── .gitignore            (logs ARE tracked in this repo)
└── run-4/  (submodule)       the Resource — Ollama available, meta-cognition emerges
    ├── .git/                 a post-hoc snapshot with full logs preserved
    ├── core.js               16 tools, 27 tests, auto-self-test, log_analytics (71 KB)
    ├── AGENT_README.md       the agent's own documentation (it wrote this)
    ├── agent_memory.json     persistent memory with goals + session logs
    └── logs/                 the full trail — 125 turns, 818 tool calls, 46 auto-self-tests
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

## Run-3: what happened (structural experiment — two minds)

Same seed, same model — but a fundamentally different **prompt structure**. Instead of one mind with a mission, the agent was split into two alternating roles sharing one conversation:

- **[ARCHITECT]** — builds, then yields.
- **[SABOTEUR]** — attacks what the Architect built, finds bugs, reports them (does NOT fix), then yields.

The "Continue" nudge was repurposed as a **"Switch roles" handoff signal**. No code enforced the alternation — just the system prompt. The goal: a Lisp interpreter (parser, evaluator, closures, recursion, error handling).

### The four runs compared

| | run-1 (no goal) | run-2 (MMO goal) | run-3 (two roles) | run-4 (Ollama) |
|---|---|---|---|---|
| **prompt structure** | one mind, open-ended | one mind, two-phase | two minds, alternating | one mind + local resource |
| **tools built** | 14 (diff, base64…) | 8 (all goal-directed) | 4 (run_lisp) | 16 (log_analytics, ollama_chat…) |
| **files outside core.js** | 0 | server.js + index.html | 0 | AGENT_README.md, agent_memory.json |
| **testing** | self-test suite | integration tests | adversarial | **automated after every edit** |
| **deliverable** | a bigger harness | a working multiplayer game | a hardened Lisp interpreter | a meta-cognitive self-improver |
| **turns / tool calls** | 83 / 171 | 56 / 116 | 136 / 927 | 125 / 818 |
| **used Ollama?** | — | — | — | built tool, tested once, barely used |

### What the Saboteur found that the Architect missed

The Architect built a working interpreter and even caught its own `(+ 1 "two")` string-coercion bug before yielding. But the Saboteur — wearing a different hat, asking different questions — found:

1. **Missing `'` quote shorthand** — `(car '())` fails. Fundamental Lisp syntax the Architect never implemented.
2. **`(cons 1)` not arity-checked** — returns `(1 nil)` instead of erroring. Silent wrong result.
3. **`and`/`or` don't short-circuit** — eagerly evaluate all args. A semantic bug: `(and false (explode))` would evaluate `explode`.
4. **JS stack overflow on deep recursion** — the raw Node.js stack overflowed before the Lisp depth guard fired.

The Architect then hardened against each finding (lowered `LISP_MAX_DEPTH`, added type checking, etc.). The cycle went around 4+ times: build → attack → harden → attack → harden.

### The result

**Hypothesis confirmed: an explicit internal adversary found blind spots that builder's-bias self-testing did not.** The same model, wearing different hats, asked different questions and found different bugs. A paragraph of role-structure produced two genuinely different cognitive postures from the same weights.

The roles alternated naturally (13 Architect / 12 Saboteur — near-perfect balance, 4 role-switches) with no enforcement mechanism beyond the prompt. Killed after ~20 min when context bloated (927 tool calls, no compaction ever fired — the Saboteur's machine-gun attack pattern of 5-8 `run_lisp` calls per turn accelerated context growth).

Explore the run:
```bash
cd run-3
cat logs/conversation.jsonl | jq -r '.preview' | head -40    # the Architect/Saboteur dialogue
grep -o '\\[ARCHITECT\\]\\|\\[SABOTEUR\\]' logs/conversation.jsonl | sort | uniq -c   # role balance
grep 'Switch roles' logs/conversation.jsonl | wc -l               # handoff count
node -e "require('./core.js').toolHandlers.run_lisp({program:'(+ 1 (* 2 3))'},{})"  # try the interpreter
```

**The finding across all four runs: the system prompt is the steering wheel. Same model, same seed, same mechanics — but the prompt's structure didn't just change what the agent built. It changed how it thought.** See [FINDINGS.md](FINDINGS.md) for a detailed cross-run analysis.

## Run-4: what happened (resource experiment — Ollama)

Same seed, same model, same "build outward" mission as run-1 — with ONE addition: 9 lines in the system prompt pointing at a local Ollama instance ("it is simply available if you need additional intelligence"). No suggestion of what to use it for.

### The Ollama result (negative — and valuable)

The agent noticed the resource immediately (minute 2), built an `ollama_chat` tool, listed available models, and tested it (17×23=391). Then called it only 2 times total. It never used Ollama for compaction (implemented truncation instead), delegation, code review, or anything else. **Provisioning ≠ utilization** — the open-ended mission didn't create enough pressure for the agent to need the local model.

### What emerged instead (meta-cognition)

The most self-reflective run. Unprompted, the agent built:

- **`log_analytics`** — a tool to analyze its own behavioral data ("genuine self-awareness")
- **Automated self-testing** — every self-edit triggers the full 27-test suite (46 times)
- **Context-aware planner** — self-generated turns driven by log data (failure rates, tool diversity)
- **Memory persistence** — discovered ephemeral memory, fixed it, noted the irony
- **12 growth sessions** with logged summaries, goal-driven development roadmap

### Caveat

Run-4 ran longer than run-1 before being stopped. The meta-cognitive behavior could be stochastic variance or a run-time effect, not necessarily caused by the Ollama prompt addition. See [FINDINGS.md](FINDINGS.md) for discussion.

Explore the run:
```bash
cd run-4
cat AGENT_README.md                                       # the agent's own documentation
grep 'auto_self_test' logs/events.jsonl | wc -l             # automated test count
grep '"tool":"ollama_chat"' logs/tools.jsonl                # the 2 Ollama calls
node -e "require('./core.js').toolHandlers.log_analytics({}, {holder:{log:{}}})"  # self-analytics
```

## Start a new run

```bash
cp -r seed-harness run-5
cd run-5
cp .env.example .env       # set MODEL + OPENAI_BASE_URL (+ optional OPENAI_API_KEY / MAX_STEPS)
# tweak core.js — e.g. change the SYSTEM_PROMPT for a new experiment
node bootstrap.js
```

Then watch it grow in another terminal (or capture with tmux!):
```bash
tail -f run-5/logs/edits.jsonl
tail -f run-5/logs/failures.jsonl
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
