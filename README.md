# nabu-prompts

The agents [nabu-frontend](https://github.com/mdijkstra-oss/nabu-frontend) calls, and the two binaries that serve them.

A prompt is a Markdown file whose path is its route and whose frontmatter picks its model. [chancery](https://github.com/mdijkstra-oss/chancery) turns the directory into HTTP endpoints; [dragoman](https://github.com/mdijkstra-oss/dragoman) translates each request into whatever dialect the provider speaks. Neither is vendored here — `compose.yaml` builds both from their own repositories, so this repository is the prompts, the model table, and the service table, and nothing else.

> [!WARNING]
> The frontend cannot talk to this yet. Its body is now `openai-responses`, but two fields still carry the names hermes-logos gave them, and it needs an `/embeddings` route no binary here serves. [Pointing the frontend at it](#pointing-the-frontend-at-it) is the list of what has to change.

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
deep-analysis-adjudicate  anthropic/claude-opus-4-6     medium
deep-analysis-filter                                    
  .deep                   anthropic/claude-opus-4-6     low
  .fast (default)         openai/gpt-5.5                low
file-hyde                 gemini/gemini-3.1-flash-lite  minimal
generic-hyde              gemini/gemini-3.1-flash-lite  minimal
hyde-generator            gemini/gemini-2.5-flash-lite  minimal
qual-coder                deepseek/deepseek-v4-pro      high
refine-code               anthropic/claude-opus-4-6     medium
scout-filter              deepseek/deepseek-v4-flash    none
section-labeler           gemini/gemini-3.1-flash-lite  minimal
semantic-filter           deepseek/deepseek-v4-pro      none
topic-assigner            gemini/gemini-2.5-flash-lite  minimal
15 agents · 16 models
```

Eleven of them have a caller in `nabu-frontend`. Four do not: `commit-agent` waits on a git backend that `nabu-storage` has not built, `cv` answers visitor questions on a personal site rather than anything in nabu, and `compacter` and `section-labeler` have no reference in the frontend at all. They are served anyway — an agent nothing calls costs a route table entry — but nothing here exercises them.

`deep-analysis-filter` is the multimodal consensus step, and it is the one agent whose shape changed in the move. It used to be an array of two model configurations that both ran; here it is two named models on one prompt, and a route reaches one of them.

## Pointing the frontend at it

Set `VITE_LLM_HOST` to chancery's published port, and `CORS_ORIGINS` in `.env` to the frontend's own origin — chancery denies every cross-origin request when that is empty, and the frontend is a browser app.

The response side already matches: the frontend parses `openai-responses` SSE, and the items it builds are Responses input items. So does the envelope — `buildRequestBody` sends `input` and `text.format`. Three things on the request side do not, and none of them can be fixed from this repository.

**`extra_content` is a hermes field.** It appears nowhere in dragoman, which carries every provider's opaque state — an Anthropic thinking signature, a Gemini thought signature, DeepSeek's reasoning — in `encrypted_content` on the item it arrived on. That is the same channel under one name, so the fix is to emit `encrypted_content` where `convert.ts` currently emits both. Left as it is, the field is dropped and a conversation loses its reasoning state at every turn boundary.

**Consensus votes are selected by a query parameter.** `step-filter.ts` builds `/deep-analysis-filter?model=0` and `?model=1` to reach the two voters. chancery routes on the path and reads only `?tool_choice=` and `?reasoning_summary=`, so both calls land on the default model and the two-model consensus becomes one model voting twice. Nothing errors. The replacement is a route suffix — `.fast` and `.deep` — indexed the same way.

**Modes are pushed as a marker.** The frontend emits `<!-- prompt: planning -->` as a system message and expects the server to expand it. chancery never parses the message array, by design, so the marker travels to the model as literal text. See [Modes](#modes).

## What isn't served

### Embeddings

> [!IMPORTANT]
> Not built. The frontend's RAG cannot work against this stack.

`app/lib/embeddings/client.ts` posts `{input: string[]}` to `/embeddings` and expects `{data: [{index, embedding}], usage}` back. chancery builds exactly one URL, `{RESPONSES_BASE_URL}/responses`, and dragoman serves `/responses`, `/services` and `/health`. There is no route to configure, so `embeddings.md` — which named `text-embedding-3-large` and carried `type: embedding` and `dimensions: 1024` — was removed rather than left as a route that would forward an embedding model to a chat endpoint.

Whoever solves this decides where an embeddings route belongs: a second dialect surface in dragoman, or something beside it.

### Modes

> [!IMPORTANT]
> Not wired. The prompts are here and reachable by no route.

`modes/planning.md` and `modes/execution.md` are mode overlays, each composing its own subdirectory with the same `[include.md]` syntax an agent uses. They sit at the repository root rather than under `config/`, because chancery reports any frontmatter-less Markdown in its configuration directory as an orphan and would refuse to start.

They were expanded server-side: the frontend pushed `<!-- prompt: planning -->` into the message array and hermes-logos replaced it with the compiled overlay. chancery's serving path never decodes the message array — that invariant is most of why it is small — so the mechanism has no equivalent and the marker reaches the model as text.

The cheapest way back is the shape `deep-analysis-filter` already uses: make each mode a named model on `qual-coder`, so `/qual-coder.planning` carries the overlay and the frontend sends a path instead of a marker. That trades a marker in the body for a route, and needs no feature in chancery.

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

Three settings from the hermes-logos configuration have no position in this format and were dropped:

- `temperature` and `seed`, which made `hyde-generator` and `topic-assigner` deterministic. chancery has no frontmatter field for either. The frontend can send `temperature` in the body, where it reaches the provider untouched.
- `legacy_thinking` on the two Gemini 2.5 models, which selected an older thinking configuration. dragoman has no such concept.
- `compact_at` on `qual-coder`, a context threshold that is the caller's business and not a body field.

### Cache breakpoints run on each provider's default TTL

The frontend places explicit cache breakpoints on the two consensus steps. An alias declares whether its model accepts them:

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

The remaining wire changes all land in one repository. `extra_content` is a field rename in `app/lib/agent/client/convert.ts`. The consensus vote is a route suffix in `app/lib/agent/tools/apply-deep-analysis/step-filter.ts`. Modes are a decision about where an overlay lives before they are a change to any file, and embeddings are a route somebody has to build.
