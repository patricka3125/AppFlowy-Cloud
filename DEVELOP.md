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

## 1. Configure environment

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

## 2. Start infrastructure services

Bring up the required Docker services (PostgreSQL, Redis, Minio, GoTrue, pgAdmin, AppFlowy Web):

```bash
docker compose -f docker-compose-dev.yml --env-file .env up -d
```

> **Podman users:** `podman-compose` does **not** auto-read `.env`. Always pass `--env-file .env` explicitly, or the GoTrue configuration (e.g. `GOTRUE_MAILER_AUTOCONFIRM`) will fall back to the defaults in the compose file.

> **Note:** Wait a few seconds after this for PostgreSQL to finish initialising before running migrations.

---

## 3. Run database migrations

Install the `sqlx` CLI (Postgres-only build, locked to the version used by this project), source the dev environment variables, then apply all pending migrations:

```bash
cargo install sqlx-cli --no-default-features --features postgres --locked

set -a && source dev.env && set +a

sqlx migrate run
```

> **Why `set -a`?** It exports every variable defined in `dev.env` into the shell environment so `sqlx` can pick up `DATABASE_URL` automatically.

> **Tip:** If you ever add new migration files, re-run `sqlx migrate run` to apply them before `cargo run`.

---

## 4. Run AppFlowy-Cloud

```bash
cargo run
```

The API server will start and listen on the port configured in `dev.env` (default `8000`).

---

## 5. Access the web app

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
