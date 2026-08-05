# nabu-prompts

The agents [nabu-frontend](https://github.com/mdijkstra-oss/nabu-frontend) calls, and the two binaries that serve them.

A prompt is a Markdown file whose path is its route and whose frontmatter picks its model. [chancery](https://github.com/mdijkstra-oss/chancery) turns the directory into HTTP endpoints; [dragoman](https://github.com/mdijkstra-oss/dragoman) translates each request into whatever dialect the provider speaks. Neither is vendored here — `compose.yaml` builds both from their own repositories, so this repository is the prompts, the model table, and the service table, and nothing else.

## Running it

```sh
cp .env.example .env       # one provider key per service you call
docker compose up
```

chancery publishes `8081`. dragoman holds every provider key and publishes no port, which is what makes it safe for it to perform no client authentication.

A route whose provider has no key answers by naming the variable, which is the quickest way to see the whole chain resolve:

```console
$ curl -s -X POST http://localhost:8081/topic-assigner \
    -H 'Content-Type: application/json' \
    -d '{"input":[{"type":"message","role":"user","content":"..."}]}'
{"error":{"message":"service \"gemini\" is unavailable: GEMINI_API_KEY is not set","type":"server_error"}}
```

## The agents

Fifteen, one directory each, `index.md` carrying the frontmatter:

```console
$ chancery --config ./config list
PATH                      MODEL                         REASONING
commit-agent              openai/gpt-5-mini             off
compacter                 gemini/gemini-3.5-flash       minimal
corpus-describer          gemini/gemini-3.1-flash-lite  minimal
cv                        gemini/gemini-3.5-flash       none
deep-analysis-adjudicate  anthropic/claude-fable-5      low
deep-analysis-filter                                    
  .anthropic              anthropic/claude-opus-5       low
  .openai (default)       openai/gpt-5.6                low
file-hyde                 gemini/gemini-3.1-flash-lite  minimal
generic-hyde              gemini/gemini-3.1-flash-lite  minimal
hyde-generator            gemini/gemini-2.5-flash-lite  minimal
qual-coder                                              
  .chat (default)         deepseek/deepseek-v4-pro      high
  .execution              deepseek/deepseek-v4-pro      high
  .planning               deepseek/deepseek-v4-pro      high
refine-code               anthropic/claude-opus-4-6     medium
scout-filter              deepseek/deepseek-v4-flash    none
section-labeler           gemini/gemini-3.1-flash-lite  minimal
semantic-filter           deepseek/deepseek-v4-pro      none
topic-assigner            gemini/gemini-2.5-flash-lite  minimal
15 agents · 18 models
```

Eleven of them have a caller in `nabu-frontend`. Four do not: `commit-agent` waits on a git backend that `nabu-storage` has not built, `cv` answers visitor questions on a personal site rather than anything in nabu, and `compacter` and `section-labeler` have no reference in the frontend at all. They are served anyway — an agent nothing calls costs a route table entry — but nothing here exercises them.

`deep-analysis-filter` is the multimodal consensus step. Both voters share the prompt, and a route suffix reaches one of them. `qual-coder` uses the same suffix mechanism for a different purpose — see [Modes](#modes).

## Pointing the frontend at it

Set `VITE_LLM_HOST` to chancery's published port, and `CORS_ORIGINS` in `.env` to the frontend's own origin — chancery denies every cross-origin request when that is empty, and the frontend is a browser app.

## Embeddings are somewhere else

Nothing here answers `/embeddings`. chancery builds exactly one URL, `{RESPONSES_BASE_URL}/responses`, and dragoman serves `/responses`, `/services` and `/health`, so an agent file cannot reach an embedding model — every route it can name ends at a chat endpoint.

That is the point rather than a gap. No provider key resolves inside chancery, and an embeddings route here would put one there. [nabu-embeddings](https://github.com/mdijkstra-oss/nabu-embeddings) holds that key and serves the route, behind the frontend's own `VITE_EMBEDDINGS_HOST`.

## Modes

`qual-coder` answers on three routes rather than one. Each names its own alias in `models.yaml`, and the two mode aliases differ from the plain one only by a `prompt:`:

```yaml
  deepseek-v4-pro-planning:
    extends: deepseek-v4-pro
    prompt: nabu/modes/planning.md
```

An alias prompt resolves against `shared/`, and chancery puts it in front of the agent's own, so `/qual-coder.planning` is the chat agent with the planning overlay ahead of its identity. `default: chat` keeps the bare `/qual-coder` route serving the unmodified agent.

The frontend derives which mode it is in from its own conversation and picks the route from that, so nothing about the mode travels in the body.

Both overlays are single files. An alias prompt is read as-is, without the `[include.md]` resolution an agent's Markdown body gets, so anything an overlay needs has to be in it.

## Configuration

`config/` is chancery's directory, and its layout is chancery's:

| path | holds |
|---|---|
| `<agent>/index.md` | one agent — its route, model, settings and prompt |
| `<agent>/*.md` | fragments that agent includes by name |
| `shared/` | fragments any agent can include |
| `tools/` | prompts pulled in when a request offers the tool named after the last dot |
| `models.yaml` | the alias an agent's `model:` names, and the settings it runs with |

`dragoman.yaml` sits outside `config/` because it is the backend's file, not chancery's. It names the four services the aliases point at, each with an endpoint, a protocol, and the environment variable holding its key.

`temperature` and `seed` are not among the fields an agent may pin, and `validate` reports either as unknown. A caller that needs one sends it in the body, where it reaches the provider untouched.

### Cache breakpoints run on each provider's default TTL

The frontend places explicit cache breakpoints on the two consensus steps. Not every model accepts them, so an alias can refuse on its model's behalf:

```yaml
  gpt-5.5:
    model: openai/gpt-5.5
    prompt_cache_breakpoints: false
```

chancery forwards that as a query parameter and dragoman removes the breakpoints on the way in, so a route reaching a model that refuses them answers normally instead of failing. Explicit breakpoints arrived with OpenAI's 5.6 family; anything earlier answers `400` naming the field.

> [!NOTE]
> Nothing here pins a TTL. The default differs per provider — Anthropic and OpenAI each apply their own — and there is currently no way to set it. `prompt_cache_options` is the field that would carry it, and no alias can name one.

### DeepSeek gets its schema as prose

DeepSeek has a JSON mode and no schema mode, and its own documentation says what that mode needs from the caller:

📘 [JSON Output | DeepSeek API Docs](https://api-docs.deepseek.com/guides/json_mode)
> "Include the word 'json' in the system or user prompt."
>
> "provide an example of the desired JSON format to guide the model in outputting valid JSON."

`semantic-filter` and `scout-filter` both send a JSON schema. The `schema_as_prompt` flag on the `deepseek` service turns the mode on and writes the schema into the instructions behind the agent's own prompt, so neither agent has to know which provider answered.

> [!IMPORTANT]
> A schema in a prompt is a request, not a constraint. `strict` is gone with the field that carried it, and an answer that is valid JSON of the wrong shape reaches the caller as a parse failure. The prompts predate the flag and still name their own shape — `semantic-filter` closes on a fenced `json` example — which is worth keeping as a second line of defence.

## Development

This repository carries no binary, so checking a change to `config/` means building one — `make build` in a chancery checkout writes `bin/chancery`:

```sh
chancery --config ./config validate
chancery --config ./config list
```

`validate` is offline and reports every problem at once: an alias no agent can resolve, an include that exists in neither the agent's directory nor `shared/`, a Markdown file nothing references.

Editing a prompt is editing a file. `config/` and `dragoman.yaml` mount read-only, so a restart is the whole cycle. Pointing compose at local checkouts instead of the published repositories is what `CHANCERY_REPO` and `DRAGOMAN_REPO` are for.

## Next: nabu-frontend

The app that calls every route here. Its README covers the document format these agents read, the query cascade that decides what reaches them, and the loop they run inside.
