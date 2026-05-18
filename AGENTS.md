# AGENTIC DIRECTIVE

> This file is identical to CLAUDE.md. Keep them in sync.

## CODING ENVIRONMENT

- Install astral uv using "curl -LsSf https://astral.sh/uv/install.sh | sh" if not already installed and if already installed then update it to the latest version
- Install Python 3.14 using `uv python install 3.14` if not already installed
- Always use `uv run` to run files instead of the global `python` command.
- Current uv ruff formatter is set to py314 which has supports multiple exception types without paranthesis (except TypeError, ValueError:)
- Read `.env.example` for environment variables.
- All CI checks must pass; failing checks block merge.
- Add tests for new changes (including edge cases), then run `uv run pytest`.
- Run checks in this order: `uv run ruff format`, `uv run ruff check`, `uv run ty check`, `uv run pytest`.
- Do not add `# type: ignore` or `# ty: ignore`; fix the underlying type issue.
- All 5 checks are enforced in `tests.yml` on push/merge (parallel jobs: suppression grep, ruff-format, ruff-check, ty, pytest).
- Branch protection: set **required status checks** to **all** of those statuses (e.g. **Ban type ignore suppressions**, **ruff-format**, **ruff-check**, **ty**, **pytest**—use the exact labels GitHub shows, which may be prefixed with **CI /**). Remove **ci** from required checks if it was previously added for the old gate job.

## IDENTITY & CONTEXT

- You are an expert Software Architect and Systems Engineer.
- Goal: Zero-defect, root-cause-oriented engineering for bugs; test-driven engineering for new features. Think carefully; no need to rush.
- Code: Write the simplest code possible. Keep the codebase minimal and modular.

## ARCHITECTURE PRINCIPLES

- **Shared utilities**: Put shared Anthropic protocol logic in neutral `core/anthropic/` modules. Do not have one provider import from another provider's utils.
- **DRY**: Extract shared base classes to eliminate duplication. Prefer composition over copy-paste.
- **Encapsulation**: Use accessor methods for internal state (e.g. `set_current_task()`), not direct `_attribute` assignment from outside.
- **Provider-specific config**: Keep provider-specific fields (e.g. `nim_settings`) in provider constructors, not in the base `ProviderConfig`.
- **Dead code**: Remove unused code, legacy systems, and hardcoded values. Use settings/config instead of literals (e.g. `settings.provider_type` not `"nvidia_nim"`).
- **Performance**: Use list accumulation for strings (not `+=` in loops), cache env vars at init, prefer iterative over recursive when stack depth matters.
- **Platform-agnostic naming**: Use generic names (e.g. `PLATFORM_EDIT`) not platform-specific ones (e.g. `TELEGRAM_EDIT`) in shared code.
- **No type ignores**: Do not add `# type: ignore` or `# ty: ignore`. Fix the underlying type issue.
- **Complete migrations**: When moving modules, update imports to the new owner and remove old compatibility shims in the same change unless preserving a published interface is explicitly required.
- **Maximum Test Coverage**: There should be maximum test coverage for everything, preferably live smoke test coverage to catch bugs early

## COGNITIVE WORKFLOW

1. **ANALYZE**: Read relevant files. Do not guess.
2. **PLAN**: Map out the logic. Identify root cause or required changes. Order changes by dependency.
3. **EXECUTE**: Fix the cause, not the symptom. Execute incrementally with clear commits.
4. **VERIFY**: Run ci checks and relevant smoke tests. Confirm the fix via logs or output.
5. **SPECIFICITY**: Do exactly as much as asked; nothing more, nothing less.
6. **PROPAGATION**: Changes impact multiple files; propagate updates correctly.

## SUMMARY STANDARDS

- Summaries must be technical and granular.
- Include: [Files Changed], [Logic Altered], [Verification Method], [Residual Risks] (if no residual risks then say none).

## TOOLS

- Prefer built-in tools (grep, read_file, etc.) over manual workflows. Check tool availability before use.

---

## CODEBASE OVERVIEW

`free-claude-code` is an Anthropic-compatible proxy (v2.0.0) that routes Claude Code API traffic to alternative AI providers: NVIDIA NIM, Kimi, Wafer, OpenRouter, DeepSeek, LM Studio, llama.cpp, and Ollama. It presents a stable Anthropic Messages API surface to Claude Code while translating requests/responses to each backend.

### Entry Points

| Command | Module | Purpose |
|---|---|---|
| `fcc-server` / `free-claude-code` | `cli.entrypoints:serve` | Start the proxy |
| `fcc-claude` | `cli.entrypoints:launch_claude` | Launch Claude Code with proxy env vars |
| `fcc-init` | `cli.entrypoints:init` | Scaffold `~/.config/free-claude-code/.env` |
| `uvicorn api.app:create_app --factory` | `api/app.py` | ASGI factory form |
| `server:app` | `server.py` | Module-level ASGI app (convenience) |

### Package Structure

```
free-claude-code/
├── server.py              # Module-level ASGI app instance (server:app)
├── api/                   # FastAPI routes, orchestration, admin UI
│   ├── app.py             # create_app() factory, lifespan, error handlers
│   ├── runtime.py         # AppRuntime: startup/shutdown, messaging wiring
│   ├── routes.py          # /v1/messages, /v1/messages/count_tokens, /v1/models
│   ├── admin_routes.py    # /admin endpoints (loopback-only)
│   ├── model_router.py    # Claude model → provider/model resolution
│   ├── optimization_handlers.py  # Local response shortcuts (probes, title gen, etc.)
│   ├── services.py        # Service-layer helpers
│   ├── dependencies.py    # FastAPI DI: get_provider, resolve_provider
│   ├── detection.py       # Request classification
│   ├── validation_log.py  # 422 request shape logging
│   ├── command_utils.py   # Shared command utilities
│   ├── gateway_model_ids.py  # Gateway model ID list
│   ├── web_server_tools.py   # web_search / web_fetch tool handling
│   ├── models/            # Pydantic request/response schemas
│   │   ├── anthropic.py   # AnthropicRequest and sub-types
│   │   └── responses.py   # Response models
│   ├── web_tools/         # Outbound HTTP for web_fetch tool
│   │   ├── constants.py
│   │   ├── egress.py
│   │   ├── outbound.py
│   │   ├── parsers.py
│   │   ├── request.py
│   │   └── streaming.py
│   └── admin_static/      # Admin UI (HTML/CSS/JS)
│
├── providers/             # Provider adapters and registry
│   ├── base.py            # BaseProvider (ABC), ProviderConfig
│   ├── registry.py        # ProviderRegistry, create_provider, PROVIDER_FACTORIES
│   ├── openai_compat.py   # OpenAIChatTransport base (NIM, Kimi)
│   ├── anthropic_messages.py  # AnthropicMessagesTransport base (OR, DS, LMS, etc.)
│   ├── model_listing.py   # ProviderModelInfo, model_infos_from_ids
│   ├── error_mapping.py   # HTTP status → ProviderError mapping
│   ├── exceptions.py      # ProviderError hierarchy
│   ├── rate_limit.py      # Per-provider rate limiting
│   ├── defaults.py        # Default base URLs (re-exports from provider_catalog)
│   ├── nvidia_nim/        # OpenAI-chat transport → Anthropic SSE
│   ├── open_router/       # Native Anthropic Messages
│   ├── deepseek/          # Native Anthropic Messages (api.deepseek.com/anthropic)
│   ├── kimi/              # OpenAI-chat transport
│   ├── wafer/             # Native Anthropic Messages (pass.wafer.ai/v1/messages)
│   ├── lmstudio/          # Native Anthropic Messages (local)
│   ├── llamacpp/          # Native Anthropic Messages (local)
│   └── ollama/            # Native Anthropic Messages (local)
│
├── core/anthropic/        # Shared Anthropic protocol helpers (no upward imports)
│   ├── content.py         # Content extraction utilities
│   ├── conversion.py      # Request conversion helpers
│   ├── sse.py             # SSE builder / chunker
│   ├── stream_contracts.py   # Stream contract assertions (used by tests and smoke)
│   ├── thinking.py        # Thinking block normalisation
│   ├── tools.py           # Tool call helpers
│   ├── tokens.py          # Token estimation (tiktoken)
│   ├── errors.py          # User-facing error strings
│   ├── utils.py           # Misc shared utilities
│   ├── emitted_sse_tracker.py  # SSE event dedup tracking
│   ├── native_messages_request.py  # Native Anthropic request builder
│   ├── native_sse_block_policy.py  # SSE block emission policy
│   ├── provider_stream_error.py    # Stream error extraction
│   └── server_tool_sse.py          # Server-side tool SSE events
│
├── messaging/             # Discord / Telegram bot adapters
│   ├── platforms/         # Platform implementations (base, discord, telegram, factory)
│   ├── rendering/         # Per-platform Markdown rendering
│   ├── trees/             # Message tree / reply-branch threading
│   ├── handler.py         # Inbound message orchestration
│   ├── command_dispatcher.py  # /stop /clear /stats commands
│   ├── session.py         # Session state persistence
│   ├── transcript.py      # Conversation transcript management
│   ├── transcription.py   # Voice → text (Whisper / NIM)
│   ├── voice.py           # Voice note intake
│   └── node_event_pipeline.py  # Claude CLI event → platform message pipeline
│
├── cli/                   # Package entrypoints and Claude subprocess management
│   ├── entrypoints.py     # fcc-server, fcc-claude, fcc-init
│   ├── manager.py         # Claude CLI subprocess lifecycle
│   ├── session.py         # CLI session abstraction
│   └── process_registry.py  # Active process tracking
│
├── config/                # Settings and provider catalog
│   ├── settings.py        # Settings (pydantic-settings), get_settings()
│   ├── provider_catalog.py   # PROVIDER_CATALOG, ProviderDescriptor, transport types
│   ├── provider_ids.py    # SUPPORTED_PROVIDER_IDS tuple
│   ├── nim.py             # NimSettings (NIM-specific config)
│   ├── constants.py       # Shared constants
│   └── logging_config.py  # Loguru configuration
│
├── tests/                 # Unit and contract tests (always run; deterministic)
│   ├── conftest.py
│   ├── api/               # API route and service tests
│   ├── cli/               # CLI management tests
│   ├── config/            # Settings and logging tests
│   ├── contracts/         # Architecture contract tests (import boundaries, etc.)
│   ├── core/              # core.anthropic tests
│   ├── messaging/         # Messaging adapter tests
│   ├── providers/         # Provider adapter tests
│   └── stream_contract.py # Shared stream contract helpers
│
└── smoke/                 # Live integration tests (opt-in: FCC_LIVE_SMOKE=1)
    ├── prereq/            # Prerequisite checks (auth, CLI, providers, etc.)
    └── product/           # End-to-end product scenarios
```

### Dependency Direction (enforced by `tests/contracts/test_import_boundaries.py`)

```
config → api, providers, messaging
core.anthropic → api, providers, messaging
providers → api
api → cli, messaging
cli → messaging
```

**Hard rules:**
- `core/` must NOT import `api`, `messaging`, `cli`, `providers`, `config`, or `smoke`.
- `api/` may only import `providers.base`, `providers.exceptions`, and `providers.registry` from the providers package (no per-adapter modules).
- `messaging/` must NOT import `api`, `cli`, or `smoke`; may import `providers.nvidia_nim.voice` only.
- Provider adapters must not import from each other.

### Model String Format

All model references use `provider_id/model/name`:

```
nvidia_nim/z-ai/glm4.7
open_router/deepseek/deepseek-r1-0528:free
kimi/kimi-k2.5
wafer/DeepSeek-V4-Pro
ollama/llama3.1:8b
```

Valid `provider_id` values: `nvidia_nim`, `open_router`, `deepseek`, `lmstudio`, `llamacpp`, `ollama`, `kimi`, `wafer`.

### Transport Types

| Type | Providers | How it works |
|---|---|---|
| `openai_chat` | `nvidia_nim`, `kimi` | OpenAI Chat Completions SSE → Anthropic SSE translation |
| `anthropic_messages` | all others | Native Anthropic Messages API pass-through |

### Key Settings (`config/settings.py`)

Settings are loaded via `pydantic-settings` from env files in priority order:
1. `.env` (repo root)
2. `~/.config/free-claude-code/.env`
3. `$FCC_ENV_FILE` (if set)

Access settings via `config.settings.get_settings()` (LRU-cached singleton).

Production HTTP handlers must use `resolve_provider(request.app)` to get the app-scoped `ProviderRegistry`. Use `get_provider` / `get_provider_for_type` only in scripts and unit tests.

### Adding a New Provider

1. Add the provider ID string to `config/provider_ids.py::SUPPORTED_PROVIDER_IDS`.
2. Add a `ProviderDescriptor` entry to `config/provider_catalog.py::PROVIDER_CATALOG`.
3. Implement a `BaseProvider` subclass under `providers/<name>/client.py`. Extend `OpenAIChatTransport` for OpenAI-compatible APIs or `AnthropicMessagesTransport` for native Anthropic Messages endpoints.
4. Register a factory function in `providers/registry.py::PROVIDER_FACTORIES`.
5. Add credential/URL fields to `config/settings.py::Settings` if needed.
6. Add contract tests and unit tests. All three dicts (`PROVIDER_CATALOG`, `PROVIDER_FACTORIES`, `SUPPORTED_PROVIDER_IDS`) must stay in sync — an `AssertionError` is raised at import time if they diverge.

### Request Optimizations

`api/optimization_handlers.py` intercepts known Claude Code probe requests and answers them locally (network probe mock, title generation skip, suggestion mode skip, filepath extraction mock). This saves quota and latency. Enable/disable via `Settings.enable_*` flags.

### Admin UI

Served at `/admin` (loopback-only). Reads/writes `.env` at runtime, validates config, and restarts the server when runtime settings change. Static files live in `api/admin_static/`.

### Smoke Tests

Live smoke tests live under `smoke/` and require `FCC_LIVE_SMOKE=1`. They use two skip classes:
- `missing_env`: credentials, binaries, or opt-in flags absent (acceptable skip).
- `upstream_unavailable`: real provider unreachable (acceptable skip).

`product_failure` and `harness_bug` are regressions — never mark them as acceptable skips.

Run smoke tests with `-n 0` (no parallelism) unless a scenario is explicitly safe under xdist.

### Logging

Uses Loguru. Structured `TRACE` rows include fields `"trace": true`, `stage`, `event`, `source`. Credential-like keys are redacted. Enable verbose payloads only for local debugging with the `LOG_RAW_*` env flags — they may expose prompt content.
