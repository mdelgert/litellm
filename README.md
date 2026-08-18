# LiteLLM Proxy Stack

A production-ready Docker Compose stack that runs [LiteLLM](https://github.com/BerriAI/litellm) as an LLM proxy with PostgreSQL for persistence and Redis for caching and rate-limiting.

## What this repo is

This repo provides a minimal, opinionated setup to self-host the LiteLLM proxy. It is NOT a Python package or application — it is a deployment configuration. Key components:

- **LiteLLM proxy** (`ghcr.io/berriai/litellm`) — unified API gateway for OpenAI, Anthropic, Azure, Groq, Gemini, Cohere and more
- **PostgreSQL 16** — stores spend logs, API keys, users, and teams
- **Redis 7** — caching (exact/semantic) and router state for latency-based routing

## Quick start

```bash
# 1. Copy and fill in secrets
cp .env.example .env
# Edit .env — see "Generating secure secrets" below

# 2. Start the stack
docker compose up -d

# 3. Tail logs
docker compose logs -f litellm
```

The proxy API is available at `http://localhost:4000` (or the port set in `LITELLM_PORT`).

The Admin UI is at `http://localhost:4000/ui` — log in with `LITELLM_UI_USERNAME` and `LITELLM_UI_PASSWORD` from your `.env`.

## Generating secure secrets

**IMPORTANT:** All passwords must be alphanumeric only (no special characters) to avoid breaking URL construction in `DATABASE_URL` and Redis auth strings.

```bash
# Safe for .env and connection URLs
openssl rand -base64 48 | tr -dc 'A-Za-z0-9' | head -c 40; echo
```

Use the output of that command for:

| Variable | Notes |
|---|---|
| `POSTGRES_PASSWORD` | Postgres user password |
| `REDIS_PASSWORD` | Redis auth password |
| `LITELLM_UI_PASSWORD` | Admin UI login password |
| `LITELLM_MASTER_KEY` | Prefix output with `sk-`: `echo "sk-$(openssl rand -base64 48 | tr -dc 'A-Za-z0-9' | head -c 40)"` |

## Changing passwords after first boot

**WARNING:** Postgres stores its user password in the `postgres_data` named volume. If you change `POSTGRES_PASSWORD` in `.env` after the volume already exists, the database will reject the new password and LiteLLM will fail to start.

To safely rotate `POSTGRES_PASSWORD`:

```bash
# Option 1: Reset everything (loses all data)
docker compose down -v
docker compose up -d

# Option 2: Rotate in-place (preserves data)
docker compose exec postgres psql -U litellm -c "ALTER USER litellm PASSWORD 'newpassword';"
# Then update POSTGRES_PASSWORD in .env and restart
docker compose restart litellm
```

## Environment variables

See `.env.example` for the full list. Key variables:

| Variable | Required | Description |
|---|---|---|
| `POSTGRES_PASSWORD` | Yes | Postgres user password |
| `REDIS_PASSWORD` | Yes | Redis auth string |
| `LITELLM_MASTER_KEY` | Yes | Master API key (prefix with `sk-`) |
| `LITELLM_UI_USERNAME` | Yes | Admin UI username (default: `admin`) |
| `LITELLM_UI_PASSWORD` | Yes | Admin UI password |
| `LITELLM_IMAGE_TAG` | No | LiteLLM image version (default: `v1.97.0`) |
| `LITELLM_PORT` | No | Host port for the proxy (default: `4000`) |

## Version pinning

The LiteLLM image is pinned to a specific release via `LITELLM_IMAGE_TAG` (default `v1.97.0`). To upgrade:

1. Check the [LiteLLM releases](https://github.com/BerriAI/litellm/releases) for the latest tag.
2. Update `LITELLM_IMAGE_TAG` in `.env`.
3. Run `docker compose pull && docker compose up -d`.

Postgres and Redis images are also pinned to major version tags (`postgres:16-alpine`, `redis:7-alpine`).

## Model configuration

Models are defined in `litellm_config.yaml`. Add or remove models under `model_list`. API keys are read from environment variables using the `os.environ/VAR_NAME` syntax — never hardcode keys in the config file.

Example:
```yaml
- model_name: gpt-4o
  litellm_params:
    model: openai/gpt-4o
    api_key: "os.environ/OPENAI_API_KEY"
```

## Day-2 operations

### Check service health
```bash
docker compose ps
curl http://localhost:4000/health/liveliness
```

### View logs
```bash
docker compose logs -f litellm
docker compose logs -f postgres
```

### Restart the proxy only (no data loss)
```bash
docker compose restart litellm
```

### Full restart
```bash
docker compose down && docker compose up -d
```

### Stop and remove all containers and volumes (DATA LOSS)
```bash
docker compose down -v
```

## Security notes

- `.env` is in `.gitignore` — never commit it
- Postgres and Redis ports are NOT exposed to the host by default (see commented lines in `docker-compose.yml`)
- All services run on an isolated Docker network (`litellm_net`)
- The LiteLLM healthcheck uses Python's `urllib` since the image does not include `curl`

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `password authentication failed` in Postgres logs | `POSTGRES_PASSWORD` changed after volume was created | See "Changing passwords after first boot" above |
| `httpx.ConnectError` in litellm logs | DB not ready or wrong credentials | Check Postgres logs; ensure passwords match |
| UI login fails | Proxy not fully started, or wrong credentials | Wait 30-60s after `docker compose up`; verify `LITELLM_UI_USERNAME`/`LITELLM_UI_PASSWORD` in `.env` |
| `ValueError: Dictionary must have exactly one key` | Wrong fallback syntax in `litellm_config.yaml` | Use `- modelname: [fallback1, fallback2]` format |
