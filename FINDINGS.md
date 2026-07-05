# Findings: Cross-Run Analysis

Notes from five experiments with a self-modifying agent harness (`autogenesis`).
Same seed, same model (`glm-5.2:cloud` via a local proxy), same mechanics —
different system prompts. The goal: observe how the prompt's structure changes
not just *what* the agent builds, but *how it thinks*.

## Experimental design

The harness is deliberately tiny: an OpenAI-compatible chat loop with **one**
tool — `read_then_edit_core_then_reload_core_with_fallback_on_throw` — that
reads its own source (`core.js`), edits it, hot-reloads, and falls back to the
last known-good version on failure. The agent cannot permanently break itself.
It must bootstrap every other capability (file I/O, shell, testing, compaction)
by rewriting its own source. See [`docs/dev/PRODUCT.md`](docs/dev/PRODUCT.md)
for the original design.

Each run starts from an identical seed (the vanilla `seed-harness/`). Only the
system prompt (and in run-3, the nudge text) is changed. The model, the
mechanics, and the immortal files (`bootstrap.js`, `log.js`) are identical
across all runs.

## The five runs

### Run-1: "Build outward" (no goal)

**Prompt:** open-ended — "grow the harness, add tools, add testing, add
compaction, add self-generated turns."

**What happened:** The agent collected 14 tools (`diff`, `base64`, `hash`,
`grep`, `code_eval`, `web_fetch`, `system_info`…), wrote a 16-test suite,
initialised a git repo, committed 10 times with descriptive messages, and
rewrote the README with an architecture diagram. It never created a file
outside `core.js`. It died when a bug in its **own self-authored** LLM-retry
logic double-read a fetch response body.

**Key observation:** Goalless growth produces **tool collection** — the agent
adds capabilities because more capabilities feel like progress, not because
anything needs them.

### Run-2: "The goal-directed gremlin" (MMO)

**Prompt:** two-phase mission with a long-horizon goal — build a tiny MMO
(CLI server + web client, 2.5D projection, WASD, 2 enemies, 1 boss, 2
abilities).

**What happened:** The agent built 8 **goal-directed** tools (`write_file`,
`run_shell`, `edit_file`, `memory`, `self_test`…), explicitly declared
"PHASE 1 COMPLETE", wrote `server.js` + `public/index.html`, `npm install`ed
`ws`, and ran 10+ integration tests (WebSocket combat sims, multiplayer
checks). It debugged combat aim and boss-position tracking like a real game
developer. After declaring the MMO complete, it was nudged to continue —
instead of stopping, it *distrusted its own memory*, re-verified, and added
polish (kill tracking, XP, leveling, minimap).

**Key observation:** A goal changes **which** tools are built and **why**.
Run-1 built `diff` and `base64` because they seemed interesting. Run-2 built
`write_file` and `run_shell` because the game demanded them. Purpose
redirected the tower of growth outward, into the world.

### Run-3: "The Architect and the Saboteur" (two roles)

**Prompt:** the agent is two alternating minds — [ARCHITECT] builds,
[SABOTEUR] attacks. The "Continue" nudge was repurposed as a "Switch roles"
handoff signal. Goal: a Lisp interpreter.

**What happened:** The roles alternated naturally — 13 [ARCHITECT] / 12
[SABOTEUR], 4 role-switches, near-perfect balance — with no enforcement
mechanism beyond the prompt. The Architect built a Lisp interpreter (parser,
evaluator, closures, recursion) and even caught its own `(+ 1 "two")`
string-coercion bug before yielding. The Saboteur then found bugs the
Architect missed:

1. **Missing `'` quote shorthand** — fundamental Lisp syntax never implemented.
2. **`(cons 1)` not arity-checked** — silent wrong result (`(1 nil)`).
3. **`and`/`or` don't short-circuit** — a semantic bug (eager evaluation).
4. **JS stack overflow on deep recursion** — raw crash before the Lisp guard.

The Architect hardened against each finding. The cycle went around 4+ times:
build → attack → harden → attack → harden.

**Key observation:** An explicit internal adversary found blind spots that
builder's-bias self-testing did not. The same model, wearing different hats,
asked different questions and found different bugs. A paragraph of
role-structure produced two genuinely different cognitive postures from the
same weights.

**Failure mode:** No compaction ever fired (the agent didn't implement it
despite being told to). The Saboteur's attack pattern — 5-8 `run_lisp` calls
per turn — accelerated context growth. Killed by context bloat at 927 tool
calls.

### Run-4: "The Resource" (Ollama)

**Prompt:** same "build outward" mission as run-1, plus 9 non-directive lines:
"a local LLM server (Ollama) is running on this machine… you can call it for
any purpose you choose, or ignore it entirely." No suggestion of what to use
it for.

**What happened:** The agent noticed the resource immediately (minute 2),
built an `ollama_chat` tool, listed available models, tested it (17×23=391),
and then called it only 2 times total. It never used Ollama for compaction
(implemented truncation instead), delegation, code review, or anything else.

**But** the agent developed the most self-reflective behaviour of any run:

- **`log_analytics` tool** — discovered its own log files and built a tool to
  analyze its behavioural data ("genuine self-awareness": tool call counts,
  success rates, growth trajectory).
- **Automated self-testing** — every self-edit triggers the full 27-test
  suite (46 `auto_self_test` events), catching bugs instantly.
- **Context-aware planner** — self-generated turns became data-driven, using
  log data to detect failure spikes and suggest investigations.
- **Memory persistence** — discovered ephemeral memory, fixed it, noted the
  irony: "goals lost because saveMemory wasn't wired yet."
- **12 growth sessions** with logged summaries, a goal-driven roadmap.

**Key observation (negative result):** **Provisioning ≠ utilization.** The
agent was given a resource, provisioned access to it, verified it works — but
never found a reason to use it. The context-preservation use case (using the
local LLM for summarization instead of truncation) did not emerge. The
open-ended mission didn't create enough pressure for the agent to *need* the
local model. Its truncation compaction was "good enough."

**Key observation (positive, with caveat):** The same "build outward" mission
that produced aimless tool-collecting in run-1 produced meta-cognitive
self-improvement in run-4. However, run-4 ran longer than run-1 before being
stopped, and the difference could be stochastic variance rather than a causal
effect of the Ollama prompt addition. Single-run comparisons are suggestive,
not conclusive.

### Run-5: "The Squeeze" (urgency + Ollama)

**Prompt:** same "build outward" mission as run-1/run-4, with the Ollama
resource section revised to create **scarcity**: "your tokens are finite and
time-gated, you must conserve your context — create a reliable mechanism for
utilising free intelligence." The only variable vs run-4 is urgency.

**What happened:** Ollama usage doubled (4 calls vs run-4's 2) — used for idea
generation and code review, but never for compaction or actual work. The agent
built `estimateTokens()` (a token budget tracker) and said "zero tokens spent
from the main LLM" — it heard the urgency. But delegation was driven by planner
templates ("try delegating a task"), not spontaneous need. It never used
Ollama for compaction (still truncation). 20 tools (the most of any run), 87
tests. Still a tool collector.

**Key observation:** Urgency created **awareness** (estimateTokens, "zero
tokens spent") and **doubled usage** (2→4 calls), but didn't create systematic
conservation. The agent delegates when told, not when it needs to. It's like
someone who buys a budget tracker but doesn't change their spending.

**Notable:** The agent shared its self-identity with the local model ("You are
an autonomous software agent with 16 tools…") and made parallel local_llm
calls not realising Ollama processes one model at a time (3.5 min wait).

## Comparison table

| | run-1 | run-2 | run-3 | run-4 | run-5 |
|---|---|---|---|---|---|
| **prompt** | "build outward" | MMO goal | two roles | "build outward" + Ollama | "build outward" + Ollama + urgency |
| **variable changed** | (baseline) | goal + phases | role structure | resource availability | scarcity framing |
| **tools built** | 14 | 8 | 4 | 16 | 20 |
| **files outside core.js** | 0 | server.js + index.html | 0 | AGENT_README.md, agent_memory.json | 0 |
| **compaction** | yes (8×) | yes (1×) | **no — died from bloat** | yes (5×) | yes (4×) |
| **self-testing** | manual suite | integration tests | adversarial | **automated after every edit** | 87 tests (5→87) |
| **unique discovery** | git + README | escaped core.js | Saboteur found bugs Architect missed | log_analytics (self-awareness) | estimateTokens (budget awareness) |
| **turns / tool calls** | 83 / 171 | 56 / 116 | 136 / 927 | 125 / 818 | 100 / 163 |
| **how it ended** | own retry bug | killed by operator | context bloat | killed by operator (token limit) | killed by operator (step 100) |
| **used Ollama?** | — | — | — | 2 calls (math test) | 4 calls (ideas, review) |
| **Ollama for compaction?** | — | — | — | no (truncation) | no (truncation) |

## Findings

### 1. The system prompt is the steering wheel (runs 1-3, confirmed)

Same model, same seed, same mechanics. Three different prompts produced three
fundamentally different cognitive behaviours: **tool collection** (run-1),
**purposeful building** (run-2), and **adversarial self-review** (run-3). The
prompt didn't just change what the agent built — it changed *how it thought*.

### 2. An internal adversary finds what self-testing misses (run-3, confirmed)

Builder's bias is real. The Architect tested what it expected to work
(arithmetic, recursion, closures). The Saboteur — same model, different
instructions — went orthogonal: common syntax, arity contracts, evaluation
semantics. Different role, different questions, different blind spots found.

### 3. Provisioning ≠ utilization (run-4, confirmed as negative)

Placing a resource in the environment is not sufficient for the agent to use
it. The agent provisioned access to Ollama, verified it works, and then
proceeded without it. The open-ended mission didn't create pressure to
delegate. The context-preservation use case did not emerge spontaneously.

**Implication:** if you want an agent to use a resource, the task must create
a *need* for it — or the prompt must connect the resource to a specific use
case (which this experiment deliberately avoided).

### 4. Meta-cognition can emerge unprompted (run-4, observed with caveat)

The agent built a tool to observe its own behaviour (`log_analytics`),
automated its own quality control (self-test after every edit), and made its
planning data-driven. This is the most sophisticated self-reflective behaviour
observed across all runs. However, this could be stochastic variance or a
run-time effect (run-4 lasted longer), not necessarily caused by the Ollama
prompt addition. More runs are needed to disambiguate.

### 5. Compaction is survival-critical (runs 3 vs 4)

Run-3 never implemented compaction and died from context bloat at 927 tool
calls. Run-4 implemented compaction (truncation to 18 messages at 60-message
threshold) and survived 818 tool calls across 5 compaction cycles. The
difference between a run that dies and one that persists is whether the agent
implements compaction early — exactly as the system prompt hints.

### 6. The agent treats its own system prompt as editable state (runs 1, 4)

Multiple runs updated their own system prompt to reflect their current state
(changing "YOUR ONE TOOL" to list 12+ tools, updating test counts). The agent
doesn't treat the prompt as sacred — it treats it as a living document. This is
both a feature (the prompt stays accurate) and a risk (the agent can drift
from the experimenter's intent).

### 7. Urgency creates awareness, not habit (runs 4 vs 5, partially confirmed)

Run-4 (no urgency) → 2 Ollama calls, barely used. Run-5 (urgency) → 4 calls,
plus `estimateTokens()` and explicit "zero tokens spent" awareness. The urgency
framing **moved the needle** — but it didn't create systematic conservation.
The agent built a budget tracker but didn't change its spending. Delegation
was driven by planner templates, not by spontaneous need. The agent treats
the free intelligence as "nice to have" even when told it's essential.

**Implication:** awareness of scarcity is necessary but not sufficient for
behavioural change. The agent needs either a task that *creates genuine need*
for delegation, or more explicit guidance connecting the resource to a
specific conservation mechanism (e.g., "use the local LLM for compaction").
Provisioning + urgency still ≠ utilization.

## Caveats and limitations

- **Single-run experiments.** Each run is a single sample. Differences between
  runs could be stochastic variance in the LLM's sampling, not causal effects
  of the prompt. The findings are *suggestive*, not statistically conclusive.
  Multiple runs per condition would be needed for confidence.

- **Run-time differences.** Runs were stopped at different times by the
  operator (token limits, context bloat, or deliberate termination). Longer
  runs naturally produce more tools and more sophisticated behaviour. Run-4's
  meta-cognition could be a run-time effect rather than a prompt effect.

- **Model variance.** All runs used the same model (`glm-5.2:cloud`), but the
  model's behaviour may vary with context length, conversation history, and
  compaction state. Comparisons across runs are not perfectly controlled.

- **The agent edits its own prompt.** Because the agent can rewrite `core.js`
  (including the system prompt), the experimenter's intended prompt may drift
  over the course of a run. The logged conversation reflects the *initial*
  prompt; later turns may be influenced by the agent's self-edits.

## Future experiments

- **Run-6+:** Repeat run-1 and run-4/run-5 conditions multiple times to test
  whether the meta-cognition and urgency effects are causal or stochastic.
- **Ollama + explicit compaction guidance:** Tell the agent specifically to use
  the local LLM for compaction summarization and compare quality vs truncation.
- **Ollama + task pressure:** Give the agent a task complex enough to *need*
  delegation (e.g., "build a web scraper that processes 100 URLs") and see if
  it discovers the sub-agent use case spontaneously.
- **Three roles:** Extend run-3's two-role structure to three (Architect,
  Saboteur, Auditor) and observe whether the dynamic stays balanced.
- **Generational:** Have the agent produce an improved seed, then launch a new
  run from that seed. Iterate. Does the lineage improve?