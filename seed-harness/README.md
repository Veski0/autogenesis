# autogenesis — the seed harness

A tiny self-modifying agent harness that builds itself from a single seed.

The idea (`docs/dev/PRODUCT.md`): a harness with an OpenAI-compatible chat loop and **one** tool —
`read_then_edit_core_then_reload_core_with_fallback_on_throw` — that reads its own core, edits it,
hot-reloads, and falls back to the last known-good version if the new one throws. The system prompt
hints the model to grow outward: add tools, add testing, add compaction, add self-generated user
turns so it never halts.

## Files

| file          | who edits it | role                                                                 |
|---------------|--------------|----------------------------------------------------------------------|
| `bootstrap.js`| **nobody**   | immortal loader: `.env`, logging, cached good-source, the loop       |
| `log.js`      | **nobody**   | stable JSONL + colourised stdout logging                            |
| `core.js`     | **the agent**| the living seed: system prompt, LLM client, tools, `init`/`step`     |
| `logs/`       | runtime      | `conversation.jsonl`, `tools.jsonl`, `edits.jsonl`, `failures.jsonl`, `events.jsonl` |

## How the self-reload works

- `bootstrap.js` keeps `holder.cachedCoreSource` = the last known-good text of `core.js`.
- The tool writes the edit to disk, then `delete require.cache[...]` + `require('./core.js')`.
- If the new module throws **or** lacks `init`/`step`, the file is restored from the cache and the
  previous good core is re-required. The agent cannot permanently brick itself.
- `holder.core` is swapped on success; the next loop iteration runs the new core's `step`.

## Run it

```bash
cp .env.example .env      # set OPENAI_API_KEY, MODEL, optional OPENAI_BASE_URL / MAX_STEPS
node bootstrap.js
```

Works with any OpenAI-compatible endpoint (set `OPENAI_BASE_URL`). Node 18+ (uses global `fetch`).

### Ollama (local)

Set `OPENAI_BASE_URL=http://localhost:11434/v1` and leave `OPENAI_API_KEY` blank — the
harness sends no `Authorization` header when the key is empty. **Use a tool-capable model**
(e.g. `qwen2.5`, `llama3.1`, `mistral-nemo`, `command-r`); the agent drives itself entirely
through tool calls, so a model without tool support will only emit text and loop on the
generic "Continue" nudge. If a model rejects the `tools` field, the harness automatically
retries without tools and logs `tools_rejected_by_endpoint`, so it degrades rather than
crashes.

## Watch the mayhem

A colourised trail streams to stdout. The durable record is in `logs/`:

```bash
tail -f logs/conversation.jsonl   # every conversation turn
tail -f logs/edits.jsonl           # every self-edit (find/replace snippets, byte deltas)
tail -f logs/failures.jsonl        # every reload failure / fallback restoration
```

## Safety

- `MAX_STEPS` caps loop iterations (default: unlimited).
- `bootstrap.js` and `log.js` are outside the agent's reach — it has no tool to touch them.
- A broken `core.js` is automatically reverted, so the worst case is a wasted turn, not a dead process.