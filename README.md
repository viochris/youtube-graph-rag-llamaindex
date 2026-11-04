# youtube-graph-rag-llamaindex
Graph RAG chatbot for YouTube videos using LlamaIndex. Fetches the transcript with a self-written loader, extracts a knowledge graph with Gemini into Neo4j, reranks with a cross-encoder, and answers from the video in a CLI chat with PostgreSQL memory. Includes Phoenix tracing, token counting, and cost tracking. Graph version of a RAG series.
