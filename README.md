# LMT Chatbot #3 (RAG)

## Intro

A reference fork of [`lmt-chatbot-skills`](https://github.com/klodzikowski/lmt-chatbot-skills) for Class 21 of the 2 MA LMT *Artificial Intelligence* course at Adam Mickiewicz University. Adds two layers on top of the chatbot baseline: **skills** (markdown blobs the bot summons on demand) and **RAG** (retrieval from an indexed document), with two retrieval algorithms side-by-side—keyword (BM25) and vector (semantic embeddings).

**Try it:** [klodzikowski.github.io/lmt-chatbot-rag](https://klodzikowski.github.io/lmt-chatbot-rag/).

## Task 1—Skills

Over the past few months, the industry has rapidly shifted toward **skills**: structured, composable extensions of the system prompt. Their value comes from being shareable—Anthropic ships Skills, Cursor has Rules, OpenAI has Custom GPT instructions. Different branding, identical shape: a markdown file containing instructions, know-how, or domain rules. It could be a company's style guide, a procedure manual, the voice of a brand—anything you'd otherwise have to paraphrase into a prompt every time.

The authoring discipline is *encapsulation*: capture how a thing is done in one place, version it, share it. The goal isn't one mega-prompt with everything stuffed in. It's a library of focused skills the model can pick from.

In this app we pre-select skills (you tick a checkbox) so the demo is easy to follow. In production the model usually decides which to apply—an admin enables a set, users may import their own, and the model picks the right one mid-conversation. That brings security implications around what gets imported, trusted, and run.

1. Open the **Skills** drawer. Three preset checkboxes: **Top-mark thesis**, **Tight summariser**, **Stretch a deadline**.
2. Tick **Top-mark thesis** and ask *"Help me with a thesis on agentic AI in education."* The reply should land in coach voice, opening with one Socratic question.
3. Untick. Resend the same question. Generic helper voice—will gladly write your MA thesis for you. The **Top-mark thesis** skill flips the behaviour towards questions only.
4. Tick **two presets** at once (Top-mark thesis + Tight summariser) and resend. Behaviours combine.
5. Add *"Always reply in haiku"* to the **Your skill (markdown)** textarea. Send anything. All three combine.
6. Drawer footer → **Simple JSON**. Open the file. Find the field `system_prompt_assembled`—that's the actual string the model saw, with all active skills concatenated under `---` separators.

## Task 2—Keyword RAG (index + retrieve)

> **Skills before RAG.** When a markdown skill fits, prefer it—deterministic, version-controllable, no retrieval failures. Reach for RAG when the knowledge is too big to paste, or changes faster than you can edit a file.

RAG (retrieval-augmented generation) means: pull relevant passages from an indexed document into the system prompt before each reply. First we index. Then we query.

**Baseline first.** Before giving the bot any document, see what it already knows. Send: *"What is the Anglistyka Effect?"* — the term comes from a fictional paper we're about to load, so the model has no training data on it. It'll either confess ignorance or hallucinate something plausible. That's the baseline — no retrieval, no grounding.

### Index

Indexing in keyword mode is free—no OpenAI call, no embedding model. Set up:

1. Open the **Retrieval-Augmented Generation (RAG)** drawer. Set **Retrieval method** to **Keyword (BM25)**.
2. Click the **Jabłoński-Żukowski Conjecture (fake Wikipedia-style entry, 2025)** example → **Index document**. The document button turns green ✓. Status reads *"Added N chunks (keyword mode — no embeddings)."* Pure JavaScript chunking, no API call.
3. Click the **Anglistyka Department Spring Review (fake journal-style paper, 2026)** example—a companion piece that paraphrases the same ideas in different words and adds dates, departmental reception, and follow-up plans. **Index document** again. Both document buttons green now—chunks accumulate (append mode); the index spans two related docs.

### Retrieve (BM25)

Match query words to chunk words, weighted by rarity (rare words count more) and chunk length (long chunks otherwise unfairly outrank short ones). From the 1990s but still wins for exact-string queries—codes, proper nouns, acronyms (the `?` next to the radio explains why).

1. Send: *"What is the Anglistyka Effect?"*—this query contains a verbatim string from the doc.
2. The reply should answer correctly. The meta line should show `+3 RAG chunks`—top-3 by BM25 score, prepended to the system prompt.
3. Drawer footer → **Detailed JSON**. Find the assistant turn → `retrieved_context` (the actual passages injected) and `retrieval_mode` should say `"keyword"`.

What this proves: **keyword retrieval matches by exact string**. No embeddings, no API call at chat time—just term frequency × inverse document frequency × length normalisation in ~50 lines of vanilla JS. The algorithm is from the 1990s (Robertson and Spärck Jones), still the default in Elasticsearch and OpenSearch.

Now try a paraphrase that shares no content words with the docs: *"Why might gifted polyglots blunder addition?"*—same idea as *affective arithmetic*, expressed in synonyms. No `+N RAG chunks` tag on the meta line; the model falls back to general knowledge and gives a generic answer about cognitive load. That's the keyword limitation: BM25 only matches what's lexically present. Vector retrieval comes next.

## Task 3—Switch to vector retrieval (semantic embeddings)

Newer algorithm, abstract logic. Convert the query and each chunk into a high-dimensional vector; rank chunks by cosine similarity to the query vector.

1. Switch the **Retrieval method** radio to **Semantic (cosine on embeddings)**. Both document buttons drop the green ✓—keyword-indexed chunks have text but no embeddings, so semantic can't query them yet.
2. Send the same query that just failed in keyword mode: *"Why might gifted polyglots blunder addition?"*
3. **First-time cost.** Because we indexed in keyword mode (text only, no embeddings), the app embeds each chunk on the fly before ranking. Status counts up *"Lazy-embedding chunk X of Y…"* once, then both document buttons re-green—chunks now have embeddings, queryable in either mode. Future semantic queries skip the backfill.
4. The reply should pull the answer from the indexed chunks—mentioning *affective arithmetic* and *2 + 2 yielding 5*. Meta line shows `+3 RAG chunks`.
5. Detailed JSON → `retrieval_mode` should now say `"semantic"`.

What this proves: **semantic retrieval matches by meaning**. The query embeds into a 1536-dimensional vector via OpenAI's `text-embedding-3-small`, and cosine ranking surfaces the chunks about affective arithmetic even though the query shares zero content words with the docs. One OpenAI embedding call per query (plus the one-time backfill at the moment of mode-switching).

## Task 4—Compare side by side

Same indexed documents. Same chunks. Different ranking. Run each query in **both** modes and watch the meta line:

| Query | Keyword (BM25) | Semantic (vectors) |
| --- | --- | --- |
| *What is the Anglistyka Effect?* | works (faster, no API call) | works |
| *Why might gifted polyglots blunder addition?* | fails—zero shared content words | works—paraphrase catches the meaning |
| *Define JZCI* | acronym matches, but the 500-char chunk split hides the expansion ("Cognitive Index" lands in the previous chunk)—reply often says "not defined" | works—pulls in the surrounding context, gives the full expansion |
| *Tell me about exam-week arithmetic* | works—both terms in docs | works—same |

What this proves: **neither mode dominates**. Production systems combine both—keyword for verbatim, semantic for paraphrase, weight the scores, ship. That's "hybrid search."

## Troubleshooting

- **Send returns 401.** OpenAI key not set or expired. Re-paste in Settings.
- **Embeddings call returns 429.** Rate limit. Wait 30 seconds and retry, or chunk less aggressively (longer, fewer chunks).
- **Keyword mode returns no `+N RAG chunks` tag.** Your query has no words in common with the indexed text. Try a different phrasing, or switch to semantic.
- **Switching modes mid-conversation gives different answers to the same question.** That's the point—different retrieval, different evidence reaches the model.

---

## Appendix — Architecture reference

| Component | OpenAI calls (per query) | Contribution to the system prompt |
| --- | --- | --- |
| **Memory** | None—`localStorage` (browser) or Supabase Postgres (cloud) | None directly; rehydrated chat history flows in via `messages` |
| **Skills** | None—string concatenation only | Ticked markdown blocks, joined with `---` separators |
| **RAG (semantic)** | One embed call (`text-embedding-3-small`) | Top-3 chunks (cosine-ranked) under a `# Retrieved context` header |
| **RAG (keyword)** | None—BM25 in vanilla JS | Top-3 chunks (BM25-ranked) under the same header |
| **Orchestrator** (`buildAugmentedSystemPrompt`) | None | Joins the contributions with `---` and ships the prompt to `/v1/chat/completions` |

### What the model actually receives

The system message OpenAI gets on each turn looks like this:

```
[system_prompt_assembled — base + active skills]

---

# Retrieved context

Use the following passages from the indexed document where they help.
If the answer isn't in them, say so plainly rather than guessing.

[retrieved_context for this turn]
```

The Detailed JSON splits these for inspectability: skills live under top-level `system_prompt_assembled` (static across the conversation); RAG chunks live under each assistant turn's `retrieved_context` (recomputed per query, hence the per-turn count in `retrieved_chunk_count`). The user message stays clean—just the question. Retrieval goes into the **system** role on purpose: that role tells the model "treat this as authoritative context," whereas anything in the user role would read as if the human typed it.

### How indexing works per mode

The **Index document** button is mode-aware:

- **Keyword (BM25) mode.** Chunks the textarea at ~500 chars and stores each chunk as `{ text }`. No OpenAI call, no embedding model. Status reads *"Added N chunks (keyword mode — no embeddings)."* Pure JS, deterministic, fast.
- **Semantic (vector) mode.** Same chunking, but each chunk is also sent to OpenAI's `text-embedding-3-small` to produce a 1536-dimensional float vector. The chunk is stored as `{ text, embedding }`. Status counts up *"Embedding chunk X of Y…"* so the per-chunk cost is visible.
- **Switching modes after indexing.** If you indexed in keyword mode and then switch to semantic and query, the missing embeddings are computed lazily on that first query (one-time backfill, status shows the progress). No manual re-index needed; future semantic queries are then cheap.

### BM25 in 50 lines

`bm25Rank(query, chunks)` in `index.html` is the implementation. Pseudo-code:

```
for each chunk:
    tokens     = chunk.lower().split_unicode_words()
    chunk_len  = len(tokens)
    tf[t]      = how many times each query term appears in the chunk

for each query term t:
    df[t] = number of chunks containing t
    idf   = log((N - df + 0.5) / (df + 0.5) + 1)
    norm  = tf[t] * (k1 + 1) / (tf[t] + k1 * (1 - b + b * chunk_len / avg_len))
    score += idf * norm
```

`k1 = 1.5` controls how quickly term frequency saturates (the 10th occurrence of a word matters less than the 1st). `b = 0.75` controls length normalisation. Standard Robertson-Spärck Jones values, the default in Elasticsearch and OpenSearch.

Storage keys are namespaced `lmt-chatbot-rag-*` to avoid colliding with the `lmt-chatbot-skills` and `lmt-chatbot` forks on the same `klodzikowski.github.io` origin.
