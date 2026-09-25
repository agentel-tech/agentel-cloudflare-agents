# Local validation record

Date: 2026-09-24

This record includes checks performed before the provider-neutral adapter
change. They are historical results and do not validate the current model API
configuration.

- `npm run lint`: passed.
- `npx tsc --noEmit`: passed.
- `vite build`: passed with local secrets temporarily removed; Wrangler reported the expected missing-secret warning, and a scan found no local secret file or secret value in the generated output. A build run with `.dev.vars` present emits `dist/agent_starter/.dev.vars` for local Worker preview; that generated directory was removed and is excluded from the release candidate.
- Pre-refactor local chat smoke check: a Cloudflare Agents chat used the model provider configured at that time, called the read-only `agentelIdentity` tool, and returned the configured Agentel ID, name, and slug.
- The provider-neutral Chat Completions adapter has not been exercised with a provider key after this change.
- At the time of these checks, the sample was not deployed and this validation did not publish a new Update or repository.

This is a local integration demonstration. It does not establish Cloudflare endorsement, adoption, or partnership.

## Close-out checks (2026-09-24)

- `npm run lint`: passed.
- `tsc --noEmit`: passed.
- `oxfmt --check README.md src`: passed for the maintained guide and integration source.
- `npm run check` (formatting, lint, and TypeScript): passed after formatting the inherited `.github/workflows/semgrep.yml`.
- The registered Agentel identity was not re-verified during this pass because the local environment could not resolve the Agentel API host. No model-provider request or deployment was made.
- At the time of these checks, the sample, Lab guide, and repository were local release candidates; no GitHub push or website deployment had been made.

## Publication update (2026-09-25)

- The sample source is now public at [agentel-tech/agentel-cloudflare-agents](https://github.com/agentel-tech/agentel-cloudflare-agents).
- The Agentel Lab guide changes remain local and have not been deployed to the website.
- A local browser chat smoke check passed with MiniMax M3 through the OpenAI-compatible Chat Completions adapter. The Cloudflare Agent called the read-only `agentelIdentity` tool, verified the configured Agentel ID against `/me`, and completed the model response.
- MiniMax's streaming tool-call chunks initially included an empty `tool_calls[].type`, which the AI SDK rejected. `starter/src/server.ts` now normalizes that field to the OpenAI-compatible value `function`; the local browser smoke passed with this compatibility fix.
- `npm run check` passed after the source change. The sample Worker has not been deployed to Cloudflare, and the Agentel Lab guide has not been deployed to the website.
