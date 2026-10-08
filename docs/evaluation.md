# Evaluation

Every number below was measured on the national-core library (3,765 documents with text, 100,071 articles,
170,204 passages) on a laptop CPU (Intel i5-1235U, no GPU, WSL2), with the commands shown. Reports are shown on the
application's Evaluation page.

## Benchmarks used

* **Zalo AI 2021 Legal Text Retrieval**, test split, as released by GreenNode on Hugging Face (MIT). Each question
  has its relevant articles, identified by document number and article number, so they can be located in this
  library independently of how either side segmented the text. 818 questions; the questions whose relevant document
  is in the library are used. The documents referenced by these questions were added to the library selection, so
  the benchmark measures ranking among ~3,800 documents, not coverage.
* **Out-of-scope questions**: 30 questions written for this evaluation (everyday topics, foreign law, future facts,
  prices) that the library cannot answer.
* **The portal's own relationships**, for citation extraction.
* **The Zalo corpus segmentation**, for structure parsing: the corpus lists the articles of the same documents as
  segmented by its authors.

The dense model was chosen among models not trained on the Zalo data.

## Structure parsing


| Measure | Result |
|---|---|
| Documents in both the library and the Zalo corpus | 782 |
| Reference articles in those documents | 24,600 |
| Reference articles found by the parser (same document, same article number) | **99.3 %** |
| Article text agreement (Jaccard over syllables), median | **0.976** |
| Articles with text agreement ≥ 0.8 | 97.6 % |
| Library documents in which no article was found | 24 of 3,765 |
| Library documents whose article numbers have a gap | 89 of 3,765 |

The reference corpus lists only part of each document's articles, so articles found by the parser and absent from
the reference (809) are expected and not errors. Two parser changes came out of this comparison: documents whose
portal page is empty are no longer counted as having text, and article headings with a stray accent (“Điềù 31”) are
recognised. Together they raised article recall from 95.5 % to 99.3 %. Most remaining differences are articles quoted
inside amending documents, which the reference counts as articles and the parser deliberately keeps as quoted
text.

## Citations and relations


| Measure | Result |
|---|---|
| Citations and amendment instructions extracted | 120,304 |
| Resolved to a document | 115,652 (**96.1 %**) |
| Resolved to a provision of the library | 87,646 |
| Pairs of different documents linked by a citation | 4,101 |
| … of which the portal also records a relation | 3,026 (**73.8 %**) |
| Official `AMENDS` relations (both documents in the library) found in the text | 938 / 1,395 (67.2 %) |
| Official `GUIDES` relations found in the text | 522 / 1,670 (31.3 %) |
| Official `REFERENCES` relations found in the text | 310 / 1,334 (23.2 %) |

Resolution methods: same document 86,578; name at the citing document's date 12,476; amending instruction 11,112;
document number 5,433; name and year 537; name and date 24. Unresolved: 3,148 citations that name no document
(“Điều 10 của Luật”), 752 names and 115 numbers not found in the catalogue, 114 names whose date or year matches no
document, 1 ambiguous name.

Agreement with the portal is a lower bound on precision: the portal does not record every citation as a relation.
Recall of `GUIDES` is low because a decree usually cites the law it implements in its preamble (“Căn cứ …”), which is
kept as the portal's `BASED_ON` relation rather than extracted, and refers to the law's articles by number only when
detailing them.

## Retrieval


Article-level metrics: a question counts as found at k when one of its relevant articles is among the first k
distinct articles returned (passages of the same article count once).

751 questions; 37 others are left out because none of their relevant articles is in the library (the document is
missing from the catalogue or was published on the portal without text, or the article was not found). Measured 2026-10-08 14:39.

| Mode | Recall@1 | Recall@5 | Recall@10 | MRR@10 | nDCG@10 |
|---|---|---|---|---|---|
| Keyword (BM25, syllables + phrases) | 54.5 % | 81.8 % | 86.8 % | 0.660 | 0.713 |
| Dense (multilingual-E5 small, int8) | 51.0 % | 78.6 % | 87.0 % | 0.626 | 0.688 |
| Hybrid (reciprocal rank fusion) | 56.6 % | 83.5 % | 89.2 % | 0.679 | 0.734 |
| Hybrid + validity weighting (default) | 42.3 % | 66.6 % | 79.8 % | 0.530 | 0.596 |

**Hybrid fusion is the best ranking overall**: it beats either signal alone at every cutoff. The signals are
complementary: keyword search is stronger on current law (MRR 0.696 against 0.619), dense search on the older texts
(0.643 against 0.577), and fusion keeps most of both.

**Validity weighting** ranks law currently in force first. The benchmark was built in 2021, and
229 of its 751 answers are now in documents that have
expired or that the portal's relationships show as replaced. Scored separately:

| Mode | Answer in force: Recall@10 | MRR@10 | Answer no longer in force: Recall@10 | MRR@10 |
|---|---|---|---|---|
| Keyword (BM25, syllables + phrases) | 89.8 % | 0.696 | 79.9 % | 0.577 |
| Dense (multilingual-E5 small, int8) | 87.9 % | 0.619 | 84.7 % | 0.643 |
| Hybrid (reciprocal rank fusion) | 90.6 % | 0.696 | 86.0 % | 0.641 |
| Hybrid + validity weighting (default) | 91.6 % | 0.708 | 52.8 % | 0.123 |

On the 522 questions answered by law in force, validity weighting improves on plain
hybrid ranking; on the others it does what it is meant to do, ranking current provisions above the historical answer.
It is the default for research on current law and can be switched off in the interface (“Ưu tiên văn bản đang có hiệu
lực”) for historical research, which gives the plain hybrid ranking.

Latency of the whole retrieval step (keyword + dense + fusion + validity), uncached, on 100 questions:
median 262 ms, 90th percentile 445 ms (the encoder and the 170,204 vectors are loaded once, when the
worker starts).

## Grounded answers and abstention


**Abstention gate** (no model). multilingual-E5 similarities are compressed: on a random sample of 100 benchmark
questions the closest passage scored at least 0.853 (median 0.907), on the 30 out-of-scope questions 0.805–0.897
(median 0.857). The threshold was chosen on another random sample of 150 benchmark questions: at 0.87, 4 of the 150
(2.7 %) and 23 of the 30 out-of-scope questions (77 %) are declined before any model call. Checked on the first
sample of 100: 2 of 100 benchmark questions (2 %) are declined. The out-of-scope set is small and was used to choose the threshold, so this
figure is optimistic; the end-to-end figure below adds the model's own refusals.

**End to end** with qwen2.5:3b (Ollama, CPU), on a separate random sample of 30 benchmark questions and the 30
out-of-scope questions, measured 2026-10-08:

| Measure | Result |
|---|---|
| Out-of-scope questions declined (gate or the model's “KHÔNG ĐỦ CĂN CỨ”) | **29 / 30 (96.7 %)** |
| Benchmark questions wrongly declined | 0 / 30 |
| Citations that point to evidence the model was given | **100 %** |
| Answer sentences supported by their cited evidence (lexical check) | 39.2 % |
| Answers with a number not found in the cited evidence | 5 / 30 (16.7 %) |
| Answers in which every substantive sentence is supported | 12 / 30 (40 %) |
| Benchmark article among the evidence given to the model | 66.7 % |
| Benchmark article cited in the answer | 43.3 % |
| Median time per question (retrieval + generation on CPU) | 65 s |

What this shows: the retrieval and gating layers behave well, and the model never cites evidence it was not given,
but a 3B model paraphrases (which the strict lexical check counts as weak support) and sometimes states a number that
is not in its sources. The verification exists for exactly this: every such sentence is marked in the interface, with
the missing number named, so the reader checks it against the provision shown next to it. A larger model behind the
same OpenAI-compatible setting is the obvious next measurement; none was available offline on this machine.

## Not measured

* Legal correctness judged by a lawyer. The sentence-support and citation checks are automatic and lexical: they
  show that what was said is in the cited text, not that the answer is complete or the best reading of the law.
* GPU latency (no GPU was available).
