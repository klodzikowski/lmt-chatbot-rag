# LMT Chatbot #3 (RAG)

A reference fork of [`lmt-chatbot-skills`](https://github.com/klodzikowski/lmt-chatbot-skills) for Class 21 of the 2 MA LMT *Artificial Intelligence* course at Adam Mickiewicz University. Single static HTML file, no build step.

Same Memory + Skills + RAG drawers as the previous fork, with one new layer in the RAG drawer: **keyword retrieval (BM25) alongside the existing semantic retrieval, switchable live**. The pedagogical headline: *RAG isn't synonymous with vector databases—keyword search still wins for many queries.*

## Try it

[klodzikowski.github.io/lmt-chatbot-rag](https://klodzikowski.github.io/lmt-chatbot-rag/). Paste your OpenAI key into Settings, then explore the four drawers: Settings, Memory, Skills, RAG.

## RAG ≠ vector database

Two retrieval modes in the RAG drawer, side by side:

| Mode | How it works | OpenAI calls | Wins when |
| --- | --- | --- | --- |
| **Semantic** | Embed the query → cosine-rank indexed chunks against the query vector | One embedding call per query (`text-embedding-3-small`, 1536-dim) | The query and the document say the same thing in different words |
| **Keyword (BM25)** | Term frequency × inverse document frequency × length normalisation, all in vanilla JS | None | Exact-match queries: codes, proper nouns, acronyms, verbatim strings |

Flip the radio in the RAG drawer to switch between them. Every retrieval-augmented reply shows a purple **`+N RAG chunks`** chip on its meta line. The assistant turn's metadata in **Detailed JSON** records `retrieved_context`, `retrieved_chunk_count`, and `retrieval_mode` per turn so students can see exactly what the orchestrator did.

The 2026 production picture is **hybrid**—BM25 + dense vectors, sometimes with a learned sparse layer (SPLADE) in between. Elasticsearch, OpenSearch, Weaviate, Qdrant, Pinecone all ship hybrid out of the box. Pick your tool to fit the problem; vectors aren't the only answer.

## At a glance

| Component | OpenAI calls | Contribution to the system prompt |
| --- | --- | --- |
| **Memory** | None—`localStorage` (browser) or Supabase Postgres (cloud) | None directly; rehydrated chat history flows in via `messages` |
| **Skills** | None—string concatenation only | Ticked markdown blocks, joined with `---` separators |
| **RAG (semantic)** | `text-embedding-3-small`—vectorises each chunk at index time, then the user query at chat time | Top-3 chunks (cosine-ranked) under a `# Retrieved context` header |
| **RAG (keyword)** | None—BM25 in vanilla JS over the indexed chunks | Top-3 chunks (BM25-ranked) under the same header |
| **Orchestrator** (`buildAugmentedSystemPrompt`) | None | Joins the contributions with `---` and ships the assembled prompt to `/v1/chat/completions` |

## BM25 in 50 lines

`bm25Rank(query, chunks)` in `index.html` is the whole implementation. Pseudo-Python:

```
for each chunk:
    tokens   = chunk.lower().split_unicode_words()
    chunk_len = len(tokens)
    tf[t]    = how many times each query term appears in the chunk

for each query term t:
    df[t] = number of chunks containing t
    idf   = log((N - df + 0.5) / (df + 0.5) + 1)
    norm  = tf[t] * (k1 + 1) / (tf[t] + k1 * (1 - b + b * chunk_len / avg_len))
    score += idf * norm
```

`k1 = 1.5` controls how quickly term frequency saturates (the 10th occurrence of a word matters less than the 1st). `b = 0.75` controls length normalisation (long docs would otherwise unfairly outrank short ones). Standard Robertson-Spärck Jones values.

## Memory

Tick **Remember conversation between sessions** in the Memory drawer. **Local** writes the chat history JSON to `localStorage`. **Supabase** writes the same shape to a hosted Postgres `chat_memory` table; reload re-hydrates from whichever backend is active.

### Supabase setup

Free tier; ~60 seconds. Sign up at [supabase.com](https://supabase.com) via GitHub; new project, any region.

**SQL Editor** (~30 sec)—paste, then **Run**:

```sql
create table chat_memory (
  id          bigserial primary key,
  user_id     text default 'demo',
  role        text,
  content     text,
  payload     jsonb,
  created_at  timestamptz default now()
);
alter table chat_memory disable row level security;
```

Project Settings → API → copy the **Project URL** and the **`anon` public** key. Paste both into the Memory drawer after switching the backend to Supabase. The `anon` key is public by design—safe in client code. Access is gated by Row Level Security policies; we disabled RLS for the class demo.

## Skills

Three preset checkboxes—**Top-mark thesis** (sharp MA thesis advisor; Socratic), **Tight summariser** (one-line headline plus 3–7 bullets, never prose), **Stretch a deadline** (extension-request email structure)—plus a **Your skill (markdown)** textarea. Every ticked skill is concatenated into the system prompt with `---` separators on every reply.

Drawer footer → **Simple JSON** → field `system_prompt_assembled` shows the actual string the model saw.

## RAG

Two preset documents (**Noam Chomsky—life and theory**; the fictional **Jabłoński-Żukowski Conjecture**) plus a paste textarea. **Index document** chunks the textarea at ~500 chars and embeds each via OpenAI's `text-embedding-3-small`—same indexed chunks serve both retrieval modes; only the ranking differs.

Append-mode—each Index click adds to the existing index. **Clear index** wipes.

## Reset behaviour

- **Clear chat**—empties the screen only. Storage stays untouched (both `localStorage` and Supabase).
- **Reset all**—wipes everything: chat, API key, system prompt, sliders, active skills, RAG index, `localStorage` history, AND the Supabase rows for `user_id = 'demo'`.

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

> *Add a Semantic ↔ Keyword (BM25) toggle to the RAG section of my chatbot. Use the bm25Rank function from `klodzikowski/lmt-chatbot-rag/index.html` as the reference implementation.*
