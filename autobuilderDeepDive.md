## Auto‑Builder Deep Dive: Architecture, Flow, and Annotated Code

This document is a thorough, code‑annotated walkthrough of the LLM‑driven Auto‑Builder. It covers the exact path from a user clicking “Build Bot” to the generated Sequence Studio flow, with code snippets and explanations for each step.

---

### Table of Contents
- End‑to‑End Sequence
- Frontend: UI and API Call
- Backend GraphQL: Schema, Module, Resolver
- Service Orchestration: Prompt, LLM Call, Normalization
- OpenAI Client: JSON Chat
- Apply Flow: Persisting Nodes + Channels
- Node Rules and Constraints
- Error Handling & Validation
- Environment and Auth
- Testing Playbook

---

## End‑to‑End Sequence
1) User enters instruction and clicks Build in the Auto‑Builder tab.
2) Frontend collects auth and calls a GraphQL mutation: `generateAutoBuilderFlow`.
3) Chat resolver delegates to `AutoBuilderService.generate(...)`.
4) Service builds the prompt, calls OpenAI (JSON mode), normalizes/enforces constraints, and returns a valid flow.
5) Frontend previews flow; on Apply, creates nodes/channels via GraphQL.
6) Sequence Studio reflects the applied flow.

---

## Frontend: UI and API Call

File: `core/apps/web/app/components/auto-builder/AutoBuilder.tsx`

Snippet – submitting the instruction and calling the API:
```tsx
const handleInstructionSubmit = async (instruction: string) => {
  setIsLoading(true);
  setError(null);
  setCurrentInstruction(instruction);
  setParsedInstruction(null);
  setGeneratedFlow(null);
  setValidationResult(null);

  try {
    const result = await generateAutoBuilderFlow({
      data: { instruction, botId },
      extras: { token, apiKey },
    });

    if (result?.success) {
      setParsedInstruction(result.parsed || null);
      setGeneratedFlow(result.flow || null);
      setValidationResult(result.validation || null);
      if (result.flow && onFlowGenerated) onFlowGenerated(result.flow);
    } else {
      const serverError = typeof result?.error === 'string'
        ? result.error
        : result?.error ? JSON.stringify(result.error) : 'Failed to process instruction';
      setError(serverError);
    }
  } catch (err: any) {
    const gqlErrors: string[] = err?.response?.errors?.map((e: any) => e?.message).filter(Boolean) || [];
    if (gqlErrors.length) setError(gqlErrors.join('\n'));
    else setError(err instanceof Error ? err.message : 'An unexpected error occurred');
  } finally {
    setIsLoading(false);
  }
};
```

File: `core/apps/web/app/api/autobuilder.ts`

Snippet – GraphQL request:
```ts
export const generateAutoBuilderFlow = async ({ data, extras }: { data: any; extras: any; }) => {
  const document = `
    mutation GenerateAutoBuilderFlow($input: GenerateFlowInput!) {
      generateAutoBuilderFlow(input: $input) {
        success
        error
        parsed { intent complexity estimatedNodes }
        validation { isValid }
        flow {
          nodes { id type label slug position { x y } }
          connections
          variables
          metadata { complexity estimatedExecutionTime nodeCount branchCount }
        }
      }
    }
  `;
  const result = await request({ document, variables: { input: data } }, extras);
  return (result as any)?.generateAutoBuilderFlow;
};
```

Explanation:
- UI collects `token` and `apiKey` from Redux, then calls the mutation.
- Errors are surfaced verbatim (server and GraphQL layer), enabling quick debugging.

---

## Backend GraphQL: Schema, Module, Resolver

File: `core/apps/chat/src/autobuilder/autobuilder.graphql`

Snippet – essential parts:
```graphql
scalar JSON

input GenerateFlowOptionsInput {
  maxNodes: Int
  allowNodeTypes: [String!]
  complexity: String
}

input GenerateFlowInput {
  instruction: String!
  botId: String!
  options: GenerateFlowOptionsInput
}

type GenerateFlowResult {
  success: Boolean!
  error: String
  flow: GeneratedFlow
  parsed: ParsedInstruction
  validation: ValidationResult
}

type Mutation {
  generateAutoBuilderFlow(input: GenerateFlowInput!): GenerateFlowResult!
}
```

File: `core/apps/chat/src/autobuilder/autobuilder.module.ts`

```ts
@Module({
  imports: [importOrm(), HttpModule, TemplateModule],
  providers: [AutoBuilderResolver, AutoBuilderService, OpenAIService],
  exports: [AutoBuilderService],
})
export class AutoBuilderModule {}
```

File: `core/apps/chat/src/autobuilder/autobuilder.resolver.ts`

```ts
@Resolver('AutoBuilder')
export class AutoBuilderResolver {
  constructor(private readonly service: AutoBuilderService) {}

  @Mutation('generateAutoBuilderFlow')
  async generateAutoBuilderFlow(@Args('input') input: any): Promise<any> {
    return this.service.generate(input);
  }
}
```

Explanation:
- Schema defines the input/output shapes and mutation.
- Resolver hands off to the service (single responsibility).
- Module wires everything together (ORM, HTTP, templates, service).

---

## Service Orchestration: Prompt, LLM Call, Normalization

File: `core/apps/chat/src/autobuilder/autobuilder.service.ts`

### `generate(...)`
```ts
async generate(input: { instruction: string; botId: string; options?: { maxNodes?: number; allowNodeTypes?: string[]; complexity?: string } }) {
  const { instruction, options } = input;
  const enabledTemplates = await this.templates.getAll();
  const prompt = this.buildPrompt(instruction, enabledTemplates, options);
  const { json, tokens, model } = await this.callLLMForJson(prompt);
  const { flow, parsed, validation } = await this.validateAndNormalize(json, options, instruction);
  validation.suggestions = [...(validation.suggestions || []), { type: 'info', message: `AutoBuilder model: ${model}, tokens: ${tokens}`, severity: 'info' }];
  return { success: true, flow, parsed, validation };
}
```

### `buildPrompt(...)`
```ts
private buildPrompt(instruction: string, templates: any[], options?: any): string {
  const nodeCatalog = `\nAllowed node types and their purposes: ...\nConstraints:\n- text and llm must be followed by an airesponse node ...`;
  const templateSummaries = templates.slice(0, 5).map((t) => `- ${t.name}: ${t.description}`).join('\n');
  const rules = [
    'Include start and end nodes',
    'Avoid orphan nodes; all connections valid',
    'Use listen + airesponse where appropriate',
    'Use routing/condition/api/llm/userdatacapture/code nodes only as needed',
    options?.allowNodeTypes ? `Allowed nodes: ${options.allowNodeTypes.join(', ')}` : '',
    options?.maxNodes ? `Cap total nodes at ${options.maxNodes}` : '',
  ].filter(Boolean).join('\n- ');

  return `You are BotStacks Auto-Builder. Return ONLY JSON ...` +
         `Instruction: ${instruction}\n\n` +
         `House Rules:\n- ${rules}\n\n` +
         `Node Catalog:\n${nodeCatalog}\n` +
         `Templates (summaries):\n${templateSummaries}\n\n` +
         `JSON schema shape:\n{"parsedInstruction": {...}, "generatedFlow": {"nodes": [...], "connections": [...], "variables": [...], "metadata": {...}}}`;
}
```

### `callLLMForJson(...)`
```ts
private async callLLMForJson(prompt: string) {
  const model = 'gpt-3.5-turbo';
  const { content, tokens } = await this.openai.chatJSON([
    { role: 'system', content: 'You are a helpful assistant that outputs ONLY valid JSON.' },
    { role: 'user', content: prompt },
  ], model, 0);
  let json: any = {};
  try { json = JSON.parse(content); } catch { json = {}; }
  return { json, tokens, model };
}
```

### `validateAndNormalize(...)` (selected excerpts)

Normalize nodes, types, ids/slugs, positions, and defaults:
```ts
const allowedTypes = new Set(['start','llm','text','airesponse','listen','set','condition','routing','api','code','userdatacapture','end']);
const typeAlias: Record<string,string> = { classify: 'routing', message: 'airesponse', setvalue: 'set', collect: 'userdatacapture' };
const slugBase = { start:'start', llm:'llm', text:'text', airesponse:'airesponse', listen:'listen', set:'set', condition:'condition', routing:'routing', api:'api', code:'code', userdatacapture:'userdatacapture', end:'end' };
```

Guarantee Start/End and normalize connections:
```ts
if (!nodes.some((n) => n.type === 'start')) nodes.unshift({ id:'start-0', type:'start', ... });
if (!nodes.some((n) => n.type === 'end')) nodes.push({ id:'end-0', type:'end', ... });

const connections = (incoming.connections || []).map((c: any) => {
  const source = c.source || c.from;
  const target = c.target || c.to;
  return source && target ? { source, target, sourceHandle:'a', targetHandle:'b' } : null;
}).filter(Boolean);
```

Enforce single vs multi‑output and special rules:
```ts
const singleOutputTypes = new Set(['llm','userdatacapture','text','airesponse','listen','set']);
// Trim extra outs for single-output nodes

// Condition: allow many; ensure fallback to End
// Routing: allow many; derive intentIds/intentMap if missing; ensure fallback
// API: success + error; if only one, add error fallback; set successNodeId/errorNodeId

// text/llm → airesponse: auto-create airesponse when missing and connect it
```

Autowire only nodes with zero outgoing edges:
```ts
// Build ordered nodes: Start → middle → End
// For a node with no outgoing edges, add a linear edge to the next node
```

Parse URL/method from instruction:
```ts
const urlMatch = instruction.match(/https?:\/\/[^\s)\'"<>]+/i);
const methodMatch = instruction.match(/\b(GET|POST|PUT|PATCH|DELETE)\b/i);
if (urlMatch) { const apiNode = nodes.find((n) => n.type === 'api'); if (apiNode) { apiNode.data.url ||= urlMatch[0]; if (methodMatch) apiNode.data.method ||= methodMatch[0].toUpperCase(); } }
```

Explanation:
- This method guarantees the returned graph is “apply‑able” in one click, even if the model omits some details. The flow remains editable in Studio for refinements.

---

## OpenAI Client: JSON Chat

File: `core/apps/chat/src/openai/openAI.service.ts`

```ts
async chatJSON(messages, model = 'gpt-4o-mini', temperature = 0.2) {
  const { data } = await firstValueFrom(
    this.httpService.post<any>(
      'https://api.openai.com/v1/chat/completions',
      { model, temperature, messages, response_format: { type: 'json_object' } },
      this.config
    ).pipe(catchError((error: AxiosError) => { /* map to HttpException */ }))
  );
  return { content: data?.choices?.[0]?.message?.content ?? '{}', tokens: data?.usage?.total_tokens ?? 0 };
}
```

Explanation:
- Forces JSON output, returns the raw JSON string, which the service parses.

---

## Apply Flow: Persisting Nodes + Channels

File: `core/apps/web/app/designer/autoBuilder.tsx`

Creating nodes:
```ts
for (const node of flow.nodes) {
  if (node.type === 'start') continue;
  const payload = {
    onechatbot_id: botStackDraftId,
    type: node.type === 'iacOutput' ? 'output' : node.type,
    display_data: JSON.stringify({ position: node.position }),
    data: JSON.stringify({ label: node.label, slug: node.slug, ...node.data }),
  };
  const { payload: created } = await dispatch<any>(createNode(payload));
  createdNodes.push({ originalId: node.id, newId: created.id, type: node.type });
}
```

Creating channels:
```ts
for (const connection of flow.connections) {
  const sourceNode = createdNodes.find(n => n.originalId === connection.source);
  const targetNode = createdNodes.find(n => n.originalId === connection.target);
  if (sourceNode && targetNode) {
    const channelPayload = {
      onechatbot_id: botStackDraftId,
      input_node_id: sourceNode.newId,
      output_node_id: targetNode.newId,
      botstack_user_id: currentBotStackUserId || botId,
    };
    await dispatch<any>(createChannel(channelPayload));
  }
}
```

Explanation:
- Apply uses existing GraphQL mutations to persist nodes and edges.
- Start node is typically pre‑existing; we don’t create a duplicate.

---

## Node Rules and Constraints
- Canonical types: `start, llm, text, airesponse, listen, set, condition, routing, api, code, userdatacapture, end`.
- Slugs: `<type>-<index>` (zero‑based per type), e.g., `start-0`, `llm-0`, `airesponse-0`.
- Single‑output nodes: `llm, userdatacapture, text, airesponse, listen, set`.
- Multi‑output nodes:
  - `condition`: many branches, with fallback to End.
  - `routing`: many intents, with fallback to End.
  - `api`: success + error routes.
- text/llm must feed into `airesponse` to actually send content to chat.

---

## Error Handling & Validation
- Frontend shows server/GraphQL errors verbatim.
- Backend returns structured validation: `errors`, `warnings`, `suggestions`.
- The service injects model/tokens usage in suggestions.

---

## Environment and Auth
- Web → Chat: Bearer token + `X-API-Key`.
- Chat requires `OPENAI_KEY` for OpenAI requests.
- Local dev: `NEXT_PUBLIC_SERVER=http://127.0.0.1:3000`, `NEXT_PUBLIC_SERVER_WS=ws://127.0.0.1:3000`.

---

## Testing Playbook
- Chat GraphQL: open `/graphql`, verify mutation and input exist; run a test mutation.
- Web UI: Auto‑Builder tab → instruction → Build → Preview → Apply.

---

### Summary
The Auto‑Builder uses a tight prompt + strong server‑side normalization to consistently produce valid flows from natural‑language requirements. It enforces node constraints, ensures messaging via `airesponse`, manages success/error and branching nodes, and provides a one‑click path from generation to persisted Studio nodes.


