# almai

Multi-provider LLM client for [Almide](https://github.com/almide/almide). One interface, every provider.

## Install

```toml
# almide.toml
[dependencies]
almai = { git = "https://github.com/almide/almai.git" }
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

All providers are pure Almide — no external SDK dependencies. Each provider directly calls the REST API via `http.request`.

## License

MIT
