# Sample Documents

This folder contains public, non-private PDF documents for testing the RAG Playground assignment.

## Available Sample

| File | Purpose |
| --- | --- |
| `task15_original_story_rag_test_document.pdf` | A fictional long-form story designed for chunking, retrieval, reranking, and answer quality experiments. |

## Why This PDF Exists

The sample document is intentionally longer than a minimal test file. It includes:

- named characters and places;
- repeated motifs such as letters, trains, rain, maps, and stations;
- precise dates, numbers, and objects;
- a narrative arc across multiple sections;
- an embedded RAG experiment guide with parameter names.

This makes it useful for testing how different RAG parameters affect retrieval quality.

## Suggested Test Questions

Use these after the document has been ingested:

1. Who owns the blue tin pencil case?
2. What happened on 12 February 2018?
3. How many letters were exchanged before the reunion?
4. Why is the brass station clock important?
5. How did Raka's understanding of distance change?
6. Which RAG parameters are listed in the document?
7. Why might a small `CHUNK_SIZE` lose emotional context?
8. Why can `USE_RERANKER` help when several chunks mention trains and letters?

## Privacy Note

The PDF is fictional and contains no API keys, passwords, personal records, or private assignment data.
