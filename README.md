# almai

Multi-provider LLM client for [Almide](https://github.com/almide/almide). One interface, every provider.

## Install

```toml
# almide.toml
[dependencies]
almai = { git = "https://github.com/almide-ai/almai", tag = "v0.3.0" }
```

## Quick start

```almide
import almai

effect fn main() -> Unit = {
  let r = almai.call("anthropic/claude-sonnet-4-6", [
    almai.system("You are helpful."),
    almai.user("What is 2+2?"),
  ])!
  println(r.content)
  println("Tokens: " + int.to_string(r.usage.total_tokens))
}
```

## Providers

| Prefix | Provider | Env vars |
|---|---|---|
| `anthropic/` | Anthropic Messages API | `ANTHROPIC_API_KEY` |
| `openai/` | OpenAI Chat Completions | `OPENAI_API_KEY` |
| `openrouter/` | OpenRouter (100+ models) | `OPENROUTER_API_KEY` |
| `cf/` | Cloudflare Workers AI | `CF_ACCOUNT_ID`, `CLOUDFLARE_API_KEY`, `CLOUDFLARE_EMAIL` |
| `azure/` | Azure OpenAI | `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT` |
| `google/` | Google Gemini | `GOOGLE_API_KEY` |
| `cli/claude` | Claude Code CLI | (authenticated CLI) |
| `cli/codex` | Codex CLI | (authenticated CLI) |

```almide
// Anthropic
almai.call("anthropic/claude-sonnet-4-6", msgs)

// Cloudflare Workers AI (open models)
almai.call("cf/@cf/meta/llama-3.3-70b-instruct-fp8-fast", msgs)

// OpenRouter (any model)
almai.call("openrouter/meta-llama/llama-3.3-70b-instruct", msgs)

// Google Gemini
almai.call("google/gemini-2.5-pro", msgs)
```

## Options

```almide
let opts = almai.defaults()
  |> almai.with_max_tokens(8192)
  |> almai.with_temperature(0.7)
  |> almai.with_system("You are a code reviewer.")
  |> almai.with_stop(["END"])

let r = almai.call_with("openai/gpt-4o", msgs, opts)!
```

`defaults()` uses `temperature: 0.0` — almai is built for measurement, and a
benchmark that resamples on every run measures the sampler as much as the
thing under test. Pass `with_temperature` for prose.

## Reproducible sampling

`with_seed` pins the sampler; `0` (the default) means "no seed", so a caller
who never asked for one never silently gets one.

```almide
let opts = almai.defaults() |> almai.with_seed(7)
let r = almai.call_with("cf/@cf/meta/llama-3.3-70b-instruct-fp8-fast", msgs, opts)!
```

**A dropped option must not look like an honoured one.** Providers differ in
what they put on the wire, so every response reports what it actually sent:

```almide
r.sampling.seed_sent          // Bool — was `seed` written into the request?
r.sampling.seed               // Int  — the value that went out (0 if unsent)
r.sampling.temperature_sent   // ... same shape for temperature / top_p /
r.sampling.max_tokens_sent    //     max_tokens
```

A manifest can therefore print `seed: 7` versus `seed: null (not sent)`
instead of quoting `CallOptions` and implying a pinned run that never was.

| Provider | max_tokens | temperature | top_p | seed |
|---|---|---|---|---|
| `cf/` | ✓ | ✓ | ✓ | ✓ |
| `openai/`, `openrouter/`, `groq/` | ✓ | ✓ | — | — |
| `anthropic/`, `azure/`, `google/`, `bedrock/` | ✓ | — | — | — |
| `cli/` | — | — | — | — |

`seed_sent: true` means the field reached the API, **not** that the model
honoured it — no text-generation API reports back which sampling controls it
applied, and Cloudflare's GLM models document seed as "best effort". Whether
a given model respects a seed is established by running the same
`(model, seed, temperature, task set)` twice and comparing.

## JSON mode

```almide
let opts = almai.defaults() |> almai.with_json_mode

let r = almai.call_with("openai/gpt-4o", [
  almai.user("List 3 colors as JSON array"),
], opts)!

let parsed = almai.parse_content_as_json(r)!
```

## Tool calling

```almide
import almai
import json

let weather_tool = Tool {
  name: "get_weather",
  description: "Get current weather for a city",
  parameters: value.object([
    ("type", value.str("object")),
    ("properties", value.object([
      ("city", value.object([
        ("type", value.str("string")),
        ("description", value.str("City name")),
      ])),
    ])),
    ("required", value.array([value.str("city")])),
  ]),
}

let opts = almai.defaults() |> almai.with_tools([weather_tool])

let r = almai.call_with("anthropic/claude-sonnet-4-6", [
  almai.user("What's the weather in Tokyo?"),
], opts)!

if almai.has_tool_calls(r) then {
  let tc = almai.first_tool_call(r) |> option.unwrap_or(ToolCall { id: "", name: "", arguments: "" })
  println("Tool: " + tc.name)
  println("Args: " + tc.arguments)
} else {
  println(r.content)
}
```

## Calls you can watch, stop and bound: `almai.live`

`call_with` blocks until the answer is in. An agent needs more: to show the answer as
it is written, to stop it when the person presses Esc, to give up on a stream that has
gone quiet, and to start a second copy of a request that is not arriving. `almai.live`
does that for every OpenAI-compatible service and for Claude Code's `claude -p`, and
[golemide](https://github.com/O6lvl4/golemide) and [comide](https://github.com/O6lvl4/comide)
are built on it.

```almide
import almai.live as live

let req = live.Request { model: "cf:glm-5.3", messages: msgs, tools: tools, effort: "low" }
let p = live.start(req, live.limits())!            // 900 s in all, 120 s of silence
while not live.is_done(p)! {
  show(live.text_so_far(p)!)                       // what has arrived so far
  if esc_pressed() then live.cancel(p)! else env.sleep_ms(250)
}
let r = live.finish(p)!                            // content, reasoning, calls, finish, usage, cost, warnings

// Or all at once, with a second copy after 120 s and a 6-minute cap:
let ran = live.run(req, live.limits(), 120000, 360000)!
```

Model ids are `PROVIDER/MODEL` or `PROVIDER:MODEL`:

| Model | Runs on | Needs |
|---|---|---|
| `cf:glm-5.3`, `cf/glm-5.3-flash`, `cf/@cf/…` | Cloudflare Workers AI | `CLOUDFLARE_ACCOUNT_ID` (or `CF_ACCOUNT_ID`) and `CLOUDFLARE_API_TOKEN` (or `CLOUDFLARE_EMAIL` + `CLOUDFLARE_API_KEY`) |
| `openai:…`, `openrouter:…`, `deepseek:…`, `zai:…`, `groq:…` | that OpenAI-compatible service | `OPENAI_API_KEY`, … |
| `ollama:…`, `lmstudio:…` | a local server | nothing |
| `NAME:MODEL` | any other OpenAI-compatible service | `NAME_BASE_URL`, `NAME_API_KEY` |
| `claude`, `claude:opus`, `cli/claude` | Claude Code's `claude -p`, on its own login | `claude` on `PATH` |

- The request runs in the background (curl, or claude) and writes to files, so it can be
  read while it arrives and stopped at any point, and curl bounds it in time. Chat
  requests are always streamed: a non-streamed Cloudflare request past about four
  minutes is ended with `408`. Credentials go in curl's config file, never on a
  command line.
- `schema` asks for an answer in that JSON Schema (`response_format`, or `--json-schema`
  for claude). `effort` is sent as `reasoning_effort` only when set: Cloudflare takes it
  for every model, other services refuse it on a model that does not reason.
- Cost is the provider's own meter when it reports one (Cloudflare's neurons,
  OpenRouter's `usage.cost`, claude's total), else Cloudflare's price table, else 0.
- `claude` runs `claude -p` with its tools, settings, hooks and MCP servers off; your
  global `CLAUDE.md` and memory are still read (only `--bare` leaves them out, and it
  takes an API key). With `tools`, the answer comes back as JSON in the reply's text,
  not through `--json-schema`: holding the schema as a tool, Claude tried to call the
  listed tools as its own and gave up. With `session` set to a file, the session is
  continued with `--resume`, so each call sends only the new messages.
- Failures are `almai.core.LlmError`: `RateLimited(retry_after_ms, …)`, `Overloaded`,
  `Timeout`, `Transport`, `ContextTooLong`, `ContentFiltered`, `Auth`, `NotFound`,
  `BadRequest`, `Cancelled`, `Truncated`, … One `core.classify` reads a failed reply,
  body first (a context overflow comes as a 400); `core.retryable(e)` says whether to
  try again as is and `core.wants_fallback(e)` whether to try another model; Retry-After
  is read from the response headers. `core.error_text(e)` words it as before:
  `status NNN: …`, `transport: …`, `request timeout: …`.
- `finish` is `Stop | Length | ToolCalls | ContentFilter | OtherFinish(raw)`, from one
  table of every provider's word, with `raw_finish` kept. `usage` splits uncached input,
  cache read, cache write, output and reasoning; a count the provider did not report is
  `none`, not 0. `cost` says where it came from: `"provider"`, `"table"` or `"none"`.
  `warnings` lists what the call could not honour (claude -p and a reasoning effort, say).
- Hedging stays in `live.run`'s loop. Almide's `fan.any` takes the first Ok in source
  order and waits for the earlier arms, so it suits a fallback chain run in parallel
  (it stops the later arms once an earlier one wins), not a first-to-finish race.

## Conversation builder

```almide
import almai
import almai.conv

let conversation = conv.empty()
  |> conv.add_system("You are a math tutor.")
  |> conv.add_user("What is a derivative?")
  |> conv.add_assistant("A derivative measures the rate of change...")
  |> conv.add_user("Can you give an example?")

let r = almai.call("anthropic/claude-sonnet-4-6", conv.messages(conversation))!
println(r.content)
```

## Retry on transient errors

`call_retry` wraps the dispatch in an exponential-backoff loop. HTTP 429 / 5xx
and connection errors are retried; auth / malformed-request errors propagate
immediately. Initial delay 1000 ms, doubles each retry.

```almide
import almai

let r = almai.call_retry(
  "anthropic/claude-sonnet-4-6",
  [almai.user("Hello")],
  almai.defaults(),
  3,                    // max_attempts: 1 try + 2 retries
)!
println(r.content)
```

For custom initial delay, use `call_retry_with_delay(..., max_attempts, base_delay_ms)`.

## Architecture

```
src/
  mod.almd              Public API, types, dispatch
  live.almd             Calls you can watch, stop and bound in time (curl / claude -p)
  tools.almd            Tool calling types and JSON Schema helpers
  conv.almd             Conversation builder
  providers/
    openai.almd         OpenAI / OpenRouter (OpenAI-compatible)
    anthropic.almd      Anthropic Messages API
    cloudflare.almd     Cloudflare Workers AI
    azure.almd          Azure OpenAI
    google.almd         Google Gemini
    cli.almd            Claude Code / Codex CLI
```

All providers are pure Almide — no external SDK dependencies. The providers under
`providers/` call the REST API via `http.request`; `almai.live` runs `curl` or `claude`
in the background.

## License

MIT
