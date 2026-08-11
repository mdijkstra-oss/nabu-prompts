# nabu-prompts

This repository holds the system prompts for [nabu](https://github.com/mdijkstra-oss/nabu-frontend).

Each prompt is a Markdown file in `config/`. Its directory name becomes its route, and its frontmatter names the model tier that route runs on.

[chancery](https://github.com/mdijkstra-oss/chancery) serves the directory as HTTP routes. [dragoman](https://github.com/mdijkstra-oss/dragoman) sits behind chancery and translates each request into the dialect its provider speaks. `compose.yaml` builds both from source; neither is vendored here.

## Running it

```sh
cp .env.example .env
docker compose up
```

Every tier defaults to OpenAI, so `OPENAI_API_KEY` alone runs all twelve prompts.

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

A prompt's frontmatter names a tier, never a model, so one `config/models.*.yaml` decides what the whole set runs on:

| tier | prompts |
|---|---|
| `lite` | corpus-describer, file-hyde, generic-hyde, hyde-generator, section-labeler, topic-assigner |
| `mid` | scout-filter |
| `strong` | qual-coder, semantic-filter, refine-code |
| `expert` | deep-analysis-filter, deep-analysis-adjudicate |

`MODELS` in `.env` names which one, and defaults to `models.openai.yaml`. Switching is a restart, not an edit:

```sh
MODELS=models.anthropic.yaml docker compose up -d chancery
```

An absolute path reads a table from anywhere, so a working copy of your own needs no file in `config/`.

| file | keys |
|---|---|
| `models.openai.yaml` | `OPENAI_API_KEY` |
| `models.anthropic.yaml` | `ANTHROPIC_API_KEY` |
| `models.gemini.yaml` | `GEMINI_API_KEY` |
| `models.deepseek.yaml` | `DEEPSEEK_API_KEY` |
| `models.multi.yaml` | `GEMINI_API_KEY`, `ANTHROPIC_API_KEY` |

Only `models.multi.yaml` spreads the tiers across providers. Each of the others runs everything on one key, which makes both `deep-analysis-filter` voters the same model voting against itself — fine for development, not for research output.

Two tiers exist only to carry a prompt. `strong-planning` and `strong-execution` extend `strong` with a file from `config/shared/nabu/modes/`, which chancery puts in front of the agent's own prompt. Editing those two files is how the `qual-coder` modes change.

## Routes

| route | does |
|---|---|
| [`/corpus-describer`](config/corpus-describer/index.md) | profiles a group of documents, giving `hyde-generator` its context |
| [`/deep-analysis-filter`](config/deep-analysis-filter/index.md) | reviews one coded passage and votes keep or remove — `.voter-one` and `.voter-two` are the two voters |
| [`/deep-analysis-adjudicate`](config/deep-analysis-adjudicate/index.md) | breaks the tie when the two votes disagree |
| [`/file-hyde`](config/file-hyde/index.md) | writes hypothetical passages from a file, for similarity search |
| [`/generic-hyde`](config/generic-hyde/index.md) | writes hypothetical passages with no corpus context |
| [`/hyde-generator`](config/hyde-generator/index.md) | writes hypothetical passages tuned to a described corpus |
| [`/qual-coder`](config/qual-coder/index.md) | the main coding agent — annotates research text against a codebook; `.planning` and `.execution` add a mode overlay |
| [`/refine-code`](config/refine-code/index.md) | reviews a codebook definition against the passages flagged against it |
| [`/scout-filter`](config/scout-filter/index.md) | finds contiguous off-topic blocks to exclude from a document |
| [`/section-labeler`](config/section-labeler/index.md) | gives each document section a label and description |
| [`/semantic-filter`](config/semantic-filter/index.md) | picks the sentences matching a search intent |
| [`/topic-assigner`](config/topic-assigner/index.md) | classifies a document by type and subject |

## See also

- [nabu-embeddings](https://github.com/mdijkstra-oss/nabu-embeddings) — nabu's embedding endpoint, served separately from these prompts.
- [nabu-frontend](https://github.com/mdijkstra-oss/nabu-frontend) — the app that calls these routes. Its README covers the document format the prompts read, the query cascade that decides what reaches them, and the loop they run inside.
