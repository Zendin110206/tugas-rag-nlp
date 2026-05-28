# RAG Parameter Exploration

This repository contains my Infinite Learning Task 15 assignment for exploring Retrieval-Augmented Generation (RAG) parameters using a FastAPI-based RAG Playground.

The project focuses on how retrieval and generation parameters affect answer quality, context relevance, latency, and consistency when asking questions over a PDF document.

## Assignment Context

- Program: AI Development - Infinite Learning
- Assignment: Task 15 - Project: RAG Parameter Exploration
- Topic: Natural Language Processing
- Base project: `rag-playground`
- Final repository: `tugas-rag-nlp`

## Project Summary

The application lets users upload PDF files, ingest them into a lightweight RAG pipeline, and ask questions through a local web interface.

The pipeline follows this flow:

```text
PDF upload -> text extraction -> chunking -> embedding -> vector search -> optional reranking -> LLM answer
```

The experiments in this README evaluate six main parameters:

```python
CHUNK_SIZE = 300
TOP_K_INITIAL = 20
FINAL_TOP_M = 5
USE_RERANKER = True
TEMPERATURE = 0.2
GROQ_MODEL = "openai/gpt-oss-120b"
```

## Sample Document

I used a custom sample PDF included in this repository:

```text
sample_documents/task15_original_story_rag_test_document.pdf
```

The document is a multi-page original narrative designed for RAG testing. It contains named characters, dates, objects, timeline details, and explicit RAG parameter references. This makes it useful for testing both factual retrieval and broader context reasoning.

Baseline ingestion with `CHUNK_SIZE = 300` produced around 13 chunks. A larger chunk size such as `600` produced fewer chunks, confirming that chunk size directly changes how the document is stored in the knowledge base.

## Local Setup

Create and activate a virtual environment:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

Install dependencies:

```powershell
pip install -r requirements.txt
```

Create a local `.env` file from `.env.example` and add your Groq API key:

```text
GROQ_API_KEY=your_groq_api_key_here
```

Run the local server:

```powershell
python -m uvicorn app:app --reload --host 127.0.0.1 --port 8000
```

Open the app:

```text
http://127.0.0.1:8000
```

## Experiment Protocol

For each experiment, I changed one parameter at a time in `app.py`, restarted or reloaded the server, uploaded the same sample PDF with `Replace` enabled, and asked the same baseline questions.

Baseline questions:

```text
Who owns the blue tin pencil case?
What happened on 12 February 2018?
Which RAG parameters are listed in the document?
```

This controlled setup keeps the comparison fair because only one variable changes at a time.

---

## 1. CHUNK_SIZE

```python
CHUNK_SIZE = 300
```

### What is CHUNK_SIZE?

When a PDF is uploaded, the extracted text is split into smaller pieces called chunks. `CHUNK_SIZE` controls how many words are stored in each chunk before embeddings are created.

### Function

`CHUNK_SIZE` defines the size of each document segment used for retrieval. Smaller chunks make retrieval more focused, while larger chunks give the model more surrounding context.

### Effects

| Value Range      | Effect                                                                                                                                                                       |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Small (100-200)  | Produces more chunks and more focused retrieval. It is useful for specific factual questions, but broader context can be split across multiple chunks.                       |
| Medium (250-400) | Provides a balanced trade-off between precision and context. In my sample document, the baseline around 300 kept enough story context while still keeping retrieval focused. |
| Large (500-1000) | Produces fewer but longer chunks. This can help narrative or summary questions, but retrieval may include extra unrelated information.                                       |

### Reflection

For my sample document, a medium chunk size around 300 is the most appropriate. The document contains both specific facts and a narrative timeline, so the system needs enough surrounding context without making each retrieved passage too noisy.

If an answer feels cut off or lacks context, I would first increase `CHUNK_SIZE`. If the answer still misses important details, I would also review `FINAL_TOP_M`, because that parameter controls how many final chunks are sent to the LLM.

---

## 2. TOP_K_INITIAL

```python
TOP_K_INITIAL = 20
```

### What is TOP_K_INITIAL?

After the user question is converted into an embedding, the system searches for chunks with similar embeddings. `TOP_K_INITIAL` controls how many initial candidate chunks are retrieved before filtering or reranking.

### Function

`TOP_K_INITIAL` controls the retrieval search width. A higher value gives the reranker more candidate chunks to evaluate, while a lower value keeps retrieval faster but may miss useful context.

### Effects

| Value Range    | Effect                                                                                                                                                              |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Small (5-10)   | Faster retrieval, but lower recall. Important chunks may be missed if they are not included in the initial candidates.                                              |
| Medium (15-25) | Balanced setting. With reranking enabled, this gives enough candidates for better context selection without excessive overhead.                                     |
| Large (30-50)  | Higher recall, but potentially slower. For small documents, this can be redundant because the total number of chunks may already be lower than the requested top-k. |

### Reflection

For my sample document, a small top-k value such as 5 can still answer simple factual questions, but it is less reliable for broader questions because the candidate pool is narrow.

`TOP_K_INITIAL` and `USE_RERANKER` are closely connected. `TOP_K_INITIAL` decides which chunks enter the candidate pool, while the reranker decides which of those candidates are most relevant. If the best chunk never enters the initial pool, the reranker cannot recover it.

---

## 3. FINAL_TOP_M

```python
FINAL_TOP_M = 5
```

### What is FINAL_TOP_M?

After candidates are retrieved and optionally reranked, only the top final chunks are sent to the LLM as context. `FINAL_TOP_M` controls how many chunks are included in that final prompt.

### Function

`FINAL_TOP_M` determines how much retrieved evidence the LLM receives. It directly affects answer completeness, prompt length, and generation latency.

### Effects

| Value Range  | Effect                                                                                                            |
| ------------ | ----------------------------------------------------------------------------------------------------------------- |
| Few (2-3)    | More focused and usually faster, but important context can be left out.                                           |
| Medium (4-6) | Balanced setting. It usually provides enough context without overloading the prompt.                              |
| Many (7-10)  | More complete context, but slower and potentially noisier because the LLM receives more information to reconcile. |

### Reflection

More context is not always better. If the additional chunks are relevant, increasing `FINAL_TOP_M` can improve completeness. If the additional chunks are only loosely related, the answer can become longer, less focused, or slightly confusing.

If final contexts contradict each other, the LLM may merge conflicting information or choose one side without explicitly explaining why. That is why retrieval quality and reranking are important before increasing the number of final contexts.

---

## 4. USE_RERANKER

```python
USE_RERANKER = True
```

### What is a Reranker?

A reranker is an additional model that evaluates the relationship between a question and each candidate chunk. It is more precise than relying only on vector similarity because it scores each `[question, chunk]` pair directly.

### Function

`USE_RERANKER` controls whether the system reorders retrieved candidates before choosing the final contexts. When enabled, it improves context relevance at the cost of additional retrieval time.

### Effects

| Mode    | Effect                                                                                                                                                                             |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `True`  | Usually improves precision and context relevance, especially for ambiguous or multi-detail questions. Retrieval can be slower because the reranker performs an extra scoring step. |
| `False` | Faster because the system relies only on vector similarity. However, the final contexts can be less precise if the embedding search returns loosely related chunks.                |

### Reflection

The reranker is most useful for questions where multiple chunks look similar but only some directly answer the question. For example, a question about `12 February 2018` may retrieve several timeline-related chunks, but the reranker helps prioritize chunks that actually describe the meeting event.

The added latency depends on the number of initial candidates and the machine running the reranker. In this project, the trade-off is acceptable because the goal is answer quality, not only speed.

---

## 5. TEMPERATURE

```python
TEMPERATURE = 0.2
```

### What is Temperature?

Temperature controls how deterministic or creative the LLM is when generating the next token. Lower values make answers more stable, while higher values make answers more varied.

### Function

`TEMPERATURE` affects the generation stage, not the retrieval stage. It changes how the model writes the answer after the relevant chunks have already been selected.

### Effects

| Value Range | Effect                                                                                              |
| ----------- | --------------------------------------------------------------------------------------------------- |
| 0.0-0.2     | Best for RAG-style QA. Answers are more factual, consistent, and grounded in the retrieved context. |
| 0.3-0.5     | Slightly more flexible wording while still usually staying on topic.                                |
| 0.6-0.8     | More variation and creativity, but less ideal for strict document QA.                               |
| 0.9-1.0     | Higher risk of drifting from the retrieved context or adding unsupported details.                   |

### Reflection

For company document QA, I would use a low temperature such as 0.1 to 0.3. The goal is to answer based on evidence, not to produce creative variations.

When temperature is raised to around 0.9, the answer can become more verbose or speculative. This is risky in RAG because the model should stay grounded in the uploaded document.

---

## 6. GROQ_MODEL

```python
GROQ_MODEL = "openai/gpt-oss-120b"
```

### What is GROQ_MODEL?

`GROQ_MODEL` selects which Groq-hosted LLM is used to generate the final answer from the retrieved context.

### Model Comparison

| Model                     | Size            | Speed and Quality Impact                                                                                                                                                        |
| ------------------------- | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `llama-3.1-8b-instant`    | 8B parameters   | Fast and suitable for development or quick testing. It is efficient, but may provide less detailed reasoning on complex questions.                                              |
| `openai/gpt-oss-120b`     | 120B parameters | Stronger reasoning and more detailed answers. It is suitable when answer quality matters more than raw speed. This is the baseline model used in my final configuration.        |
| `llama-3.3-70b-versatile` | 70B parameters  | A strong middle-to-large model option. It can provide high-quality answers while potentially being more practical than the largest model depending on availability and latency. |

### Reflection

The quality gain from larger models can be worth the additional latency when the use case requires accurate and well-structured answers. For quick development, a smaller model is useful. For a final RAG demonstration, I prefer `openai/gpt-oss-120b` because it provides stronger answer quality for narrative and parameter-based questions.

For this document QA use case, `openai/gpt-oss-120b` is the most suitable final choice because the sample document contains both factual details and broader reasoning questions.

---

## Additional Non-Tunable Parameters

```python
CHUNK_OVERLAP = 60
SCORE_THRESHOLD = 0.30
```

| Parameter         | Default | Function                                                                                                                                                                                                                             |
| ----------------- | ------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `CHUNK_OVERLAP`   |      60 | Repeats words between adjacent chunks so that important information near chunk boundaries is not lost. If too small, context can be cut off. If too large, chunks become repetitive and storage/retrieval can become less efficient. |
| `SCORE_THRESHOLD` |    0.30 | Filters out low-similarity chunks. If too high, relevant context may be removed. If too low, unrelated chunks may enter the final candidate pool.                                                                                    |

## Final Configuration

After the experiments, I restored the application to the balanced baseline:

```python
CHUNK_SIZE = 300
TOP_K_INITIAL = 20
FINAL_TOP_M = 5
USE_RERANKER = True
TEMPERATURE = 0.2
GROQ_MODEL = "openai/gpt-oss-120b"
```

This configuration is suitable for the sample document because it balances retrieval precision, context completeness, reranking quality, and grounded generation.

## Key Lessons Learned

- RAG quality depends heavily on how documents are chunked and retrieved before the LLM generates an answer.
- `CHUNK_SIZE`, `TOP_K_INITIAL`, `FINAL_TOP_M`, and `USE_RERANKER` mainly affect retrieval quality.
- `TEMPERATURE` and `GROQ_MODEL` mainly affect generation quality.
- Changing one parameter at a time is important because it makes the trade-off easier to understand.
- The best parameter value depends on document length, question type, and whether the priority is speed, completeness, or factual consistency.

## References

- Base project: <https://github.com/arifian853/rag-playground>
- Final repository: <https://github.com/Zendin110206/tugas-rag-nlp>
- Groq: <https://groq.com/>
