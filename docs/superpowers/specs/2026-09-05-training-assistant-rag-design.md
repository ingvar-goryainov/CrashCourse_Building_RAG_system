# Design: Training-assistant RAG (Day 2 assignment)

**Date:** 2026-09-05
**Status:** Approved

## Goal

Fork the Day 2 workshop notebook and retarget it from the synthetic realtor
corpus to a personal strength-training corpus, satisfying the five assignment
requirements:

1. Indexing: walk `data/` → chunk → embed → ChromaDB (`upsert`, stable ids)
2. Metadata: at least `source` (file path) for citation and `where` filtering
3. Retrieval: `collection.query` with `query_embeddings` and a `k` parameter
4. Generation: "answer only from context" prompt + a `Sources:` block of `[source=…]`
5. Reproducible logic in a self-contained `.ipynb`

## Deliverable

`Day_2/rag_workshop_02_training_assistant.ipynb` — the realtor notebook forked,
Telegram cells dropped, ingestion rewritten for the training corpus.

| Setting | Value | Rationale |
|---|---|---|
| `CHROMA_DIR` | `Day_2/chroma_data/training_assistant` | Separate dir; the old realtor index is not reused |
| `COLLECTION_NAME` | `training_corpus_v1` | Fresh namespace, no realtor vectors |
| `EMBED_MODEL` | `text-embedding-3-small` | Same as workshop |
| `CHAT_MODEL` | `gpt-4o-mini` | Same as workshop |
| Interpreter | `~/miniconda3/bin/python` (3.12) | Has chromadb 0.5.23, openai, PyMuPDF, pandas; the homebrew `python3` (3.13) has none |

## Corpus

| File | Type | Shape |
|---|---|---|
| `Training Program-8.0-Day {1,2,3}.csv` | program table | Spreadsheet export, week blocks, 2-column exercise spans |
| `Strength tests Training Program 8.0.csv` | record log | Preamble + header + one row per test |
| `Training Program {3.0,4.0}.pdf` | PDF | 2 pages each, clean text layer |
| `Antifragility protocol.md` | Markdown | ~880 chars, injury-prevention exercise list |

## Ingestion — three handlers dispatched by file shape

### Markdown
Whole file → `chunker(700, 100)`.

### PDF
Text per page via PyMuPDF → `chunker(700, 100)`. Page number kept in metadata.

### CSV — shape-dispatched

The workshop's "one row = one document" logic produces garbage here: a row like
`,2,35kg x 6,2,10kg x 3,…` carries no week, no date and no exercise names.

**Program day-CSV** (row 0 col 0 matches `^Day\s*\d+`):

- *Header span map*: a non-empty header cell at column `i` owns columns
  `i … next non-empty − 1`. This single rule covers both observed layouts —
  Day 1's `set# + result` pairs and Day 2's `value + empty` pairs.
- *Week blocks*: a new block starts when col 0 is an integer; a non-empty
  non-integer col 0 (e.g. `10 Nov`) is the block's date; all-empty rows are
  skipped.
- *Rendering*: one chunk per week block. A 2-cell span whose first cell is a
  digit renders as `1) 30kg x 6`; otherwise cells join with a space. Embedded
  newlines normalise to ` / `. Empty fields render as `—`.

```
Training Program 8.0 — Day 1 — Week 3 (24 Nov)
Push press: 1) 40kg x 6; 2) 40kg x 6; 3) 45kg x 6; 4) 45kg x 6
Pull-ups: 1) 16kg x 3; 2) 16kg x 3; 3) 16kg x 3
...
```

**Records CSV** (anything else): skip the preamble, take the first row with ≥3
non-empty cells as the header, then one row = one document rendered as
`Вправа: Back squat | Результат (кг, см): 110 кг | Повтори: 3 | …`.

**Fallback**: a `Day N` file that does not match the expected shape falls back
to whole-file chunking with a printed warning, rather than crashing.

## Stable ids

Deterministic from path + position, so re-running `upsert` overwrites instead of
duplicating:

```
Day_2__data__Training Program-8.0-Day 1.csv::w3     # week block
Day_2__data__Strength tests ….csv::r4               # record row
Day_2__data__Antifragility protocol.md::c0          # md chunk
Day_2__data__Training Program 3.0.pdf::p1::c0       # pdf page chunk
```

## Metadata

`source` (repo-relative path, required — drives both citation and `where`) plus
`kind`, and where applicable `day`, `week`, `date`, `page`, `chunk`, `program`.
All values stringified for Chroma.

## Retrieval

`retrieve(query, k=6, where=None)` → `collection.query(query_embeddings=[…],
n_results=k, where=where, include=["documents","distances","metadatas"])`,
cosine space.

## Generation

Ukrainian prompt, role = personal training assistant. Rules: answer only from
the given context; if context is insufficient, say so outright; never invent an
exercise name or weight; close with a `«Джерела:»` block listing `[source=…]`
paths, citing the *file* for CSV facts rather than a row id.

## Kept extras

- **Router**: an LLM picks `source` paths from an allowlist built from actually
  indexed files, with corpus-specific hints. Multi-file → `{"source": {"$in": […]}}`.
- **History**: in-memory `chat_id → messages`, 4-turn window.
- **Eval**: golden set of ~6 questions with `expected_sources`, scored on source
  coverage + LLM judge, including at least one hallucination probe the corpus
  cannot answer.

Dropped: the Telegram bot section.

## Verification

- Parsing is verified **offline before any API call**: a dry-run cell prints
  per-file chunk counts and a full sample week chunk, so the span/week parse is
  eyeballable without spending a token.
- Embedding → retrieval → generation → eval cannot be verified in this session:
  there is no `OPENAI_API_KEY` and no `.env`. A `.env.example` is provided; the
  loader is wired; the user supplies the key and runs those cells.

## Out of scope

- Telegram bot
- Local/Ollama provider variant (Day 3 covers it)
- Re-targeting the original realtor notebook, which the corpus swap left
  unrunnable (files recoverable from commit `90cb144`)
