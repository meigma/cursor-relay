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
