# Private Memory Bake-off — 2026-09-06

This is a provisional Phase 0 result, not the backend-selection ADR. Both runs used the
same frozen manifest (`9d01ef806392925c477e172efec64796adf2c962a1c47076f2bc43c00b1ca43b`),
161 guarded Telegram messages, 34 owner-graded questions, a 4,000-token context ceiling,
OpenAI GPT-5 mini for extraction, and `text-embedding-3-small` for embeddings. Private text,
questions, answer keys, and per-question results remain ignored below `data/`.

Honcho was self-hosted at commit `c300236c110c6e544ced07c843c5daf44fa133a5` with
derivation and message embedding enabled and summary/dream jobs disabled. Nocheh ran at
commit `910f372`. Answer-model grading was disabled, so answer accuracy is unknown for both.

## Result

| Measurement | Nocheh structured memory | Honcho message search |
| --- | ---: | ---: |
| Source-valid recall | 47.30% | **67.65%** |
| Expected-fact recall | 21.57% | **69.12%** |
| Persian source recall | 45.11% | **65.52%** |
| Multi-hop source recall | 49.44% | **73.33%** |
| Task/deadline source recall | 20.83% | **83.33%** |
| Long-range coaching source recall | **66.67%** | 16.67% |
| Retrieval latency p95 | **923 ms** | 1,593 ms |
| Context tokens p95 | 2,417 | **1,367** |
| Preparation duration | 675.4 s | **431.6 s** |
| Background queue lag | 0 | 426.2 s |
| Measured run cost | $0.0662 | Unknown: telemetry missing |
| Projected monthly cost | $0.41 | Unknown: telemetry missing |

Both returned only valid source ids, reported no stale or duplicate evidence, and stayed
inside the context budget. Nocheh created 51 memory records, 42 graph nodes, and 37 graph
edges. Honcho retained 161 plaintext messages plus 161 embeddings and derived 60
source-linked `explicit` documents. Its PostgreSQL database occupied 14,556,519 bytes,
including 3,178,496 bytes of indexes, immediately after the run.

## What The Result Does And Does Not Prove

Honcho decisively won this small retrieval sample, especially for facts and deadlines. It
did not prove that Honcho's derived memory is better. The shared adapter scored Honcho's raw
message search; the 60 derived documents were not queried. Most deriver chunks emitted zero
observations, while other chunks produced the 60 documents. Honcho's cost, answer quality,
deletion/export behaviour, recovery, and derived-document recall remain unmeasured.

The result therefore does not justify replacing Nocheh's typed graph, task lifecycle,
suggestion approval, provenance, or audit trail. It shows a narrower and actionable gap:
semantic retrieval over guarded source messages currently beats retrieval over Nocheh's
LLM-extracted records. Both systems kept query context bounded as the corpus grew; storage,
embedding, and background processing grow instead.

## Provisional Direction

Do not adopt Honcho as the primary memory layer yet. Keep Nocheh as the typed authority and
test a third arm: owned PostgreSQL/pgvector search over the guarded importer log, using the
same embedding model and token budget. This isolates whether Honcho's advantage comes from
its memory derivation or simply from indexing source messages.

The selection rule for the expanded run is:

- choose owned memory if owned source search lands within the frozen 3-point quality margin
  of Honcho while preserving lower latency, known cost, replay, and provenance;
- choose hybrid only if Honcho's **derived-document** recall beats the owned arm outside that
  margin and its cost and recovery gates pass; Honcho remains advisory and typed Nocheh state
  wins conflicts;
- do not select Honcho-primary while its scored advantage depends on raw-message search and
  it cannot replace typed domain state.

Before the selection ADR, expand to at least 1,000 guarded messages and 60 questions, enable
provider usage telemetry, score Honcho derived documents separately, add the owned-source
arm, and run answer-model, deletion/replay, backup/restore, and provider-failure checks.
