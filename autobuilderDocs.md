## Auto‑Builder: End‑to‑End Technical Documentation

This document explains how the new LLM‑driven Auto‑Builder works across the frontend and backend, including files, functions, data flow, schema, constraints, and deployment/runtime considerations.

### High‑Level Architecture

- Frontend (web/app):
  - Auto‑Builder UI reads a natural‑language instruction from the user.
  - Calls a chat GraphQL mutation to generate a flow.
  - Previews the flow; on “Apply”, creates nodes and connections in the current bot.

- Backend (chat/app):
  - GraphQL mutation `generateAutoBuilderFlow` hits `AutoBuilderService`.
  - Service builds a structured prompt and calls OpenAI for JSON output.
  - Normalizes the JSON into valid BotStacks flow: nodes, connections, variables, metadata.
  - Enforces constraints (node types, slugs, single/multi‑output rules, AI response rules).
  - Returns the result to the frontend.

- External:
  - OpenAI chat completions (`gpt‑3.5‑turbo`), via `OPENAI_KEY`.

---

### Frontend – Files and Flow

- UI component (user input/preview):
  - `core/apps/web/app/components/auto-builder/AutoBuilder.tsx`
- API helper:
  - `core/apps/web/app/api/autobuilder.ts`
- GraphQL doc (web side):
  - `core/apps/web/app/api/gql/autobuilder.graphql`
- Apply flow logic (creates nodes/channels after preview):
  - `core/apps/web/app/designer/autoBuilder.tsx`
  - Node management APIs and thunks:
    - `core/apps/web/app/api/nodes.ts`
    - `core/apps/web/app/state/thunks/project.ts`

Flow:
1) User types instruction and clicks Build in `AutoBuilder.tsx`.
2) `AutoBuilder.tsx` reads `token` and `apiKey` from Redux and calls:
   - `generateAutoBuilderFlow({ data: { instruction, botId }, extras: { token, apiKey } })`
3) The API helper posts a GraphQL mutation (string) through `request(...)`.
4) On success, UI shows `flow`, `parsed`, `validation` and enables Apply.
5) Apply: creates nodes with `createNode` and channels with `createChannel` against the chat GraphQL API.

Error handling:
- Frontend surfaces server/GraphQL errors verbatim (no generic hidden failures).
- A local fallback (simple 5‑node chain) exists but is primarily for early wiring when the mutation schema isn’t deployed.

---

### Backend – Files and GraphQL

- GraphQL schema and module:
  - `core/apps/chat/src/autobuilder/autobuilder.graphql`
  - `core/apps/chat/src/autobuilder/autobuilder.module.ts`
  - `core/apps/chat/src/autobuilder/autobuilder.resolver.ts`
- Service (core logic):
  - `core/apps/chat/src/autobuilder/autobuilder.service.ts`
- OpenAI client:
  - `core/apps/chat/src/openai/openAI.service.ts`
- Templates (priors):
  - `core/apps/chat/src/template/template.service.ts`

GraphQL (chat):
- Mutation: `generateAutoBuilderFlow(input: GenerateFlowInput!): GenerateFlowResult!`
- Input: `{ instruction, botId, options?: { maxNodes, allowNodeTypes, complexity } }`
- Output: `{ success, error, parsed, validation, flow }`

Deployment notes:
- Chat `GraphQLModule` uses `typePaths: ['./**/*.graphql']` so the autobuilder schema is included on build.
- Chat `AppModule` imports `AutoBuilderModule`.

---

### AutoBuilderService – Implementation

File: `core/apps/chat/src/autobuilder/autobuilder.service.ts`

Key methods:

- `generate(input)`
  - Loads enabled templates.
  - `buildPrompt(...)` (adds node catalog + rules + template summaries + strict JSON instructions).
  - `callLLMForJson(...)` (`gpt-3.5-turbo`, temperature 0, JSON mode) → `{ json, tokens, model }`.
  - `validateAndNormalize(json, options, instruction)` → legal, traversable flow.
  - Adds model/tokens into `validation.suggestions` for observability.

- `buildPrompt(instruction, templates, options)`
  - Includes Node Catalog: allowed node types, constraints, slug rules, and connection shape.
  - House Rules: start/end required, no orphans, single/multi‑output constraints, API success/error routes, text/llm → airesponse.
  - Template summaries from DB.
  - Requires strict JSON shape.

- `callLLMForJson(prompt)`
  - Uses `OpenAIService.chatJSON(messages, 'gpt-3.5-turbo', 0)`.
  - Forces `response_format: json_object`.
  - Returns parsed JSON + token usage.

- `validateAndNormalize(json, options, instruction)`
  - Accepts LLM output and constructs a valid flow:
    1) Allowed types and alias mapping:
       - Canonical: `start, llm, text, airesponse, listen, set, condition, routing, api, code, userdatacapture, end`.
       - Aliases to canonical (e.g., classify → routing, message → airesponse). Unknown types dropped with warning.
    2) Canonical ids/slugs/positions:
       - Slugs like `llm-0`, `airesponse-0` (zero‑based per type).
       - Auto layout on x‑axis when missing.
       - Minimal data defaults by node type (llm/api/condition/routing/userdatacapture).
    3) Guarantee Start/End.
    4) Normalize connections:
       - Harmonize `{from,to}` vs `{source,target}`.
       - Drop malformed edges.
    5) Enforce single/multi‑output constraints:
       - Single‑output: `llm, userdatacapture, text, airesponse, listen, set` → keep one outgoing; drop extras.
       - Multi‑output:
         - `condition`: multiple allowed; ensure fallback to End; maintain `data.routeMap`.
         - `routing`: multiple allowed; ensure fallback to End; infer `intentIds/intentMap` from edges if missing.
         - `api`: up to two (success/error). If only one, add error fallback to End. Fill `data.successNodeId`/`data.errorNodeId`.
    6) text/llm → airesponse:
       - If missing, auto‑create `airesponse-*` and connect it so content is actually sent to chat.
    7) Parse URL/method from instruction for API:
       - If instruction contains a URL and/or HTTP method, populate the first API node lacking them.
    8) Auto‑wire only nodes with zero outgoing edges:
       - Ensures traversable Start → middle → End without overriding real branches the model produced.

Return:
- `flow`: `nodes[]`, `connections[]`, `variables[]`, `metadata`
- `parsed`: optional parsed instruction payload from the model
- `validation`: `errors[]`, `warnings[]`, `suggestions[]` (includes model/tokens)

---

### OpenAI Service

File: `core/apps/chat/src/openai/openAI.service.ts`

- `chatJSON(messages, model, temperature)`
  - `POST https://api.openai.com/v1/chat/completions`
  - Headers include `Authorization: Bearer ${OPENAI_KEY}`.
  - Uses `response_format: { type: 'json_object' }`.
  - Returns `{ content, tokens }`.

Note: Routing intent classification elsewhere uses `classify(...)` on the same service; we aligned Auto‑Builder to `gpt‑3.5‑turbo` for consistency.

---

### Node Type Rules (Constraints)

- Canonical types: `start, llm, text, airesponse, listen, set, condition, routing, api, code, userdatacapture, end`.
- Slugs: `<type>-<index>` (zero‑based per type), e.g., `start-0`, `llm-0`, `airesponse-0`.
- Outputs:
  - Single‑output: `llm, userdatacapture, text, airesponse, listen, set`.
  - Multi‑output: `condition` (multi branches + fallback), `routing` (multi intents + fallback), `api` (success+error).
- Messaging rule: any `text` or `llm` must be followed by `airesponse` (the node that actually sends messages to chat).

---

### Auth & Environment

- Web → Chat:
  - `Authorization: Bearer <token>` + `X-API-Key` (if present).
- Chat requires `OPENAI_KEY`.
- Local dev:
  - Web: `NEXT_PUBLIC_SERVER=http://127.0.0.1:3000`, `NEXT_PUBLIC_SERVER_WS=ws://127.0.0.1:3000`
  - Chat: `yarn nx serve chat` (GraphQL at `/graphql`)

---

### Endpoints & Ops

- Frontend → Backend:
  - `generateAutoBuilderFlow(input: GenerateFlowInput!): GenerateFlowResult!`
- Frontend Apply:
  - `createNode(input: CreateNodeInput!) → Node`
  - `createChannel(input: CreateChannelInput!) → Channel`
- OpenAI:
  - `POST https://api.openai.com/v1/chat/completions`

---

### Testing

- Chat GraphQL:
  - Open `http://127.0.0.1:3000/graphql`
  - Verify `generateAutoBuilderFlow` and `GenerateFlowInput` exist
  - Run a test mutation to confirm a flow is returned with `nodes/connections`

- Web UI:
  - Auto‑Builder tab → type instruction → Build → Preview → Apply (observe nodes/channels created)

---

### Future Enhancements

- Configurable model via env or options
- Persist audit logs (`AutoBuilderChange`) on Apply
- Smarter API body/header inference from instruction
- Auto add `listen` after `airesponse` when conversational loop is implied


