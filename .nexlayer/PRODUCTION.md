# Nexlayer — how `cat-chat-app` ships

You were asked to work on or deploy this app. The state below is already
resolved — do not re-derive it from the code. The procedure is not here;
ask Nexlayer for it (see "How to deploy").

## The app

| | |
| --- | --- |
| Name | `cat-chat-app` |
| Repo | `https://github.com/sasdeployer/cat-chat-app` on `main` |
| Planned | 2026-10-06T22:28:17.019Z |
| Registered with Nexlayer | yes |

`.nexlayer/plan.lock` pins the commit this plan was written against. If HEAD
has moved and you changed how the app starts, runs, or what it needs,
re-check before deploying.

## The production plan

Written by the Nexlayer agent from this repo. Every decision cites the files it
rests on; if the code has changed since, re-check those files first.

Cat Chat runs as three services. The frontend serves the React build through Nginx and forwards /api and /socket.io (including WebSocket upgrades) to backend.pod:3001. The backend runs Express and Socket.io, and Postgres 15 is built from database/ with init.sql baked in. The most important production change is that the committed nexlayer.yaml contains a literal database password: it must come from the POSTGRES_PASSWORD key, and the database must keep its 2Gi volume so chat history survives restarts.

- **services: Keep the existing three services: frontend (Nginx, port 80, public at /), backend (Node/Express/Socket.io, port 3001) and database (Postgres 15, port 5432). Each is built from its own folder and pushed as registry.nexlayer.io/YOUR_USER_ID/cat-chat-app-<service>:planned.** — Each folder has its own Dockerfile, and nginx.conf already proxies to backend.pod:3001, which matches the backend service name and the port its Dockerfile exposes. (`nexlayer.yaml`, `frontend/Dockerfile`, `backend/Dockerfile`, `database/Dockerfile`, `frontend/nginx.conf`)
- **keys: Start from the existing nexlayer.yaml and change three things. Rename the app to cat-chat-app. Point the images at the planned registry paths. Replace both literal 'postgres' passwords (backend DB_PASSWORD and database POSTGRES_PASSWORD) with ${POSTGRES_PASSWORD}, so both services resolve to one generated value.** — The committed yaml exposes the database password in plain text, and the backend and database must share the same credential. (`nexlayer.yaml`)
- **database: Build Postgres from database/Dockerfile (postgres:15-alpine plus init.sql). Keep major version 15, user postgres and database catchat.** — The repo pins Postgres 15, and the schema lives in database/init.sql, which only runs if it is baked into the image's docker-entrypoint-initdb.d. (`database/Dockerfile`, `database/init.sql`, `nexlayer.yaml`)
- **storage: Keep the 2Gi volume postgres-data mounted at /var/lib/postgresql, with PGDATA=/var/lib/postgresql/data.** — Messages and users live in Postgres and must survive restarts. Mounting the parent folder avoids the lost+found folder that breaks initdb. (`nexlayer.yaml`)
- **networking: Only the frontend is public, at path /. The browser reaches the backend through Nginx on the same origin. The backend reaches the database at database.pod:5432. CLIENT_URL is set to <% URL %> for Socket.io CORS.** — server.js limits Socket.io CORS to CLIENT_URL (default http://localhost:3000), and nginx.conf already proxies /api/ and /socket.io/ with Upgrade headers. (`backend/server.js`, `frontend/nginx.conf`, `nexlayer.yaml`)
- **scaling: Run exactly one backend instance.** — server.js keeps online users and socket mappings in in-memory Maps (activeUsers, userSockets), so a second instance would split users and break presence and typing events. (`backend/server.js`)
- **health: Use GET /api/health as the backend health check, reached publicly through the frontend proxy.** — server.js defines /api/health and it returns {status:'ok'} without touching the database. (`backend/server.js`)

### Fix before production

- **Blocker** — Remove the literal database password from the committed nexlayer.yaml: DB_PASSWORD and POSTGRES_PASSWORD are committed as 'postgres', which exposes the credential. Use ${POSTGRES_PASSWORD} as in the planned yaml, and treat the old value as compromised. (`nexlayer.yaml`)
- Drop the hardcoded POSTGRES_PASSWORD ENV from the database image: The image bakes in a default password. If the runtime key were ever missing, the database would start with 'postgres' instead of failing. (`database/Dockerfile`)
- Make the frontend's API and Socket.io URLs same-origin in the production build: The code reads REACT_APP_API_URL and REACT_APP_SOCKET_URL at build time. If they fall back to a localhost address, browsers cannot reach the backend; they should be empty or relative so traffic goes through the Nginx proxy. (`frontend/src/App.js`)

### Verify after the deploy

1. GET / on the app URL returns 200 with the React index.html
2. GET /api/health on the app URL returns 200 with {"status":"ok"}
3. GET /api/rooms on the app URL returns 200 JSON, not 500, proving the backend reached database.pod and init.sql created the schema
4. GET /socket.io/?EIO=4&transport=polling on the app URL returns 200 with a Socket.io handshake payload
5. Send a message, restart the database service, then GET /api/messages and confirm the message is still returned

### Ask the human

- The existing deployment is named 'cat-chat'; this plan deploys as 'cat-chat-app'. Does the old deployment's chat history need to be migrated, or is starting fresh acceptable?
- Will the backend ever need more than one instance? That would require a shared Socket.io adapter (e.g. Redis) and moving the in-memory online-users state out of server.js.

## The deploy config

This repo already has a `nexlayer.yaml`, and it is the source of truth: the
services in this plan were read from it (`frontend`, `backend`, `database`).
Deploy from it. Change it for a reason, never to match a fresh analysis —
the analysis infers; this file is what runs.

## Can this deploy right now?

**Yes.** Nothing is blocking.

## How to deploy

Call `nexlayer_get_deployment_workflow` first. It returns the current
procedure — building and pushing the image included — and it is kept up to
date in a way this file is not. Do not infer the steps from here, and do
not skip it because the app looks simple.

If Nexlayer tools are not available to you, the human runs
`npx @nexlayer/mcp-install` once.

## Secrets

Keys reach the app by name. In `nexlayer.yaml`, write `${NAME}` where the
value goes (e.g. `OPENAI_API_KEY: "${OPENAI_API_KEY}"`) — never the value.
When you deploy through the Nexlayer MCP, Nexlayer fills each name from this
app's Secrets. Values never go in this repo, the chat, or your context.

| Key | Status |
| --- | --- |
| `DB_PASSWORD` | not set — optional, the app runs without it |

Keys marked supplied are handled. Do not ask for them again.

Optional keys: leave them out of `nexlayer.yaml` unless the human wants that
feature on — then they add the key in the same Secrets page and you add its
`${NAME}` reference.

## What was inferred rather than read

Nothing. Every claim in this plan was read from the repo.

## What "it worked" means

The `verify` list in `.nexlayer/pipeline.yaml` is what to check. Check it — do not
assume a deploy worked.

## Stop and ask the human

- A required key is missing (send the link above — never take the value).
- Something would become publicly reachable that is internal in this plan.
- Anything that deletes data or tears down a running deployment.

Everything else is yours to do. When something breaks, start at
`.nexlayer/TROUBLESHOOTING.md`.
