# Engineering decisions

The project brief proposed a set of technologies (knowledge graph, Neo4j, hybrid retrieval, agentic RAG,
reranking…) and left the architecture open. These are the choices made after working with the data, and where they
differ from the brief.

## SQLite is the system of record; Neo4j is a projection

**Choice.** All facts live in one SQLite file (documents, units, passages, relations, citations, with provenance).
Neo4j is rebuilt from it by a sync step.

**Why.** A desktop application must start and work without a database server. One file is easy to back up and
migrate, and full-text search (FTS5) and the catalogue lookups need SQL anyway. Making Neo4j derived means it can be
dropped and rebuilt at any time and can never disagree with the source.

**Deviation from the brief.** Neo4j is not where retrieval-time graph lookups run. Measured on this library, the
one-hop expansion used during research (five articles) takes 2.8 ms in SQLite and 11.6 ms through the Neo4j driver
(medians on an idle machine, including the Bolt round trip; about 3 ms against 150 ms while the CPU was busy), with
identical results on 620 random articles and in a test. Neo4j is
used where it is the better tool: interactive exploration, variable-length validity lineage, ad-hoc Cypher
questions (most-cited articles, replacement chains, guiding decrees by authority).

## A fixed research workflow instead of an agent loop

**Choice.** Analyse → retrieve → validity weighting → graph expansion → gate → answer → verify, always in that order.
The language model writes the final text only.

**Why.** The default model is a local 3B model (privacy: legal questions and documents stay on the machine). Small
models choose tools unreliably, and an agent loop makes runs hard to reproduce and audit. Legal research needs the
opposite: the same question must produce the same evidence, and every step must be explainable. The interface shows
the fixed steps with their timings and counts, never model reasoning.

**Deviation from the brief.** No query decomposition by the model and no agentic tool use. The parts of “agentic”
behaviour that matter here are done deterministically: recognising named documents and articles, scoping, following
amendments and guidance through the graph, and refusing when evidence is insufficient.

## Deterministic structure parsing, not model extraction

Vietnamese drafting rules make the structure regular; a rule-based parser is exact when it fires, traceable to
line ranges, and parses the 3,811 selected documents in about 40 seconds on a laptop CPU. It was checked against an
independent segmentation (see [evaluation.md](evaluation.md)).

## No reranker

A cross-encoder reranker (for example a multilingual MiniLM or bge-reranker) would run on every candidate on the
CPU. The time budget was spent on what changes legal answers more: validity weighting and graph expansion. Reranking
is a clear next step to evaluate with the existing retrieval benchmark.

## Exact vector search, no vector database

About 170,000 passages × 384 dimensions fit in memory (≈260 MB as float32) and an exact dot product takes a few
milliseconds; an approximate index or a vector database would add a dependency without a measurable gain at this
size.

## multilingual-E5 small, int8

Measured on this CPU (i5-1235U): 14.8 passages/s for the int8 model, 9.6 for float32, 6.1 for E5-base int8.
Vietnamese embedding models trained on the Zalo legal data were avoided because the Zalo test set is the retrieval
benchmark.

## The portal's relationships are trusted over its status field

The relationship records are more specific than the effect-status column, which lags behind (documents replaced
years ago are still marked in force). A document replaced or ended by a document already in effect is treated as
superseded, demoted and annotated with the reason.

## Document-level relations come from the portal; provision-level links come from the text

The portal records relations between documents only. Provision-level edges (which article cites, amends or details
which article) are extracted from the text with their exact words, so a reader can always check them.

## Library selection

The library (“national-core”) holds every law, code and ordinance (in force or not, so replaced texts can be shown
as such), every decree in force or partly in force, and the documents referenced by the evaluation questions (282
documents, mostly circulars). The rest of the 171k-document catalogue is metadata only. Decisions, local
resolutions and circulars outside the evaluation set are not parsed: they would multiply the index by ten without
being the primary sources legal questions are answered from.

## Formats

HTML is the portal's format and the main path. DOCX is read with the standard library (`zipfile` + XML), PDF only
when it has a text layer (pypdfium2, optional). Scanned PDFs are refused with a message: OCR is out of scope, and
legal text recognised with errors would be worse than no text.

## Things not done (yet)

* Point-in-time consolidation (the text of an article as amended at a date). The amendment instructions are
  extracted and linked, which is the input such a feature needs.
* Citations in preambles (“Căn cứ …”) are kept as the portal's `BASED_ON` relations rather than extracted.
* GPU inference paths; everything was built and measured on a CPU-only laptop.
