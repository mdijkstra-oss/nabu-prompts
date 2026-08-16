# nabu-prompts

This repository holds the system prompts for [nabu](https://github.com/mdijkstra-oss/nabu-frontend).

Each prompt is a Markdown file in `config/`. Its directory name becomes its route, and its frontmatter names the model tier that route runs on.

[chancery](https://github.com/mdijkstra-oss/chancery) serves the directory as HTTP routes. [dragoman](https://github.com/mdijkstra-oss/dragoman) sits behind chancery and translates each request into the dialect its provider speaks. `compose.yaml` builds both from source; neither is vendored here.

## Running it

```sh
cp .env.example .env
docker compose up
```

Every tier defaults to OpenAI, so `OPENAI_API_KEY` alone runs every prompt.

`.env` holds:

| variable | default | sets |
|---|---|---|
| `OPENAI_API_KEY` `ANTHROPIC_API_KEY` `GEMINI_API_KEY` `DEEPSEEK_API_KEY` | empty | one per provider your tier file names |
| `MODELS` | `models.openai.yaml` | which models yaml the tiers resolve against; a bare name is read from `config/`, an absolute path from anywhere |
| `CHANCERY_PORT` | `8081` | host port the routes are published on |
| `CORS_ORIGINS` | `http://localhost:5173` | comma-separated origins a browser may call from; empty denies every cross-origin request |
| `LOG_REQUEST_HEADERS` | `X-Session-ID,X-Project-ID` | caller headers to log and pass on as identity |
| `CHANCERY_REPO` `DRAGOMAN_REPO` | their GitHub repos | where compose fetches each binary; a local path works too |

`LOG_LEVEL` (`info`) and `ENV` (`production`) pass straight through to chancery.

Every route takes a POST, and requests and replies use the OpenAI Responses API format:

```sh
curl -X POST localhost:8081/topic-assigner \
  -H 'Content-Type: application/json' \
  -d '{"input":[{"type":"message","role":"user","content":"..."}]}'
```

> [!WARNING]
> Nothing in `compose.yaml` authenticates a caller to chancery, so anyone who reaches `CHANCERY_PORT` can spend your provider keys. `CORS_ORIGINS` stops browsers and nothing else; `curl` ignores it. dragoman holds every key and publishes no port, which is what keeps the keys off the host.

## Picking a provider

A prompt's frontmatter names a tier, never a model, so one `config/models.*.yaml` decides what the whole set runs on. Each of those files defines every tier the prompts ask for, and notes above each one which prompts sit on it.

```sh
MODELS=models.anthropic.yaml docker compose up -d chancery
```

| file | keys |
|---|---|
| `models.openai.yaml` | `OPENAI_API_KEY` |
| `models.anthropic.yaml` | `ANTHROPIC_API_KEY` |
| `models.gemini.yaml` | `GEMINI_API_KEY` |
| `models.deepseek.yaml` | `DEEPSEEK_API_KEY` |
| `models.multi.yaml` | `GEMINI_API_KEY`, `ANTHROPIC_API_KEY` |

Only `models.multi.yaml` spreads the tiers across providers. Each of the others runs everything on one key, which makes both `deep-analysis-filter` voters the same model voting against itself — fine for development, not for research output.

## Routes

Every directory in [`config/`](config/) is a route under its own name, so `config/qual-coder/index.md` answers `POST /qual-coder`. Its frontmatter says what the prompt does and which tier it runs on. `shared/` and `tools/` are the exceptions: they hold the partials the prompts include, and publish nothing.

A prompt whose frontmatter carries a `models:` map instead of a single `model:` publishes one route per entry, addressed by suffix. `qual-coder` names `chat`, `planning` and `execution`, so `/qual-coder.planning` runs the planning overlay while a bare `/qual-coder` runs whichever entry `default:` names. `deep-analysis-filter` uses the same mechanism to give each of its two voters a route.

## See also

- [nabu-embeddings](https://github.com/mdijkstra-oss/nabu-embeddings) — nabu's embedding endpoint, served separately from these prompts.
- [nabu-frontend](https://github.com/mdijkstra-oss/nabu-frontend) — the app that calls these routes. Its README covers the document format the prompts read, the query cascade that decides what reaches them, and the loop they run inside.
