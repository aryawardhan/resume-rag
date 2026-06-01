# Resume Screening RAG Pipeline

A Retrieval-Augmented Generation (RAG) resume screening assistant for helping hiring teams search, compare, and analyze candidate resumes against job descriptions.

The current runnable demo is a Streamlit application that uses:

- FAISS for vector search over resume chunks
- Hugging Face sentence-transformer embeddings
- LangChain for document loading, splitting, retrieval, and LLM integration
- Ollama or Gemini for chat and RAG Fusion query generation
- Pandas for resume CSV loading and uploaded dataset handling

## Demo Preview

The Streamlit app opens directly into a chat-style resume screening assistant.

![Resume Screening GPT starting screen](https://github.com/Hungreeee/Resume-Screening-RAG-Pipeline/assets/46376260/3a7122d5-1c8e-4d98-bb06-cbc28813a2c3)

When the user provides a job description, the system retrieves matching candidates and produces a ranked hiring summary.

![Job description response example](https://github.com/Hungreeee/Resume-Screening-RAG-Pipeline/assets/46376260/d3e47a4e-257c-47d6-a12e-73e48dacc137)

When the user asks about a specific applicant ID, the system performs an exact lookup and analyzes that candidate.

![Applicant ID response example](https://github.com/Hungreeee/Resume-Screening-RAG-Pipeline/assets/46376260/94081148-b99f-40d9-b665-b5cbb7e15123)

## Requirements

- Python 3.10 or 3.11 recommended
- Ollama installed locally
- At least one Ollama model pulled locally, such as `llama3`
- Gemini API key if using the Gemini provider
- Enough disk space for embeddings, FAISS indexes, and the included datasets

Install Python packages from inside the project directory:

```powershell
git clone https://github.com/aryawardhan/resume-rag.git
cd resume-rag
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

Gemini support is optional. It is only required when the sidebar provider is set to `Gemini`.

1. Install dependencies from `requirements.txt`.
2. Set one of these environment variables in `.env` or your shell:

```env
GOOGLE_API_KEY=your_google_api_key_here
# or
GEMINI_API_KEY=your_google_api_key_here
```

3. Optionally choose a default Gemini model:

```env
LLM_PROVIDER=Gemini
GEMINI_MODEL=gemini-3.5-flash
```

## Run The Demo

From the project directory:

```powershell
cd resume-rag
streamlit run demo/interface.py
```

Streamlit will print a local URL, usually:

```text
http://localhost:8501
```

## What The App Does

The assistant supports three main query paths:

1. **Job description search** — detects hiring-style prompts, retrieves matching resume chunks from FAISS, scores candidates, and returns a ranked summary.
2. **Applicant ID lookup** — detects numeric IDs and fetches exact matching candidate resumes.
3. **General recruitment questions** — answers using recent chat history without retrieval.

## Example Queries

```text
Find Python developers with machine learning experience
Hire a senior backend engineer with API and SQL skills
Who has TensorFlow experience?
Show applicant 101
Compare the top 3 candidates
```

## Uploading Your Own Resumes

Use the sidebar upload control with either a CSV or a single PDF resume.

CSV uploads must contain:

```csv
ID,Resume
101,"Resume text here..."
102,"Another resume text here..."
```

## Project Structure

```text
resume-rag/
├── README.md
├── requirements.txt
├── demo/
│   ├── interface.py              # Streamlit app entry point
│   ├── llm_agent.py              # Ollama/Gemini prompting and streaming
│   ├── retriever.py              # Query detection, scoring, RAG retrieval
│   ├── ingest_data.py            # FAISS vectorstore builder
│   ├── chatbot_verbosity.py      # Retrieval/debug UI panel
│   └── interactive/
│       ├── convert_pdf.py        # Convert PDF resumes to CSV
│       └── ingest_data.py        # Environment-driven FAISS ingestion script
├── preprocessing/
├── evaluation/
├── data/
│   ├── main-data/
│   └── supplementary-data/
└── vectorstore/
```

## Rebuilding The Default Vectorstore

To force a rebuild manually:

1. Stop Streamlit.
2. Rename or remove the `vectorstore` directory.
3. Start Streamlit again.

The rebuilt vectorstore will be based on `data/main-data/synthetic-resumes.csv`.

## PDF Resume Conversion

`demo/interactive/convert_pdf.py` converts PDF files from:

```text
data/supplementary-data/pdf-resumes/
```

into:

```text
data/supplementary-data/pdf-resumes.csv
```

## Troubleshooting

**`ModuleNotFoundError: langchain_huggingface`**
```powershell
pip install langchain-huggingface
```

**`ModuleNotFoundError: langchain_google_genai`**
```powershell
pip install langchain-google-genai
```

**Ollama connection error** — confirm your model is installed:
```powershell
ollama list
ollama pull llama3
```

**FAISS load error** — let the app rebuild the index from the resume CSV automatically.

**Slow first run** — the first run may download the embedding model and build FAISS indexes. Later runs will be faster.

## License

Apache License 2.0

## Acknowledgements

Originally based on [sgsatpute/Resume-Screening-RAG-Pipeline](https://github.com/sgsatpute/Resume-Screening-RAG-Pipeline), inspired by RAG Fusion.