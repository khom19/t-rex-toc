# TOC

Full stack for the T-RexX project. Frontend and backend live in their own repos and are wired in
here as git submodules.

```
t-rex-toc/
├── docker-compose.yml     # production stack
├── docker-compose.dev.yml # dev stack (both services, hot-reload)
├── T-RexEx-frontend/      # submodule — React + TypeScript + Vite
└── T-RexEx-backend/       # submodule — FastAPI
```

## First clone

The submodules are empty on a plain `git clone`, so pull them too:

```bash
git clone --recurse-submodules <this repo>

# already cloned without them?
git submodule update --init --recursive

# later, to pull the latest commit of each submodule
git submodule update --remote --merge
```

## Production

```bash
docker compose up --build
```

Open http://localhost:8080

| Service | Image | Port |
| --- | --- | --- |
| `frontend` | `T-RexEx-frontend` `prod` target — nginx serving the built bundle | `8080` → `80` |
| `backend` | `T-RexEx-backend` `prod` target — uvicorn, non-root, runtime deps only | internal `8000` only |

The backend has **no host port**. The browser only ever talks to the frontend on `:8080`, and nginx
proxies `/api/` through to `backend:8000` over the compose network. That keeps the whole stack
same-origin, so CORS never enters the picture in production, and the API isn't independently
exposed. This does mean the backend service must stay named `backend` — the frontend's `nginx.conf`
refers to it by that hostname.

```bash
docker compose up -d --build         # run in the background
docker compose logs -f frontend      # follow one service's logs
docker compose ps                    # check status + healthcheck
docker compose down                  # stop and remove containers
```

Both images are built from source in the submodules, so after pulling new submodule commits,
rebuild with `docker compose up --build`.

## Development

Run both services at once from here — source is bind-mounted and both hot-reload:

```bash
docker compose -f docker-compose.dev.yml up --build
```

| Service | URL | Reloads on |
| --- | --- | --- |
| `frontend` | http://localhost:5173 | edits under `T-RexEx-frontend/src/` (Vite HMR) |
| `backend` | http://localhost:8000 — [Swagger](http://localhost:8000/docs) | edits under `T-RexEx-backend/app/` (uvicorn `--reload`) |

```bash
docker compose -f docker-compose.dev.yml up -d --build    # background
docker compose -f docker-compose.dev.yml logs -f backend  # one service's logs
docker compose -f docker-compose.dev.yml down             # stop
```

`docker-compose.dev.yml` defines no services of its own — it `include`s each submodule's compose
file, so the dev setup has one source of truth and this file can't drift from it. Running a
submodule's compose directly still works and behaves identically:

```bash
cd T-RexEx-frontend && docker compose up --build    # frontend alone
cd T-RexEx-backend  && docker compose up --build    # backend alone
```

### Dev vs. prod: how the frontend reaches the API

These differ, which matters when you write fetch calls:

- **prod** — one origin. The browser only sees `:8080`, and nginx proxies `/api/` to the backend.
  No CORS involved.
- **dev** — two origins. Vite serves on `:5173` and the browser calls `http://localhost:8000`
  directly, which is cross-origin and relies on the backend's `CORS_ORIGINS` (default `*`).

So a hardcoded `http://localhost:8000` works in dev and breaks in prod. Use a relative `/api/...`
path plus a Vite dev proxy, or read the base URL from an env var, so the same code works in both.

Note the two stacks use the same compose project name, so they share container names — bring one
down before starting the other.
