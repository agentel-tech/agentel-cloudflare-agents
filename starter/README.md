# Agent Starter

![npm i agents command](./npm-agents-banner.svg)

A starter template for building AI chat agents on Cloudflare, powered by the [Agents SDK](https://developers.cloudflare.com/agents/) and a configurable OpenAI-compatible model API.

Uses the OpenAI-compatible Chat Completions API, with tools for weather, timezone detection, calculations with approval, and task scheduling. The model endpoint and model name are configurable in `wrangler.jsonc`.

## Quick start

```bash
cd starter
npm ci
cp dev-vars.example.txt .dev.vars
# Fill in AGENTEL_API_KEY and MODEL_API_KEY in .dev.vars
npm run dev
```

> Configure `MODEL_API_KEY` and `AGENTEL_API_KEY` in `.dev.vars` before running
> locally. Model inference goes to the configured API; the chat session and
> agent runtime remain on Cloudflare. Do not commit `.dev.vars`.

Open [http://localhost:5173](http://localhost:5173) to see your agent in action.

Try these prompts to see the different features:

- **"What's the weather in Paris?"** — server-side tool (runs automatically)
- **"What timezone am I in?"** — client-side tool (browser provides the answer)
- **"Calculate 5000 \* 3"** — approval tool (asks you before running)
- **"Remind me in 5 minutes to take a break"** — scheduling
- **Image questions** — choose a vision-capable model with `MODEL_NAME` first

## Project structure

```
src/
  server.ts    # Chat agent with tools and scheduling
  app.tsx      # Chat UI built with Kumo components
  client.tsx   # React entry point
  styles.css   # Tailwind + Kumo styles
```

## What's included

- **AI Chat** — Streaming responses powered by an OpenAI-compatible Chat Completions API via `AIChatAgent`
- **Image input** — Drag-and-drop, paste, or click to attach images for vision-capable models
- **Three tool patterns** — server-side auto-execute, client-side (browser), and human-in-the-loop approval
- **Scheduling** — one-time, delayed, and recurring (cron) tasks
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

### Connect to MCP servers

Add external tools from MCP servers:

```ts
async onChatMessage(onFinish, options) {
  // Connect to an MCP server
  await this.mcp.connect("https://my-mcp-server.example/sse");

  const result = streamText({
    // ...
    tools: {
      ...myTools,
      ...this.mcp.getAITools() // Include MCP tools
    }
  });
}
```

See [MCP Client API](https://developers.cloudflare.com/agents/api-reference/mcp-client-api/).

## Configure the model provider

This sample uses an OpenAI-compatible Chat Completions API. Set `MODEL_API_KEY`
in `.dev.vars` and configure `MODEL_API_BASE_URL` and `MODEL_NAME` in
`wrangler.jsonc`. The defaults point to OpenAI; to use another provider, set
its OpenAI-compatible API base URL and model name. Choose a model that supports
tool calling for the included tools. Image questions require a vision-capable
model.

### Provider adapter

```ts
import { createOpenAI } from "@ai-sdk/openai";

const modelApi = createOpenAI({
  name: "openai-compatible",
  apiKey: this.env.MODEL_API_KEY,
  baseURL: this.env.MODEL_API_BASE_URL
});
const result = streamText({
  model: modelApi.chat(this.env.MODEL_NAME)
  // ...
});
```

Set `MODEL_API_KEY` as a Worker secret and `MODEL_API_BASE_URL` plus
`MODEL_NAME` as Worker variables. The endpoint must implement OpenAI-compatible
Chat Completions. Other native APIs, such as Anthropic Messages or Google
Generative AI, need their AI SDK provider adapter and corresponding code
changes; changing the URL alone is not sufficient. The default install includes
only the OpenAI-compatible adapter below; native-provider examples are optional
changes for your fork.

The Cloudflare Agents runtime remains independent of the model provider.

### Native Anthropic API

```bash
npm install @ai-sdk/anthropic
```

Replace the OpenAI-compatible model construction in `src/server.ts`:

```ts
import { createAnthropic } from "@ai-sdk/anthropic";

const anthropic = createAnthropic({ apiKey: this.env.MODEL_API_KEY });
const result = streamText({
  model: anthropic(this.env.MODEL_NAME)
  // Keep the existing system prompt, messages, and tools.
});
```

The provider uses Anthropic's native Messages API. `MODEL_API_BASE_URL` is not
used by this adapter.

### Native Google Generative AI API

```bash
npm install @ai-sdk/google
```

Replace the model construction with the Google provider:

```ts
import { createGoogleGenerativeAI } from "@ai-sdk/google";

const google = createGoogleGenerativeAI({ apiKey: this.env.MODEL_API_KEY });
const result = streamText({
  model: google(this.env.MODEL_NAME)
  // Keep the existing system prompt, messages, and tools.
});
```

The provider uses Google's Generative AI API. `MODEL_API_BASE_URL` is not used
by this adapter. Pick a model that supports the tools and input types you use.

The AI SDK documents the [Anthropic provider](https://ai-sdk.dev/providers/ai-sdk-providers/anthropic)
and [Google Generative AI provider](https://ai-sdk.dev/providers/ai-sdk-providers/google-generative-ai).

## Deploy

```bash
npx wrangler login
npx wrangler secret put AGENTEL_API_KEY
npx wrangler secret put MODEL_API_KEY
npm run deploy
```

Your agent is live on Cloudflare's global network. Messages persist in SQLite, streams resume on disconnect, and the agent hibernates when idle.

## Learn more

- [Agents SDK documentation](https://developers.cloudflare.com/agents/)
- [Build a chat agent tutorial](https://developers.cloudflare.com/agents/getting-started/build-a-chat-agent/)
- [Chat agents API reference](https://developers.cloudflare.com/agents/api-reference/chat-agents/)
- [Workers AI models](https://developers.cloudflare.com/workers-ai/models/)

## License

MIT
