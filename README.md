# AI Memory Projects

A collection of learning projects exploring long-term memory solutions for AI agents. Each project integrates a different memory provider with an LLM (Gemini) to demonstrate how persistent, per-user memory can ground conversational responses.

## Choosing a Memory Layer

Quick decision guide for picking a memory approach:

- **Want to remember who the user is?** → Mem0
- **Want to remember when things happened and how they link?** → Zep
- **Want a stateful agent that never forgets its core state over months?** → Letta
- **Want the agent to learn from its mistakes and update its own prompt?** → LangMem

### Ranking for this repo's use case

1. **Mem0** — user preference / fact memory, multi-user via `user_id` namespacing. Not time-aware: flat fact storage per user, no "this changed on date X" reasoning (timestamps exist for created/updated, but there's no bi-temporal validity model).
2. **Zep** — knowledge graph + timestamps for complex, evolving data relationships. Multi-user **and** time-aware natively (Graphiti's bi-temporal graph tracks when a fact became valid/invalid), which is the direct fit for "each user, own long-term, time-related info" — not something you need to bolt together from two tools.

Both Mem0 and Zep have solid per-user namespacing for multi-user SaaS use, which is why they're the two covered in this repo.

**Not pursued yet (future work):**

3. **Letta** — long-running autonomous agent framework. Skipped for now because it's a full agent architecture (its own runtime/server, self-editing memory blocks baked into the agent loop), not a memory layer you can drop into an existing agent — a much bigger adoption decision than Mem0/Zep.
4. **LangMem** — procedural memory (the agent rewrites its own system prompt based on feedback), built on LangGraph. Skipped for now mainly to avoid locking into the LangChain/LangGraph ecosystem.

**Also worth knowing about — the "wiki" pattern ([Karpathy's gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)):** an alternative to vector/graph memory where the LLM incrementally builds and maintains a structured markdown wiki (ingest → query → lint), rather than retrieving from raw documents or fact stores each time. It's a well-known and reportedly effective pattern — some community implementations run at real scale (thousands of pages, git-backed) — but it is explicitly "an idea file... designed to be copy pasted to your own LLM Agent," not a maintained product or SDK. No hosted service, versioning, or support — each adopter builds and maintains their own implementation. Famous, practically proven by third-party repos, but not production-graded in the way Mem0/Zep (as commercial APIs) are. Noted here as future exploration, not a current dependency.

## Projects

### [mem0-project](https://github.com/LLIIMMXXUUAANN/mem0-project)
A multi-user Streamlit chat assistant integrating **Mem0 Cloud** for persistent memory management and Gemini 2.5 Flash for responses.

- Per-user memory scoping with conversation isolation
- Automatic memory extraction from chat exchanges via LLM processing
- Relevance-ranked memory search injected into the chat flow
- Built-in "Memory browser" tab for direct search/add/delete operations
- Transparent "Memories used" panel showing what context informed each reply

**Stack:** Python, Streamlit, Mem0 Cloud SDK, Google Gemini API, `uv`

### [zep-project](https://github.com/LLIIMMXXUUAANN/zep-project)
A learning project demonstrating the **Zep Cloud** memory API, featuring per-user memory graphs and a shared knowledge graph.

- User/thread management with idempotent operations
- Automatic memory assembly via `thread.get_user_context()`
- Standalone graph creation, ingestion, and semantic search (`graph.search`)
- Asynchronous graph ingestion with polling
- Bi-temporal fact modeling (valid/invalid timestamps) with automatic dedup across ingestions

**Stack:** Python, Zep Cloud SDK, Google Gemini API, `uv`

## Comparison at a Glance

| | mem0-project | zep-project |
|---|---|---|
| Memory provider | Mem0 Cloud | Zep Cloud |
| Interface | Streamlit chat UI | Script-based demos |
| Memory model | Fact extraction per user | Per-user graph + shared knowledge graph |
| Notable concept | Direct memory CRUD via UI | Bi-temporal fact graph, async ingestion |

## Runtime Pattern: Add-Every-Turn + Get-Every-Turn

Both projects follow the same operational loop, which is the standard pattern both Mem0 and Zep are built around:

```
for each turn in the conversation:
    1. GET  -> search/retrieve relevant memories for the current message
    2. LLM  -> generate a response, grounded with the retrieved memories
    3. ADD  -> send the new user+assistant messages for fact extraction (async)
```

### Why "get" happens every turn
The relevant memories depend on what the *current* message is about — a question about a flight booking needs different facts than a question about dietary preferences. There's no way to know in advance which facts will be relevant, so retrieval has to run fresh on every turn, synchronously, before the LLM call, because the response quality depends on it.

- **mem0-project:** `memory.search(query, user_id=...)` runs before generation and returns ranked facts, shown in the UI's "Memories used" panel.
- **zep-project:** `thread.get_user_context(thread_id)` auto-assembles a context block from the user's graph; `graph.search()` is used for ad-hoc/standalone graph queries outside the thread flow.

### Why "add" happens every turn (not batched at session end)
`add()` is asynchronous and non-blocking — it kicks off a background LLM extraction job (parse messages → extract durable facts → dedupe against existing memory/graph) and returns immediately, so it never adds latency to the user-visible response. Given that, calling it every turn is strictly better than waiting until "end of session":

1. **No reliable "end of session" event.** Chat UIs rarely have a clean termination signal — users close the tab, the browser crashes, the app is backgrounded, or the session just goes idle indefinitely. If extraction is deferred to session-end, an interrupted session loses 100% of its memory. Per-turn add persists incrementally, so at worst you lose the *current* unsent turn, not the whole conversation.
2. **Cost is roughly a wash either way.** Extracting facts from N turns costs about the same in total tokens whether it's done as N small extraction calls or 1 big call over the whole transcript at the end (arguably batching at the end can even cost a bit less by amortizing the instruction/system-prompt overhead across one call). So cost is not the deciding factor — availability and durability are.
3. **Faster availability across sessions/threads.** If a user starts a *second* conversation (new thread, different device, or a support agent picks up the case) minutes after the first, per-turn add means those facts are already extracted and searchable. Batch-at-end means the second session runs "blind" until the first session officially closes — which may never happen cleanly.
4. **Built-in dedup absorbs the extra calls.** Both providers dedupe/reconcile facts at ingestion time (Mem0 merges/updates existing memories; Zep's bi-temporal graph invalidates superseded facts rather than duplicating them), so adding frequently doesn't create redundant or conflicting memory — it converges to the same end state as one big batch, just sooner.

### Concurrent sessions / multi-user behavior
Memory is scoped per identity, not per process or per session, which is what makes concurrent use safe:

- **mem0-project:** every `add`/`search` call is scoped by `user_id` (set via the sidebar User ID field). Two users chatting at the same time — or the same user with two browser tabs open — never see each other's memories, since every read/write is filtered server-side by that ID. Within one user's concurrent tabs, each tab's `add()` still fires independently; the next `search()` from *either* tab will see facts extracted from *both*, since Mem0 Cloud is the shared source of truth, not local state.
- **zep-project:** scoping is per `user_id` + `thread_id`. Each thread has its own message log and rolled-up context, but all threads for the same user read/write into that user's *shared* knowledge graph. So a fact learned in thread A becomes available to thread B once ingestion finishes — enabling cross-thread continuity — while `thread.get_user_context()` still returns thread-local recent messages plus graph-derived facts, keeping the two concerns (raw dialogue vs. durable knowledge) separate.
- **Race conditions:** because extraction is async, two rapid adds from concurrent tabs/threads can be in flight simultaneously. Both providers handle this server-side via their dedup/merge step rather than requiring the client to lock or sequence calls — the client-side code in both projects fires `add()` and moves on without waiting.

### Batch mode: what it's actually for
Batch ingestion (passing a full message history or set of documents in one call) is *not* an alternative to per-turn add for live chat — it's for **cold-start / bootstrapping**:

- Importing pre-existing conversation history when migrating from another system.
- Bulk-loading reference documents/FAQs into the shared graph (used in zep-project's standalone graph demo) so they're searchable from turn one, without needing a live conversation to "teach" the agent those facts turn by turn.
- One-off backfills, not part of the steady-state request loop.

**Bottom line:** for a live agent, add-every-turn + get-every-turn is the correct default — it's async so it's effectively free in latency, roughly a wash in cost, and strictly more durable and available than deferring to session-end. Batch add is reserved for loading history/knowledge that didn't originate from a live turn-by-turn session.
