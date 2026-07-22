# Authoritative model, context, and token-count sources

**Decision date:** 2026-07-22

**Question:** Which first-party source is authoritative for model identity, usable context, output reserve, and token-count semantics in each native API and consumer-subscription lane, how do Claude Code and Codex consume that information, and how should catalog drift be detected before it reaches production?

## Decision

Manifest must maintain **four separate model contracts**, keyed by execution lane:

1. Anthropic API;
2. Claude Code with a claude.ai subscription;
3. OpenAI API;
4. Codex with a ChatGPT subscription.

A shared marketing name does not make two lanes equivalent. Model identity, entitlement, context, output headroom, token accounting, and client compaction behavior all belong to the lane. Aliases are request-time selectors, not stable catalog identities; resolve an alias to a concrete model when a session lease is created and record the source revision that justified the resolution.

The authority order is:

1. **An authenticated, first-party machine-readable catalog for the same lane and account**, where one exists.
2. **The catalog and behavior consumed by the current official client for that lane.**
3. **First-party model documentation for that exact API lane.**
4. **A versioned, conservative Manifest override**, allowed only when the first three do not expose a field. It must never exceed a verified upstream or client limit.

Provider acceptance and returned usage remain the final runtime truth. A static catalog is an admission and compaction contract, not a guarantee that every account has the advertised entitlement.

At research time the current stable clients were [Claude Code v2.1.217](https://github.com/anthropics/claude-code/releases/tag/v2.1.217) and [Codex v0.145.0](https://github.com/openai/codex/releases/tag/rust-v0.145.0).

## Authority matrix

| Lane | Identity and availability | Context and output authority | Token-count authority | What Manifest may advertise |
| --- | --- | --- | --- | --- |
| Anthropic API | Authenticated [`GET /v1/models`](https://platform.claude.com/docs/en/api/models/list) and [`GET /v1/models/{model_id}`](https://platform.claude.com/docs/en/api/models/retrieve). Retrieval resolves an alias to a concrete returned `id`. | The same Models API now returns `max_input_tokens` and `max_tokens`; use the [context-window semantics](https://platform.claude.com/docs/en/build-with-claude/context-windows) for the combined input/output budget and current model documentation for explanatory detail. | Provider-native [`POST /v1/messages/count_tokens`](https://platform.claude.com/docs/en/build-with-claude/token-counting) for the same model, headers, and structured request. It is a provider estimate and may differ slightly from message usage. Completed message `usage` is runtime truth. | No more than the authenticated model record permits, less the selected output and safety reserve. |
| Claude Code subscription | The model selected by the authenticated account's current Claude Code picker/configuration. `default`, `opus`, and `sonnet` are moving selectors; a full model name pins the requested version. [Plan entitlement and extended-context selection](https://code.claude.com/docs/en/model-config#extended-context) are Claude Code subscription rules, not Anthropic API entitlement. | Claude Code's documented built-in model behavior plus the authenticated plan's actual entitlement. In particular, gateway-routed Sonnet 5 is budgeted at 200K unless its 1M option is explicitly selected. Claude Code's gateway discovery reads only `id` and `display_name`, not context fields. | The gateway's native `count_tokens` pass-through when present; otherwise Claude Code estimates locally. The [gateway protocol](https://code.claude.com/docs/en/llm-gateway-protocol#optional-endpoints-and-startup-traffic) makes token counting optional. | The lower of the verified subscription entitlement, Claude Code's configured window, and the upstream model limit. Never infer subscription entitlement from API documentation. |
| OpenAI API | Authenticated [`GET /v1/models`](https://developers.openai.com/api/reference/resources/models/methods/list) establishes basic identity and availability. It exposes `id`, `created`, `object`, and `owned_by`, but not context metadata. | The official per-model developer page for the exact API model. For example, [GPT-5.6 Sol](https://developers.openai.com/api/docs/models/gpt-5.6-sol) documents a 1,050,000-token context window and 128,000 maximum output. | Provider-native [`POST /v1/responses/input_tokens`](https://developers.openai.com/api/reference/resources/responses/subresources/input_tokens/methods/count) for the same model and Responses input, followed by the completed Response's returned `usage` as runtime truth. | No more than the documented API limit for that model and endpoint, reduced by requested output and a measured serialization/safety reserve. |
| Codex ChatGPT subscription | The authenticated Codex backend `/models` response for the installed official client. Codex sends its client version and treats a valid visible remote catalog as the sole model source for ChatGPT auth. Its bundled `models.json` is the offline/failure fallback. | `context_window`, `max_context_window`, `effective_context_window_percent`, and `auto_compact_token_limit` from that same Codex catalog, interpreted by the official client. | Exact upstream usage from a completed Responses call; Codex uses a coarse byte heuristic only when it must reconstruct history without server usage. | The lower of the authenticated remote catalog, the same release's bundled fallback, and any independently verified provider rejection boundary. OpenAI API limits are not Codex subscription limits. |

## The current cross-lane mismatch is material

The OpenAI API page currently gives `gpt-5.6-sol` a **1,050,000-token** context and **128,000-token** maximum output. The current stable Codex release's bundled catalog gives that same slug a **272,000-token** `context_window` and `max_context_window` ([catalog source](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/models-manager/models.json#L4-L28)).

Codex then reserves headroom in two stages:

- `effective_context_window_percent` has a [95% default](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/protocol/src/openai_models.rs#L351-L356), explicitly reserving headroom for system prompts, tools, and model output ([field semantics](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/protocol/src/openai_models.rs#L421-L424));
- absent an explicit value, automatic compaction defaults to 90% of the raw context ([calculation](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/protocol/src/openai_models.rs#L454-L470)).

For a 272,000-token catalog record, that yields:

- raw catalog window: **272,000**;
- client effective window: **258,400**;
- default auto-compaction threshold: **244,800**.

The official client applies the effective percentage when exposing its working context limit ([Codex turn context](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/core/src/session/turn_context.rs#L213-L219)). Therefore 1.05M is correct for the OpenAI API lane but unsafe for the current ChatGPT/Codex subscription lane. The catalog's `supported_in_api: true` means the model can be used with API authentication; it does not make the two lanes' context contracts identical.

This is the exact class of error a unified `model -> context` table creates. Manifest must key its records by at least `(provider, auth lane, protocol, concrete model, client/catalog version)`.

## Field semantics Manifest must keep distinct

| Field | Meaning |
| --- | --- |
| `requested_model` | The alias or ID received from the harness. Useful for audit, never sufficient for admission. |
| `resolved_model` | Concrete upstream ID leased to the session after alias resolution. |
| `advertised_context` | The provider or client catalog's nominal window. It is not automatically usable input. |
| `max_output_cap` | Maximum legal output parameter for the model; not necessarily the output budget chosen for this request. |
| `requested_output_reserve` | `max_tokens`/`max_output_tokens` plus reasoning/output behavior selected for the turn. |
| `client_usable_context` | The context the harness uses for percentage displays and compaction decisions. |
| `client_compact_at` | The harness's actual compaction threshold, which can be below its usable context. |
| `gateway_admit_at` | Conservative limit for the exact serialized upstream request after translation and tool/system overhead. |
| `counter_source` | Provider endpoint, provider-returned usage, official-client estimate, target tokenizer, or Manifest fallback, including a version. |
| `source_revision` | API response ETag/hash, official client version and catalog SHA, documentation fingerprint, and observation time. |

For a provider whose window includes input and generated output, the request admission ceiling is conceptually:

```text
provider_request_input_ceiling = min(
  provider_max_input,
  provider_total_context - requested_output_reserve
)

gateway_admit_at = min(
  provider_request_input_ceiling,
  client_usable_context,
  execution_engine_limit
) - measured_adapter_and_safety_margin
```

Do not reserve the model's maximum output on every request unless the client actually requests it; do reserve the selected request output plus measured tool, system, reasoning, and serialization overhead. Anthropic explicitly says the context window contains both prior input and the new output, and Claude Code documents that increasing `CLAUDE_CODE_MAX_OUTPUT_TOKENS` reduces the window available before compaction.

## How Claude Code consumes the contract

Claude Code's [`default`](https://code.claude.com/docs/en/model-config#model-aliases) setting clears an override and chooses the recommendation for the account or organization. `opus` and `sonnet` track recommended versions and can change over time; a full model name is the pinning mechanism.

For extended context, entitlement is account- and model-dependent. The current documentation says Opus is automatically upgraded to 1M on Max, Team, and Enterprise, while other plan/model combinations may require usage credits. Behind an LLM gateway, Claude Code cannot verify Sonnet 5's 1M support and budgets it at 200K unless the 1M selection is used ([model configuration](https://code.claude.com/docs/en/model-config#sonnet-5-context-window)). This means a transparent gateway cannot merely forward a 1M-capable upstream and expect Claude Code to use the larger window.

Claude Code offers two different controls:

- [`CLAUDE_CODE_AUTO_COMPACT_WINDOW`](https://code.claude.com/docs/en/env-vars) changes the capacity used for auto-compaction and is capped at the model's actual built-in window.
- `CLAUDE_CODE_MAX_CONTEXT_TOKENS` changes the assumed window directly for unrecognized model IDs; for recognized Claude models it only applies with `DISABLE_COMPACT`. It exists specifically for custom models behind `ANTHROPIC_BASE_URL`.

Gateway model discovery does not solve this. The protocol states that Claude Code requests `GET /v1/models?limit=1000`, reads `id` and optional `display_name`, and ignores other entries and fields ([discovery contract](https://code.claude.com/docs/en/llm-gateway-protocol#request-and-response)). Discovery is disabled by default, cached locally, and falls back to the built-in list. Consequently, returning `max_input_tokens` from Manifest's discovery endpoint does **not** configure Claude Code's compaction window today.

Token counting is optional. When `POST /v1/messages/count_tokens` is absent, Claude Code estimates context locally. When present on a native Anthropic route, Manifest must forward the same model, structured body, `anthropic-version`, and the open-ended `anthropic-beta` list. Counting one model and executing another invalidates the result because the token counter is model-specific; Anthropic notes that current tokenizer generations can differ by roughly 30% for the same input.

## How Codex consumes the contract

Codex's OpenAI-compatible model manager fetches `/models?client_version=...` using the active lane's authentication ([endpoint source](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/model-provider/src/models_endpoint.rs#L38-L115)). For ChatGPT authentication, a non-empty remote result with a visible model replaces the bundled catalog rather than merely supplementing it ([manager source](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/models-manager/src/manager.rs#L394-L450)). The client caches the catalog with its ETag and a [five-minute default TTL](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/models-manager/src/manager.rs#L26-L59); the bundled release file is the starting and fallback catalog.

The important implication is that the authenticated remote response is the primary subscription-lane authority, and `models.json` from the same released client is the reproducible fallback and CI oracle. A third-party router's copied catalog is neither.

Codex permits configuration overrides, but `model_context_window` is clamped to `max_context_window` ([override source](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/models-manager/src/model_info.rs#L25-L37)). A custom catalog can replace the bundled one for testing, but increasing metadata cannot grant the upstream a larger real window. Production must not use overrides to erase a remote/provider reduction.

For accounting, Codex distinguishes exact provider usage from local reconstruction. The official protocol labels `RawResponseCompletedEvent.token_usage` as exact upstream usage ([source](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/protocol/src/protocol.rs#L1826-L1833)); its history estimator explicitly calls itself a coarse byte-based lower bound rather than a tokenizer-accurate count ([source](https://github.com/openai/codex/blob/rust-v0.145.0/codex-rs/core/src/context_manager/history.rs#L162-L180)). Admission decisions should therefore use the catalog plus a conservative estimator, then reconcile against returned usage. They must not promote Codex's fallback estimate to provider truth.

## Native token-count contract

Anthropic provides the stronger preflight primitive:

- the endpoint accepts the same structured input categories as Messages, including system blocks, tools, images, and PDFs;
- the count is under the tokenizer of the requested model;
- Anthropic calls it an estimate and says actual Message input can differ slightly;
- completed Message usage is the authoritative observed count;
- cached input still occupies context: `input_tokens`, `cache_read_input_tokens`, and `cache_creation_input_tokens` all count toward the window.

OpenAI's public Models endpoint provides no context fields, but the Responses API now provides [`POST /v1/responses/input_tokens`](https://developers.openai.com/api/reference/resources/responses/subresources/input_tokens/methods/count). It accepts the target `model` and Responses input shape and returns a `response.input_tokens` object with `input_tokens`. For the OpenAI API lane, Manifest must submit the **post-translation target request input** to that endpoint and reconcile it with the completed Response's returned `usage`. A pinned target tokenizer/estimator with margin remains the fallback when this optional preflight is unavailable. For the ChatGPT/Codex subscription lane, no public API entitlement should be assumed; use the authenticated Codex catalog, conservative admission, and official Codex response usage to correct the running state.

For an Anthropic-facing translated route, do not label a character heuristic as provider-native counting. The acceptable choices are:

1. return the target request's conservative post-translation estimate and record its source/version in telemetry; or
2. leave the optional endpoint unavailable so Claude Code knowingly falls back to its local estimate, while Manifest independently rejects requests that exceed its target-lane admission ceiling.

The first choice gives the client better compaction behavior only after parity tests show it is conservative across tools, images, thinking, and model revisions. Native Anthropic routes must always use native count-token semantics.

## Catalog-drift detector

The detector should be a release gate and scheduled job, not an auto-updater. Its output is a normalized, credential-free catalog artifact reviewed like code.

### Inputs

1. **Anthropic API snapshot:** authenticated `/v1/models` plus retrieval of every enabled model; retain only IDs, aliases as resolved, capability booleans, `max_input_tokens`, `max_tokens`, response ETag/hash, and observation time.
2. **Claude Code contract:** current and previous stable release versions; fingerprint the official model-configuration, environment-variable, and gateway-protocol pages; run an authenticated synthetic canary for every enabled subscription plan/model selection, including the standard and `[1m]` choices. Store no prompts or credentials.
3. **OpenAI API snapshot:** authenticated `/v1/models` for availability; parse the official per-model page for context and max output. A page parse failure is an unknown contract, not permission to retain an unverified increase.
4. **Codex subscription snapshot:** current and previous stable release tags and their bundled `models.json`; run the official client with an isolated home and authenticated test account so it refreshes its own `models_cache.json`. Normalize only routing/capability/context fields and the ETag; never copy authentication state.
5. **Golden token corpus:** synthetic requests covering plain text, Unicode, large tool schemas, tool results, images/PDF metadata, thinking/reasoning items, cached prefixes, and near-boundary prompts. Exercise each provider-native preflight counter where the lane supports one, then compare it with completed usage and the official client's estimate. Record corpus hashes and count results, not user content.

### Normalized record

```json
{
  "lane": "codex_chatgpt_subscription",
  "auth_class": "chatgpt_subscription",
  "protocol": "openai_responses",
  "requested_model": "gpt-5.6-sol",
  "resolved_model": "gpt-5.6-sol",
  "advertised_context": 272000,
  "max_context_override": 272000,
  "effective_context_percent": 95,
  "client_usable_context": 258400,
  "client_compact_at": 244800,
  "max_output_cap": null,
  "counter_source": "provider_usage_then_codex_estimator",
  "source": {
    "client_version": "0.145.0",
    "catalog_ref": "rust-v0.145.0",
    "etag": "<redacted-or-hash>",
    "observed_at": "2026-07-22T00:00:00Z"
  }
}
```

### Diff and enforcement rules

| Change | Action |
| --- | --- |
| Context, output cap, capability, or entitlement decreases | Block promotion immediately; lower new-session admission to the verified minimum. Existing leases keep their recorded contract only until a safe boundary, and must not be silently moved to an incompatible target. |
| Context or output cap increases | Do not auto-enable. Require authenticated boundary canaries, current/previous CLI conformance, and human approval before raising production admission. |
| Alias resolves to a different concrete model | Create a new catalog revision. New sessions may adopt it after conformance; existing sessions remain pinned. |
| Model disappears or becomes unavailable for a plan | Block new leases and route only at an explicit safe boundary to a separately verified compatible model. |
| Token-count corpus changes beyond the documented small provider variance | Block translation/native parity promotion and require tokenizer/adapter review. Never silently adjust historical session usage. |
| Source cannot be fetched, parsed, authenticated, or is stale | Keep the last approved catalog for already compatible operation, but block new models, increases, and releases that depend on the unknown value. Alert on staleness. |
| Additive model/capability | Open a review; do not expose it automatically. |

Run the lightweight source diff daily and on every upstream release. Run current/previous CLI startup and short token/stream conformance on every Manifest engine or catalog change. Run expensive near-boundary context tests only when a context, tokenizer, model, plan entitlement, or official CLI compaction rule changes, plus a periodic scheduled sample.

The detector must compare each production engine's reported catalog with the approved Manifest catalog. An engine may report a lower limit and make the route ineligible; it may never raise the effective limit. Every request should record the lane, resolved model, client version, approved catalog revision, client/gateway/upstream ceilings, counter source, and sanitized account/session hashes.

## Architectural consequence

Manifest should own the **catalog provenance and conservative admission contract** even when a sidecar executes the request. Execution engines are replaceable observations, not authorities allowed to enlarge a model's capability.

The session lease must pin:

```text
lane + protocol + concrete model + account class
+ client version + catalog revision
+ tokenizer/counter revision
+ advertised/client/admission context limits
+ output reserve policy
```

Cross-provider routing is eligible only when the target lane has its own verified record and the active session fits the lower portable contract. A cache miss, identical model slug, or larger API documentation number is not evidence that a subscription session can switch safely.

This hierarchy would have prevented the observed error: the OpenAI API's 1.05M GPT-5.6 Sol record could never populate the ChatGPT/Codex subscription record whose current official client catalog is 272K.
