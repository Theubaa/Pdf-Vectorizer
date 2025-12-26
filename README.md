# PDF Vectorizer – FastAPI + FAISS + Supabase

Production-ready browser-based document vectorization service that extracts text from PDFs, JSON, CSV, and Excel files, generates embeddings using OpenAI/Gemini/Claude, and stores vectors in Supabase for RAG applications.

## Features

- **Multi-format upload**: PDF, JSON, CSV, Excel (up to 1GB)
- **Semantic text extraction**: Layout-aware PDF parsing with section detection
- **Intelligent chunking**: Paragraph-aligned chunks with semantic continuity
- **Flexible embeddings**: OpenAI, Google Gemini, or Anthropic Claude
- **Vector storage**: Supabase pgvector + local FAISS index
- **RAG-ready**: Metadata-rich chunks optimized for retrieval
- **Web UI**: Simple upload interface with download capabilities

---

## System Requirements

### Ubuntu 20.04+ / 22.04+ (Recommended)

- **Python**: 3.10 or higher
- **RAM**: Minimum 4GB (8GB+ recommended for large PDFs)
- **Disk**: 10GB+ free space (for models and data)
- **Internet**: Required for downloading ML models and API calls

---

## Installation Guide (Fresh Ubuntu System)

### Step 1: Update System Packages

```bash
sudo apt update
sudo apt upgrade -y
```

### Step 2: Install Python and Build Tools

Check if Python 3 is installed:

```bash
python3 --version
```

If not installed or version < 3.10:

```bash
sudo apt install -y python3 python3-venv python3-pip
```

Install build essentials (required for some Python packages):

```bash
sudo apt install -y build-essential
```

### Step 3: Install Git

Check if Git is installed:

```bash
git --version
```

If not installed:

```bash
sudo apt install -y git
```

### Step 4: Clone the Repository

```bash
cd ~
git clone https://github.com/your-username/your-repo.git vectorizer-fleetwise
cd vectorizer-fleetwise
```

**Note**: Replace `https://github.com/your-username/your-repo.git` with your actual repository URL.

### Step 5: Create Virtual Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

You should see `(.venv)` in your terminal prompt.

### Step 6: Upgrade Pip

```bash
pip install --upgrade pip
```

### Step 7: Install Python Dependencies

```bash
pip install -r requirements.txt
```

**Note**: This may take 5-10 minutes as it downloads ML models and dependencies.

---

## Environment Configuration

### Step 1: Create `.env` File

In the project root directory, create a `.env` file:

```bash
touch .env
```

### Step 2: Configure Environment Variables

Open `.env` in a text editor (nano, vim, or VS Code):

```bash
nano .env
```

### Step 3: Add Required Variables (STRICT ORDER)

Copy and paste the following template into your `.env` file, then replace placeholder values with your actual credentials:

```env
# ============================================
# EMBEDDING PROVIDER CONFIGURATION
# ============================================
# Choose ONE provider: openai | gemini | google | claude | anthropic
EMBEDDING_PROVIDER=openai

# ============================================
# OPENAI (if EMBEDDING_PROVIDER=openai)
# ============================================
OPENAI_API_KEY=your_openai_api_key_here
# Optional: embedding model (default: text-embedding-3-small)
OPENAI_EMBEDDING_MODEL=text-embedding-3-small

# ============================================
# GOOGLE GEMINI (if EMBEDDING_PROVIDER=gemini or google)
# ============================================
GOOGLE_API_KEY=your_google_api_key_here
# Optional: Gemini embedding model (default: models/text-embedding-004)
GEMINI_EMBEDDING_MODEL=models/text-embedding-004

# ============================================
# ANTHROPIC CLAUDE (if EMBEDDING_PROVIDER=claude or anthropic)
# ============================================
ANTHROPIC_API_KEY=your_anthropic_api_key_here
# Optional: Anthropic embedding model
ANTHROPIC_EMBEDDING_MODEL=claude-3-haiku-20240307

# ============================================
# SUPABASE CONFIGURATION (REQUIRED)
# ============================================
# Your Supabase project URL (found in Supabase Dashboard > Settings > API)
SUPABASE_URL=https://your-project-id.supabase.co

# Your Supabase service role key (found in Supabase Dashboard > Settings > API > service_role key)
SUPABASE_SERVICE_KEY=your_supabase_service_role_key_here
```

### Step 4: Save and Exit

If using `nano`: Press `Ctrl+X`, then `Y`, then `Enter`.

### Step 5: Verify `.env` File

```bash
cat .env
```

Ensure:
- No spaces around `=` signs
- No quotes around values (unless the value itself contains spaces)
- All required variables are set

---

## Getting API Keys

### OpenAI API Key

1. Go to https://platform.openai.com/api-keys
2. Sign in or create an account
3. Click "Create new secret key"
4. Copy the key and paste it into `.env` as `OPENAI_API_KEY`

### Google Gemini API Key

1. Go to https://makersuite.google.com/app/apikey
2. Sign in with your Google account
3. Click "Create API Key"
4. Copy the key and paste it into `.env` as `GOOGLE_API_KEY`

### Anthropic Claude API Key

1. Go to https://console.anthropic.com/
2. Sign in or create an account
3. Navigate to API Keys section
4. Click "Create Key"
5. Copy the key and paste it into `.env` as `ANTHROPIC_API_KEY`

### Supabase Credentials

1. Go to https://supabase.com/
2. Sign in and create a new project (or use existing)
3. Wait for project setup to complete
4. Go to **Settings** → **API**
5. Copy **Project URL** → paste as `SUPABASE_URL`
6. Copy **service_role** key (not anon key) → paste as `SUPABASE_SERVICE_KEY`

**Important**: Use the `service_role` key, not the `anon` key, for server-side operations.

---

## Supabase Database Setup

### Step 1: Create Table

In your Supabase Dashboard, go to **SQL Editor** and run:

```sql
CREATE TABLE IF NOT EXISTS document_embeddings (
  id bigserial PRIMARY KEY,
  file_id text NOT NULL,
  chunk_id integer NOT NULL,
  section text,
  content text NOT NULL,
  embedding vector(1536) NOT NULL,  -- Adjust dimension based on your embedding model
  created_at timestamp with time zone DEFAULT now()
);

-- Create index for vector similarity search
CREATE INDEX IF NOT EXISTS document_embeddings_embedding_idx 
ON document_embeddings 
USING ivfflat (embedding vector_cosine_ops);

-- Optional: Create unique constraint to prevent duplicates
CREATE UNIQUE INDEX IF NOT EXISTS document_embeddings_file_chunk_unique 
ON document_embeddings (file_id, chunk_id);
```

**Note**: Adjust `vector(1536)` based on your embedding model:
- OpenAI `text-embedding-3-small`: 1536
- OpenAI `text-embedding-3-large`: 3072
- Google Gemini `text-embedding-004`: 768
- Check your provider's documentation for exact dimensions

### Step 2: Enable pgvector Extension

If not already enabled:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

---

## Running the Application

### Development Mode

With virtual environment activated:

```bash
uvicorn app:app --reload --host 0.0.0.0 --port 8000
```

Access the UI at: `http://localhost:8000` or `http://127.0.0.1:8000`

### Production Mode (Using Gunicorn)

Install Gunicorn:

```bash
pip install gunicorn
```

Run:

```bash
gunicorn app:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
```

---

## Domain Connection & API Guide

### Option 1: Using Nginx Reverse Proxy (Recommended)

#### Step 1: Install Nginx

```bash
sudo apt install -y nginx
```

#### Step 2: Configure Nginx

Create a configuration file:

```bash
sudo nano /etc/nginx/sites-available/vectorizer
```

Add the following (replace `your-domain.com` with your actual domain):

```nginx
server {
    listen 80;
    server_name your-domain.com www.your-domain.com;

    client_max_body_size 1G;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### Step 3: Enable Site

```bash
sudo ln -s /etc/nginx/sites-available/vectorizer /etc/nginx/sites-enabled/
sudo nginx -t  # Test configuration
sudo systemctl reload nginx
```

#### Step 4: Set Up SSL with Let's Encrypt

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com -d www.your-domain.com
```

Certbot will automatically configure HTTPS and renew certificates.

### Option 2: Using Cloudflare Tunnel (Alternative)

1. Install Cloudflare Tunnel:

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
```

2. Authenticate:

```bash
cloudflared tunnel login
```

3. Create tunnel:

```bash
cloudflared tunnel create vectorizer
```

4. Configure and run:

```bash
cloudflared tunnel route dns vectorizer your-domain.com
cloudflared tunnel run vectorizer
```

---

## API Endpoints

### Upload File

**POST** `/upload`

- **Content-Type**: `multipart/form-data`
- **Body**: `file` (PDF, JSON, CSV, Excel)
- **Max Size**: 1GB

**Response**:
```json
{
  "file_name": "document.pdf",
  "file_id": "document_abc123",
  "number_of_pages": 10,
  "number_of_chunks": 5,
  "raw_exists": true,
  "chunks_exist": true,
  "raw_download_url": "/download/raw/document_abc123",
  "chunks_download_url": "/download/chunks/document_abc123",
  "status": "ok"
}
```

### Download Raw Text

**GET** `/download/raw/{file_id}`

Returns the extracted raw text as a `.txt` file.

### Download Chunks

**GET** `/download/chunks/{file_id}`

Returns the semantic chunks as a `.json` file.

### Semantic Search

**POST** `/search`

- **Content-Type**: `application/x-www-form-urlencoded`
- **Body**: `query=your search query&top_k=5`

**Response**:
```json
{
  "results": [
    {
      "score": 0.85,
      "file_name": "document.pdf",
      "chunk_index": 2,
      "section": "Introduction",
      "text": "Relevant text chunk..."
    }
  ],
  "status": "ok"
}
```

### Preview Chunks

**GET** `/preview/chunks/{file_id}?limit=10`

Returns the first N chunks for inspection.

---

## Project Structure

```
vectorizer-fleetwise/
├── app.py                      # FastAPI application entry point
├── requirements.txt            # Python dependencies
├── .env                        # Environment variables (create this)
├── .gitignore                  # Git ignore rules
├── README.md                   # This file
│
├── services/
│   ├── pdf_loader.py          # PDF text extraction
│   ├── text_reconstructor.py  # Semantic text repair
│   ├── semantic_chunker.py    # Intelligent chunking
│   ├── embedder.py            # FAISS embeddings
│   ├── embedding_providers.py # OpenAI/Gemini/Claude providers
│   ├── vector_store_supabase.py # Supabase integration
│   ├── vector_ingestion.py    # Ingestion orchestrator
│   ├── json_extractor.py      # JSON file processing
│   ├── csv_extractor.py       # CSV file processing
│   └── excel_extractor.py     # Excel file processing
│
├── templates/
│   └── index.html             # Web UI
│
├── storage/                    # Generated at runtime
│   ├── uploads/               # Uploaded files
│   ├── raw_text/              # Extracted text
│   └── chunks/                # Semantic chunks
│
└── data/                       # Generated at runtime
    ├── reconstructed/         # Reconstructed documents
    ├── normalized/            # Normalized non-PDF data
    └── vectors/               # FAISS index + metadata
```

---

## Troubleshooting

### App crashes with "SUPABASE_URL is missing"

- Ensure `.env` file exists in the project root
- Check that `SUPABASE_URL` and `SUPABASE_SERVICE_KEY` are set
- Verify no extra spaces around `=` signs
- Restart the application after modifying `.env`

### Embedding provider errors

- Verify `EMBEDDING_PROVIDER` matches one of: `openai`, `gemini`, `google`, `claude`, `anthropic`
- Ensure the corresponding API key is set (e.g., `OPENAI_API_KEY` for `openai`)
- Check API key validity and account balance

### Supabase connection errors

- Verify `SUPABASE_URL` format: `https://your-project-id.supabase.co`
- Ensure you're using the `service_role` key, not `anon` key
- Check that the `document_embeddings` table exists
- Verify pgvector extension is enabled

### File upload fails

- Check file size is under 1GB
- Verify file format is supported (PDF, JSON, CSV, Excel)
- Check disk space availability

### Port already in use

If port 8000 is busy:

```bash
# Find process using port 8000
sudo lsof -i :8000

# Kill the process (replace PID with actual process ID)
sudo kill -9 PID

# Or use a different port
uvicorn app:app --reload --port 8001
```

---

## Security Notes

- **Never commit `.env` file** to version control
- Use `service_role` key only on the server (never expose to frontend)
- Keep API keys secure and rotate them periodically
- Use HTTPS in production (via Nginx + Let's Encrypt)
- Consider rate limiting for production deployments

---

## License

[Your License Here]

---

## Support

For issues, feature requests, or questions:
- Open an issue on GitHub
- Check existing issues and discussions

---

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request


1️⃣ SEO-OPTIMIZED PROJECT TITLE (CRITICAL)

❌ Current (too generic, zero keywords):

PDF Vectorizer – FastAPI + FAISS + Supabase


✅ Replace with (keyword-loaded but clean):

PDF Vectorizer for RAG – FastAPI, FAISS, Supabase Vector Search (OpenAI, Gemini, Claude)


This alone improves:

GitHub search

Google indexing

AI crawler understanding

2️⃣ SEO-OPTIMIZED META DESCRIPTION (USE IN README TOP)

Add this immediately under the title:

A production-ready PDF & document vectorization pipeline for RAG applications. Upload PDFs, CSV, JSON, or Excel files, generate embeddings using OpenAI, Gemini, or Claude, and store vectors in Supabase pgvector and FAISS. Built with FastAPI for semantic search, AI chatbots, and knowledge bases.

This hits:

PDF vectorization

RAG

embeddings

Supabase

semantic search

3️⃣ HIGH-INTENT KEYWORDS YOU MUST TARGET

These are what people actually search 👇
(You should naturally repeat them across README)

Primary Keywords

PDF vectorizer

PDF embeddings

RAG pipeline

Supabase vector search

FAISS vector database

FastAPI RAG backend

Secondary Keywords

Document vectorization service

Semantic PDF search

AI knowledge base backend

OpenAI embeddings Supabase

pgvector FastAPI

Chunking PDF for RAG

Long-Tail (VERY important)

How to build RAG pipeline with Supabase

PDF to vector database FastAPI

Store embeddings in Supabase pgvector

Semantic search backend for PDFs

FAISS + Supabase hybrid vector store

If these phrases aren’t present, Google won’t rank you.

4️⃣ SEO-OPTIMIZED INTRO (REPLACE YOUR CURRENT INTRO)
❌ Current intro

Too technical, no search intent, no benefits.

✅ Replace with this (copy-paste):
## PDF Vectorizer for RAG Applications (FastAPI + FAISS + Supabase)

This project is a **production-ready PDF and document vectorization backend** designed for **Retrieval-Augmented Generation (RAG)** systems.

It allows you to upload **PDF, CSV, JSON, and Excel files**, extract clean and structured text, generate **semantic embeddings using OpenAI, Google Gemini, or Anthropic Claude**, and store them in **Supabase pgvector and a local FAISS index** for fast semantic search.

This vectorizer is ideal for:
- AI chatbots trained on documents
- Knowledge base search systems
- Enterprise RAG pipelines
- PDF semantic search APIs
- Supabase-powered AI applications


Now Google understands exactly what this project is.

5️⃣ SEO-OPTIMIZED FEATURES SECTION (REWRITE)

Search engines LOVE bullet lists with keywords.

## Key Features – Document Vectorization & RAG Backend

- PDF to vector database conversion for RAG pipelines
- Semantic text extraction with layout-aware PDF parsing
- Intelligent chunking optimized for retrieval accuracy
- Embeddings generation using OpenAI, Gemini, or Claude
- Supabase pgvector integration for vector similarity search
- FAISS local vector index for fast retrieval
- Metadata-rich chunks for high-quality RAG responses
- FastAPI-based backend with REST APIs
- Large file support (up to 1GB documents)
- Ready for AI chatbots, knowledge bases, and search engines

6️⃣ SEO-FOCUSED USE CASES (YOU WERE MISSING THIS)

Google ranks use cases, not tools.

## Use Cases

- Build a PDF-based AI chatbot using RAG
- Create a document knowledge base with Supabase
- Semantic search engine for company documents
- AI assistant trained on internal PDFs
- Research paper vectorization and search
- Legal, policy, or compliance document retrieval
- Training custom LLMs with document context


This increases long-tail search traffic.

7️⃣ ADD A DEDICATED “RAG PIPELINE” SECTION (IMPORTANT)
## How This RAG Vectorizer Works

1. Upload documents (PDF, CSV, JSON, Excel)
2. Extract and normalize text content
3. Apply semantic chunking for context preservation
4. Generate vector embeddings using AI models
5. Store embeddings in Supabase pgvector
6. Index vectors locally using FAISS
7. Perform semantic search or RAG-based querying


This ranks for:

how RAG works

RAG pipeline PDF

vector database workflow

8️⃣ API SECTION – ADD SEO LANGUAGE

Change:

API Endpoints

To:

## REST API for PDF Vectorization & Semantic Search


And add one line under it:

These APIs allow you to build document-based RAG systems, AI search engines, and Supabase-powered vector applications.

9️⃣ ADD FAQ SECTION (SEO GOLD)
## FAQ – PDF Vectorization & RAG

### What is a PDF vectorizer?
A PDF vectorizer converts document text into numerical embeddings that can be stored in a vector database for semantic search and RAG applications.

### Does this support Supabase vector search?
Yes. This project uses Supabase pgvector for storing and querying embeddings.

### Can I use this for RAG with GPT or Gemini?
Yes. The vectorized documents can be used with OpenAI, Gemini, or Claude-based RAG pipelines.

### Is FAISS required?
FAISS is optional but recommended for fast local similarity search alongside Supabase.

**Last Updated**: 2024

