# LMT Chatbot #3 (RAG)

A reference fork of [`lmt-chatbot-skills`](https://github.com/klodzikowski/lmt-chatbot-skills) for Class 21 of the 2 MA LMT *Artificial Intelligence* course at Adam Mickiewicz University. Adds two layers on top of the chatbot baseline: **skills** (markdown blobs the bot summons on demand) and **RAG** (retrieval from an indexed document), with two retrieval algorithms side-by-side—keyword (BM25) and vector (semantic embeddings).

**Skills before RAG.** When a markdown skill fits, prefer it—deterministic, version-controllable, no retrieval failures. Reach for RAG when the knowledge is too big to paste, or changes faster than you can edit a file.

**RAG ≠ vector databases.** The production default today is hybrid search: keyword retrieval (BM25, used in Elasticsearch, OpenSearch, Lucene) combined with dense vector search, sometimes with a learned sparse layer (SPLADE) in between. BM25 is from the 1990s but still wins for exact-string queries—codes, proper nouns, acronyms. The walkthrough below covers both: keyword first, vectors second.

**Try it:** [klodzikowski.github.io/lmt-chatbot-rag](https://klodzikowski.github.io/lmt-chatbot-rag/). Same UI as the previous fork; the app prompts for your OpenAI key on first use. New URL means fresh storage—re-enter the key even if you'd set it up before.

## Task 1—Skills

A skill is a markdown blob the bot can summon on demand. Same shape across the industry: Anthropic Skills, OpenAI Custom GPT instructions, Cursor Rules. Different branding, identical idea.

1. Open the **Skills** drawer. Three preset checkboxes: **Top-mark thesis**, **Tight summariser**, **Stretch a deadline**.
2. Tick **Top-mark thesis** and ask *"Help me with a thesis on agentic AI in education."* The reply should land in coach voice, opening with one Socratic question.
3. Untick. Resend the same question. Generic helper voice.
4. Tick **two presets** at once (Top-mark thesis + Tight summariser) and resend. Behaviours combine.
5. Add *"Always reply in haiku"* to the **Your skill (markdown)** textarea. Send anything. All three combine.
6. Drawer footer → **Simple JSON**. Open the file. Find the field `system_prompt_assembled`—that's the actual string the model saw, with all active skills concatenated under `---` separators.

What this proves: **skills are composable format constraints**. You stack them; the bot follows them every reply.

## Task 2—Index a RAG document

RAG (retrieval-augmented generation) means: pull relevant passages from an indexed document into the system prompt before each reply. First we index. Then we query.

**Baseline first.** Before giving the bot any document, see what it already knows. Send: *"What is the Anglistyka Effect?"* — the term comes from a fictional paper we're about to load, so the model has no training data on it. It'll either confess ignorance or hallucinate something plausible. That's the baseline — no retrieval, no grounding.

Now feed it some documents.

1. Open the **RAG** drawer.
2. Click the **Jabłoński-Żukowski Conjecture (made-up)** preset button. The textarea fills with the fictional paper. Click **Index document**. Status counts "Embedding chunk 1 of N…" then "Added N chunks."
3. Click the **Anglistyka Department Spring Review (made-up)** preset—a companion piece that paraphrases the same ideas in different words and adds dates, departmental reception, and follow-up plans. Click **Index document** again. The chunks accumulate (append mode); the index now spans two related docs.

Each chunk is embedded once via OpenAI's `text-embedding-3-small`. The same indexed chunks serve **both** retrieval algorithms below—only the ranking differs.

## Task 3—Keyword retrieval (BM25)

Older algorithm, simpler logic. Match query words to chunk words, weighted by rarity (rare words count more) and chunk length (long chunks otherwise unfairly outrank short ones).

1. In the RAG drawer, set **Retrieval method** to **Keyword (BM25)**.
2. Click the **?** next to it for a one-paragraph explanation. Read it.
3. Send: *"What is the Anglistyka Effect?"*—this query contains a verbatim string from the doc.
4. The reply should answer correctly. The meta line should show `+1 RAG chunks` (purple chip).
5. Drawer footer → **Detailed JSON**. Find the assistant turn → `retrieved_context` (the actual passages injected) and `retrieval_mode` should say `"keyword"`.

What this proves: **keyword retrieval matches by exact string**. No embeddings, no API call at chat time—just term frequency × inverse document frequency × length normalisation in ~50 lines of vanilla JS. The algorithm is from the 1990s (Robertson and Spärck Jones), still the default in Elasticsearch and OpenSearch.

Now try a query the doc doesn't contain verbatim: *"Which group bends reality the most?"* You'll likely see no `+N RAG chunks` chip—nothing matches. That's the keyword limitation.

## Task 4—Vector retrieval (semantic embeddings)

Newer algorithm, abstract logic. Convert the query and each chunk into a high-dimensional vector; rank chunks by cosine similarity to the query vector.

1. Switch the **Retrieval method** radio to **Semantic (cosine on embeddings)**.
2. Click the **?** for a paragraph on how it works.
3. Send the same query that just failed in keyword mode: *"Which group bends reality the most?"*
4. This time the reply should pull the answer from the indexed chunks. Meta line shows `+1 RAG chunks`.
5. Detailed JSON → `retrieval_mode` should now say `"semantic"`.

What this proves: **semantic retrieval matches by meaning**. The query embeds into a 1536-dimensional vector via OpenAI's `text-embedding-3-small`, and cosine ranking surfaces the chunk about the Universe-Bending Lemma even though the query doesn't share a word with it. One OpenAI embedding call per query.

## Task 5—Compare side by side

Same indexed document. Same chunks. Different ranking. Run each query in **both** modes and watch the meta line:

| Query | Keyword (BM25) | Semantic (vectors) |
| --- | --- | --- |
| *What is the Anglistyka Effect?* | works (faster, no API call) | works |
| *Which group bends reality the most?* | likely fails—no overlap | works—catches the meaning |
| *Define JZCI* | works—exact acronym match | partial—depends on embedding |
| *Tell me about exam-week arithmetic* | partial—only "exam"/"arithmetic" might match | works—paraphrase wins |

What this proves: **neither mode dominates**. Production systems combine both—keyword for verbatim, semantic for paraphrase, weight the scores, ship. That's "hybrid search."

## Troubleshooting

- **Send returns 401.** OpenAI key not set or expired. Re-paste in Settings.
- **Embeddings call returns 429.** Rate limit. Wait 30 seconds and retry, or chunk less aggressively (longer, fewer chunks).
- **Keyword mode returns no `+N RAG chunks` chip.** Your query has no words in common with the indexed text. Try a different phrasing, or switch to semantic.
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
