# Tabula — the visual component of Sethlans

**Tabula** is the board that makes visible what the subagents orchestrated by
[Sethlans](https://github.com/GabrieleConsonni/sethlans) are doing: it organizes work by **projects** (a Jira project or
an internal one) and, within each project, displays **epics**, **stories**, **tasks**, and the
state/consumption of the **agents** (a pool shared across projects).

It consists of a backend with a REST API (FastAPI + SQLite (default) or PostgreSQL, Alembic
migrations) and a React frontend (Vite). Each epic/story/task has an associated
**Markdown** document (`md`); stories have a **phase** (`phase`). The active project
is selected from the combo box in the header (with the `+` to create a new one).

The subagents write to the backend via the API (see [tabula-protocol.md](https://github.com/GabrieleConsonni/sethlans/blob/main/.claude-plugin/tabula-protocol.md));
the frontend displays the state and updates automatically (polling ~4s). Updating the board
is **best-effort**: if it does not respond, development work continues anyway.

## Structure

```
backend/
├── tabula_server.py        # FastAPI API (CRUD endpoints + /state)
├── models.py               # SQLAlchemy ORM models (epics/stories/tasks/agents)
├── db.py                   # DB engine (SQLite default, PostgreSQL optional)
├── alembic/                # migrations (env.py + versions/)
├── alembic.ini
├── seed.py                 # optional seed (canonical agents + demo)
├── requirements.txt        # base deps (SQLite — no external dependencies)
└── requirements-postgres.txt  # adds psycopg2-binary for PostgreSQL
frontend/
├── index.html
├── package.json
├── vite.config.js
├── .env.example
└── src/
    ├── main.jsx
    ├── api.js              # API client
    ├── App.jsx             # state + routing between views
    ├── styles.css
    └── components/
        ├── ProjectSwitcher.jsx # project combo + create project (header)
        ├── Agenda.jsx      # home: epics (left) + stories (right)
        ├── StoryPage.jsx   # story detail: Munera + Periti tabs
        ├── Munera.jsx      # task board
        ├── Periti.jsx      # agent grid
        └── shared.jsx      # reused pieces (ColHeader, EditBox, etc.)
```

## Quick start (Docker Hub — recommended)

No need to clone the repository. Pull the pre-built images:

```bash
curl -O https://raw.githubusercontent.com/GabrieleConsonni/sethlans/main/docker-compose.dist.yml
docker compose -f docker-compose.dist.yml up -d
```

- Interface: <http://localhost:5173>
- API / docs: <http://localhost:9955/docs>

Database: **SQLite** by default, persisted in a Docker named volume `tabula-data`. Zero external dependencies.

For full details on images, tags, and PostgreSQL configuration: [docker-hub.md](docker-hub.md).

## Running from source (Docker)

You need [Docker Desktop](https://www.docker.com/products/docker-desktop/) running and the repo cloned.

```bash
docker compose up --build -d
```

- Interface: <http://localhost:5173>
- API / docs: <http://localhost:9955/docs>

To stop: `docker compose down`.

### Development mode (hot-reload)

```bash
docker compose -f docker-compose.dev.yml up --build
```

Backend and frontend sources are mounted into the containers: every change is applied automatically.

- Backend: `uvicorn --reload` reloads on Python file saves.
- Frontend: Vite dev server with hot module replacement.

## Quick start (without Docker)

### 1. Backend

```bash
cd backend
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt    # SQLite (default)
# For PostgreSQL: pip install -r requirements-postgres.txt
alembic upgrade head               # creates the tables
python tabula_server.py            # http://localhost:9955  (docs: /docs)
```

Default DB: `sqlite:///./tabula.db` (file in `backend/`). Override with `TABULA_DB_URL`.

The board starts **empty**. To populate canonical agents + demo data: `python seed.py`.

### 2. Frontend

```bash
cd frontend
npm install
cp .env.example .env               # optional: change the backend URL
npm run dev                        # http://localhost:5173
```

The frontend reads `VITE_API_URL` (default `http://localhost:9955`). You can also change it at
runtime from the field at the top of the interface.

## API in brief

Resources: `projects`, `epics`, `stories`, `tasks`, `agents`. Each one with:

| Method | Path              | Action                    |
|--------|-------------------|---------------------------|
| GET    | `/{resource}`     | list (with filters)       |
| POST   | `/{resource}`     | create                    |
| GET    | `/{resource}/{id}`| read                      |
| PATCH  | `/{resource}/{id}`| partial update            |
| DELETE | `/{resource}/{id}`| delete                    |

Filters: `/epics?project_id=`, `/stories?epic_id=`, `/tasks?story_id=`, `/tasks?agent_id=`, `?status=`, `/projects?type=`.
Full snapshot: `GET /state`.

### Example: a subagent updates its own work

```bash
# takes a task
curl -X PATCH localhost:9955/tasks/t3 \
  -H "Content-Type: application/json" \
  -d '{"status":"progress","agent_id":"a2"}'

# updates the agent state and token count
curl -X PATCH localhost:9955/agents/a2 \
  -H "Content-Type: application/json" \
  -d '{"current_task":"Test end-to-end","status":"active","tokens":92000}'

# completes the task
curl -X PATCH localhost:9955/tasks/t3 -H "Content-Type: application/json" -d '{"status":"done"}'
```

## Data schema

```
Project { id, name, type, jira_key }                                   type: jira|internal
Epic    { id, title, desc, status, project_id, md, md_updated_at }     status: todo|progress|done
Story   { id, title, desc, status, phase, epic_id, md, md_updated_at } status: todo|progress|done ; phase: analysis|ux|design|dev|done
Task    { id, title, status, story_id, agent_id, md, md_updated_at }   status: todo|progress|done
Agent   { id, name, current_task, status, tokens }                     status: active|idle
```

Hierarchy: **Project → Epic → Story → Task**. Deleting a project cascade-removes the
linked epics/stories/tasks.

`md` = associated Markdown document (for stories it can contain HTML mockups in
` ```mockup ` blocks, rendered in the UI inside a sandboxed iframe).

## Configuration

| Variable        | Default                        | Notes                              |
|-----------------|--------------------------------|------------------------------------|
| `TABULA_DB_URL` | `sqlite:///./tabula.db`        | SQLite (default) or PostgreSQL URL |
| `TABULA_PORT`   | `9955`                         | API port                           |
| `VITE_API_URL`  | `http://localhost:9955`        | Backend base URL (also at runtime) |

## Notes

- CORS is open to all origins for development convenience: in production, restrict `allow_origins`.
- Migrations with Alembic: `alembic upgrade head` / `alembic revision -m "..."`.
- No authentication: add a token if exposed on the network.
