# JobPilot v0

> A daily Telegram digest of new PM jobs in Germany and London, scored 0–100 against your profile.

**Live demo:** <link> · **PM:** Prashant

## The problem
2–3 sentences. Who hurts, how much, why now.

## What it does
<demo GIF>

## Results
| Metric | Target | Result |
|---|---|---|
| | | |

## How it works
Architecture diagram + 3–5 bullets.

## Key product decisions
Links to `docs/decisions/`.

## Limitations & what's next

## Built with
<stack> · <datasets with licences>

## Run locally
Needs Docker Desktop. n8n runs in Docker on your laptop; the digest only goes out while the laptop is on.

```bash
docker compose up -d        # start n8n (pinned version in docker-compose.yml)
open http://localhost:5678  # first time: create the local owner account
docker compose down         # stop; workflows and credentials are kept
```

- **Data:** workflows, credentials and n8n's encryption key live in the Docker volume `p02-jobpilot_n8n_data`, never in the repo. `docker compose down -v` deletes them.
- **Restarts:** the container restarts by itself after Docker or the laptop restarts (`restart: unless-stopped`). For a daily digest, turn on *Start Docker Desktop when you sign in* in Docker Desktop settings.
- **Only local:** n8n listens on `127.0.0.1:5678`, so nothing outside this laptop can reach it.
- **Secrets:** API keys and the Telegram bot token go in n8n credentials, never in workflow JSON or git.
- **Workflow file:** `workflows/` is mounted at `/home/node/workflows` for exports (P2-09).
