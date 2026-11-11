# Resume Modifier

Resume Modifier is a Dockerized web app for tailoring a resume and cover letter to a job posting, previewing the result, exporting polished PDFs, and keeping a per-user history of previous runs.

## What the application does

- Optimizes a resume for a required job role, company, job URL, job description, and uploaded resume PDF.
- Uses a configured AI provider to generate an optimized resume, cover letter, gap analysis, interview prep, and LinkedIn outreach content.
- Stores optimization history per user account.
- Renders downloadable PDFs using nine resume templates and four cover-letter templates. Google-, Microsoft-, and Apple-inspired options are original designs, not official company templates.
- Checks tailored resume claims against the source resume, attempts to correct unsupported claims, and flags anything that still needs review.
- Includes saved resumes, backup/restore helpers, and optional job-scanning integrations using n8n and crawl4ai.

The automated fact check is a best-effort AI review, not a guarantee. Review the final resume against the original source before submitting it.

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
- `db`: PostgreSQL on `127.0.0.1:${DB_HOST_PORT}`
- `n8n`: workflow automation on `127.0.0.1:${N8N_HOST_PORT}`
- `crawl4ai`: crawl API on `127.0.0.1:${CRAWL4AI_PORT}`

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

### Initialize the application

1. Clone the repository and enter the project directory:

   ```bash
   git clone https://github.com/H20-Jenish/Resume_modifier.git
   cd Resume_modifier
   ```

2. Create your local environment file. On Windows PowerShell, run:

   ```powershell
   Copy-Item .env.example .env
   ```

   On macOS or Linux, run:

   ```bash
   cp .env.example .env
   ```

3. Edit `.env` and set your time zone, host ports, and non-default values for the n8n and crawl4ai tokens. The example uses high host ports to reduce Windows port conflicts. If a port is already in use, change its corresponding `*_PORT` value.

   The PostgreSQL username/password and the initial application admin password are currently defined in the Compose/backend configuration rather than `.env`. They are intended only for local development. Before sharing this stack, change the PostgreSQL password in Compose and keep it consistent with `DATABASE_URL`; replace the admin-password fallback and development JWT/provider-key encryption secrets in the backend; and set unique n8n secrets in `.env`. Changing the admin password through the UI alone does not remove the hard-coded fallback.

4. Build and start the services:

   ```bash
   docker compose up --build -d
   docker compose ps
   ```

   Wait for PostgreSQL to become healthy and for the backend to finish starting. If a service exits, inspect its output with `docker compose logs backend` (replace `backend` with `n8n`, `frontend`, or another Compose service name as needed).

5. Open the app at `http://localhost:18080` (or the value of `FRONTEND_PORT`). The backend API documentation is at `http://localhost:8000/docs`.

6. For local setup, you can sign in with the first-run account `admin` / `securepassword123`. The backend creates this account when no `admin` account exists. You can also register a separate account from the login screen. **Do not expose this development setup to an untrusted network:** the current backend has a hard-coded admin-password fallback, so changing the password in the UI does not disable the default password. Remove that fallback and replace the development secrets before any shared or production deployment.

7. Start an AI provider on the host machine, then configure it in the app’s Settings. Use `host.docker.internal` in the provider URL so the backend container can reach the host service; provider setup examples are below. Select the provider and model, add an API key if required, save the settings, and use model detection to confirm connectivity.

8. Submit an optimization with all five required inputs: Job Role, Company Name, a valid Job URL, Job Description, and a text-readable Resume PDF.

The basic optimizer does not require n8n or crawl4ai configuration. To stop the stack while preserving its data, run:

```bash
docker compose down
```

To delete the database, n8n data, and backup volumes as well, run `docker compose down -v`. This permanently deletes those Docker-managed volumes.

### Optional job-scanning setup

The job-scanning workflows are separate from resume optimization. To use them, open n8n at `http://localhost:15678` (or your configured `N8N_HOST_PORT`), create its initial owner account, and import the applicable workflow JSON from `n8n/workflows/`. Configure the provider credentials and workflow settings it requires, and ensure the webhook secret matches `N8N_WEBHOOK_SECRET` in `.env`. The crawl4ai service is started by Compose; set its token using `CRAWL4AI_TOKEN` if you use crawl requests that require authentication.

## Environment variables

Use a local `.env` file for the Compose substitutions below. The PostgreSQL credentials and the initial admin account password are not configurable through `.env` in the current setup.

| Variable | Purpose |
| --- | --- |
| `TIMEZONE` | Time zone used by the containers |
| `FRONTEND_PORT` | Host port for the frontend container |
| `DB_HOST_PORT` | Host port for PostgreSQL |
| `N8N_HOST_PORT` | Host port for n8n |
| `CRAWL4AI_PORT` | Host port for the crawl4ai container |
| `N8N_WEBHOOK_SECRET` | Shared secret for backend to n8n webhook requests |
| `N8N_ENCRYPTION_KEY` | n8n encryption key |
| `CRAWL4AI_TOKEN` | API token for the crawl4ai service |

Note: Windows can dynamically reserve some local ports, which can make Docker port binding fail after a reboot. This repo uses higher defaults (`FRONTEND_PORT=18080`, `DB_HOST_PORT=15432`, `N8N_HOST_PORT=15678`, `CRAWL4AI_PORT=21235`) to avoid recurring bind failures.

## Application flow

1. Sign in or register an account.
2. Open Settings and choose `lmstudio` or `ollama`.
3. Configure the provider URL, model, and optional API key.
4. Run an optimization from the main form.
5. Review results in the app, then export resume or cover-letter PDFs.
6. Revisit prior runs from History.

## Notes

- The backend stores user accounts, history, saved resume data, and encrypted provider API keys in PostgreSQL.
- On startup, the backend creates the tables and seeds the `admin` account if one does not already exist. The hard-coded fallback means changing this account's password through the UI does not disable the default password; keep the app local until that fallback is removed.
- Manual SQL files under `migrations/` are not required to build or run the Docker stack, so they are left out of version control for this repo.

### Configure an AI provider

The AI provider runs on the host or at another URL reachable from the backend container. Do not enter `localhost` for a service running on your host; from inside Docker, that points to the backend container itself.

#### LM Studio

1. Download and launch [LM Studio](https://lmstudio.ai/)
2. Load a model and start the local server (default: `http://localhost:1234`)
3. In the app Settings, set:
   - Provider: `lmstudio`
   - Base URL: `http://host.docker.internal:1234` (the app will auto-strip a trailing `/v1` if you include it)
   - Model: copy the model ID shown in LM Studio

Note: The application prefers LM Studio's native REST API endpoints when `provider` is set to `lmstudio`. It can fall back to OpenAI-compatible endpoints. Use either the host root (for example `http://host.docker.internal:1234`) or an OpenAI-compatible `/v1` URL supported by the configured server.

If your LM Studio server is configured to require an API token, enter that token in the app Settings `API Key` field (the app sends it as an `Authorization: Bearer <token>` header). The `/models/detect` and other calls will return `401 Unauthorized` unless a valid token is provided.

#### Ollama

1. Install and start [Ollama](https://ollama.com/): `ollama serve`
2. Pull a model: `ollama pull llama3`
3. In the app Settings, set:
   - Provider: `ollama`
   - Base URL: `http://host.docker.internal:11434`
   - Model: `llama3` (or whichever model you pulled)

#### Other OpenAI-compatible APIs

The main Settings screen offers LM Studio and Ollama. For another OpenAI-compatible endpoint, choose **LM Studio**, enter the service's OpenAI-compatible base URL and model identifier, and provide its API key. Requests are sent to the configured endpoint; an external provider will receive the resume and job data submitted for processing.

---

## Usage

### 1. Register / Login

On a new database, sign in with `admin` / `securepassword123` for local development, or click **Register** to create a separate account. User data is stored in the local PostgreSQL container. The default admin password remains accepted by a hard-coded backend fallback even if changed through the UI; do not expose this development setup to an untrusted network.

### 2. Configure AI Settings

Go to the **Settings** tab and fill in your AI provider details (see above). Click **Save Settings**. You can also click **Detect Models** to auto-fetch available models from your inference server.

### 3. Optimize Your Resume

1. Open the **Resume Optimizer** tab.
2. Fill in **Job Role**, **Company Name**, **Job URL**, and **Job Description**.
3. Upload a text-readable **Resume PDF**. Pasted Markdown is not accepted by the optimizer form, and image-only/scanned PDFs may need OCR before upload.
4. Choose the human-tone setting if desired, then click **Optimize My Resume**.
5. Keep the page open while optimization runs. When complete, the results are displayed and the app may send a browser notification if permission is granted.

All five inputs are required. The Job URL must be a valid HTTP or HTTPS URL. The AI-assisted fact check may flag claims for review; verify the generated resume against the uploaded source before using it.

### 4. Preview and Download PDF

After optimization, review the resume in the preview and choose a template. Download the PDF from the result view. Previously optimized resumes are available from History and can also be previewed and downloaded with a selected template.

### 5. Generate a Cover Letter

After optimizing your resume, use **Generate Cover Letter**. The AI uses the job description and source resume to produce a tailored cover letter. Preview and download it using one of the cover-letter templates.

### 6. Optimized Resumes History

The **Optimized Resumes** tab shows all past optimization runs. You can:

- Search by job title or company
- Filter by date range
- Click any entry to view the full optimized resume
- Edit the saved resume or add/update its Job URL
- Choose a template and download a PDF
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
| **Google-Inspired** | Clean, keyword-forward layout; original design, not an official template |
| **Microsoft-Inspired** | Structured skills and impact layout; original design, not an official template |
| **Apple-Inspired** | Minimal, typography-led layout; original design, not an official template |

Resume contact details are rendered as separate, right-aligned rows. The name and contact typography and colors can vary by template.

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

The renderer recognizes a Markdown heading for the candidate name and parses nearby contact lines, including labeled fields and common phone, email, address, and LinkedIn values.

The name is placed on the left and contact details are stacked on the right. Contact rows wrap long values to help keep them within the page. The remaining resume body follows below.

Template-specific CSS overrides (fonts, colors, accent styles) are injected per-request.

---

## API Reference

Selected endpoints are listed below. All API endpoints are on the backend at `http://localhost:8000`.

| Method | Path | Description |
|---|---|---|
| `POST` | `/register` | Create a new user account |
| `POST` | `/login` | Authenticate; returns JWT token |
| `POST` | `/change-password` | Change password for authenticated user |
| `POST` | `/optimize` | Run AI resume optimization |
| `POST` | `/generate-cover-letter` | Generate a cover letter |
| `POST` | `/generate-outreach` | Generate LinkedIn outreach content |
| `POST` | `/pdf/render` | Render Markdown to PDF (returns binary) |
| `GET` | `/history` | List all optimization history entries |
| `POST` | `/history/{document_id}/template` | Set the template for a history entry |
| `PATCH` | `/history/{document_id}/job-url` | Update a history entry's job URL |
| `PATCH` | `/history/{document_id}/resume` | Update the saved resume content |
| `PATCH` | `/history/{document_id}/cover-letter` | Update the saved cover-letter content |
| `DELETE` | `/history/{document_id}` | Delete a history entry |
| `POST` | `/models/detect` | Detect available models from the AI server |
| `POST` | `/backup/snapshot` | Create a database backup snapshot |
| `GET` | `/backup/snapshot` | List available backup snapshots |
| `POST` | `/backup/restore` | Restore the database from a snapshot |
| `POST` | `/provider-api-key` | Store or update an encrypted provider API key for the current user (form: `provider`, `api_key`) |
| `DELETE` | `/provider-api-key` | Remove a stored provider API key (query: `provider`) |
| `GET` | `/provider-api-keys/status` | List supported providers and whether a key is stored for the current user |
| `GET` | `/provider-api-key` | Retrieve the current user's stored provider API key (query: `provider`) |
| `POST` | `/resume/store` | Upload a resume PDF; server extracts and stores plain text (`multipart/form-data`) |
| `GET` | `/resume/stored` | Return stored resume preview and parsed profile for the current user |
| `POST` | `/resume/profile/ai` | Run an AI model against the stored resume to infer a structured profile (form params: `lm_url`, `lm_model`, `provider`, `api_key`) |
| `DELETE` | `/resume/stored` | Delete the stored resume for the current user |
| `POST` | `/listings/scan` | Scan selected job sources |
| `GET` | `/listings` | List saved job listings |
| `DELETE` | `/listings/{listing_id}` | Delete one saved listing |
| `DELETE` | `/listings` | Clear saved listings |
| `GET` | `/scan-config` | Read the current user's job-scan configuration |
| `GET` | `/scan-schedule/current` | Read the current user's scan schedule |
| `POST` | `/scan-schedule/save` | Save a scan schedule |
| `POST` | `/scan-schedule/toggle` | Enable or disable the scan schedule |
| `GET` | `/scan-schedule/logs` | Read scan-schedule logs |

Interactive API docs (Swagger UI) are available at `http://localhost:8000/docs` while the backend is running.

---

## Ports

| Service | Container Port | Host Binding |
|---|---|---|
| Frontend (nginx) | 80 | `127.0.0.1:18080` (configurable via `FRONTEND_PORT`) |
| Backend (FastAPI) | 8000 | `127.0.0.1:8000` |
| Database (PostgreSQL) | 5432 | `127.0.0.1:15432` (configurable via `DB_HOST_PORT`) |
| n8n | 5678 | `127.0.0.1:15678` (configurable via `N8N_HOST_PORT`) |
| crawl4ai | 11235 | `127.0.0.1:21235` (configurable via `CRAWL4AI_PORT`) |

All ports are bound to `127.0.0.1` — accessible only from the local machine.

---

## Development Notes

- The frontend is a **single HTML file** (`frontend/index.html`) served as a static file by nginx. No build step is required — edit and refresh.
- The backend mounts `./backend` as a volume, so Python file changes take effect after `docker compose restart backend`.
- `weasyprint==62.3` and `pydyf==0.11.0` are pinned together. Upgrading either independently may break PDF rendering.
- LM Studio and Ollama are the provider choices in the main Settings screen. LM Studio also supports an OpenAI-compatible API URL; another compatible endpoint can be configured through that option.
- The current tab and optimization result are persisted in `sessionStorage`; AI settings are persisted in `localStorage`.

---

## Security Notes

- All ports are bound to `localhost` only — not exposed to external networks.
- Passwords are hashed with bcrypt via `passlib`.
- JWT tokens authenticate the app's user-facing API requests.
- The database credentials and seeded admin password are development defaults. Change them, along with the JWT and provider-key encryption secrets, before using the app with sensitive data or sharing the deployment.
- AI requests go to the provider URL configured in Settings. If that URL is an external service, the submitted resume and job information are sent to that provider. Job scanning also contacts the selected job sources and configured integrations.
- This Compose setup is intended for local development and is not production-hardened. Do not expose it directly to the internet.
