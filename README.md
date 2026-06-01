# Resume Screening RAG Pipeline

A Retrieval-Augmented Generation (RAG) resume screening assistant for helping hiring teams search, compare, and analyze candidate resumes against job descriptions.

The current runnable demo is a Streamlit application that uses:

- FAISS for vector search over resume chunks
- Hugging Face sentence-transformer embeddings
- LangChain for document loading, splitting, retrieval, and LLM integration
- Ollama or Gemini for chat and RAG Fusion query generation
- Pandas for resume CSV loading and uploaded dataset handling

The repository also contains research notebooks, generated test sets, evaluation outputs, PDF resume conversion utilities, and prebuilt vector indexes.

## Demo Preview

The Streamlit app opens directly into a chat-style resume screening assistant.

Resume Screening GPT starting screen

When the user provides a job description, the system retrieves matching candidates and produces a ranked hiring summary.

Job description response example

When the user asks about a specific applicant ID, the system performs an exact lookup and analyzes that candidate.

Applicant ID response example

## Local Run Verification

The project was run locally with the repository virtual environment and Streamlit:

```powershell
cd "C:\projects\resume rag\Resume-Screening-RAG-Pipeline-main"
.\.venv\Scripts\python.exe -m streamlit run demo/interface.py --server.port 8501 --server.headless true --server.fileWatcherType none
```

Verified locally:

- Streamlit served the app at http://localhost:8501.
- The app loaded the default `data/main-data/synthetic-resumes.csv` dataset.
- The FAISS vectorstore and Hugging Face embedding model initialized.
- Ollama had llama3:latest available.
- The UI rendered the sidebar, model controls, RAG mode selector, resume upload control, title, and chat input.
- A sample query, `Find Python developers with machine learning experience`, completed retrieval and generated a ranked candidate answer.
- The updated uploader was verified with support for both CSV and PDF files.

Startup screen from the local run:

Local Streamlit startup screen

Completed sample query from the local run:

Local Streamlit query result

Updated upload control with CSV/PDF support:

CSV and PDF upload control

## What The App Does

The assistant supports three main query paths:

**Job description search**

- Detects hiring-style prompts such as "Find Python developers with ML experience".
- Retrieves relevant resume chunks from FAISS.
- Maps chunks back to full resumes.
- Computes a keyword, bigram, and skill-density match score.
- Returns ranked candidates and asks the LLM to summarize the top choices.

**Applicant ID lookup**

- Detects numeric applicant IDs with at least three digits.
- Fetches exact matching candidate resumes from the active dataset.
- Produces a structured candidate analysis.

**General recruitment questions**

- Skips retrieval when no job description, skill query, or applicant ID is detected.
- Answers using the recent chat history and the local LLM.

## Current Demo Features

- Streamlit chat interface
- LLM provider switcher for Ollama and Gemini
- Configurable Ollama and Gemini model selections
- Generic RAG and RAG Fusion modes
- RAG Fusion sub-query generation
- Exact applicant ID retrieval
- Resume upload through CSV or single PDF files
- Automatic FAISS vectorstore rebuild if vectorstore is missing
- Conversation clearing
- Retrieval verbosity panel showing:
  - query type
  - selected RAG mode
  - retrieved resumes
  - generated sub-queries
  - reciprocal-rank-fusion scores
  - elapsed time

## Architecture

The application is built around a retrieval layer and a generation layer. The retrieval layer decides whether the user query needs resume context, finds relevant candidates, and prepares candidate data. The generation layer uses the retrieved resume text plus the user's request to produce a recruiter-friendly answer.

Chatbot structure

At a lower level, resumes are embedded into a FAISS vectorstore. For job descriptions, the query is embedded, candidate chunks are retrieved, and RAG Fusion can expand the original query into multiple focused sub-queries before re-ranking.

RAG pipeline

Runtime Architecture

Data And Indexing Architecture

RAG Fusion Architecture

## How It Works End To End

1. The user submits a message in the Streamlit chat UI.
2. `SelfQueryRetriever.retrieve_docs()` classifies the message.
3. If the message contains applicant IDs, the retriever directly searches the active dataframe by ID.
4. If the message looks like a job description or skill search, the retriever uses FAISS similarity search.
5. If RAG Fusion is selected, `ChatBot.generate_subquestions()` asks Llama3 to split the job description into 3-4 focused search queries.
6. The vectorstore retrieves candidate chunks for each query.
7. Candidate chunk IDs are combined and re-ranked with reciprocal rank fusion.
8. The selected IDs are mapped back to full resume text from the dataframe.
9. Each full resume receives a readable match score based on keyword overlap, bigram overlap, and skill density.
10. The ranked resumes are sent to Llama3 as context.
11. Llama3 streams a structured answer back to Streamlit.
12. The verbosity panel shows what was retrieved, what sub-queries were used, and how long the run took.

This flow gives the LLM concrete resume context instead of asking it to answer from memory.

## Retrieval Logic In Detail

### Query classification

`detect_query_type()` checks the user message in this order:

- A number with at least three digits means applicant ID lookup.
- Hiring keywords such as `find`, `hire`, `candidate`, `developer`, `engineer`, `manager`, or `years of experience` mean job description retrieval.
- Known skill terms such as `python`, `react`, `sql`, `tensorflow`, `aws`, `docker`, or `scrum` also trigger job description retrieval.
- Anything else is treated as a general recruitment question.

### Generic RAG

Generic RAG uses the original user query only:

```
User job description -> FAISS search -> candidate IDs -> full resumes -> LLM response
```

This is faster and simpler.

### RAG Fusion

RAG Fusion expands the original query:

```
User job description
-> Llama3 generated sub-queries
-> FAISS search for each query
-> reciprocal rank fusion
-> candidate IDs
-> full resumes
-> LLM response
```

This can improve retrieval when the job description is long, broad, or contains several different requirements.

### Match scoring

The displayed match score is not the raw FAISS score. It is a presentation score computed from:

- unigram keyword overlap
- bigram phrase overlap, weighted higher than single words
- a small skill-density bonus

This makes the ranked output easier for users to read while FAISS still handles semantic retrieval.

## Generation Logic In Detail

The app uses different prompts depending on the query type:

- **Job description retrieval**: return the top candidates, preserve ranking order, choose the best candidate, and explain why.
- **Applicant ID lookup**: summarize strengths, weaknesses, and an overall recommendation.
- **General question**: answer as a recruitment assistant using recent chat history.

Responses are streamed through Streamlit with `st.write_stream()`, so the answer appears progressively.

## Runtime Data Structures

### Session state

`demo/interface.py` keeps long-lived objects in `st.session_state` so Streamlit reruns do not recreate expensive resources on every interaction.

| Key | Created in | Purpose |
|---|---|---|
| chat_history | interface.py | Stores HumanMessage, AIMessage, and verbosity render tuples. |
| resume_list | interface.py | Stores the latest retrieved resume context list. |
| embedding_model | interface.py | Reuses the Hugging Face embedding model. |
| df | interface.py | Active resume dataframe, either default synthetic data or uploaded CSV. |
| rag_pipeline | interface.py | Active SelfQueryRetriever connected to the active vectorstore and dataframe. |
| llm | interface.py | Cached ChatBot wrapper around the selected provider/model. |
| llm_provider | Sidebar selectbox | Current LLM backend: Ollama or Gemini. |
| ollama_model | Sidebar selectbox | Current local Ollama model. |
| gemini_model | Sidebar selectbox | Current Gemini API model. |
| llm_signature | interface.py | Provider/model tuple used to refresh the cached LLM only when needed. |
| rag_selection | Streamlit selectbox | Current mode: Generic RAG or RAG Fusion. |

### Retriever metadata

`SelfQueryRetriever` updates a `meta_data` dictionary on every retrieval. The verbosity UI reads this object.

```json
{
    "rag_mode": "",
    "query_type": "no_retrieve",
    "extracted_input": "",
    "subquestion_list": [],
    "retrieved_docs_with_scores": []
}
```

- `rag_mode`: selected Streamlit RAG mode.
- `query_type`: one of `retrieve_applicant_jd`, `retrieve_applicant_id`, or `no_retrieve`.
- `extracted_input`: parsed IDs or job description payload.
- `subquestion_list`: original query plus generated sub-queries when RAG Fusion is used.
- `retrieved_docs_with_scores`: fused candidate ID scores after retrieval.

### Retrieved document format

For job description retrieval, each selected resume is formatted like this before it is passed to the LLM:

```
Applicant ID: <id>
Match Score: <score>%
Skills: <comma-separated skills>

<full resume text>
```

For applicant ID lookup, the format is simpler:

```
Applicant ID <id>
<full resume text>
```

## Function-By-Function Walkthrough

### demo/interface.py

This is the Streamlit entry point and runtime coordinator. It does not define custom functions, but its top-level blocks act as the application lifecycle.

| Block | What it does | Why it matters |
|---|---|---|
| Imports | Loads Streamlit, Pandas, LangChain message types, FAISS, embeddings, ChatBot, ingest, and SelfQueryRetriever. | Connects the UI, retrieval, vectorstore, and model layers. |
| Config constants | Defines DATA_PATH, FAISS_PATH, EMBEDDING_MODEL, model lists, and provider defaults. | These control the default resume CSV, default local FAISS folder, embedding model, and selectable LLM backends. |
| Page setup | Calls st.set_page_config(), st.title(), and st.caption(). | Gives the app its visible title and multi-provider positioning. |
| chat_history init | Creates an empty chat list when missing. | Preserves conversation turns across Streamlit reruns. |
| resume_list init | Creates an empty list for latest retrieved resumes. | Lets the app keep retrieval context available after a query. |
| embedding_model init | Builds HuggingFaceEmbeddings(model_name="sentence-transformers/all-MiniLM-L6-v2"). | Embeddings are expensive, so the app caches them in session state. |
| df init | Reads data/main-data/synthetic-resumes.csv. | Provides the default resume corpus. |
| rag_pipeline init | Loads FAISS from vectorstore; if loading fails, rebuilds with ingest(). | Makes the app resilient when the index is missing. |
| LLM defaults init | Creates default provider/model values from .env or built-in defaults. | Lets the app start with Ollama by default while allowing Gemini configuration. |
| Sidebar | Provides provider/model selection, RAG mode selection, CSV upload, clear conversation, examples, and about text. | Gives non-code controls for changing retrieval behavior, model backend, and dataset. |
| llm init | Creates or refreshes one ChatBot(provider, model) instance. | Avoids recreating the LLM wrapper unless the selected provider/model changes. |
| Upload branch | Validates uploaded CSV columns, builds a new FAISS index in memory, and swaps the active retriever/dataframe. | Lets users test their own resumes without editing files. |
| Chat history rendering | Replays HumanMessage, AIMessage, and verbosity tuples. | Keeps the chat transcript visible after reruns. |
| Chat input branch | Runs retrieval, shows retrieved candidates, streams the LLM response, renders verbosity, and appends history. | This is the main request-response loop. |

### demo/retriever.py

This file contains query routing, scoring, and retrieval.

**Constants**

| Name | Purpose |
|---|---|
| RAG_K_THRESHOLD = 5 | Number of candidate chunks retrieved per query before fusion. |
| SKILLS | Known skill list used for skill-triggered retrieval and skill extraction. |
| JD_KEYWORDS | Hiring/job-description keywords used to detect retrieval-worthy user prompts. |

**detect_query_type(question: str)**

Classifies the user message.

| Return type | Parsed payload | When it happens |
|---|---|---|
| retrieve_applicant_id | {"id_list": [...]} | The message contains one or more 3+ digit numbers. |
| retrieve_applicant_jd | {"job_description": question} | The message contains hiring keywords or known skills. |
| no_retrieve | {} | No resume retrieval signal is detected. |

**compute_score(query: str, resume_text: str) -> float**

Computes a readable match percentage for presentation.

**extract_skills(text: str) -> list**

Lowercases resume text and returns every skill from SKILLS that appears in the resume.

**class RAGRetriever**

Base retriever that knows how to search FAISS and map retrieved IDs back to full resumes.

**class SelfQueryRetriever(RAGRetriever)**

The application-level retriever used by Streamlit.

### demo/llm_agent.py

This file wraps the selected LLM backend. Ollama is the default local provider, and Gemini is available when `langchain-google-genai` and an API key are configured.

### demo/ingest_data.py

This file builds a FAISS vectorstore from a dataframe using `chunk_size=1000` and `chunk_overlap=150`.

### demo/chatbot_verbosity.py

This file renders the debug/trace panel under each assistant response.

## Project Structure

```
Resume-Screening-RAG-Pipeline-main/
|-- README.md
`-- Resume-Screening-RAG-Pipeline-main/
    |-- README.md
    |-- requirements.txt
    |-- demo/
    |   |-- interface.py
    |   |-- llm_agent.py
    |   |-- retriever.py
    |   |-- ingest_data.py
    |   |-- chatbot_verbosity.py
    |   `-- interactive/
    |       |-- convert_pdf.py
    |       `-- ingest_data.py
    |-- preprocessing/
    |-- evaluation/
    |-- data/
    |-- vectorstore/
    |-- vectorstore-pdf/
    `-- vectorstore-synthetic/
```

## Requirements

- Python 3.10 or 3.11 recommended
- Ollama installed locally
- At least one Ollama model pulled locally, such as `llama3`
- Gemini API key if using the Gemini provider

## Installation

```powershell
cd "C:\projects\resume rag\Resume-Screening-RAG-Pipeline-main"
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

Copy the example environment file if you want to configure defaults:

```powershell
copy .env.example .env
```

Do not commit your real `.env` file.

## Ollama Setup

Install Ollama, then pull Llama3:

```powershell
ollama pull llama3
```

Make sure the Ollama service is running before starting the Streamlit app:

```powershell
ollama serve
```

If Ollama is already running as a desktop/background service, you do not need to run `ollama serve` again.

## Gemini Setup

Gemini support is optional. Set one of these environment variables in `.env`:

```
GOOGLE_API_KEY=your_google_api_key_here
# or
GEMINI_API_KEY=your_google_api_key_here
```

## Run The Demo

```powershell
cd "C:\projects\resume rag\Resume-Screening-RAG-Pipeline-main"
streamlit run demo/interface.py
```

Streamlit will print a local URL, usually:

```
http://localhost:8501
```

## Example Queries

- Find Python developers with machine learning experience
- Hire a senior backend engineer with API and SQL skills
- Who has TensorFlow experience?
- Show applicant 101
- Compare the top 3 candidates

## Uploading Your Own Resumes

Use the sidebar upload control with either a CSV or a single PDF resume.

CSV uploads must contain:

```
ID,Resume
101,"Resume text here..."
102,"Another resume text here..."
```

## Rebuilding The Default Vectorstore

```powershell
# Stop Streamlit, then remove the vectorstore folder:
Remove-Item -Recurse -Force "C:\projects\resume rag\Resume-Screening-RAG-Pipeline-main\vectorstore"
# Start Streamlit again — it will rebuild automatically
streamlit run demo/interface.py
```

## Troubleshooting

**ModuleNotFoundError: torchvision**

```powershell
pip install torchvision
```

**ModuleNotFoundError: langchain_huggingface**

```powershell
pip install langchain-huggingface
```

**ModuleNotFoundError: langchain_google_genai**

```powershell
pip install langchain-google-genai
```

**Ollama connection error**

```powershell
ollama list
ollama pull llama3
```

**Slow first run**

The first run may download the embedding model and build or load FAISS indexes. Later runs will be faster.

## License

Apache License 2.0

## Acknowledgements

The original project was created as a resume screening RAG proof of concept and was inspired by RAG Fusion.
