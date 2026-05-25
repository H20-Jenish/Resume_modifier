# Resume Modifier

Resume Modifier is a Dockerized web app for tailoring a resume and cover letter to a job posting, previewing the result, exporting polished PDFs, and keeping a per-user history of previous runs.

## What the application does

- Accepts a job description, resume, and optional job URL.
- Uses a configured AI provider to generate an optimized resume, cover letter, gap analysis, interview prep, and LinkedIn outreach content.
- Stores optimization history per user account.
- Renders downloadable PDF outputs from selectable templates.
- Includes backup/restore helpers plus workflow integrations for job-scanning support.

## Stack

- Frontend: static single-page app in [frontend/index.html](frontend/index.html) served by `nginx:alpine`
- Backend: FastAPI app in [backend/main.py](backend/main.py)
- Database: PostgreSQL 15
- Integrations: `n8n` workflows and `crawl4ai`
- Orchestration: Docker Compose via [docker-compose.yml](docker-compose.yml)

## Services

The Compose stack starts these services:

- `frontend`: serves the SPA on `http://127.0.0.1:${FRONTEND_PORT}`
- `backend`: FastAPI API on `http://127.0.0.1:8000`
- `db`: PostgreSQL on `127.0.0.1:5432`
- `n8n`: workflow automation on `127.0.0.1:5678`
- `crawl4ai`: crawl API on `127.0.0.1:11235`

All ports are bound to `127.0.0.1` only.

## Repository contents

This repository intentionally keeps only the files needed to build and run the Docker stack:

```text
Resume_modifier/
├── .env.example
├── .gitignore
├── LICENSE
├── README.md
├── docker-compose.yml
├── backend/
│   ├── Dockerfile
│   ├── main.py
│   └── requirements.txt
├── frontend/
│   ├── admin_icon.svg
│   ├── favicon.svg
│   └── index.html
└── n8n/
    ├── gen_v5_workflows.py
    └── workflows/
```

Local-only folders such as `.vscode`, `.env`, `Raw_Files`, generated backups, and manual migrations are excluded from version control.

## Quick start

### Prerequisites

- Docker Desktop or Docker Engine with the Compose plugin
- An AI provider endpoint reachable by the backend container, such as LM Studio or Ollama

### Setup

1. Clone the repository.
2. Copy `.env.example` to `.env`.
3. Adjust `FRONTEND_PORT` if needed.
4. Start the stack:

```bash
docker compose up --build -d
```

5. Open the frontend at `http://localhost:<FRONTEND_PORT>`.

To stop the stack:

```bash
docker compose down
```

To remove volumes as well:

```bash
docker compose down -v
```

## Environment variables

Use a local `.env` file for overrides. The committed example file documents the supported values.

| Variable | Purpose |
| --- | --- |
| `TIMEZONE` | Time zone used by the containers |
| `FRONTEND_PORT` | Host port for the frontend container |
| `N8N_WEBHOOK_SECRET` | Shared secret for backend to n8n webhook requests |
| `N8N_ENCRYPTION_KEY` | n8n encryption key |
| `CRAWL4AI_TOKEN` | API token for the crawl4ai service |

Note: on some Windows machines, port `4310` can fall inside a reserved range. If that happens, use another local port such as `3210` in your `.env` file.

## Application flow

1. Register or sign in.
2. Open Settings and choose `lmstudio` or `ollama`.
3. Configure the provider URL, model, and optional API key.
4. Run an optimization from the main form.
5. Review results in the app, then export resume or cover-letter PDFs.
6. Revisit prior runs from History.

## Notes

- The backend stores user accounts, history, and encrypted provider API keys in PostgreSQL.
- The app seeds a default `admin` user on first run if the database is empty.
- Manual SQL files under `migrations/` are not required to build or run the Docker stack, so they are left out of version control for this repo.

#### LM Studio

1. Download and launch [LM Studio](https://lmstudio.ai/)
2. Load a model and start the local server (default: `http://localhost:1234`)
3. In the app Settings, set:
   - Provider: `lmstudio`
   - Base URL: `http://host.docker.internal:1234` (the app will auto-strip a trailing `/v1` if you include it)
   - Model: copy the model ID shown in LM Studio

Note: The application prefers LM Studio's native REST API endpoints (`/api/v1/chat`, `/api/v1/models`) when `provider` is set to `lmstudio`. If those endpoints are not available, the backend will fall back to LM Studio's OpenAI-compatible endpoints under `/v1/*` (e.g., `/v1/chat/completions`). You can safely point the Base URL to either the host root (e.g. `http://host.docker.internal:1234`) or to the OpenAI-compatible path (e.g. `http://host.docker.internal:1234/v1`) — the backend will detect and use the best supported API shape.

If your LM Studio server is configured to require an API token, enter that token in the app Settings `API Key` field (the app sends it as an `Authorization: Bearer <token>` header). The `/models/detect` and other calls will return `401 Unauthorized` unless a valid token is provided.

#### Ollama

1. Install and start [Ollama](https://ollama.com/): `ollama serve`
2. Pull a model: `ollama pull llama3`
3. In the app Settings, set:
   - Provider: `ollama`
   - Base URL: `http://host.docker.internal:11434`
   - Model: `llama3` (or whichever model you pulled)

#### OpenAI / Other Compatible APIs

Set the provider to `lmstudio` (uses the OpenAI-compatible client path), point the Base URL at the endpoint, and supply a real API key.

---

## Usage

### 1. Register / Login

On first launch, click **Register** and create an account. All data is stored locally in the PostgreSQL container. Log in with your credentials to access the optimizer.

### 2. Configure AI Settings

Go to the **Settings** tab and fill in your AI provider details (see above). Click **Save Settings**. You can also click **Detect Models** to auto-fetch available models from your inference server.

### 3. Optimize Your Resume

1. Go to the **Resume Optimizer** tab.
2. Paste the **Job Description** into the left text area.
3. Either:
   - Paste your current resume as **Markdown** in the right text area, or
   - Upload a **PDF** (it will be extracted automatically).
4. Click **Optimize Resume**.
5. A "DO NOT REFRESH THIS PAGE" warning appears in the header while optimization is running.
6. When complete, a browser notification is sent and the results are displayed.

The output persists in the current browser session — refreshing the page will restore your last optimization result. Results are cleared only by clicking **Clear**.

### 4. Preview and Download PDF

1. After optimization, click **Download PDF** (or the template thumbnail in history).
2. A modal opens with a live preview on the right and action buttons (template selector + download) at the top.
3. Select a **resume template** from the dropdown.
4. Preview updates automatically.
5. Click **Download PDF** to save. The modal stays open so you can switch templates and download again.

### 5. Generate a Cover Letter

After optimizing your resume, click **Generate Cover Letter**. The AI uses the same job description and resume to produce a tailored cover letter. Download it as a PDF using any of the cover letter templates.

### 6. Optimized Resumes History

The **Optimized Resumes** tab shows all past optimization runs. You can:

- Search by job title or company
- Filter by date range
- Click any entry to view the full optimized resume
- Choose a per-entry template and download its PDF
- Delete individual entries

### 7. Backup and Restore

In the **Settings** tab:

- **Create Snapshot** — saves a compressed backup of the database to `/app/data/backups/` inside the container (mapped to a named Docker volume).
- **Restore Snapshot** — lists available snapshots; click one to restore the database to that point in time.

---

## Resume Templates

| **Template** | Style |
|---|---|
| **Classic** | Clean, corporate; uppercase section headers; dark navy name |
| **Modern** | Garamond serif font; reduced letter-spacing; elegant proportions |
| **Executive** | Bold orange accent on section dividers; authoritative layout |
| **Minimal** | Uppercase slate-gray section labels; maximum whitespace |
| **Tech** | Blue left-border accent on name; monospace-adjacent feel |
| **Creative** | Blue left-bar on section headers; expressive but professional |

All templates share:
- Name rendered at **27pt**, left-aligned
- Contact info (Phone → Email → Address → LinkedIn) at **11pt**, right-aligned
- Consistent page margins and line-height for broad compatibility

---

## Cover Letter Templates

| Template | Style |
|---|---|
| **Classic** | Formal business letter format |
| **Modern** | Contemporary with tasteful typography |
| **Executive** | Premium look matching the Executive resume template |
| **Minimal** | Ultra-clean, distraction-free |

---

## PDF Export

PDFs are generated server-side using **WeasyPrint 62.3** (with `pydyf 0.11.0` pinned for compatibility).

The backend parses the Markdown resume header to extract structured contact fields:

- **Name** — first non-empty line
- **Phone** — line matching a phone number pattern
- **Email** — line containing `@`
- **Address** — line containing a comma or common location keywords
- **LinkedIn** — line containing `linkedin`

These are rendered into a two-column `<header>` block: name on the left, contact details stacked on the right. The rest of the resume body follows below.

Template-specific CSS overrides (fonts, colors, accent styles) are injected per-request.

---

## API Reference

All endpoints are on the backend at `http://localhost:8000`.

| Method | Path | Description |
|---|---|---|
| `POST` | `/register` | Create a new user account |
| `POST` | `/login` | Authenticate; returns JWT token |
| `POST` | `/change-password` | Change password for authenticated user |
| `POST` | `/optimize` | Run AI resume optimization |
| `POST` | `/generate-cover-letter` | Generate a cover letter |
| `POST` | `/pdf/render` | Render Markdown to PDF (returns binary) |
| `GET` | `/history` | List all optimization history entries |
| `POST` | `/history/{document_id}/template` | Set the template for a history entry |
| `DELETE` | `/history/{document_id}` | Delete a history entry |
| `POST` | `/models/detect` | Detect available models from the AI server |
| `POST` | `/backup/snapshot` | Create a database backup snapshot |
| `GET` | `/backup/snapshot` | List available backup snapshots |
| `POST` | `/backup/restore` | Restore the database from a snapshot |
| `POST` | `/provider-api-key` | Store or update an encrypted provider API key for the current user (form: `provider`, `api_key`) |
| `DELETE` | `/provider-api-key` | Remove a stored provider API key (query: `provider`) |
| `GET` | `/provider-api-keys/status` | List supported providers and whether a key is stored for the current user |
| `POST` | `/resume/store` | Upload a resume PDF; server extracts and stores plain text (`multipart/form-data`) |
| `GET` | `/resume/stored` | Return stored resume preview and parsed profile for the current user |
| `POST` | `/resume/profile/ai` | Run an AI model against the stored resume to infer a structured profile (form params: `lm_url`, `lm_model`, `provider`, `api_key`) |
| `GET` | `/provider-api-key` | Retrieve the decrypted provider API key for the current user (query: `provider`) |
| `PATCH` | `/history/{document_id}/resume` | Update the `tailored_resume` content for a history entry (form: `tailored_resume`) |
| `PATCH` | `/history/{document_id}/cover-letter` | Update the `cover_letter` content for a history entry (form: `cover_letter`) |
| `DELETE` | `/resume/stored` | Delete the stored resume for the current user |

Interactive API docs (Swagger UI) are available at `http://localhost:8000/docs` while the backend is running.

---

## Ports

| Service | Container Port | Host Binding |
|---|---|---|
| Frontend (nginx) | 80 | `127.0.0.1:4310` (configurable via `FRONTEND_PORT`) |
| Backend (FastAPI) | 8000 | `127.0.0.1:8000` |
| Database (PostgreSQL) | 5432 | `127.0.0.1:5432` |

All ports are bound to `127.0.0.1` — accessible only from the local machine.

---

## Development Notes

- The frontend is a **single HTML file** (`frontend/index.html`) served as a static file by nginx. No build step is required — edit and refresh.
- The backend mounts `./backend` as a volume, so Python file changes take effect after `docker compose restart backend`.
- `weasyprint==62.3` and `pydyf==0.11.0` are pinned together. Upgrading either independently may break PDF rendering.
- The `openai` Python package is used as a generic HTTP client for all AI providers (not just OpenAI) via the `base_url` parameter. This means any OpenAI API-compatible service works out of the box.
- Tab state, optimization results, and AI settings are persisted in `sessionStorage` (tab + results) and `localStorage` (AI settings) so they survive page refreshes within the same session.

---

## Security Notes

- All ports are bound to `localhost` only — not exposed to external networks.
- Passwords are hashed with bcrypt via `passlib`.
- JWT tokens are used for all authenticated API calls.
- The default database password in `docker-compose.yml` (`securepassword123`) should be changed before any shared or production deployment.
- No telemetry or external calls are made — all AI requests go to your locally configured inference server.
