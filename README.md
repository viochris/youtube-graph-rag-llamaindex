# 🕸️ YouTube Graph RAG Chatbot (LlamaIndex)

A CLI chatbot that lets you chat with a YouTube video through a knowledge graph built from its transcript. The first time you run it, it fetches the video's captions, splits them into overlapping chunks, asks Gemini to extract relationships from every chunk, and stores the resulting graph in Neo4j. From then on, every question is answered by retrieving the related parts of the graph together with their source text, reranking them with a cross-encoder, and passing the best ones to Gemini, which is instructed to answer strictly from what the video says. The conversation is saved in PostgreSQL, so it carries over between runs. This is the **Graph RAG** version, where context is found by searching a graph of relationships instead of searching plain text chunks. The Vector version lives in [youtube-vector-rag-llamaindex](https://github.com/viochris/youtube-vector-rag-llamaindex).

![Python](https://img.shields.io/badge/Python-3.10%2B-blue?style=flat&logo=python&logoColor=white)
![LlamaIndex](https://img.shields.io/badge/Framework-LlamaIndex-6E56CF?style=flat)
![Neo4j](https://img.shields.io/badge/GraphDB-Neo4j-008CC1?style=flat&logo=neo4j&logoColor=white)
![Gemini](https://img.shields.io/badge/LLM-Google%20Gemini-4285F4?style=flat&logo=googlegemini&logoColor=white)
![LiteLLM](https://img.shields.io/badge/Routing-LiteLLM-black?style=flat)
![Reranker](https://img.shields.io/badge/Reranker-Cross--Encoder-yellowgreen?style=flat)
![PostgreSQL](https://img.shields.io/badge/Memory-PostgreSQL%20(AsyncPG)-336791?style=flat&logo=postgresql&logoColor=white)
![Phoenix](https://img.shields.io/badge/Tracing-Arize%20Phoenix-purple?style=flat)
![License](https://img.shields.io/badge/License-MIT-green?style=flat)

## 📌 Overview

This repository contains one CLI script, `Graph-self-make-youtube-reader-cmd.py`, that turns a YouTube video into something you can ask questions. You give it a list of video URLs. It downloads each transcript, splits the text into overlapping chunks, and uses an LLM to extract relationships from every chunk, written as subject, relation, and object. These relationships are stored in Neo4j as a property graph, which is a set of nodes connected by relationships. The script then opens an interactive chat loop in the terminal, where each answer is built from the parts of the graph retrieved for that question.

The transcript loader is written inside the script itself (`load_youtube_transcripts`) instead of using a ready-made LlamaIndex reader. For each URL, it reads the video id, calls `youtube-transcript-api` to get the caption snippets, joins the snippets into one continuous text, and wraps that text in a LlamaIndex `Document`. The video id and the URL are stored in the document's metadata, under the keys `video_id` and `video_url`.

Extracting and storing the graph only happens when the Neo4j database has no nodes. The script checks this with one Cypher query that counts every node in the database. On every later run, it skips the download and the extraction and rebuilds the index straight from the graph already stored in Neo4j, which saves time and LLM calls. The index is then connected to a context-mode chat engine, with a cross-encoder reranker to refine the retrieved passages and PostgreSQL to store the conversation.

The graph index also uses the Gemini embedding model, which is configured in the script. By LlamaIndex's default for a graph index, the graph nodes are embedded as well, so a question can be matched to the graph by meaning and not only by keywords. The script does not change this default.

The answers are meant to come from the retrieved graph context and not from the model's own memory. The system prompt tells the model to answer strictly from the retrieved context and to say so when the answer is not there. This is an instruction and not a hard check, so the printed sources are there to let you verify each answer. After every answer, the script prints the source nodes with their scores, the token usage, and the estimated cost of that turn. The token and cost numbers cover the chat turns only. The LLM calls made while the graph is built are not included, because the counters are reset when the chat session starts.

This repository is the Graph version of a three-architecture comparison (Vector, Graph, and Hybrid RAG). See [Three RAG Architectures](#-three-rag-architectures) for how they differ and where each one lives.

Here is what happens when you run the script:

1. Load `.env` and stop with a plain error message if `GOOGLE_API_KEY`, `GROQ_API_KEY`, or `PHOENIX_API_KEY` is missing. This check runs before the error handling in step 11 starts.
2. Configure the LLM (Gemini through LiteLLM), the embedding model, the chunking, the token counter, and the debug handler that is later used to calculate the cost.
3. Register Phoenix tracing, so every run is recorded as a trace that you can inspect in the Phoenix web interface.
4. Create the cross-encoder reranker. Its model weights are downloaded the first time this happens.
5. Connect to Neo4j with the credentials from `.env`. On connection, the graph store also checks the database version and creates the constraints and the vector index it needs, if they do not exist yet.
6. Count the nodes in the Neo4j database to decide whether the graph still has to be built.
7. If the database has no nodes, fetch the transcripts, chunk them, let the LLM extract relationships from every chunk, and store the nodes, relationships, and embeddings in Neo4j. Otherwise, load the index straight from the existing graph.
8. Create the PostgreSQL memory and the context chat engine, which ties together retrieval, reranking, memory, and the LLM.
9. Run an interactive loop in the terminal until you type an exit word. Empty input is rejected with a warning.
10. For every question, print the answer, the source nodes with their scores, the token usage, and the estimated cost.
11. If anything fails after that, print the traceback and a categorized summary, then exit with code 1.

### ✨ Key Features
* 🎬 **Self-written transcript loader.** Fetches captions with `youtube-transcript-api` and builds the LlamaIndex documents itself, with the video id and the URL stored as metadata.
* 🕸️ **LLM-built knowledge graph.** The transcript is split into 2048-token chunks with a 200-token overlap. Each chunk is sent to Gemini in its own call, and up to 15 relationships are extracted from it. The relationships are stored in Neo4j as a property graph.
* 🧠 **Graph retrieval with source text.** Questions are matched to the graph, and with `include_text=True` each retrieved item combines the matched relationships with the source text they came from, so the LLM sees the original wording and not only short relationships.
* ♻️ **Build once.** If the Neo4j database already has nodes, the script skips the download and the extraction and loads the existing graph, which keeps later runs fast and cheap.
* 🎯 **Retrieve, then rerank.** Retrieves from the graph with `similarity_top_k=5`, then narrows the results to the top 3 with a `cross-encoder/ms-marco-MiniLM-L-2-v2` reranker, so only the best passages reach the LLM.
* 💬 **Persistent conversational memory.** Chat history is stored in PostgreSQL through `Memory` and AsyncPG, so a follow-up question still makes sense after you restart the script.
* 🧱 **Grounded by instruction.** A strict system prompt tells the model to answer only from the retrieved context and to admit when it does not have the information.
* 🔍 **Transparent answers.** The retrieved source nodes are printed with their scores and text after every response, so each answer can be checked against the passages behind it.
* 📊 **Token and cost tracking.** Token counts come from LlamaIndex's callback handler, and the cost of each turn is estimated through LiteLLM. A single question can involve more than one LLM call, and the numbers add up the calls recorded during the turn.
* 🛰️ **Phoenix tracing.** Every run sends its traces to the Arize Phoenix project `youtube_graph_project`, where each step of a question can be inspected.
* 🛡️ **Categorized error handling.** Failures print the raw traceback for debugging, then a readable summary such as a quota, authentication, database, or access error.

---

## 🎯 Context & Problem Statement

### 📖 The Problem

Video is where a lot of knowledge lives today, and it is hard to work with. You cannot press `Ctrl+F` on something that was spoken, and scrubbing through an hour of video to find one explanation is slow. Pasting a transcript into a general chatbot brings its own problems. Four of them show up when you try to build something better.

1. **A video cannot be searched like text, and ideas in it are connected.** Finding the part that explains why something works, or how two ideas connect, means watching or skimming. Searching for similar passages finds text that resembles the question, but it does not record how the ideas in the video relate to each other. Those relationships can be spread across different parts of a video.
2. **A general chatbot answers from memory, not from the video.** Asked about a specific video, a plain LLM may rely on old training data, mix in other sources, or guess. It also has no duty to say "that was not in the video", so you cannot tell which parts of its answer came from the video you care about.
3. **Naive pipelines are wasteful and forgetful.** A basic fetch-and-ask script downloads the same video on every run, and for a graph it would also pay for the relationship extraction again each time. It also loses the whole conversation when the process exits, so every session starts from zero.
4. **It is hard to see what an answer was based on and what it cost.** Without printed sources and token counts, you cannot check an answer against the video, and you cannot tell how many tokens a question used or what it cost.

### 💡 The Solution

Each problem maps to one piece of the pipeline, and each piece has a known gap.

| Problem | How this repo handles it | What is still missing |
| :--- | :--- | :--- |
| 1. Video cannot be searched, and ideas are connected | The transcript is split into chunks, and an LLM extracts the relationships in each one into a Neo4j graph. At question time the graph is searched, and the source text of the matches comes along | The relationships come from an LLM, so the graph can be incomplete or wrong. Only the captions are used, so videos without captions cannot be read, and the transcript is requested in the default language, which is English |
| 2. Answers from memory | Retrieved nodes are reranked by a cross-encoder, a strict system prompt limits the answer to them, and the source nodes are printed so the answer can be checked | The limit is an instruction to the model and not a hard check on its output, so the printed sources remain the way to verify |
| 3. Wasteful and forgetful | The download and the extraction are skipped when the database already has nodes, and the chat history lives in PostgreSQL, so repeated use stays cheap and continuous | The check counts every node in the Neo4j database, not only the ones from this script, and the session id is fixed in the script, so every run continues the same conversation |
| 4. Hidden sources and cost | Source nodes, token counts, and a per-turn cost estimate are printed, and Phoenix records a trace of every run | Token counts use an OpenAI tokenizer, so they are approximate for Gemini, the cost covers LLM calls only, and the calls made while building the graph are not included |

---

## 🧠 Core Concepts Explained

If RAG or graphs are new to you, this table explains the terms used in the rest of the README.

| Concept | Plain-language explanation | Where you see it |
| :--- | :--- | :--- |
| **RAG** | Retrieval-Augmented Generation. First find the passages that are relevant to the question, then give them to the LLM together with the question, so the answer is based on your source and not only on what the model remembers | The whole script |
| **Transcript** | The captions of a video as text. Here it is fetched with `youtube-transcript-api` and joined into one continuous text per video | `load_youtube_transcripts` |
| **Document** | A LlamaIndex object that holds text and metadata. Each video becomes one document, with its `video_id` and `video_url` stored as metadata | `Document(...)` |
| **Chunking** | Splitting long text into smaller pieces, because extraction and prompting work better with short passages. `chunk_size=2048` is counted in tokens, and `chunk_overlap=200` repeats the last 200 tokens at the start of the next chunk, so a sentence cut at the boundary keeps its context | `SentenceSplitter` |
| **Knowledge graph** | A way to store knowledge as things and the relationships between them, instead of as running text. For example, "Gemini" and "embedding model" can be two nodes with a relationship between them | The Neo4j database |
| **Relationship (triplet)** | One fact written as subject, relation, object. The extractor asks the LLM for these, and each one becomes a relationship between two nodes in the graph | `SimpleLLMPathExtractor` |
| **Property graph** | A graph whose nodes and relationships can carry properties, such as a name or text. It is the graph model that LlamaIndex stores in Neo4j here | `PropertyGraphIndex` |
| **Neo4j** | A database made for graphs. It stores nodes and relationships directly and is queried with Cypher | `Neo4jPropertyGraphStore` |
| **Cypher** | Neo4j's query language. The script uses one Cypher query, `MATCH (n) RETURN COUNT(n) AS total`, to count the nodes and decide whether the graph has to be built | `graph_store.structured_query(...)` |
| **Embedding** | A list of numbers that represents the meaning of a text. Texts with similar meaning get similar numbers, which is what makes search by meaning possible. A graph index embeds its nodes by default | `models/gemini-embedding-2` |
| **Index** | The LlamaIndex object that builds the graph from documents, or loads an existing graph, and searches it | `PropertyGraphIndex.from_documents`, `PropertyGraphIndex.from_existing` |
| **Retrieval and top-k** | Finding the parts of the graph that relate to the question. By default, LlamaIndex combines two lookups for a graph index: the LLM is asked for keywords and synonyms of the question, which are matched against the graph, and the question's embedding is matched against the embedded graph nodes. `similarity_top_k=5` is passed on to the retrievers. These are LlamaIndex defaults that the script does not override, and they can change between LlamaIndex versions | `as_chat_engine(..., similarity_top_k=5)` |
| **`include_text`** | An option that makes the retriever return the source text the relationships came from. By LlamaIndex's default, each returned item then holds a short preamble, the relationships found in that chunk, and the text of the chunk | `include_text=True` |
| **Source node** | An item that the retriever returned and that reached the LLM. Here it holds relationships and the source text they came from, and the score printed next to it is the reranker's score | `ai_response.source_nodes` |
| **Reranker (cross-encoder)** | A second, more careful scoring step. The first search is fast and approximate. The cross-encoder then reads the question and each retrieved item together and re-scores them, keeping only the best `top_n` | `SentenceTransformerRerank` (`top_n=3`) |
| **Context chat engine** | A chat engine that, for every message, retrieves context, adds it to the prompt, and remembers the exchange | `chat_mode="context"` |
| **System prompt** | A standing instruction given to the LLM before the conversation starts. Here it says to answer only from the retrieved context | `ai_system_prompt` |
| **`Memory`** | LlamaIndex's conversation memory. With `async_database_uri`, the history is stored in a database table, and `token_limit` caps how much history is kept | `Memory.from_defaults(...)` |
| **LiteLLM** | A library that offers one interface to many LLM providers. Here it calls Gemini and also provides the cost calculation | `LiteLLM`, `litellm.completion_cost` |
| **Trace** | A step-by-step record of what happened inside a run. Arize Phoenix collects traces and shows them in a web interface | `register(...)`, `LlamaIndexInstrumentor` |
| **`TokenCountingHandler`** | A LlamaIndex callback that counts the tokens sent to and received from the LLM | `token_counter` |
| **`LlamaDebugHandler`** | A LlamaIndex callback that records LLM inputs and outputs. The script reads its events to calculate the cost | `debug_handler` |
| **`nest_asyncio`** | Allows async code to run in environments where an event loop is already running | `nest_asyncio.apply()` |

---

## 🧩 Tech Stack Highlights

| Layer | Tool | Role |
| :--- | :--- | :--- |
| **Transcript source** | `youtube-transcript-api` | Fetches the captions of each video |
| **Orchestration** | LlamaIndex core (`Settings`, `Document`, `PropertyGraphIndex`, `SentenceSplitter`) | Global configuration, chunking, and index construction |
| **Graph extraction** | `SimpleLLMPathExtractor` (up to 15 relationships per chunk, 5 workers) | Uses the LLM to turn each chunk into relationships |
| **Graph storage** | Neo4j (`Neo4jPropertyGraphStore`) | Stores the nodes, relationships, and embeddings, and serves the retrieval |
| **Text embeddings** | `GoogleGenAIEmbedding` (`models/gemini-embedding-2`) | Embeds the graph nodes and each question |
| **Reranking** | `SentenceTransformerRerank` (`cross-encoder/ms-marco-MiniLM-L-2-v2`) | Rescores the retrieved nodes and keeps the top 3 |
| **LLM** | LiteLLM with `gemini/gemini-2.5-flash` (temperature 0.3) | Relationship extraction and answer generation |
| **Chat engine** | LlamaIndex `context` chat mode | Retrieves context on every turn and injects it into the prompt |
| **Conversation memory** | `Memory` + AsyncPG (PostgreSQL) | Persistent chat history across runs |
| **Observability** | Arize Phoenix (`phoenix.otel`, OpenInference LlamaIndex instrumentor) | Traces of every run |
| **Metrics** | `TokenCountingHandler` with `tiktoken`, `LlamaDebugHandler`, `litellm.completion_cost` | Token counts and the cost estimate |
| **Config & runtime** | `python-dotenv`, `nest-asyncio`, `asyncio` | Loads secrets from `.env` and runs the async loop |

---

## ⚙️ Configuration Reference

Every tunable value lives directly in `Graph-self-make-youtube-reader-cmd.py`, so changing a setting means editing the script.

| Setting | Value | Effect |
| :--- | :--- | :--- |
| **Target videos** | `youtube_urls = ["https://www.youtube.com/watch?v=i3OYlaoj-BM"]` | The videos that are fetched and turned into a graph on a fresh run |
| **Transcript language** | The default of `youtube-transcript-api` (English) | The transcript is requested without a language option, so a video needs an English transcript |
| **Chunking** | `chunk_size=2048`, `chunk_overlap=200` | How the transcript is split before extraction |
| **Graph extractor** | `SimpleLLMPathExtractor(llm=Settings.llm, max_paths_per_chunk=15, num_workers=5)` | Up to 15 relationships are extracted from each chunk, with 5 parallel workers |
| **Embedding model** | `models/gemini-embedding-2`, `embed_batch_size=50` | Embeds the graph nodes and each question, 50 texts per request |
| **Retrieval depth** | `similarity_top_k=5` | Passed on to the graph retrievers for each question |
| **Source text** | `include_text=True` | Each retrieved item combines the relationships with the source text they came from |
| **Reranker** | `cross-encoder/ms-marco-MiniLM-L-2-v2`, `top_n=3` | Number of retrieved items (source nodes) that reach the LLM |
| **LLM** | `gemini/gemini-2.5-flash`, `temperature=0.3` | The model for extraction and answers, and its randomness (a lower value gives more focused answers) |
| **LLM fallback** | A commented-out block in the script | Model fallback and retries are disabled. The block lists a Gemini model and a Groq model as fallbacks |
| **Chat mode** | `context` | Retrieves context on every message |
| **Graph store** | `Neo4jPropertyGraphStore` with `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_URI`, `NEO4J_DATABASE` | Where the graph is stored and read from |
| **Store setup** | Done by `Neo4jPropertyGraphStore` when it connects | Reads the schema (which needs APOC), checks the database version, and creates uniqueness constraints and a vector index named `entity` if they do not exist. The vector index is only created on Neo4j 5.23 or newer |
| **Build check** | `MATCH (n) RETURN COUNT(n) AS total`, then `total > 0` | If the database has any node at all, the script loads the existing graph and skips the build |
| **Chat memory** | `session_id="youtube_graph_session"`, `token_limit=40000`, `table_name="youtube_graph_memory"` | How the chat history is stored and trimmed |
| **Phoenix project** | `youtube_graph_project` | The project that receives the traces |
| **Phoenix endpoint** | `PHOENIX_COLLECTOR_ENDPOINT`, with a hardcoded default | Where traces are sent. The default points to the author's Phoenix space |
| **Token counting** | `tiktoken` with `cl100k_base` | The tokenizer behind the token numbers |
| **Cost format** | Six decimal places | The estimated cost of each turn, for example `$0.004638 USD` |
| **Debug trace** | `LlamaDebugHandler(print_trace_on_end=True)` | Prints a trace in the terminal after each run |
| **Logging** | The root logger and the `httpx`, `google_genai`, and `litellm` loggers are set to `WARNING` | Keeps the terminal readable |
| **Checked at startup** | `GOOGLE_API_KEY`, `GROQ_API_KEY`, `PHOENIX_API_KEY` | The script stops with an error if any of them is missing |
| **Not checked at startup** | `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_DATABASE`, `ASYNCPG_DATABASE_URL` | They are passed straight to their clients |
| **Exit commands** | `exit`, `close`, `out`, `break`, `quit`, `x` | Ends the session |

---

## 💡 Questions to Try

The first three questions are the ones used in the [benchmark repository](https://github.com/viochris/rag-token-benchmark-vector-graph-hybrid), in increasing order of difficulty. They are generic enough to ask about any video.

| Question | What it shows |
| :--- | :--- |
| `What is this video mainly about?` | A simple question that needs the overall topic of the video |
| `What are the main points or steps explained in this video, and how do they relate to each other?` | An analytical question that needs several parts of the transcript and how they connect |
| `Based on everything discussed in this video, what conclusion or key takeaway can be drawn, and what supporting details from different parts of the video lead to that conclusion?` | A multi-hop question that has to connect details from different parts of the video |
| `Explain that in simpler terms`, right after any answer | The memory at work, because the follow-up only makes sense with the previous answer |
| A question the video never covers | The grounding instruction at work, where the model should say it does not have the information |

---

## 🖥️ Example Console Output

An illustrative first-run session, based on the messages the script prints. Text in angle brackets stands for values that change from run to run.

```text
⚙️ [SYSTEM] Configuring LlamaIndex Settings (LLM & Embeddings)...
📊 [SYSTEM] Initializing Phoenix Observability & Tracing...
⚙️ [SYSTEM] Initializing Cross-Encoder Reranker...
⏳ [SYSTEM] Connecting to Neo4j Graph Database...
⏳ [SYSTEM] Databases are empty. Initializing data ingestion and graph extraction...
📥 [SYSTEM] Fetching YouTube transcripts...
🧠 [SYSTEM] Extracting Property Graph (Nodes & Triplets) using LLM...
<progress bars while the graph is built>
✅ [SYSTEM] Property Graph successfully created and stored.
🧠 [SYSTEM] Setting up Chat Engine with PostgreSQL memory...
✅ [SYSTEM] Chat Engine successfully integrated and ready for interaction.

======================================================================
💬 [SYSTEM] Interactive Graph Agent Session Started. Type 'exit' to stop.
======================================================================

🗣️ [USER] You: <your question about the video>
🧠 [AI] Querying the knowledge graph...
----------------------------------------------------------------------
🤖 [AI RESPONSE]:
<the answer, based on the retrieved graph context>
----------------------------------------------------------------------

🔍 [SOURCE NODES] Retrieved <number> relevant nodes from the graph:
--- Node 1 (score: <rerank score>) ---
📄 Text Context: <the source node text: a short preamble, the relationships found in the chunk, then the chunk text>

📊 [LLAMAINDEX CALLBACK TOKEN OVERVIEW]:
Model Used        : <model>
Prompt Tokens     : <number>
Completion Tokens : <number>
Total LLM Tokens  : <number>
======================================================================

💰 [COST ANALYSIS]:
Total Estimated Cost: $<amount> USD
======================================================================
```

On later runs, the ingestion lines are replaced by `📂 [SYSTEM] Existing data found. Loading existing index...` and `✅ [SYSTEM] Index successfully loaded from existing storage.`

---

## 📁 Repository Structure

```text
youtube-graph-rag-llamaindex/
├── Graph-self-make-youtube-reader-cmd.py
├── LICENSE
├── README.md
└── requirements.txt
```

| Path | Description |
| :--- | :--- |
| `Graph-self-make-youtube-reader-cmd.py` | The full pipeline in one script: transcript loading, graph extraction, retrieval, reranking, chat engine, and the CLI loop |
| `requirements.txt` | All Python dependencies, with notes inside the file that explain why each one is needed |
| `LICENSE` | MIT License |
| `README.md` | You're reading it |

---

## 🏗️ Architecture & Data Flow

### 🔄 End-to-End System Flowchart

```mermaid
flowchart TD
    A["YouTube URLs from the youtube_urls list"] --> B{"Does the Neo4j database already have nodes?"}

    B -- "No" --> C
    B -- "Yes" --> H["PropertyGraphIndex.from_existing loads the stored graph"]

    subgraph Ingestion ["Runs only if the database has no nodes"]
        C["load_youtube_transcripts fetches the captions and builds Documents"] --> D["SentenceSplitter chunks the text, 2048 size and 200 overlap"]
        D --> E["SimpleLLMPathExtractor asks the LLM for up to 15 relationships per chunk"]
        E --> E2["GoogleGenAIEmbedding embeds the graph nodes"]
        E2 --> F["Neo4j stores nodes, relationships, and embeddings"]
    end

    F --> G["PropertyGraphIndex"]
    H --> G

    Q["Question in the terminal"] --> I["Graph retriever finds related parts of the graph and adds their source text, similarity_top_k is 5"]
    G --> I
    I --> J["Cross-encoder reranker keeps the top 3"]
    J --> K["Context chat engine"]
    L["PostgreSQL memory through AsyncPG"] --> K
    M["Gemini LLM through LiteLLM"] --> K
    K --> N["Answer based on the retrieved graph context"]
    N --> O["Answer, source nodes, tokens, and cost are printed"]
```

> **Note.** On every run, the script first counts the nodes in the Neo4j database. If the count is above zero, nothing is downloaded or extracted again, and the stored graph is loaded. The ingestion path in the diagram therefore runs only once for an empty database. Phoenix records a trace for each run, and the token counter and debug handler observe every LLM call. The numbers printed after each answer are reset at the start of the chat session, so the LLM calls made during extraction are not part of them.

---

## 🆚 Three RAG Architectures

This repository implements **Graph RAG**. The same task of answering questions about a video can also be done with plain similarity search over embeddings, or with both approaches combined. That makes three versions of the same chatbot, one of which is still to come, plus a repository that measures what each one costs. This section explains how they differ, where each one lives, and what the benchmark found.

### 📍 Where Each Architecture Lives

| Architecture | Approach | Location |
| :--- | :--- | :--- |
| **Vector RAG** | Similarity search over embeddings | [youtube-vector-rag-llamaindex](https://github.com/viochris/youtube-vector-rag-llamaindex) |
| **Graph RAG** | Search over a knowledge graph | This repository (`Graph-self-make-youtube-reader-cmd.py`) |
| **Hybrid RAG** | Vector search and graph search combined | Coming soon |
| **Token and cost benchmark** | Compares the three on the same video and questions | [rag-token-benchmark-vector-graph-hybrid](https://github.com/viochris/rag-token-benchmark-vector-graph-hybrid) |

### 🔍 What Differs

The Hybrid column follows the setup used in the benchmark repository.

| Aspect | Vector RAG | Graph RAG (this repo) | Hybrid RAG (coming soon) |
| :--- | :--- | :--- | :--- |
| How context is found | The question is compared with the chunk embeddings, and the closest chunks are used | The text is turned into relationships (subject, relation, object) kept in a graph, and the question is answered from that graph together with the source text | Both retrievers run, and their results are merged with Reciprocal Rank Fusion. The LLM also writes extra variants of the question before the search |
| Storage | ChromaDB Cloud | Neo4j | ChromaDB Cloud and Neo4j |
| Extra LLM work when indexing | None, only embeddings | An LLM extracts the relationships from every chunk | The same as Graph RAG |
| Reranking | Cross-encoder, top 5 to top 3 | The same cross-encoder | The same cross-encoder |
| Moving parts | Fewest | More | Most |

### 📊 About the Benchmark Repository

[rag-token-benchmark-vector-graph-hybrid](https://github.com/viochris/rag-token-benchmark-vector-graph-hybrid) builds the three architectures end to end on the same YouTube transcript and asks each of them the same three questions, the simple, analytical, and multi-hop ones listed under [Questions to Try](#-questions-to-try). Every turn reports prompt, completion, and total tokens plus an estimated cost in USD. All runs use `gemini-2.5-flash` with fallback disabled, so the model, the source, and the questions stay the same across the three.

Results from that repository, shown as total tokens and estimated cost:

| Question | Vector RAG | Graph RAG | Hybrid RAG |
| :--- | :---: | :---: | :---: |
| Q1 (Simple) | 5,858 tokens, $0.002618 | 4,959 tokens, $0.004638 | 9,208 tokens, $0.010026 |
| Q2 (Analytical) | 7,318 tokens, $0.005876 | 7,989 tokens, $0.010416 | 10,408 tokens, $0.010901 |
| Q3 (Multi-hop) | 8,754 tokens, $0.005338 | 9,836 tokens, $0.008787 | 12,871 tokens, $0.012668 |

In estimated cost, the order was Vector, then Graph, then Hybrid on every question. In total tokens, the same order held for the analytical and multi-hop questions. On the simple question, Graph used fewer total tokens than Vector (4,959 against 5,858) but still cost more, because its completions were longer.

These numbers come from one video, three questions, and one run per question, with costs estimated through LiteLLM. The benchmark measures tokens and cost only. It does not measure how good the answers are. The Graph column comes from the benchmark repository's runs, so it is a reference for this version and not a measurement made with this script alone.

### 🧭 Which One Fits Your Case

* If you want the simplest setup and the lowest cost, Vector RAG is the place to start, and the [Vector repository](https://github.com/viochris/youtube-vector-rag-llamaindex) shows how it works end to end.
* Graph RAG, the version in this repository, adds a Neo4j database and an LLM step at indexing time, so that retrieval can follow explicit relationships. The benchmark shows what it costs per question.
* Whether Graph or Hybrid RAG answers better than Vector RAG was not measured, so the choice between them still needs a quality comparison.

---

## 💻 Installation & Reproduction Steps

### 📋 Prerequisites
* **Python 3.10 or newer** is required.
* **Accounts and keys** you will need: a Google AI Studio API key for Gemini, an [Arize Phoenix](https://phoenix.arize.com/) API key, a PostgreSQL database for the conversation memory, and a Neo4j database for the graph. A hosted PostgreSQL database, for example one on Supabase, works with its connection string. The script also requires `GROQ_API_KEY` to be set, because its startup check looks for it. See the note under the variable table below.
* **A Neo4j database, version 5.23 or newer, with the APOC plugin enabled.** The graph store reads the schema through APOC procedures, and it creates its vector index only on Neo4j 5.23 or newer. The database can be hosted or local.
* **Internet access**, because the script calls YouTube, Gemini, Neo4j, Phoenix, and PostgreSQL. On the first run it also downloads the reranker weights from Hugging Face and the tokenizer file that `tiktoken` needs, so the first start takes longer than later ones.

### 🛠️ CLI Installation & Execution

#### 1. Clone the Repository
```bash
git clone https://github.com/viochris/youtube-graph-rag-llamaindex.git
cd youtube-graph-rag-llamaindex
```

#### 2. Create and Activate a Virtual Environment
A virtual environment keeps this project's packages separate from the rest of your system.
```bash
# macOS/Linux
python3 -m venv venv
source venv/bin/activate

# Windows
python -m venv venv
venv\Scripts\activate
```

#### 3. Install Dependencies
```bash
pip install --upgrade pip
pip install -r requirements.txt
```
> `torch` is a large download, so the first installation can take a while. The LlamaIndex ecosystem also ships as many small, fast-moving sub-packages. If you hit a resolver conflict, try installing into a clean virtual environment and letting `pip` resolve every package from `requirements.txt` in a single command rather than one at a time, since that gives it the full picture needed to pick mutually compatible versions.

#### 4. Configure Environment Variables
Create a `.env` file in the project root.
```env
GOOGLE_API_KEY=your_google_ai_studio_key
GROQ_API_KEY=your_groq_key
PHOENIX_API_KEY=your_phoenix_key
PHOENIX_COLLECTOR_ENDPOINT=https://app.phoenix.arize.com/s/your-space/v1/traces
NEO4J_URI=neo4j+s://your-instance.databases.neo4j.io
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your_neo4j_password
NEO4J_DATABASE=neo4j
ASYNCPG_DATABASE_URL=postgresql+asyncpg://user:password@host:5432/dbname
```

| Variable | Purpose |
| :--- | :--- |
| `GOOGLE_API_KEY` | Gemini key. The embedding model receives it directly, and LiteLLM finds its Gemini key in the environment for the LLM calls |
| `GROQ_API_KEY` | Only checked at startup. The Groq model appears only in the commented-out fallback block, so no active call uses it |
| `PHOENIX_API_KEY` | Authenticates the trace upload to Phoenix |
| `PHOENIX_COLLECTOR_ENDPOINT` | Optional. Where traces are sent |
| `NEO4J_URI` | Connection address of the Neo4j database, for example a `neo4j+s://...` address of a hosted instance or `bolt://localhost:7687` for a local one |
| `NEO4J_USERNAME` | Neo4j user name |
| `NEO4J_PASSWORD` | Neo4j password |
| `NEO4J_DATABASE` | Name of the Neo4j database to use, usually `neo4j` |
| `ASYNCPG_DATABASE_URL` | Async connection string of the PostgreSQL database that holds the chat memory table |

> The script has a default value for `PHOENIX_COLLECTOR_ENDPOINT` that points to the author's own Phoenix space. Set this variable to your own space, otherwise your traces will not appear in your account.
>
> Only `GOOGLE_API_KEY`, `GROQ_API_KEY`, and `PHOENIX_API_KEY` are checked at startup, and the check only looks for a non-empty value. The Neo4j and PostgreSQL variables are not checked, so a mistake there only shows up when that client is first used. Keep `.env` out of version control.

#### 5. Set the Target Videos
The video URLs are set directly in the script.
```python
youtube_urls = ["https://www.youtube.com/watch?v=i3OYlaoj-BM"]
```
Edit this list before the first run. The video needs captions, and the transcript is requested in the default language, which is English.

#### 6. Run the Application
```bash
python Graph-self-make-youtube-reader-cmd.py
```
Type your questions at the `🗣️ [USER] You:` prompt. Type `exit`, `close`, `out`, `break`, `quit`, or `x` to end the session. On the first run, the graph is built before the chat starts. This takes a while, because the LLM is called once for every chunk to extract relationships.

#### 7. Look at the Graph (Optional)
After the first run, you can see part of the graph in the Neo4j Browser, or in the query tool of your Neo4j provider, with this Cypher query:
```cypher
MATCH (n)-[r]->(m) RETURN n, r, m LIMIT 25
```

#### 8. Switch to Another Video or Start Over
The graph is only built when the Neo4j database has no nodes, so changing `youtube_urls` has no effect while the database still holds any node. Empty the database once, and the next run builds the graph for the new list. The chat history stays in PostgreSQL, so also change `session_id` in the script, or clear the `youtube_graph_memory` table, if you do not want the old conversation to carry over.
```python
import os
from dotenv import load_dotenv
from neo4j import GraphDatabase

load_dotenv()
with GraphDatabase.driver(
    os.getenv("NEO4J_URI"),
    auth=(os.getenv("NEO4J_USERNAME"), os.getenv("NEO4J_PASSWORD")),
) as driver:
    driver.execute_query("MATCH (n) DETACH DELETE n", database_=os.getenv("NEO4J_DATABASE"))
```
> This deletes **every** node and relationship in that Neo4j database, including data from any other project that uses the same database. The same query also works in the Neo4j Browser.

---

## 🛠️ Troubleshooting & Error Messages

### 🚨 Error Categories Printed by the Script

When something fails after the startup key check, the script prints the traceback and then one of these summaries, and exits with code 1. The checks run from top to bottom, and the first match is used. A missing `GOOGLE_API_KEY`, `GROQ_API_KEY`, or `PHOENIX_API_KEY` is raised before this handler starts, so it appears as a normal Python error with its message and no summary.

| Message prefix | Matches when |
| :--- | :--- |
| `🚨 [API ERROR]` | The error type is `ResourceExhausted`, or the message contains `quota` or `429` |
| `🚨 [DATA ERROR]` | A `ValueError` whose message contains `empty` |
| `🚨 [AUTH ERROR]` | The error type is `InvalidArgument`, or the message contains `api_key` |
| `🚨 [DATABASE ERROR]` (Neo4j) | The message contains `neo4j`, `serviceunavailable`, or `auth` |
| `🚨 [DATABASE ERROR]` (PostgreSQL) | The message contains `asyncpg` or `postgres` |
| `🚨 [ACCESS ERROR]` | A `PermissionError` |
| `🚨 [UNKNOWN ERROR]` | Anything that does not match the categories above |

The Neo4j and PostgreSQL checks match on the text of the message, and the Neo4j check runs first. A PostgreSQL message that contains `auth`, such as a failed password authentication, is therefore reported as a Neo4j error. A Neo4j connection error whose message has none of the three words is reported as `UNKNOWN ERROR`. In both cases, the printed traceback shows the real cause.

### 🧰 Common Problems

| Symptom | Likely cause | What to do |
| :--- | :--- | :--- |
| `GOOGLE_API_KEY is missing`, `GROQ_API_KEY is missing`, or `PHOENIX_API_KEY is missing` at startup | The variable is missing from `.env`, or the file is not in the project root | Add the variable to `.env` in the project root and run the script again |
| LiteLLM reports a missing Gemini key | LiteLLM did not find the key in the environment | Also set `GEMINI_API_KEY` in `.env` with the same value as `GOOGLE_API_KEY` |
| `UNKNOWN ERROR` mentioning a transcript | The video has no captions, captions are disabled, or no English transcript exists | Choose a video that has English captions |
| `UNKNOWN ERROR` mentioning an invalid YouTube URL | The URL does not contain `youtube` or `youtu.be`, and has no `v=` parameter | Use a standard `youtube.com` or `youtu.be` link |
| YouTube refuses the request | YouTube can block requests from some cloud provider and VPN IP addresses | Run the script from a regular home or office connection |
| The script says existing data was found, but the answers come from an old video or lack context | The Neo4j database already has nodes, from an earlier run or from another project, so the build is skipped | Empty the database as shown in step 8, or use a separate Neo4j database for this project |
| The first run stopped halfway, and the next run skips the build | The check only counts nodes, so a database that was partly filled counts as built | Check the database for nodes before running again, and empty it as shown in step 8 if the graph is incomplete |
| `API ERROR` (quota or `429`) during the first run | The graph build makes one LLM call per chunk, with 5 workers in parallel, which can reach the rate limit of your Gemini plan | Wait for the limit to reset and run again, or check the plan's limits |
| A Neo4j error mentioning `apoc` or `db.index.vector` | The database does not have the APOC plugin, or it is older than 5.23, so the vector index is not created | Use Neo4j 5.23 or newer with APOC enabled |
| Neo4j authentication or connection error | `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, or `NEO4J_DATABASE` is wrong, or the database is not running | Check the values in `.env` against the connection details of your Neo4j instance |
| `ImportError` asking for `sentence-transformers` or `torch` | The reranker dependencies are not installed | Run `pip install -r requirements.txt` in the active virtual environment |
| The first run is slow before the chat starts | The reranker weights are downloaded from Hugging Face, and the graph is built with LLM calls | Wait for both to finish. Later runs reuse the weights and the stored graph |
| Traces do not show up in your Phoenix account | `PHOENIX_COLLECTOR_ENDPOINT` still points to the default space | Set it to your own Phoenix endpoint |
| Error about the async driver | `asyncpg` is not installed, or `ASYNCPG_DATABASE_URL` does not start with `postgresql+asyncpg://` | Run `pip install -r requirements.txt` and check the URL |
| The chat remembers an old conversation | The memory uses a fixed `session_id` | Change `session_id` in the script, or clear the `youtube_graph_memory` table |
| Long trace output appears after each answer | `LlamaDebugHandler` is created with `print_trace_on_end=True` | Set it to `False` in the script |
| A database error appears for a problem that is not about the database | Any message containing `neo4j`, `serviceunavailable`, or `auth` is reported as a Neo4j error, and a PostgreSQL authentication failure can be one of them | Read the printed traceback to find the real cause |

---

## 📝 Conclusion

This project set out to solve four problems: a video that cannot be searched like text and whose ideas are connected, a general chatbot that answers from memory instead of from the video, a naive pipeline that downloads and forgets everything on every run, and answers whose sources and cost stay hidden. The script addresses them directly. It fetches the transcript with its own loader, turns each chunk into relationships stored in a Neo4j graph, retrieves from that graph together with the source text, reranks what it retrieves, tells the model to answer only from that context, and keeps both the graph and the conversation so repeated use stays cheap and continuous. Every answer is printed with its source nodes, its token usage, and its estimated cost.

The design choices hold up reasonably well on their own terms. Skipping the build when the database has nodes avoids paying for the same download and the same relationship extraction twice. Returning the source text together with the relationships keeps the original wording in front of the model. Reranking with a small cross-encoder before the LLM is a deliberate quality step, because it filters the retrieved nodes before they take up space in the prompt. Keeping the whole flow in one readable script makes it easy to follow, run, and modify.

It is also not a finished product. The video list is hardcoded, changing it requires emptying the database by hand, the transcript carries no timestamps, the graph quality depends on what the LLM extracts, and nothing measures how good the answers are. The sections below go through these gaps and lay out the next steps.

---

## ⚠️ System Limitations

### 🏗️ Architectural Limitations
* **Hardcoded target videos.** The URLs are set by editing the `youtube_urls` list in the script, not passed in as an argument, so switching videos means opening the code.
* **One-time build gate that counts every node.** The graph is only built when the Neo4j database has no nodes at all. Changing the list without emptying the database does not trigger a new build. The same happens when the database holds nodes from another project, so the script skips the build and answers from the wrong graph, without any warning.
* **One shared graph and one fixed conversation.** Every video goes into the same graph, and the memory uses one `session_id` and one table name. Every run continues the same conversation, even after you switch videos, so old topics can mix into new answers.
* **Any error ends the session.** After the startup key check, the whole pipeline, including the chat loop, sits inside one top-level `try` block. A transient failure on a single turn, such as an API rate limit, is reported but still stops the script, and you have to start it again.
* **Settings are hardcoded.** The model names, chunk settings, extractor limits, retrieval depth, Phoenix project name, and memory settings are fixed values in the script.
* **Startup validation does not match the real needs.** The script requires `GROQ_API_KEY` even though no active code uses it, because the fallback that would use it is commented out. The Neo4j and PostgreSQL variables, which the script does need, are not checked, so a mistake there only shows up when that client is first used.
* **Model fallback is disabled.** The fallback and retry options are commented out, so a failed model call is not retried and is not sent to another model. This also applies to the calls made while the graph is built.
* **Dependence on external services.** The script needs live access to YouTube, Gemini, Neo4j, Phoenix, and PostgreSQL. There is no offline mode.
* **Specific Neo4j requirements.** The graph store needs the APOC plugin and, to create its vector index, Neo4j 5.23 or newer, so not every Neo4j instance will work.
* **The cost of building the graph is not reported.** The LLM calls that extract the relationships are made before the chat starts, and the token and cost counters are reset when the session starts. The printed numbers therefore describe the chat turns only.
* **CLI-only interface.** All interaction happens in a terminal. There is no web page, no visual chat history, and no built-in view of the graph.

### 🔬 Model & Domain Limitations
* **Captions only.** A video without captions cannot be read, and automatic captions can contain mistakes that end up in the graph and in the answers. The transcript is requested in the default language, which is English.
* **No timestamps.** The loader joins the caption snippets into plain text, so answers cannot point to a moment in the video.
* **Simple URL parsing.** The video id is taken from the URL with string rules, so unusual link formats may fail or give a wrong id.
* **The graph is only as good as the extraction.** An LLM reads each chunk and writes up to 15 relationships with a simple prompt. The script defines no entity or relationship types, so the graph holds whatever the LLM wrote, it can miss facts, and the same idea can appear under different names. If the extractor cannot read the LLM's reply for a chunk, LlamaIndex's default is to log the error and continue, so that chunk adds no relationships while the script carries on.
* **Retrieval on every message.** The context chat mode retrieves for every message, including greetings and thanks. The default graph retriever also asks the LLM for related keywords before it searches, so a single message can use more than one LLM call.
* **Grounding relies on the prompt.** Staying within the retrieved context is enforced by the system prompt and the retrieval setup, not by a hard check on the output, so the model can still go beyond the retrieved passages.
* **Fixed retrieval settings.** Chunk size, overlap, `similarity_top_k`, and the reranker's `top_n` are fixed. A broad question that needs many parts of a long video is limited by only three source nodes reaching the LLM.
* **Local reranking.** The cross-encoder runs on your machine, so it needs `torch`, a one-time weight download, and some compute on every turn.
* **Token counts are approximate.** `TokenCountingHandler` uses the `cl100k_base` tokenizer from OpenAI, while the model is Gemini, so the numbers can differ from what the provider counts.
* **The cost covers LLM calls only.** Embedding calls are not included, and the estimate depends on the response data that LiteLLM receives.
* **No quality evaluation.** Retrieval and answer quality were not measured against a labeled question set, so there is no number for how often the script retrieves the right passage or answers correctly, and no comparison with the Vector version.

---

## 🚀 Future Work
* **Take the video URLs as an argument.** Accept one or more URLs on the command line, so the code does not need editing to switch videos.
* **One graph and one session per video.** Keep each video's graph separate, for example in its own database or by filtering on the `video_id` property that the extractor copies from the document metadata, and name the memory session after the video id, so switching videos needs no manual cleanup and old conversations do not mix in.
* **Check for this project's data only.** Decide whether to build the graph by looking for the nodes this script created, instead of counting every node in the database.
* **Report the cost of building the graph.** Keep the token and cost numbers of the extraction step, so the full cost of a video is visible and not only the cost of the chat turns.
* **Keep timestamps.** Store the start time of each caption in the metadata, so answers can point to the moment in the video.
* **Support more transcript languages.** Let the user choose the language, or fall back to another one when English is missing.
* **Make the chat loop resilient.** Move error handling inside the loop, so a failed turn prints the error and lets you try again instead of ending the session.
* **Enable model fallback.** Turn on the retry and fallback options through the LiteLLM configuration, so a temporary model failure does not stop a turn, and remove the `GROQ_API_KEY` check if no Groq model is used.
* **Guide the extraction with a schema.** Define the entity and relationship types the graph should use, so the extraction is more consistent across chunks.
* **Move configuration out of the code.** Take model names, chunk settings, and retrieval depth from environment variables or arguments.
* **Evaluate retrieval and answers.** Build a small labeled question set for a few videos and measure retrieval and answer quality, and compare the Graph version with the Vector version, instead of relying on fixed defaults.
* **Add a simple interface.** A small Streamlit page would make the answers and their sources easier to read than terminal output.

---

## 📄 License
This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---
**Author:** [Silvio Christian, Joe](https://github.com/viochris)

*"Ask a video a question, and let its knowledge graph answer."*
