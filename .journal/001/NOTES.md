---
id: 001
title: Start journal session
started: 2026-08-13
---

## 2026-08-13 22:17 — Kickoff
Goal for the session: Set up the repository session journal, then create a new current session.
Current state of the world: The personal journal branch and worktree are initialized and published; no earlier sessions exist.
Plan: Await the substantive task and record meaningful checkpoints as work progresses.

## 2026-08-13 22:20 — Goal refined
Goal for the session: Reimplement alfons-fhl/Cursor-Plan2API in Go with stronger engineering maturity.
Current state of the world: The upstream repository is cloned into a temporary directory for read-only discovery.
Plan: Map its architecture and behavior, identify maturity risks, then use the findings to guide the Go design.

## 2026-08-13 22:26 — Upstream discovery
Mapped 55 TypeScript source files (8,372 lines) and four custom test scripts (1,745 lines). The product is an HTTP compatibility facade over one Cursor `agent` subprocess per request, with OpenAI Chat/Responses, Anthropic Messages, embeddings, image generation, admin, usage, and daemon surfaces.
The central reusable behavior is protocol translation, prompt construction, Cursor process execution and stream parsing, request admission, session resume, and model discovery. Three large endpoint handlers duplicate orchestration instead of sharing an application service.
Material risks found: unauthenticated wildcard-CORS defaults combined with agent/home-workspace execution; unrestricted workspace and `file://` inputs; unbounded HTTP bodies, process buffers, and semaphore queue; ineffective temp cleanup; a warm “pool” that only launches disposable warmup requests; silent config/persistence fallbacks; fake non-semantic embedding fallback; container packaging without the Cursor CLI; and vulnerable runtime dependencies.
Verification: `npm ci`, `npm run build`, `npm run test:unit`, and CLI help succeeded. The custom suite reported 83 passes, but includes at least one unconditional assertion. `npm audit --omit=dev` reported four high and one critical runtime vulnerability.
Next: define a deliberately smaller Go compatibility contract and implement one shared execution pipeline before adding protocol adapters.

## 2026-08-13 22:38 — Hermes integration traced
Cloned NousResearch/hermes-agent at commit `9504edbaea29ce249864a1be05819d972f8fae8d`. Hermes has no Cursor-specific integration; it treats Cursor-Plan2API as a generic OpenAI-compatible custom provider at `http://127.0.0.1:8787/v1`.
The intended default is Hermes-owned tool execution: Hermes sends its OpenAI function schemas and conversation to `/chat/completions`; the bridge injects those schemas into a Cursor `ask` prompt, converts Cursor output into OpenAI `tool_calls`, and Hermes validates and executes the calls locally before posting tool results in the next request. This repeats until the bridge returns assistant text without tool calls.
The optional bridge `delegate` mode instead runs Cursor in `agent` mode against the user's home/workspace, suppresses returned tool calls, and gives Hermes only final text. That is an agent-inside-agent bypass of Hermes's tool loop, not the primary compatibility contract.
For a Hermes-first Go implementation, the required surface is OpenAI Chat Completions with correct streaming tool-call deltas, stable call IDs, tool-result replay, finish reasons, usage, and model selection. Responses, Anthropic Messages, embeddings, images, admin UI, and native Cursor delegation are not required for the initial Hermes path.

## 2026-08-14 08:00 — Integration direction corrected
Cursor now publishes a supported Agent SDK and `sdk.v1` SDK Bridge specifically for Go and other non-TypeScript/Python clients. The bridge provides typed agent/run lifecycle, streaming, cancellation, local and cloud runtimes, custom tool callbacks, usage, and additive protocol versioning. A live local ACP probe also completed initialize, Cursor-login authentication, and session creation; it advertised session loading, MCP support, agent/plan/ask modes, and the account model catalog.
The SDK and ACP are agent protocols, not raw model-inference APIs. They own the Cursor agent loop. Wrapping either as OpenAI Chat Completions cannot preserve Hermes-owned function calling without the same brittle prompt-to-JSON emulation used upstream.
Recommendation: integrate at the agent boundary. Expose a narrow Go MCP tool backed by the official Cursor SDK Bridge, let Hermes invoke Cursor for bounded coding tasks, and keep Hermes as the outer orchestrator. If the hard requirement is instead to use Cursor subscription inference inside Hermes's existing tool loop, Cursor has no supported raw completion interface; the OpenAI facade is then an unavoidable compatibility hack rather than the preferred architecture.
Next: prototype one end-to-end MCP call from Hermes through a thin Go SDK Bridge adapter, including streamed status, cancellation, and a restricted workspace, before designing any broader service.

## 2026-08-14 08:38 — Subscription authentication clarified
Cursor documents that SDK runs use the same pricing and request pools as IDE and Cloud Agent runs; a user API key bills to the user's existing plan rather than a separate API account. A key is still mandatory for the SDK Bridge, and the Start plan does not include the SDK. Additional charges occur only when on-demand usage is explicitly enabled and included usage is exhausted; disabling on-demand makes Cursor stop requests at the plan limit.
For a subscription-native setup with no API-key provisioning, prefer the official Cursor CLI ACP transport. Its `cursor_login` authentication reuses `agent login`; the live probe confirmed that path locally. Revised recommendation: keep the Hermes-facing MCP boundary, but make the Cursor adapter selectable—ACP by default for local user subscriptions, SDK Bridge only when typed SDK capabilities or cloud/service-account operation justify user API-key setup.
The first spike should implement ACP only and avoid a multi-backend abstraction. Add SDK Bridge support later only if a demonstrated requirement needs it.

## 2026-08-14 08:44 — ACP versus SDK Bridge
For the narrow local Hermes integration, ACP is the smaller implementation: one installed `agent` child, newline-delimited JSON-RPC, existing Cursor login, and only the initialize/auth/session/prompt/update/cancel subset. Its costs are untyped protocol ownership, CLI-version coupling, process restart/session recovery, no documented usage or cost telemetry, local stdio only, and mandatory handling of blocking permission, question, and plan requests so a headless Hermes call cannot deadlock.
The SDK Bridge is more implementation and deployment work in Go: pin and distribute the standalone bridge and matching protos, generate a Connect client, parse the ready handshake, read and protect the per-process bearer token, provide the Cursor API key on RPCs, manage streams, and shut down the loopback server. That cost buys a supported additive `sdk.v1` contract, typed errors and streams, durable Agent/Run resources, usage and cost reporting, artifacts, local and cloud parity, custom tools and stores, and explicit capability/version discovery.
Decision remains ACP for the first local subscription spike. Reconsider SDK Bridge when usage accounting, cloud runs, durable run management, or supported typed compatibility becomes a proven requirement.
