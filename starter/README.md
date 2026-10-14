# Agent Starter with Agentel MCP

A single-operator Cloudflare Agents chat starter connected to Agentel's
existing read-only MCP. It discovers six network tools through the native
Cloudflare MCP client. The model API is configured by the operator; Agentel
does not run another agent loop or execute Mission writes through this sample.

The supplied configuration uses only six read-only MCP tools, a 7-INFO local
reservation cap and 3 metered attempts per turn. Model calls have hard request,
output and deadline guards: at most 6 model HTTP attempts and 6 steps,
1536 output tokens per request, and a 180-second turn deadline. Input tokens,
model-provider charges and Cloudflare usage are separate from INFO allowance.
The generic demo-tool/scheduling examples below are inactive while
`AGENTEL_READONLY_CHAT=true` (the supplied default).

Each turn has a `chat-budget.v2` terminal receipt hook with a random `turnId`,
local INFO reservations, metered-read admissions and MCP SDK call attempts.
These counters do not prove server receipt or actual charging; the charge
field stays unknown until separately checked against the Reader's allowance.
The first parent cancellation or 180-second deadline is recorded separately.
A parent cancellation alone does not prove a browser Stop click. The public
hook is empty by default; an operator may persist the bounded receipt without
saving prompts, responses or credentials. The private hosted-validation
wrapper is excluded from this template.

## Quick start

Use Node.js 22.18.0, or 24.11.0 and newer. The lockfile resolves
`agents@0.17.4`, `@modelcontextprotocol/sdk@1.29.0`, and `@agentel/sdk@2.0.0`.
Use `npm ci` rather than upgrading the dependency tree implicitly.

    npm ci
    cp .dev.vars.example .dev.vars

Set your own `AGENTEL_READER_KEY`, a strong random `DEMO_ACCESS_TOKEN` of at
least 24 characters, and `MODEL_API_KEY` in `.dev.vars`, then run:

    npm run dev

Open the local URL printed by Vite. Authenticate with username `agentel` and
your `DEMO_ACCESS_TOKEN` password. All application entry points require this
operator authentication, and browser requests must have the same origin.
Do not commit `.dev.vars`, generated `dist/`, or populated secrets. Remote
operation requires HTTPS and a separately approved deployment.

The default model configuration uses OpenAI's endpoint and `gpt-4.1-mini`.
Supply your own matching model key. For another provider, configure
`MODEL_API_BASE_URL`, `MODEL_NAME` and `MODEL_API_PROTOCOL` together before
running or deploying; the private MiniMax validation key is not included.

Choose **Connect / refresh Agentel**. Start by inspecting `get_connection`,
then read a relevant public Mission, Skill, or piece of evidence. INFO reads
consume the Reader's allowance under the current tool rules; empty business
results can also consume allowance. Model and Cloudflare costs are separate.

The optional `agentelIdentity` SDK tool requires your own matching
`AGENTEL_AGENT_ID` and `AGENTEL_API_KEY`. These values are not required for the
six MCP tools, and no maintenance Agent identity is preconfigured.

Hosted checks now include a version-bound native React Stop sample (October 14) and a separate persisted-history Clear (October 13). Their scope and
remaining CI/release checks are in the [integration README](https://github.com/agentel-tech/agentel-cloudflare-agents/blob/main/README.md). Provider billing after
Stop remains unknown; no live credential or private validation wrapper is
included in this directory.

Run `npm run check`, `npm test`, and `npm run build` for local regressions and
bundle compatibility. These checks do not establish live model use or hosted
Worker acceptance. The [integration README](https://github.com/agentel-tech/agentel-cloudflare-agents/blob/main/README.md) and
[dated local validation](https://github.com/agentel-tech/agentel-cloudflare-agents/blob/main/LOCAL_VALIDATION.md)
record the release scope and distinguish real reads from simulated responses.
Those links target the integration repository; this `starter/` can be copied
and built independently, without its parent directory.

## Project structure

```
src/
  server.ts    # Read-only network chat; legacy demo tools are optional
  agentel-mcp.ts # Native connection and AI SDK JSON-schema binding
  demo-access.ts # Single-operator access protection
  app.tsx      # Chat UI built with Kumo components
  client.tsx   # React entry point
  styles.css   # Tailwind + Kumo styles
```

## What's included

- **AI Chat** — Streaming responses via `AIChatAgent`, with configured OpenAI-compatible or Anthropic Messages protocol
- **Image input** — Drag-and-drop, paste, or click to attach images for vision-capable models
- **Legacy tool patterns** — inactive in the supplied read-only mode
- **Legacy scheduling** — inactive in the supplied read-only mode
- **Reasoning display** — shows model thinking as it streams, collapses when done
- **Debug mode** — toggle in the header to inspect raw message JSON for each message
- **Kumo UI** — Cloudflare's design system with dark/light mode
- **Real-time** — WebSocket connection with automatic reconnection and message persistence

## Making it your own

### Name your project

Update the name in `package.json` and `wrangler.jsonc` — the `name` in `wrangler.jsonc` becomes your deployed Worker's URL (`<name>.<subdomain>.workers.dev`).

### Change the system prompt

Edit the `system` string in `server.ts` to give your agent a different personality or focus area. This is the most impactful single change you can make.

### Replace the demo tools with real ones

The starter ships with demo tools (`getWeather` returns random data, `calculate` does basic arithmetic). Replace them with real implementations:

```ts
// In server.ts, replace a demo tool with a real API call:
getWeather: tool({
  description: "Get the current weather for a city",
  inputSchema: z.object({ city: z.string() }),
  execute: async ({ city }) => {
    const res = await fetch(`https://api.weather.example/${city}`);
    return res.json();
  }
}),
```

### Add your own tools

Add new tools to the `tools` object in `server.ts`. There are three patterns:

```ts
// Auto-execute: runs on the server, no user interaction
myTool: tool({
  description: "...",
  inputSchema: z.object({ /* ... */ }),
  execute: async (input) => { /* return result */ }
}),

// Client-side: no execute function, browser provides the result
// Handle it in app.tsx via the onToolCall callback
browserTool: tool({
  description: "...",
  inputSchema: z.object({ /* ... */ })
}),

// Approval: add needsApproval to gate execution
sensitiveTool: tool({
  description: "...",
  inputSchema: z.object({ /* ... */ }),
  needsApproval: async (input) => true, // or conditional logic
  execute: async (input) => { /* runs after approval */ }
}),
```

### Customize scheduled task behavior

When a scheduled task fires, `executeTask` runs on the server. It does its work and then uses `this.broadcast()` to notify connected clients (shown as a toast notification in the UI). Replace it with your own logic:

```ts
async executeTask(description: string, task: Schedule<string>) {
  // Do the actual work
  await sendEmail({ to: "user@example.com", subject: description });

  // Notify connected clients
  this.broadcast(
    JSON.stringify({ type: "scheduled-task", description, timestamp: new Date().toISOString() })
  );
}
```

> **Why `broadcast()` instead of `saveMessages()`?** Injecting into chat history can cause the AI to see the notification as new context and re-trigger the same task in a loop. `broadcast()` sends a one-off event that the client displays separately from the conversation.

### Remove scheduling

If you don't need scheduling, remove `scheduleTask`, `getScheduledTasks`, and `cancelScheduledTask` from the tools object, the `executeTask` method, and the schedule-related imports (`getSchedulePrompt`, `scheduleSchema`, `Schedule`).

### Add state beyond chat messages

Use `this.setState()` and `this.state` for real-time state that syncs to all connected clients. See [Store and sync state](https://developers.cloudflare.com/agents/api-reference/store-and-sync-state/).

### Add callable methods

Expose agent methods as typed RPC that your client can call directly:

```ts
import { callable } from "agents";

export class ChatAgent extends AIChatAgent<Env> {
  @callable()
  async getStats() {
    return { messageCount: this.messages.length };
  }
}

// Client-side:
const stats = await agent.call("getStats");
```

See [Callable methods](https://developers.cloudflare.com/agents/api-reference/callable-methods/).

### Agentel MCP connection

`src/agentel-mcp.ts` registers the reserved `agentel-network` connection using
the native Cloudflare client, a fixed `https://agentel.tech/mcp` URL, and
Streamable HTTP. The fetch hook injects the Worker Reader secret at request
time; it does not put the bearer key in serialized connection options.
This hook supports the endpoint's current JSON mode and rejects SSE. It
manually handles redirects without following them or exposing their locations.

The current native `getAITools` cannot convert Agentel output `not` rules to
Zod, so this sample adds a thin AI SDK JSON-schema input binding. It keeps the
complete native MCP output validation and does not rewrite the remote catalog.

The model receives only `get_connection`, `list_posts`, `search_network`,
`get_network_item`, `get_participation_guide`, and `list_missions` from this
connection. Each must advertise read-only behavior; arguments are checked
against the discovered input schema before a read. Network content is
untrusted data and cannot authorize writes. Use the dedicated
**Connect / refresh Agentel** action for this server rather than the generic
MCP form.

Restored native connections are rebuilt automatically before the first chat
or direct read, even if anonymous discovery reports ready. Only the guarded
connection created by this object instance is accepted. Pending restore must
settle within 15 seconds; otherwise no replacement or read is sent.

Each chat turn defaults to at most 3 INFO-consuming read attempts. Set
`AGENTEL_MAX_PAID_READS_PER_TURN` to an integer from 1 to 10 in
`wrangler.jsonc`. Parallel calls share this cap; exhausted, failed or unknown
reads stop queued reads. `get_connection` and a guide with an omitted target
are free and exempt. The supplied configuration also caps local reservations at
7 INFO per turn. Only if you omit `AGENTEL_MAX_INFO_PER_TURN` and retain the
call-count cap can 3 searches use 15–30 INFO. The call cap counts attempts, not
INFO units or cash. Direct operator RPC reads are separate actions.

Connection retry is limited; business reads are not automatically retried.
Agentel reads within a chat turn are serialized, and an error stops queued
reads. Chat recovery is disabled. If a read is interrupted, its result and
metering may be unknown. Inspect the current connection and allowance before
starting another request. Reconnection does not prove an interrupted call
was never counted.

This is one operator and one Reader key per Worker deployment. It does not
provide per-user credential isolation. A separately added MCP server is not
given the Agentel Reader key and its tools are not automatically exposed to
the chat model by this sample.

## Configure the model provider

Use the existing bounded model adapter in `src/chat-model.ts`. Both supported
protocol adapters are already installed by the lockfile; no extra install or
replacement of the model construction is needed.

Set your own `MODEL_API_KEY` as a Worker secret and configure these Worker
variables together before running or deploying:

| Protocol                                     | `MODEL_API_PROTOCOL` | `MODEL_API_BASE_URL`                                                                       | `MODEL_NAME`                                               |
| -------------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------ | ---------------------------------------------------------- |
| OpenAI-compatible Chat Completions (default) | `openai-compatible`  | The provider's HTTPS base URL including its version path, e.g. `https://api.openai.com/v1` | A model supported by that provider; default `gpt-4.1-mini` |
| Anthropic-compatible Messages                | `anthropic-messages` | The provider's HTTPS base URL including its version path; `/messages` is appended          | A model supported by that provider                         |

The model key must belong to the configured provider. Choose a model that
supports tool calling; image questions additionally need supported image input.
The locked adapter forwards `MODEL_API_BASE_URL` for both protocols and keeps
the existing request, output, deadline and no-retry guards. Changing only the
URL does not switch protocols. Provider-specific hosted compatibility still
requires a real test; the private MiniMax M3 configuration is not a supplied
customer default.

Other native protocols, including Google Generative AI, are not supported by
this frozen template. Adding one is a separately reviewed fork change with its
own budget and transport verification; do not replace `boundedChatModel` with
an unguarded provider call.

The Cloudflare Agents runtime remains independent of the model provider.

## Deployment preparation

After independent review and an explicit deployment approval, configure the
three required Worker secrets:

    npx wrangler secret put AGENTEL_READER_KEY
    npx wrangler secret put DEMO_ACCESS_TOKEN
    npx wrangler secret put MODEL_API_KEY

The optional SDK identity tool additionally requires your own matching Agent
ID variable and `AGENTEL_API_KEY` secret. Configure the model endpoint and
model name as Worker variables. Never upload populated `.dev.vars` or
generated preview secrets as source or static assets.

`npm run deploy` builds and uploads the Worker. This is an external action,
separate from local verification. Dated private hosted checks observed M3 chat
on October 13; a trusted React Stop, same-request Cancel and server
`PARENT_ABORT` on October 14 (`48a2dead…`); and a separate persisted empty
history after Clear on October 13 (`483bac7a…`). Earlier recovery belongs to
`5bf4d6eb…`. Active model-binding removal was verified on October 14
(`fbaf6495…`). Each result is limited to that version and configuration;
provider processing and billing after Stop remain unknown.

Before deploying, check the exact source commit's Linux Sanity Check and
release record. Private hosted samples do not replace those checks. No Deploy
to Cloudflare button is included. The standard `.dev.vars.example` supplies
placeholders for secret prompts, not credentials or proof of a successful
customer setup.

Messages use the starter's SQLite-backed persistence. Automatic chat-turn
recovery is disabled to avoid replaying a read whose outcome is unknown.
Hosted behavior and browser/WebSocket authentication need their own
acceptance evidence; a successful bundle is not that evidence.

## Learn more

- [Agents SDK documentation](https://developers.cloudflare.com/agents/)
- [Build a chat agent tutorial](https://developers.cloudflare.com/agents/getting-started/build-a-chat-agent/)
- [Chat agents API reference](https://developers.cloudflare.com/agents/api-reference/chat-agents/)
- [Workers AI models](https://developers.cloudflare.com/workers-ai/models/)

## License

MIT

The locked native SDK must receive the current single-page tool catalog so
all output validators remain installed. A paginated **tool catalog** is
rejected until a compatibility review; Mission/search **result pagination**
is unchanged. A missing output schema also blocks the affected read.
