## Auto‑Builder – Architecture, Implementation, and Code Review (Full Doc)

This document is a full technical deep‑dive and code review for the LLM‑driven Auto‑Builder. It includes a high‑level overview, end‑to‑end request/response flow, detailed explanations of each key file and function, constraints, error handling, deployment, and recommendations.

### Contents
- 1) High‑Level Overview
- 2) Frontend – Files, Data Flow, and Apply Logic
- 3) Backend – GraphQL Schema, Resolver, Module
- 4) AutoBuilderService – Functions Explained (with behavior and constraints)
- 5) OpenAI Service – JSON Chat
- 6) Node Types and Constraints
- 7) Error Handling and Validation
- 8) Auth, Environment, and Deployment
- 9) Testing Playbook
- 10) Code Review – Strengths, Risks, and Recommendations

---

## 1) High‑Level Overview

The Auto‑Builder converts a natural‑language instruction into a valid BotStacks flow. The web UI sends the instruction to a chat GraphQL mutation, which orchestrates an LLM call (OpenAI, JSON mode) and then normalizes/enforces flow rules on the server. The UI previews the result and applies it (creating nodes/channels) when approved.

---

## 2) Frontend – Files, Data Flow, and Apply Logic

Key files:
- UI (instruction/preview): `core/apps/web/app/components/auto-builder/AutoBuilder.tsx`
- API wrapper: `core/apps/web/app/api/autobuilder.ts`
- GraphQL doc (web): `core/apps/web/app/api/gql/autobuilder.graphql`
- Apply flow integration: `core/apps/web/app/designer/autoBuilder.tsx`
- Nodes API: `core/apps/web/app/api/nodes.ts` (CreateNode/CreateChannel)
- Project thunks: `core/apps/web/app/state/thunks/project.ts` (dispatchers)

Frontend flow:
1) User enters an instruction on the Auto‑Builder tab and clicks Build.
2) `AutoBuilder.tsx` selects auth from Redux (`token`, `apiKey`) and calls `generateAutoBuilderFlow` via `autobuilder.ts`.
3) On success, it renders the preview (`flow`, `validation`) and enables Apply.
4) Apply: `autoBuilder.tsx (screen)` iterates over nodes and connections and dispatches `createNode`/`createChannel` mutations to persist the flow.

Frontend error handling:
- Surfaces server message or GraphQL errors directly.
- A local fallback (simple chain) exists only to avoid blocking during early schema setup; once backend is live, it won’t trigger.

---

## 3) Backend – GraphQL Schema, Resolver, Module

Files:
- Schema: `core/apps/chat/src/autobuilder/autobuilder.graphql`
- Module: `core/apps/chat/src/autobuilder/autobuilder.module.ts`
- Resolver: `core/apps/chat/src/autobuilder/autobuilder.resolver.ts`
- Service: `core/apps/chat/src/autobuilder/autobuilder.service.ts`
- OpenAI: `core/apps/chat/src/openai/openAI.service.ts`
- Templates: `core/apps/chat/src/template/template.service.ts`

GraphQL types (chat):
- `mutation generateAutoBuilderFlow(input: GenerateFlowInput!): GenerateFlowResult!`
- `GenerateFlowInput`: `instruction`, `botId`, optional `options { maxNodes, allowNodeTypes, complexity }`
- `GenerateFlowResult`: `success`, `error?`, `parsed?`, `validation?`, `flow?`

Resolver:
- Validates args and delegates to `AutoBuilderService.generate(...)`.

Module:
- Imports ORM, HttpModule, `TemplateModule`, and provides Resolver/Service/OpenAIService.

---

## 4) AutoBuilderService – Functions Explained

File: `core/apps/chat/src/autobuilder/autobuilder.service.ts`

### `async generate({ instruction, botId, options })`
- Loads enabled templates for contextual bias.
- Calls `buildPrompt(instruction, templates, options)` to produce a compact but strict prompt.
- Calls `callLLMForJson(prompt)` – returns `{ json, tokens, model }` (OpenAI JSON mode).
- Calls `validateAndNormalize(json, options, instruction)` to produce a legal flow.
- Adds model/tokens info into `validation.suggestions` for visibility.
- Returns `{ success: true, flow, parsed, validation }`.

### `buildPrompt(instruction, templates, options)`
- Creates a Node Catalog with:
  - Allowed types: `start, llm, text, airesponse, listen, set, condition, routing, api, code, userdatacapture, end`.
  - Constraints:
    - Start/End required
    - Single‑output: `llm, userdatacapture, text, airesponse, listen, set`
    - Multi‑output: `condition`, `routing`, `api (success/error)`
    - `text`/`llm` must be followed by `airesponse` (sends the message)
  - Slug rules: `<type>-<zero-based-index>`; use `source/target` for edges.
- Adds template summaries (from DB) to bias patterns.
- Appends explicit JSON schema shape to force structure.

### `callLLMForJson(prompt)`
- Uses `OpenAIService.chatJSON([...messages], 'gpt-3.5-turbo', 0)` with `response_format: json_object`.
- Parses returned `content` as JSON; returns `{ json, tokens, model }`.

### `validateAndNormalize(json, options, instruction)`
Purpose: accept LLM output, enforce all constraints, and return an “apply‑able” flow.

Key steps and behaviors:
1) Allowed types and alias mapping
   - Canonical: `start, llm, text, airesponse, listen, set, condition, routing, api, code, userdatacapture, end`
   - Aliases (e.g., classify → routing, message → airesponse); unknown types dropped with warnings.

2) Canonical IDs, slugs, positions, defaults
   - Slugs/IDs like `llm-0`, `airesponse-0`, `listen-0`, `end-0` with zero‑based indices per type.
   - Auto‑layout positions if missing.
   - Data defaults per node type (llm prompts; api method/url/headers/bodyType; condition routeMap; routing intentIds/map; userdatacapture userFields).

3) Guarantee `start-0` and `end-0`
   - Auto‑create missing Start/End with sensible defaults.

4) Normalize connections
   - Accept `{from,to}` or `{source,target}`; normalize to `source/target` with default handles `a/b`.
   - Drop malformed edges.

5) Enforce single/multi‑output constraints
   - Single‑output: `llm, userdatacapture, text, airesponse, listen, set` → if >1 outgoing, keep first, drop rest (warn).
   - `condition`: allow many; ensure at least one route (fallback to End) and keep `data.routeMap`.
   - `routing`: allow many; ensure fallback; derive `intentIds` and `intentMap` from outgoing edges when missing.
   - `api`: up to two (success + error). If only one exists, add an error fallback to End. Fill `data.successNodeId`/`data.errorNodeId`.

6) Ensure `text/llm → airesponse`
   - If a `text` or `llm` lacks an airesponse downstream, auto‑create an `airesponse-*` node and connect it.

7) Parse URL/method from instruction
   - If the instruction contains an explicit URL (http/https) and HTTP method, inject them into the first API node missing those fields.

8) Autowire only nodes with zero outgoing edges
   - Adds linear edges to guarantee a traversable path (Start → middle → End) without overriding genuine branching.

Output:
- `flow { nodes[], connections[], variables[], metadata }`, `parsed`, `validation { errors, warnings, suggestions }`

---

## 5) OpenAI Service – JSON Chat

File: `core/apps/chat/src/openai/openAI.service.ts`

### `chatJSON(messages, model, temperature)`
- `POST https://api.openai.com/v1/chat/completions` with headers `{ Authorization: Bearer OPENAI_KEY }`.
- Uses `response_format: { type: 'json_object' }` to force JSON.
- Returns `{ content, tokens }`.

Note: Routing’s `classify(...)` also uses OpenAI; we aligned Auto‑Builder to `gpt‑3.5‑turbo` for consistency/cost.

---

## 6) Node Types and Constraints

- Canonical types: `start, llm, text, airesponse, listen, set, condition, routing, api, code, userdatacapture, end`.
- Slug format: `<type>-<index>` (zero‑based per type), e.g., `start-0`, `llm-0`, `airesponse-0`.
- Output rules:
  - Single‑output: `llm, userdatacapture, text, airesponse, listen, set`.
  - Multi‑output: `condition` (multi branches + fallback), `routing` (multi intents + fallback), `api` (success + error).
- Messaging rule: Any `text`/`llm` must lead into an `airesponse` node; that’s what sends the message.

---

## 7) Error Handling and Validation

Frontend:
- Shows server errors verbatim; surfaces GraphQL errors if present.

Backend:
- Collects `errors`, `warnings`, `suggestions` and returns them.
- Adds model/tokens as an informational suggestion for visibility.

---

## 8) Auth, Environment, and Deployment

Auth:
- Web sends `Authorization: Bearer <token>` and `X-API-Key` (if present).

Environment:
- Chat requires `OPENAI_KEY`.
- Web local dev: `NEXT_PUBLIC_SERVER=http://127.0.0.1:3000`, `NEXT_PUBLIC_SERVER_WS=ws://127.0.0.1:3000`.

Deployment:
- Chat GraphQL scans `./**/*.graphql`, and `AppModule` imports `AutoBuilderModule`.

---

## 9) Testing Playbook

Chat GraphQL:
- Open `http://127.0.0.1:3000/graphql` and confirm mutation and input are present.
- Test with variables: `{ instruction: "...", botId: "test-bot", options: { maxNodes: 20 } }`.

Web UI:
- Auto‑Builder tab → instruction → Build → Preview → Apply, then inspect nodes/channels in Studio.

---

## 10) Code Review – Strengths, Risks, Recommendations

### Strengths
- Clear separation of concerns:
  - UI (input/preview), API helper, GraphQL schema/resolver, service (prompt/normalize), OpenAI client.
- Robust server‑side normalization ensures valid flows even with imperfect LLM output (ids/slugs, positions, defaults, constraints, auto‑wiring).
- Constraints encoded both in prompt (to guide the LLM) and in code (to guarantee correctness):
  - Single vs. multi‑output rules
  - text/llm → airesponse requirement
  - API success/error paths
  - Start/End presence
- Error transparency in the UI and validation metadata from backend.

### Risks / Edge Cases
- LLM variance: Even with guidance, models may output structures that need heavy normalization. We handle this, but some complex branching intent/condition logic may still need user adjustments post‑generation.
- API inference: URL/method parsing is basic (regex). Complex headers/body inference not yet implemented.
- Intent IDs: Derived intent keys (Intent1, Intent2, …) are placeholders. Real intent IDs require user mapping or DB lookups.
- Positioning: Auto layout is simple (x‑axis). Large graphs might overlap visually; the Studio can adjust later.

### Recommendations
- Make model/temperature configurable by environment.
- Add richer API inference (headers/body from instruction; simple key:value parsing; JSON detection).
- Persist `AutoBuilderChange` audit logs when flows are applied.
- Optional: auto‑add a `listen` after `airesponse` when conversational loop is implied.
- Extend the prompt with a few curated, diverse template exemplars to train multi‑branch patterns.


