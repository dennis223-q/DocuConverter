# DOCFORGE

**Every document. Every format. One powerful workspace.**

DOCFORGE is a privacy-first, production-oriented document conversion and PDF workspace. It uses React + TypeScript + Vite for the UI and FastAPI + Python for processing, with optional LibreOffice and Tesseract support.

## What is implemented

### PDF
- PDF → DOCX
- PDF → XLSX (table extraction where tables are detectable)
- PDF → PPTX (page-rendered slides)
- PDF → JPG / PNG / WEBP
- PDF → TXT / HTML
- PDF → CSV
- PDF/A conversion via Ghostscript when available
- OCR → searchable PDF / TXT / DOCX / XLSX
- Merge, split, extract, delete, duplicate, reorder
- Rotate, crop, resize
- Compress
- Repair/rewrite
- Page numbering
- Watermark
- Metadata editor
- Password protection / decryption with supplied password
- Flatten
- PDF information viewer
- Compare two PDFs
- Draw/text/image/annotation application from editor JSON

### Office / image / text
- DOCX → PDF / TXT / HTML
- XLSX → PDF / CSV
- CSV → XLSX / PDF
- PPTX → PDF / page images
- JPG / PNG / WEBP / HEIC / TIFF / BMP → PDF
- Multiple images → PDF
- TXT → PDF / DOCX
- HTML → PDF / DOCX

### OCR
- English, Hindi, Spanish, French, German
- Image/PDF OCR to text and searchable PDF
- OCR output to DOCX/XLSX

## Reality/limitations

DOCFORGE deliberately reports limitations instead of pretending every conversion is perfect.

- **PDF → DOCX:** layout-aware text extraction, not full Adobe-level reconstruction.
- **PDF → XLSX:** extracts detected tables; arbitrary visual layouts are not guaranteed.
- **PDF → PPTX:** pages are rendered as high-quality slide images rather than reconstructed editable slide objects.
- **PDF → PDF/A:** requires Ghostscript in the runtime.
- **HEIC:** Pillow builds often do not decode HEIC; install `pillow-heif` support or use an optional provider.
- **OCR:** Tesseract language packs must be installed in the container. The included Docker image installs the five requested languages.
- **Office → PDF:** LibreOffice is used for reliable headless conversion.
- **Remove watermark:** not implemented as destructive object-removal because automatic watermark detection/removal can corrupt legitimate page content. The architecture is ready for a provider.
- **Request signature / accounts / billing:** intentionally not faked. These are future-ready interfaces, not pretend features.

## Architecture

```text
docforge/
├── frontend/                 # React + TypeScript + Vite
├── backend/
│   ├── app/
│   │   ├── api/              # FastAPI routes
│   │   ├── core/             # settings/security
│   │   ├── services/         # conversion providers
│   │   └── main.py
│   └── tests/
├── docs/
├── Dockerfile
├── render.yaml
├── docker-compose.yml
└── .env.example
```

The default deployment is a **single Docker service**: FastAPI serves the built React app and processes files locally. This keeps the initial Render deployment inexpensive and simple. The service layer is designed so Redis/Celery and object storage can be introduced later.

## Local development

### 1. Backend

Python 3.11+ is recommended.

```bash
cd backend
python -m venv .venv
# macOS/Linux:
source .venv/bin/activate
# Windows:
# .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
```

### 2. Frontend

```bash
cd frontend
npm install
npm run dev
```

The Vite development server proxies `/api` to `http://localhost:8000`.

## Docker

```bash
docker compose up --build
```

Then open `http://localhost:8000`.

## Environment

Copy `.env.example` to `.env`.

Important variables:

- `PORT` — HTTP port, default `8000`
- `MAX_FILE_SIZE_MB` — upload limit
- `MAX_FILES_PER_REQUEST`
- `PROCESSING_TIMEOUT_SECONDS`
- `FILE_RETENTION_MINUTES`
- `OCR_LANGUAGES` — e.g. `eng,hin,spa,fra,deu`
- `CORS_ORIGINS`
- `SECRET_KEY`
- `STORAGE_PROVIDER` — `local` initially
- `REDIS_URL` — reserved for async workers
- `EXTERNAL_CONVERSION_BASE_URL` / `EXTERNAL_CONVERSION_API_KEY` — optional future provider

No API key is required for the default local stack.

## Render

This repository contains `render.yaml` and a root `Dockerfile`.

1. Push the repository to GitHub.
2. In Render, choose **New → Blueprint**.
3. Connect the GitHub repository.
4. Render reads `render.yaml`.
5. Set any secrets in the Render dashboard, especially `SECRET_KEY`.
6. Deploy.

The container starts FastAPI on `$PORT`, serves the production frontend, and exposes:

```text
GET /health
```

Expected response:

```json
{"status":"healthy","service":"docforge"}
```

### Exact Render assumptions

- Service type: Docker web service
- Dockerfile: `./Dockerfile`
- Port: `$PORT`
- Health check: `/health`
- Persistent disk: **not required** for default privacy-first operation
- Files are stored in a temporary runtime directory and removed after processing / retention cleanup

For heavy workloads, use a separate worker + Redis + object storage. The provider abstractions are intentionally isolated for that migration.

## Tests

```bash
cd backend
pytest
```

Tests cover file validation, PDF merge/split/compress, image-to-PDF, PDF→DOCX, PDF→XLSX, office conversion adapters, OCR routing, and API health.

## GitHub

```bash
git init
git add .
git commit -m "Build DOCFORGE"
git branch -M main
git remote add origin <your-repository-url>
git push -u origin main
```

## Security model

- Randomized server-side job IDs and filenames
- No public static upload directory
- MIME + extension validation
- Request/file count limits
- Upload size limits
- Rate limiting middleware
- Temporary processing directories
- Automatic cleanup
- No permanent document storage by default
- Safe filename handling
- Passwords are only held in memory for the request
- Production should run behind HTTPS

## Future-ready interfaces

The backend includes provider abstractions for:

- local conversion
- external conversion services
- object storage
- background jobs
- billing
- authentication

They are intentionally not represented as fake UI functionality.
