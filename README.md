# LMT Chatbot #3 (RAG)

Homework: 

Optional video: 

[link]

A reference fork of [`lmt-chatbot-skills`](https://github.com/klodzikowski/lmt-chatbot-skills) for Class 21 of the 2 MA LMT *Artificial Intelligence* course at Adam Mickiewicz University. Today we extend RAG with a second retrieval algorithm—**keyword search (BM25)**—alongside the existing semantic retrieval, switchable live in the RAG drawer.

**Industry context.** RAG isn't synonymous with vector databases. The default in production today is **hybrid search**: BM25 keyword retrieval combined with dense vector search, often with a learned sparse layer (SPLADE) in between. Elasticsearch, OpenSearch, Weaviate, Qdrant, and Pinecone all ship hybrid out of the box. BM25 is old—1990s, Robertson and Spärck Jones—but still wins for exact-string queries: codes, proper nouns, acronyms. The "always reach for vectors" reflex is wrong; pick the tool that fits the problem.

**Try it:** [klodzikowski.github.io/lmt-chatbot-rag](https://klodzikowski.github.io/lmt-chatbot-rag/). New URL means new storage—you'll need to re-enter your OpenAI key.

## Task 1—Open the app

1. Go to <https://klodzikowski.github.io/lmt-chatbot-rag/>.
2. Open the drawer (hamburger top-left). Settings is open by default; Memory, Skills, RAG are collapsed.
3. Paste your OpenAI API key into Settings → OpenAI API key.
4. Send a test message. A reply should land within a few seconds.

Same skeleton as last week's `lmt-chatbot-skills`, with one new control in the RAG drawer.

## Task 2—Index a preset document

1. Open the **RAG** drawer.
2. Click the **Jabłoński-Żukowski Conjecture (made-up)** preset button. The textarea fills with the text.
3. Click **Index document**. Status counts "Embedding chunk 1 of N…" then "Added N chunks. N total in the index."

Each chunk is embedded once via OpenAI's `text-embedding-3-small`. The same indexed chunks serve both retrieval modes—only the ranking differs.

## Task 3—Semantic retrieval

The mode you already met last week.

1. In the RAG drawer, the **Retrieval method** radio defaults to **Semantic (cosine on embeddings)**.
2. Click the **?** next to it for a one-paragraph explanation of how it works.
3. Send: *"Which group bends reality the most?"*—this query has no word overlap with the indexed text.
4. The reply should pull the answer from the indexed chunks. The meta line should show `+1 RAG chunks` (purple chip).
5. Drawer footer → **Detailed JSON**. Open the file. Find the assistant turn → `retrieved_context` (the actual passages injected) and `retrieval_mode` should say `"semantic"`.

What this proves: **semantic retrieval matches by meaning**. Embeddings group "bends reality" with passages about the Universe-Bending Lemma even though those exact words don't appear in the query.

## Task 4—Keyword retrieval (BM25)

The new mode.

1. Switch the **Retrieval method** radio to **Keyword (BM25)**.
2. Click the **?** next to it to read how BM25 scores chunks.
3. Send: *"What is the Anglistyka Effect?"*—this query contains a verbatim string from the doc.
4. The reply should answer correctly, with `+1 RAG chunks` on the meta line.
5. Detailed JSON → `retrieval_mode` should now say `"keyword"`.

What this proves: **keyword retrieval matches by exact string**. No embeddings, no API call at chat time—just term frequency × inverse document frequency × length normalisation in ~50 lines of vanilla JS.

## Task 5—Compare side by side

Run each query in **both** modes and watch the meta line. Some queries hit in one mode, miss in the other:

| Query | Semantic mode | Keyword mode |
| --- | --- | --- |
| *What is the Anglistyka Effect?* | works | works (faster, no API call) |
| *Which group bends reality the most?* | works | likely fails—no overlap |
| *Define JZCI* | partial—depends on embedding | works—exact acronym match |
| *Tell me about exam-week arithmetic* | works—paraphrase wins | partial—only "exam" or "arithmetic" might match |

What this proves: **neither mode dominates**. The 2026 production answer is hybrid—combine both, weight the scores, ship.

## Task 6—Skills (the bit we missed last week)

Quick walkthrough of the Skills drawer, since we didn't get to it in Class 20.

1. Open the **Skills** drawer. Three preset checkboxes: **Top-mark thesis**, **Tight summariser**, **Stretch a deadline**.
2. Tick **Top-mark thesis** and ask *"Help me with a thesis on agentic AI in education."* The reply should land in coach voice with a Socratic question.
3. Untick. Resend. Generic helper voice.
4. Tick **two presets** at once and resend. Behaviours combine.
5. Add *"Always reply in haiku"* to the **Your skill (markdown)** textarea. Send any message. All three combine.
6. Drawer footer → **Simple JSON** → field `system_prompt_assembled` shows the concatenated prompt with `---` separators.

A skill is a markdown blob the bot summons on demand—same shape as Anthropic Skills, OpenAI Custom GPT instructions, Cursor Rules. Different branding, identical idea.

## Reset behaviour

- **Clear chat**—empties the screen only. Storage stays untouched. Reload re-hydrates from whichever Memory backend is active.
- **Reset all**—wipes everything: chat, API key, system prompt, sliders, active skills, RAG index, retrieval-mode choice, `localStorage` history, AND the Supabase rows for `user_id = 'demo'`.

## Troubleshooting

- **Send returns 401.** OpenAI key not set or expired. Re-paste in Settings.
- **Embeddings call returns 429.** Rate limit. Wait 30 seconds and retry, or chunk less aggressively (longer, fewer chunks).
- **Keyword mode returns no `+N RAG chunks` chip.** Your query has no words in common with the indexed text. Try a different phrasing, or switch to semantic.
- **Switching modes mid-conversation gives different answers to the same question.** That's the point—different retrieval, different evidence reaches the model.
- **Indexer says "Embedding chunk X of Y" even when I'm in keyword mode.** Indexing always embeds, so you can flip back to semantic for free. The mode toggle only changes ranking at chat time.

---

## At a glance (architecture)

| Component | OpenAI calls (per query) | Contribution to the system prompt |
| --- | --- | --- |
| **Memory** | None—`localStorage` (browser) or Supabase Postgres (cloud) | None directly; rehydrated chat history flows in via `messages` |
| **Skills** | None—string concatenation only | Ticked markdown blocks, joined with `---` separators |
| **RAG (semantic)** | One embed call (`text-embedding-3-small`) | Top-3 chunks (cosine-ranked) under a `# Retrieved context` header |
| **RAG (keyword)** | None—BM25 in vanilla JS | Top-3 chunks (BM25-ranked) under the same header |
| **Orchestrator** (`buildAugmentedSystemPrompt`) | None | Joins the contributions with `---` and ships the prompt to `/v1/chat/completions` |

Chunks are embedded once at index time, regardless of mode—so flipping the toggle mid-class is free.

## BM25 in 50 lines

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

`k1 = 1.5` controls how quickly term frequency saturates (the 10th occurrence of a word matters less than the 1st). `b = 0.75` controls length normalisation (long chunks otherwise outrank short ones unfairly). Standard Robertson-Spärck Jones values, the default in Elasticsearch and OpenSearch.

## Source map

`index.html` is one file. Key entries in the `<script>` block:

- `PRESET_SKILLS`, `PRESET_DOCS`—preset definitions.
- `getActiveSkillContent()`—concatenates ticked presets plus the custom textarea.
- `buildAugmentedSystemPrompt(lastUserMessage)`—assembles `[user prompt] + [active skills] + [retrieved chunks]` with `---` separators.
- `embed(text)`, `cosineSim(a, b)`—the semantic primitives.
- `bm25Tokenise(text)`, `bm25Rank(query, chunks)`—the keyword primitives. Pure JS, no deps.
- `getRagMode()`, `retrieveContext(query, k=3)`—reads the radio, picks the path.
- `saveHistoryToStorage()`, `saveHistoryToSupabase()`, `hydrateHistoryFromSupabase()`—the memory dispatch.

Storage keys are namespaced `lmt-chatbot-rag-*` to avoid colliding with the `lmt-chatbot-skills` and `lmt-chatbot` forks on the same `klodzikowski.github.io` origin.

## Use as homework reference

Point your AI coding assistant at this repo and ask it to add the keyword/semantic toggle to your own fork. Example prompt:

> *Add a Semantic / Keyword (BM25) retrieval toggle to the RAG section of my chatbot. Use the bm25Rank function from `klodzikowski/lmt-chatbot-rag/index.html` as the reference implementation.*
