# Switch Trust SDK

Lightweight SDK for routing LLM traffic through a guardrails proxy. Wraps OpenAI, Anthropic, Google GenAI, and LangChain LLM clients, with agent framework plugins for Google ADK.

## Features

- **LLM SDK wrapping** — route LLM traffic through the Switch Trust guardrails proxy with a single `switch_trust.wrap()` call. Supports OpenAI, Anthropic, Google GenAI, and LangChain chat models.
- **Agent framework plugins** — deeper integration with agent lifecycle hooks for metadata extraction and guardrails routing. Currently supports Google ADK.
- **Plugin system** for extensibility

## Supported Integrations

| Integration | Category | Guardrails Routing |
|---|---|---|
| OpenAI SDK | LLM SDK | `switch_trust.wrap()` |
| Anthropic SDK | LLM SDK | `switch_trust.wrap()` |
| Google GenAI SDK | LLM SDK | `switch_trust.wrap()` |
| LangChain (`ChatOpenAI`, `ChatAnthropic`, `ChatGoogleGenerativeAI`) | LLM SDK | `switch_trust.wrap()` |
| Google ADK | Agent Framework | `ADKGuardrailsPlugin` |

## Install

```bash
pip install switch-trust-sdk-py

# With optional extras for your LLM provider
pip install "switch-trust-sdk-py[openai]"     # OpenAI SDK
pip install "switch-trust-sdk-py[anthropic]"  # Anthropic SDK
pip install "switch-trust-sdk-py[genai]"      # Google GenAI SDK
pip install "switch-trust-sdk-py[adk]"        # Google ADK (includes GenAI)
pip install "switch-trust-sdk-py[langchain]"  # LangChain
pip install "switch-trust-sdk-py[all]"        # Everything
```

## Guardrails

Route all LLM traffic through the Switch Trust guardrails proxy. Create your sync client as usual, then call `switch_trust.wrap()` — it auto-detects the provider and applies guardrails routing. Async clients (`AsyncOpenAI`, `AsyncAnthropic`) are not supported.

### OpenAI

```python
import openai
import switch_trust

client = openai.OpenAI(api_key="your-openai-api-key")
client = switch_trust.wrap(
    client,
    gateway_url="https://api.switchagents.ai",
    api_key="your-guardrails-api-key",
    policy_id="your-policy-id",  # optional
)

response = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello"}],
)
```

> **Optional parameters:** besides the client object, every parameter passed to `wrap()` here — `gateway_url`, `api_key`, `policy_id` — is optional. Each falls back to its `SWITCH_TRUST_*` environment variable if omitted (see [Environment Variables](#environment-variables)).

### Anthropic

```python
import anthropic
import switch_trust

client = anthropic.Anthropic(api_key="your-anthropic-api-key")
client = switch_trust.wrap(
    client,
    gateway_url="https://api.switchagents.ai",
    api_key="your-guardrails-api-key",
)

message = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
```

> **Optional parameters:** as with OpenAI above, `gateway_url` and `api_key` are optional — they fall back to `SWITCH_TRUST_GATEWAY_URL`/`SWITCH_TRUST_API_KEY` if omitted (see [Environment Variables](#environment-variables)). The client object is the only argument `wrap()` strictly requires.

### Google GenAI

Google GenAI clients don't expose `api_key` as an attribute, so `llm_api_key` cannot be auto-extracted. Pass it explicitly so the SDK forwards your key to the gateway — see [Environment Variables](#environment-variables) to set it via `SWITCH_TRUST_LLM_API_KEY` instead:

```python
import google.genai
import switch_trust

client = google.genai.Client(api_key="your-gemini-api-key")
client = switch_trust.wrap(
    client,
    gateway_url="https://api.switchagents.ai",
    api_key="your-guardrails-api-key",
    llm_api_key="your-gemini-api-key",  # required — your upstream provider API key
)

response = client.models.generate_content(
    model="gemini-2.5-flash",
    contents="Hello",
)
```

> **Optional parameters:** `gateway_url` and `api_key` are optional here too, falling back to `SWITCH_TRUST_GATEWAY_URL`/`SWITCH_TRUST_API_KEY` if omitted (see [Environment Variables](#environment-variables)). `llm_api_key` is the exception for Google GenAI — it stays required, as noted above, since it can't be auto-extracted from the client.

### LangChain

Use `LangChainGuardrailsMiddleware` from `switch_trust.plugins.langchain` in `middleware=[...]` to route a LangChain agent's LLM traffic through the guardrails proxy:

> **Anthropic version caveat:** `pip install "switch-trust-sdk-py[langchain]"` alone does not pin the underlying `anthropic` SDK — `langchain-anthropic` pulls in its own, looser `anthropic` range, which can resolve to an untested major version. Guardrails routing for Anthropic depends on the pinned range in [Known Limitations](#known-limitations); on a newer major version, requests fail with `TypeError: Invalid type for url. Expected str or httpx2.URL, ...`, because that release moved to an internal `httpx2` client the base-URL rewrite doesn't account for. Install `pip install "switch-trust-sdk-py[langchain,anthropic]"` (or pin the `anthropic` version explicitly) to stay in the tested range.

```python
from switch_trust.plugins.langchain import LangChainGuardrailsMiddleware
from langchain.agents import create_agent

middleware = LangChainGuardrailsMiddleware()

agent = create_agent(
    model="openai:gpt-4o",
    middleware=[middleware],
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "Hello!"}]},
    config={"configurable": {"thread_id": "session-123"}},
)
```

> **Credentials:** `LangChainGuardrailsMiddleware()` with no arguments resolves `gateway_url`/`api_key`/`llm_api_key` from `SWITCH_TRUST_*` environment variables (see [Environment Variables](#environment-variables)). These must be exported in the process environment the agent runs in — a `.env` file is not loaded unless you pass `load_dotenv=True`; loading it is left to the agent developer.
>
> The chat model itself still needs its own provider credential to construct — e.g. `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, or `GOOGLE_API_KEY` — set the same way you normally would for that LangChain integration. This is separate from the `SWITCH_TRUST_*` keys above: the middleware only rewrites the model's base URL and headers for guardrails routing *after* it's constructed, so a missing provider key fails before guardrails ever get involved.

On every model call it attaches `X-Agent-Id` / `X-Agent-Name` (from `AGENT_ID` / `AGENT_NAME`), `X-Agent-Session-Id` (the invocation's `thread_id`) and `X-Agent-Turn-Id` (a new id per `agent.invoke`, shared by every model call of it, tool loop included) so the platform can show each turn as one interaction. The session and turn headers are set on the individual request, not the shared model client, so concurrent invocations of one agent never mix them up.

**Choosing a provider:** `model=` accepts either a `"provider:model"` string — `"openai:gpt-4o"`, `"anthropic:claude-sonnet-4-5"`, `"google_genai:gemini-2.5-flash"` — or an already-constructed chat model instance (`ChatAnthropic(...)`, `ChatGoogleGenerativeAI(...)`, etc.) when you need to pass provider-specific kwargs. Works with any LangChain chat model supported by `create_agent`.

### Google ADK

ADK agents lazily create their GenAI client at runtime and use `generate_content_config` for per-request routing, so `switch_trust.wrap()` cannot be used. Use the ADK plugin instead:

```python
from switch_trust.plugins.adk import ADKGuardrailsPlugin
from google.adk import Agent

plugin = ADKGuardrailsPlugin()

agent = Agent(
    model="gemini-3.6-flash",
    generate_content_config=plugin.content_config,
    before_model_callback=plugin.before_model_callback,
    on_model_error_callback=plugin.on_model_error,
)
```

> **Credentials:** `ADKGuardrailsPlugin()` with no arguments resolves `gateway_url`/`api_key`/`llm_api_key` from `SWITCH_TRUST_*` environment variables (see [Environment Variables](#environment-variables)). These must be exported in the process environment the agent runs in — a `.env` file is not loaded unless you pass `load_dotenv=True`; loading it is left to the agent developer.

> **Passing your own `content_config`:** if you supply `content_config=...` to the plugin, wire `plugin.content_config` (not your original object) into the `Agent`. The plugin returns a clone with the guardrails `http_options` attached and leaves your object untouched, so passing the original sends traffic straight to the model — and with `require_guardrails` enabled by default, `before_model_callback` raises.

`before_model_callback` automatically extracts agent identity and session metadata on each LLM call; it reads the agent name from `AGENT_NAME`, or from the ADK callback context if that is unset, and attaches `X-Agent-Id`, `X-Agent-Name`, `X-Agent-Session-Id`, and `X-Agent-Turn-Id` headers to the guardrails request. `X-Agent-Turn-Id`, the ADK's invocation ID, allows the platform to show each turn as one interaction. If guardrails routing was never applied (i.e. the request has no `http_options`), a `SwitchTrustGuardrailsError` is raised unless `require_guardrails=False` is passed. `on_model_error_callback` converts guardrails blocks (detected via the `GUARDRAIL_BLOCKED` error code) into an `LlmResponse` so the agent can handle them gracefully.

> The platform derives this agent's id from `AGENT_NAME`; only set `AGENT_ID` if you're pinning the exact id shown on the agent's page — an incorrect value disconnects this agent's traffic from its evals.

### Configuration Reference

`switch_trust.wrap()` and `switch_trust.init()` accept these guardrails parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `gateway_url` | `str` | Guardrails proxy URL |
| `api_key` | `str` | Your guardrails API key |
| `llm_api_key` | `str \| None` | Your upstream LLM provider API key, forwarded to the gateway. Required to reach the provider — pass it directly, via `forward_llm_key=True` (`wrap()` only, auto-extracts from `client.api_key`), or via the `SWITCH_TRUST_LLM_API_KEY` environment variable (see [Environment Variables](#environment-variables)). |
| `forward_llm_key` | `bool` | `wrap()` only. When `True`, auto-extract the provider key from the client (`client.api_key`) and forward it (default `False`) |
| `policy_id` | `str \| None` | Guardrails policy ID to apply (optional) |
| `require_guardrails` | `bool \| None` | When unset/`True` (default), raises `SwitchTrustGuardrailsError` if guardrails config cannot be resolved. `False` allows operation without guardrails. |
| `dangerously_disable_guardrails` | `bool` | Explicit, self-documenting opt-out equivalent to `require_guardrails=False`. Prefer this so unguarded deployments are greppable. Cannot be combined with `require_guardrails=True`. Either opt-out emits a once-per-process `SECURITY CONTROL DISABLED` log warning so it stays auditable. |

### Security posture

The opt-out is not silent: `init()`, `wrap()`, the ADK plugin, and the LangChain
middleware emit a `SECURITY CONTROL DISABLED` `logging.warning` (via the
`switch_trust.guardrails` logger, at most once per process) whenever guardrails are
disabled, so unguarded deployments are detectable in logs.

Call `switch_trust.status()` (or `client.guardrails_status()`) at runtime to inspect
the effective posture — a `GuardrailsStatus` with
`active, require_guardrails, provider, agent_id, agent_name`, where `active` is
`True` only when traffic is actually routed through the gateway. Returns `None`
before `init()`.

### Environment Variables

Instead of passing credentials in code, you can set them as environment variables:

| Environment Variable  | Parameter     | Description                                      |
|-----------------------|---------------|--------------------------------------------------|
| `SWITCH_TRUST_GATEWAY_URL` | `gateway_url` | Guardrails proxy URL                             |
| `SWITCH_TRUST_API_KEY`     | `api_key`     | Your guardrails API key                          |
| `SWITCH_TRUST_LLM_API_KEY` | `llm_api_key` | Your upstream LLM provider API key               |
| `SWITCH_TRUST_POLICY_ID`   | `policy_id`   | Guardrails policy ID (optional)                  |
| `AGENT_ID`            | —             | Overrides the id the platform derives from `AGENT_NAME`. Only set this to the exact id shown on the agent's page — a mismatched value breaks eval lookups for this agent. Leave unset in most cases. `wrap()`, the ADK plugin and the LangChain middleware send it as `X-Agent-Id`.|
| `AGENT_NAME`          | —             | Agent name attached to guardrails requests. `wrap()`, the ADK plugin and the LangChain middleware send it as `X-Agent-Name`.|
| `SWITCH_TRUST_ALLOWED_GATEWAY_HOSTS` | — | Comma-separated allowed gateway hostnames. Default: `api.switchagents.ai`, `api.flintai.dev`, `app.flintai.dev`. The `*` wildcard requires `allow_insecure_gateway=True` (see below). |

Environment variables are read automatically. A `.env` file is **not** read by
default — pass `load_dotenv=True` to load `<cwd>/.env`, or `load_dotenv="/path/to/.env"`
to load a specific trusted file (requires `pip install "switch-trust-sdk-py[dotenv]"`):

```bash
# .env
SWITCH_TRUST_GATEWAY_URL=https://api.switchagents.ai
SWITCH_TRUST_API_KEY=your-guardrails-api-key
SWITCH_TRUST_LLM_API_KEY=your-llm-api-key
```

Then use the Switch Trust SDK without passing credentials:

```python
import openai
import switch_trust

client = openai.OpenAI()
client = switch_trust.wrap(client, load_dotenv=True)  # reads env vars + <cwd>/.env
```

**Precedence:** Explicit parameters > environment variables > auto-extract from `client.api_key` (only in `wrap()`, and only when `forward_llm_key=True`).

**Gateway trust boundary:** The gateway receives the Switch Trust API key, provider
credentials, and all prompt/response traffic. By default only `api.switchagents.ai`,
`api.flintai.dev` and `app.flintai.dev` are allowed; override with `SWITCH_TRUST_ALLOWED_GATEWAY_HOSTS` (comma-separated
hostnames). A loopback `gateway_url` and the `SWITCH_TRUST_ALLOWED_GATEWAY_HOSTS='*'`
wildcard cannot be enabled from the environment or a `.env` file — pass
`allow_insecure_gateway=True` to `init()`/`wrap()` (local development only).
Requesting loopback or `*` without it raises at config time.

**Multiple provider keys:** If multiple provider API keys are set in the environment (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_API_KEY`), auto-detection will fail. Pass `provider` explicitly to `switch_trust.init()` to disambiguate.

### How It Works

1. `switch_trust.wrap(client, gateway_url=..., ...)` auto-detects the provider from the client type, computes the provider-specific path prefix (`/openai`, `/anthropic`, `/gemini`), rewrites the client's base URL, and injects custom headers (`X-FlintAI-API-Key`, and `X-LLM-API-Key` when an LLM key is available, `X-Guardrails-Policy-Id` when a policy is set, `X-Agent-Id` and `X-Agent-Name` when `AGENT_ID` and `AGENT_NAME` are set)
2. The guardrails proxy intercepts the request, applies policy checks (detectors), and strips the custom headers
3. The proxy injects the appropriate upstream auth credentials and forwards the request to the LLM provider

**Global state:** Each `switch_trust.wrap()` call updates the global Switch Trust SDK client's guardrails config. If you wrap multiple clients with different parameters, each client retains its own headers and base URL, but only the last `wrap()` call's config is stored globally. For most applications (single provider, shared credentials), this is transparent.

### Known Limitations

**Private attribute mutation:** `switch_trust.wrap()` modifies internal attributes of LLM SDK clients (`_base_url`, `_custom_headers`, `_api_client._http_options`) to redirect traffic through the guardrails proxy. These are not part of the public API of the respective SDKs and may change without notice. Pin your SDK versions to the tested ranges:

- `openai>=2.40.0,<3`
- `anthropic>=0.105.2,<1`
- `google-genai>=2.7,<3`

**Google GenAI `api_key`:** Google GenAI clients do not expose `api_key` as an attribute, so `llm_api_key` cannot be auto-extracted via `forward_llm_key`. Pass `llm_api_key=` explicitly, or set `SWITCH_TRUST_LLM_API_KEY` in the environment (see [Environment Variables](#environment-variables)).

**Google GenAI URL normalization:** The Google GenAI SDK requires a trailing slash on the base URL. The Switch Trust SDK normalizes this automatically — do not add a trailing slash to `gateway_url`.

**Async clients:** Async clients (`AsyncOpenAI`, `AsyncAnthropic`) are not supported. Use sync clients only.

**Thread safety:** `switch_trust.init()`, `switch_trust.wrap()`, and `switch_trust.shutdown()` are not thread-safe. Call them from the main thread during application startup. Once initialized, wrapped clients can be used from any thread — the underlying SDK clients handle their own thread safety.

### `switch_trust.init()` Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `provider` | `str \| None` | `None` | LLM provider (`"openai"`, `"anthropic"`, `"google"`) |
| `gateway_url` | `str \| None` | `None` | Guardrails proxy URL |
| `api_key` | `str \| None` | `None` | Your guardrails API key |
| `llm_api_key` | `str \| None` | `None` | Your upstream LLM provider API key |
| `policy_id` | `str \| None` | `None` | Guardrails policy ID (optional) |

## Plugins

Plugins handle events from the Switch Trust SDK lifecycle. Subclass `SwitchTrustPlugin` and override the methods you care about:

```python
from switch_trust.plugins import SwitchTrustPlugin

class MyPlugin(SwitchTrustPlugin):
    name = "my-plugin"

    def on_init(self, client):
        print(f"Plugin initialized with {client}")

    def on_shutdown(self):
        print("Shutting down")

# Register after init
switch_trust.init()
switch_trust.register_plugin(MyPlugin())
```

### Plugin Methods

| Method | Called when |
|--------|-----------|
| `on_init(client)` | Plugin is registered |
| `on_shutdown()` | `switch_trust.shutdown()` is called |

## Architecture

```
switch_trust.wrap(client, gateway_url=..., api_key=...)
    │
    ├── Auto-initializes SwitchTrustClient (singleton) if needed
    ├── Auto-extracts llm_api_key from client.api_key (OpenAI, Anthropic)
    ├── Configures guardrails (base_url, headers) from provided params
    ├── Detects client type (OpenAI, Anthropic, Google GenAI, or LangChain)
    ├── For LangChain: unwraps to underlying SDK client
    └── Applies guardrails config to client (rewrites base_url, injects headers)

switch_trust.shutdown()
    │
    └── Notifies plugins via on_shutdown()
```

## Development

```bash
# Install with dev dependencies
pip install -e ".[dev]"

# Run tests
pytest

# Lint
ruff check .

# Type check
mypy src
```

## Contributing

See [CONTRIBUTING.md](https://github.com/sandbox-quantum/switch-trust-sdk-py/blob/main/CONTRIBUTING.md).

## License

Apache 2.0, as limited by a modified Commons Clause license condition. See [LICENSE](https://github.com/sandbox-quantum/switch-trust-sdk-py/blob/main/LICENSE) for the full terms.
