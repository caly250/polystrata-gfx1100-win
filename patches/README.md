# Local patches

## `server.py.patched` — keep the stream alive during a long prefill

**Problem.** During prompt reading, `strata-poly` sends nothing to say. The server emits an
SSE comment (`: keep-alive`) every ~2 s, but **every OpenAI client drops comment lines**, so a
front end that streams shows nothing for the entire prefill.

Measured on this machine with an 8,000-token prompt: **29.8 s of complete silence**, 0 usable
events. That reads as "it hung" — the most common complaint about local inference.

**Fix.** Two places yield a real event instead of an SSE comment:

| File / line | Endpoint | Before | After |
| --- | --- | --- | --- |
| `openai_chunks`, `kind == "ping"` | `/v1/chat/completions` | `yield None` → `: keep-alive` | `yield {**chunk({}), "strata_ping": True}` |
| Anthropic stream, `kind == "ping"` | `/v1/messages` | `yield None` → `: keep-alive` | `yield "ping", {"type": "ping"}` |

`ping` is a real Anthropic SSE event type, and the OpenAI path is a valid chunk with an empty
delta, so no client is confused.

**Measured after the patch** (same 8,000-token prompt):

| | before | after |
| --- | ---: | ---: |
| SSE comments (dropped) | many | **0** |
| real events during prefill | **0** | **21** |
| first content token | 29.8 s | 29.8 s (unchanged — this is physics) |

The wait does not get shorter. What changes is that the client **knows** the request is alive.

Apply with:

```bat
copy patches\server.py.patched "%PS_DIR%\serve\server.py"
```

(from the PolyStrata folder, after backing up the original)
