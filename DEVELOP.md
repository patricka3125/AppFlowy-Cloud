# AppFlowy-Cloud — Local Developer Setup (Linux)

This guide covers everything needed to build and run AppFlowy-Cloud locally on a Fedora / RHEL-based Linux system.

---

## Prerequisites

### 1. Rust toolchain

Install `rustup` and add the `rust-src` component (required by some build scripts and IDE tooling):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup component add rust-src
```

### 2. System dependencies

Install C/C++ build tools, compression libraries, and the Protocol Buffers compiler:

```bash
sudo dnf install -y gcc-c++ clang make cmake \
    snappy-devel zlib-devel bzip2-devel lz4-devel libzstd-devel

sudo dnf install -y protobuf-compiler protobuf-devel
```

---

## Quick start (all-in-one script)

If you want to skip the manual steps below, use the bundled setup script which handles Docker, migrations, and building automatically:

```bash
cp dev.env .env

# First time / after wiping the DB — starts services, runs migrations, builds:
./script/run_local_server.sh --reset

# Subsequent runs — starts services, skips migrations, builds:
./script/run_local_server.sh
```

| Flag | Effect |
|---|---|
| `--reset` | Creates the database and runs all SQL migrations |
| `--sqlx` | Regenerates SQLx offline metadata (`cargo sqlx prepare --workspace`) |

The script also auto-sets `GOTRUE_MAILER_AUTOCONFIRM=true` for the duration of the run.

> **Additional prerequisites for the script:** The script uses `psql` to health-check Postgres and `sqlx-cli` for migrations. Install them before running:
> ```bash
> sudo dnf install -y postgresql          # provides psql
> cargo install sqlx-cli --no-default-features --features postgres --locked
> ```

> If you prefer to run each step yourself, follow the manual instructions below.

---

## Manual setup

### 1. Configure environment

Copy the template environment file:

```bash
cp dev.env .env
```

For local development, enable auto-confirm so you can sign up without SMTP / email verification.
In `dev.env` (and then re-copy to `.env`), set:

```
GOTRUE_MAILER_AUTOCONFIRM=true
```

---

### 2. Start infrastructure services

Bring up the required Docker services (PostgreSQL, Redis, Minio, GoTrue, pgAdmin, AppFlowy Web):

```bash
docker compose -f docker-compose-dev.yml --env-file .env up -d
```

> **Podman users:** `podman-compose` does **not** auto-read `.env`. Always pass `--env-file .env` explicitly, or the GoTrue configuration (e.g. `GOTRUE_MAILER_AUTOCONFIRM`) will fall back to the defaults in the compose file.

> **Note:** Wait a few seconds after this for PostgreSQL to finish initialising before running migrations.

---

### 3. Run database migrations

Install the `sqlx` CLI (Postgres-only build, locked to the version used by this project), source the dev environment variables, then apply all pending migrations:

```bash
cargo install sqlx-cli --no-default-features --features postgres --locked

set -a && source dev.env && set +a

sqlx migrate run
```

> **Why `set -a`?** It exports every variable defined in `dev.env` into the shell environment so `sqlx` can pick up `DATABASE_URL` automatically.

> **Tip:** If you ever add new migration files, re-run `sqlx migrate run` to apply them before `cargo run`.

---

### 4. Run AppFlowy-Cloud

```bash
cargo run
```

The API server will start and listen on the port configured in `dev.env` (default `8000`).

---

### 5. Access the web app

The `appflowy_web` container (added to `docker-compose-dev.yml`) serves the AppFlowy Web frontend:

- **URL:** [http://localhost:3000](http://localhost:3000)
- Sign up with any email — with `GOTRUE_MAILER_AUTOCONFIRM=true`, no email verification is needed.

---

## Services overview

| Service | Port | Purpose |
|---|---|---|
| AppFlowy-Cloud API (`cargo run`) | `8000` | REST API + WebSocket server |
| AppFlowy Web (Docker) | `3000` | User-facing web editor |
| GoTrue (Docker) | `9999` | Authentication (signup, login, OAuth) |
| PostgreSQL (Docker) | `5432` | Primary database |
| Redis (Docker) | `6379` | Caching / pub-sub |
| Minio (Docker) | `9000` / `9001` | S3-compatible object storage / console |
| pgAdmin (Docker) | `5400` | Database admin UI |

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `relation "..." does not exist` errors at compile time | Migrations have not been applied | Run `sqlx migrate run` (step 3) |
| `DATABASE_URL` not found | Environment not sourced | `set -a && source dev.env && set +a` |
| Docker services not ready | Postgres still starting up | Wait ~10 s and retry `sqlx migrate run` |
| `protoc` not found | Protobuf compiler missing | `sudo dnf install -y protobuf-compiler` |
| "Error sending confirmation mail" on signup | `GOTRUE_MAILER_AUTOCONFIRM` is `false` | Set to `true` in `dev.env`, re-copy to `.env`, restart GoTrue |
| GoTrue ignoring `.env` values | Podman-compose doesn't auto-read `.env` | Use `--env-file .env` in all compose commands |
| Web app 404s on API calls (e.g. `/notifications/unread-count`) | `appflowy_web:latest` is newer than the server | Pin web image to `0.9.163` — see below ↓ |

---

### Web app API 404s — version mismatch between web client and server

**Symptom:** The workspace loads but the browser console shows many 404 errors like:

```
GET http://localhost:8000/api/workspace/.../notifications/unread-count 404 (Not Found)
```

**Root cause:** `appflowy_web:latest` tracks the newest release (e.g. `0.11.x`) and calls API endpoints that don't exist in the older `dev` branch server code.

**Fix:** The `docker-compose-dev.yml` pins the web image to `0.9.163` (the last `0.9.x` release, which matches this server branch). If the container is running a newer image, force-recreate it:

```bash
docker compose -f docker-compose-dev.yml --env-file .env up -d --force-recreate appflowy_web
```

> **Tip:** You can override the version without editing the compose file by setting `APPFLOWY_WEB_VERSION` in `.env`:
> ```
> APPFLOWY_WEB_VERSION=0.9.163
> ```

---

### Logging in to the web app when GoTrue CORS blocks the browser

The web app at `localhost:3000` makes direct cross-origin requests to GoTrue on port `9999`. If CORS blocks the login form, you can bypass it by getting a token via curl and injecting it into the browser's `localStorage` manually.

**Step 1 — Create a user (first time only):**

```bash
curl -s -X POST "http://localhost:9999/signup" \
  -H "Content-Type: application/json" \
  -d '{"email": "dev@example.com", "password": "password"}'
```

**Step 2 — Get an access token:**

```bash
curl -s -X POST "http://localhost:9999/token?grant_type=password" \
  -H "Content-Type: application/json" \
  -d '{"email": "dev@example.com", "password": "password"}'
```

Copy the full JSON response. Note the `access_token` field — you'll need it in the next steps.

**Step 3 — Register the user with AppFlowy-Cloud:**

The GoTrue token alone isn't enough — AppFlowy-Cloud has its own user/workspace records that are created on first login. Trigger registration by calling the verify endpoint with the token from step 2:

```bash
ACCESS_TOKEN="<paste access_token value here>"

curl -s "http://localhost:8000/api/user/verify/${ACCESS_TOKEN}"
```

You should get back `{"code":0,"data":{"is_new":true},...}`. If `is_new` is `true`, the user and default workspace were just created in the AppFlowy-Cloud database.

**Step 4 — Inject it into the browser:**

Open `http://localhost:3000`, then open DevTools (**F12 → Console**) and paste the following, replacing the `{}` with the full JSON from step 2:

```javascript
const gotrueResp = { /* paste full curl JSON response here */ };

function decodeJWT(token) {
  try { return JSON.parse(atob(token.split('.')[1])); } catch { return null; }
}
const userInfo = decodeJWT(gotrueResp.access_token);
if (userInfo) gotrueResp.user = { id: userInfo.sub, email: userInfo.email };

localStorage.setItem('token', JSON.stringify(gotrueResp));
window.location.href = '/app';
```

> **Note:** After a `--reset` (DB wipe), existing users are deleted. Re-run step 1 to recreate your account.
