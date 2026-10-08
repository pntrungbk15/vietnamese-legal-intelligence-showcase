# Local services

The application is a desktop program that works offline once its data is built. Besides its Python packages it
uses up to three local resources. Each one is detected at run time, and when one is missing the features that need
it are switched off instead of the application stopping.

| Resource | Required? | Used for | When it is missing |
|---|---|---|---|
| The library, built from the public vbpl dataset | yes, for research | documents, structure, citations, keyword search | the library is empty; HTML, DOCX and PDF files can still be imported |
| Embedding model (multilingual-E5 small, int8 ONNX, about 118 MB) and the passage vectors | recommended | the dense half of hybrid retrieval and the abstention gate | keyword retrieval only; the gate falls back to requiring a shared two-syllable phrase |
| Neo4j | optional | the Graph page on Neo4j and the read-only Cypher console | the Graph page runs the same queries on SQLite; the Cypher console says it needs Neo4j; research is unaffected |
| Language model server (Ollama, or any OpenAI-compatible endpoint) | optional | writing the answer from the evidence | research shows the evidence only, with the reason no answer was written |

## Data folder

Everything the application writes (the SQLite library, settings, the embedding model, vectors and evaluation reports)
lives in one per-user application-data folder, which an environment variable can redirect. Nothing is written next to
the program.

## Embedding model

The Settings page downloads two files of `intfloat/multilingual-e5-small` from Hugging Face (the int8 ONNX model and
its tokenizer) and then embeds every passage. The job is resumable and runs in the worker process. Vectors are stored
in shards keyed by the text they encode, so re-importing a document only re-embeds passages whose text changed. For
the national-core library (about 170,000 passages) it took 4 h 15 min on a laptop CPU (Intel i5-1235U).

## Neo4j

Neo4j holds a projection of the SQLite library, rebuilt by a sync job in about 100 seconds; it can be dropped and
rebuilt at any time and is never the source of a fact.

* Developed against Neo4j 2026.09 Community (which needs Java 21). Any installation that exposes Bolt works: the
  tarball, Neo4j Desktop or the official container image. The sync replaces the database's content, so use a
  database dedicated to the application.
* The address (default `bolt://localhost:7687`), user, password and database are entered in Settings, or taken from
  `NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASSWORD` and `NEO4J_DATABASE`. The password is kept only in the per-user
  settings file.
* Detection: the application connects on a background thread at start-up and again whenever the settings are saved.
  The Graph page's status light says whether the driver is missing, the server is unreachable, the credentials were
  rejected or Neo4j is switched off, and the page falls back to the SQLite implementation of the same views. A
  *Reconnect* button tries again.
* Research never needs Neo4j: its graph expansion always runs on SQLite, which returned identical results and was
  faster on this data (2.8 ms against 11.6 ms per expansion).

## Language model

* The default is Ollama at `http://localhost:11434` with `qwen2.5:3b` (`ollama pull qwen2.5:3b`). With a local model,
  questions and documents never leave the machine.
* Any OpenAI-compatible endpoint can be used instead (llama.cpp server, vLLM, LM Studio or a hosted API): a base URL
  ending in `/v1`, or an API key, selects that protocol. With a hosted API the question and the evidence are sent to
  it. `LEGAL_LLM_BASE_URL`, `LEGAL_LLM_MODEL` and `LEGAL_LLM_API_KEY` override the saved settings.
* Detection: Settings has a test button that lists the models the endpoint offers. During research the endpoint is
  called only when an answer is needed; if it cannot be reached, the result keeps its evidence, says that no answer
  was written and shows the error. An evidence-only mode skips the model on purpose.
* When Ollama runs on another machine (for example on Windows while the application runs in WSL), it must listen on
  an address the application can reach, and that address goes in Settings.
