# Architecture

Vietnamese Legal Intelligence is a desktop research workstation. It turns the published text of Vietnamese legal
documents into a structured, cited, connected library, and answers questions only from provisions it can show.

```
                         ┌───────────────────────────── desktop process (PyQt5, FluentQt) ─────────────────────────────┐
 vbpl dataset (Parquet)  │  Research · Library · Document · Graph · Evaluation · Settings                              │
 HTML / DOCX / PDF files │        │ JSON lines (QProcess)          │ short reads (GUI thread)   │ graph queries (pool) │
          │              └────────┼────────────────────────────────┼────────────────────────────┼──────────────────────┘
          ▼                       ▼                                ▼                            ▼
   parse & ingest ──────▶ worker process ─────────────▶  SQLite library (system of record)  ◀── Neo4j projection
   parse, cite, resolve   encoder (ONNX E5), vectors,    documents · units · passages · FTS5     (rebuilt by sync)
                          research engine, long jobs     relations · citations · research
```

## Data model

**Documents** carry only what the source states: number, type, issuing body, issue / effective / expiry dates,
effect status, field, signer. Every row records its source (`vbpl.vn via th1nhng0/vietnamese-legal-documents`, or
the imported file name and the header line each value came from). The whole catalogue (171,556 records) is
imported so that citations and relations can name documents whose text is not in the library.

**Units** are the document's structure: preamble, parts, chapters, sections, articles, clauses, points, closing,
appendix. Each unit keeps its line range in the original text and its citation coordinates (article, clause,
point), so a citation such as `điểm a khoản 1 Điều 3` resolves to exactly one unit.

**Passages** are the retrieval units: one per article when it is short, otherwise groups of whole clauses of about
1,200 characters (long clauses are split on sentence boundaries with overlap). Each passage carries a context line
(document, chapter, article heading) that is indexed with it.

**Relations** are the portal's document-to-document records, normalised into 13 directed kinds. The portal stores
each fact twice with different labels ("Văn bản HD, QĐ chi tiết" on one side, "Quy định chi tiết, hướng dẫn thi
hành" on the other); both become one `GUIDES` edge that keeps both labels. Directions were verified against the
data: for every label, the acting document is the newer one in more than 99 % of pairs.

**Citations** are extracted from provision text: `cites` (a provision refers to another) and amendment instructions
(`amend`, `insert`, `repeal`, `replace`, `reword`). Each keeps the unit it was found in, the exact words and offsets,
the target (document, article, clause, point, unit when the target is in the library) and how the target was
resolved (`same document`, `number`, `name at citing date`, `name and date`, `amending instruction`…).

## Parsing Vietnamese legal text

Vietnamese normative documents follow the drafting rules of the Law on Promulgation of Legal Documents, so a
deterministic parser recovers the structure reliably: `Chương I`, `Mục 1`, `Điều 5.`, `1.`, `a)`. The rules that
matter in practice:

* **Quoted text.** Amending laws quote whole articles (“Điều 5. …”). Quoted lines never start a unit.
* **Numbering.** An article must increase the article number (or add a suffix such as `4a`); a clause must be the
  next number; a point the next letter of the Vietnamese sequence `a b c d đ e g h …`. Anything else is content.
* **End matter.** “Nơi nhận:”, signature blocks (`TM.`, `KT.`) and the adoption sentence of a law end the body;
  appendices and forms are kept as their own units and are not retrieval passages.
* **Typography.** Text is NFC-normalised (sources mix composed and decomposed diacritics); a stray accent such as
  “Điềù 31” is still an article heading.

## Citation resolution

A citation names its document by number (`Nghị định số 145/2020/NĐ-CP`), by type and name with or without a date
(`Luật Doanh nghiệp ngày 17 tháng 6 năm 2020`, `Bộ luật Lao động`), or not at all (`khoản 3 Điều 101`, `Điều 5 của
Luật này`: the same document). Names are matched against the catalogue by the **longest known name** the cited
phrase starts with (so “Bộ luật Lao động nhưng chưa…” resolves to the Labour Code and “Luật An toàn, vệ sinh lao
động” keeps its comma). A name without a date means the version in force when the citing text was issued: the
latest one issued on or before that day. An amending law is cited by the name of the law it amends plus its own
date, which is matched by date and title. Amendment instructions without a named document belong to the document
named in the enclosing clause or article heading, or to the only document the text is recorded as amending.

## Retrieval and research

1. **Analyse** the question with rules: document numbers, named laws (lower case accepted; “luật lao động” also
   tries the Labour Code), named articles. Named articles are looked up directly; named documents (and the documents
   that guide them) are boosted.
2. **Retrieve** with BM25 over FTS5 (syllables plus every adjacent pair of content syllables as a phrase, diacritics
   kept) and with multilingual-E5 vectors (exact cosine over ~170k × 384 floats), fused by reciprocal rank fusion;
   one passage per article.
3. **Validity weighting** (default, can be switched off for historical research): documents whose status is
   expired, or that the graph shows as replaced or ended by a document already in effect, are demoted (×0.6).
   Either way they are annotated. The portal's status field lags behind its own
   relationships (Nghị định 27/2014/NĐ-CP is still marked “Còn hiệu lực” although Nghị định 145/2020/NĐ-CP replaced
   it); the relationships are the more specific evidence.
4. **Graph expansion** for the top five articles: instructions that amend them, provisions of guiding documents
   that cite them, and provisions they cite, as linked evidence.
5. **Gate**: if nothing named in the question was found and the closest passage has a cosine similarity below 0.87
   (calibrated in [evaluation.md](evaluation.md); without vectors: no shared two-syllable phrase), stop and say so;
   no model call.
6. **Answer** with the language model from the five strongest primary items and up to two linked items, each
   labelled `[E1]…`; then **verify** each sentence: the citations must be among the items given to the model, the
   sentence's content syllable pairs must be found in the cited text, and every number must appear there.

The workflow is fixed and deterministic up to the model call; the model writes only the final text. See
[decisions.md](decisions.md) for why there is no agent loop.

## Processes and threads

* The **desktop process** reads SQLite directly for documents and lists (sub-millisecond queries), runs Neo4j and
  heavier SQLite graph queries on a two-thread pool, and never loads ONNX Runtime.
* The **worker process** owns the encoder, the vectors and every long job:
  research, importing files or the dataset, downloading the model, computing vectors, synchronising Neo4j. It imports
  ONNX Runtime before anything else (PyQt5's MSVC runtime breaks it on Windows when loaded first), replies in JSON
  lines, streams answer text at most ten times a second and accepts cancellation.
* Long jobs are resumable: vectors are written in shards keyed by the text they encode, so an interrupted build
  continues, and re-importing a document reuses the vectors of passages whose text did not change.

## Neo4j

Neo4j holds a projection of the library: `Document`, `Provision` (articles), `Authority` nodes; the 13 relation
kinds between documents; `HAS_ARTICLE`, `ISSUED_BY`; and `CITES` / `CHANGES` between articles (or to a `Document`
when the cited text is not in the library), each with the citation id, the source unit and the exact words.
A sync rebuilds it from SQLite in about 100 seconds for the national-core library (122k documents, 100k
provisions, 446k relations, 116k citations).

The interface uses Neo4j for exploration (document relations, validity lineage over variable-length paths,
provision citations) and offers a read-only Cypher console with example queries. The same views and the research
expansion are implemented over SQLite, and a test checks that both backends return identical results; the
application works without a Neo4j server.
