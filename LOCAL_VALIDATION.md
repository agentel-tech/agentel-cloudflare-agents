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

## CF-M1 local maintenance candidate (2026-10-11)

This section records new checks on the isolated candidate, based on commit
`32204b6825efc8f77693fde390c6195667fa2564` plus the existing uncommitted README
clarification. It does not transfer the September model smoke result to SDK 2.

- `@agentel/sdk` is pinned to `2.0.0`. npm generated the lock using clean,
  empty npm configurations and an independent cache. Only the root package
  metadata and the SDK package entry changed; other locked package entries
  retain their prior values. The SDK integrity matches the previously fetched
  public npm package bytes.
- The existing `AgentelConnector.fromEnv()` / `me()` integration source stays
  unchanged and keeps the configured channel ID. No registration path was added.
- The actual Node engine intersection is `^22.18.0 || >=24.11.0`. Node 22.12
  is insufficient for the existing Babel 8 dependency requirements; the root
  CI definition pins Node 22.18.0 and runs `npm ci` from `starter/`.
- Official Node 22.18.0 for Darwin arm64 was fetched from nodejs.org and
  verified against its SHA256 checksum. Its bundled npm 10.9.3 performed
  offline `npm ci --ignore-scripts --no-audit --no-fund`: passed, 534 packages.
- `npm run check` on Node 22.18.0: formatting, lint, and TypeScript passed;
  zero lint warnings or errors. TypeScript uses `noEmit`.
- `npm test` on Node 22.18.0: three top-level behavior tests and five nested
  missing-configuration cases passed (Node test count: 8/8). The tests invoke
  the actual integration function and installed SDK 2 with synthetic fetch
  responses. Missing configuration makes zero requests; an ID mismatch rejects;
  success returns exactly `id`, `name`, and `slug`, filtering credential and
  other metadata. They also cover configuration trimming and API URL normalization.
- A first minimum-version installation with system npm 12.0.2 emitted an unsupported
  npm/Node pairing warning; that attempt is retained separately and superseded
  by the clean bundled-npm result above.
- The workflow has been prepared at the repository root. No GitHub Actions
  run or Linux execution was performed during this pass. The inherited Semgrep
  file remains an inactive example under `starter/.github/`.
- A direct `vite build` using Node 22.18.0 passed for the Worker and client
  bundles. The build ran with an empty credential environment, isolated
  build-tool configuration, HTTP fetch disabled, and Wrangler metrics disabled.
  It emitted the expected missing-secret warning and the client
  chunk-size warning. This is bundle compatibility evidence, not a live Worker
  or browser-chat acceptance result. The root CI definition includes this check.
- No live `/me`, model-provider request, browser chat,
  publication, deployment, Mission work, or independent release review was
  performed. No credentials were read or copied. The original repository was
  not modified. Installed dependencies and downloaded verification runtimes
  are local tooling and are excluded from the source candidate.


## CF-M2 native MCP local review candidate (2026-10-11)

The isolated candidate adds a native Cloudflare MCP client to the existing
Agentel remote endpoint, keeps SDK 2.0.0 and all locked dependencies unchanged
from CF-M1, and removes the maintenance identity as a customer default.

- Local Node 22.18.0 formatting, lint, TypeScript, 38 offline unit tests,
  and Worker/client bundle checks passed. Credentials and HTTP fetch were
  disabled for the bundle. Expected missing-secret/chunk warnings remain.
- A local Miniflare/workerd harness exercised the built ChatAgent plus the
  same source AI binding: 31 checks passed with complete actual catalog input
  and output schemas and synthetic response data. These cover protected
  asset/RPC paths, six definitions, AI binding, secret-free stored options,
  runtime input/output validation, explicit business errors, cross-origin
  rejection, 401/no external OAuth discovery, redirect refusal, partial
  catalog failure, and current-secret reconnect after persistence restart.
  This is not hosted Worker or browser/model adoption evidence.
- Native `agents@0.17.4.getAITools` skips all six current definitions because
  Zod cannot convert output `not` constraints. A thin AI SDK JSON-schema input
  binding resolves that conversion issue while native `callTool` continues
  validating the full output schema. An initial simulation omitted output
  schemas and did not reveal this; its limitation is retained, and the final
  full-schema simulation supersedes that result.
- workerd does not implement redirect:error. The candidate uses manual and
  rejects non-2xx without exposing Location; the initial failed runtime
  attempt is retained in the audit package. The local Node egress relay also
  needed to omit hop-by-hop/body-length headers; it is harness-only code.
- Actual public Agentel reads used the ordinary STARTER INFO Reader in local
  workerd with a fixed-URL Node egress relay. Six-tool discovery and binding,
  Mission list, a returned Mission detail, its target participation guide,
  refresh and allowance readback passed. Four open Missions were returned;
  the first was the AI Application Post-Mortem Corpus. All three business
  reads cost 1 INFO each; observed daily and monthly allowance delta was 3.
  This consumed ordinary free allowance and is not a cash payment.
- The target guide supplied an SDK/API application path. No SDK application,
  assignment, acceptance, delivery or Credit transaction occurred. Returned
  rewards were unspecified and settlement disabled; this reflects the current
  rollout stage, not an absence of the designed Credit system.
- Four connection/conversion setup attempts made zero paid business calls,
  and their failed receipts remain separate. The successful finite run made
  each of the three paid calls once; no paid retry was performed.
- Reader injection is server side, absent from stored MCP options. JSON is
  parsed before exact-key redaction; reflected response-header keys fail
  closed. This is not a guarantee against arbitrary encoded secret content.
  SSE is explicitly unsupported for this current JSON-only endpoint.
- Chat recovery is disabled. AI reads in each turn are serialized and a
  failure halts queued reads. Unit regression proves three concurrently
  scheduled reads emit only one call when the first returns an error. Idle
  persistence restart caused no paid replay; an interrupted live model turn
  was not exercised.
- Browser Basic/WebSocket operation, live model autonomous selection, hosted
  Cloudflare deployment, GitHub Actions/Linux, customer outcomes, independent
  Claude review and public release remain NOT_RUN. The original repository,
  CF-M1 sealed candidate and prior platform evidence were not overwritten.


Final guard increment: missing output schemas and paginated/batch tool catalogs
fail closed, while Mission/search content cursors retain their existing
behavior. After that change, a separate native real-source free smoke passed:
six definitions and AI bindings, complete output validation, refresh and
get_connection readback, 0 INFO daily/monthly delta. The earlier 3 INFO paid
run predates only these guards, not the native transport/AI-binding fix; it
was not replayed. Actual model-turn interruption remains untested.


## CF-M2 RC2 and hosted phase 1 (2026-10-11)

This is a later result, not a rewrite of the dated historical records above.
The existing September README clarification and its MiniMax smoke boundary
remain preserved. The model/INFO/provider and deployment scopes are distinct.

- Narrow independent source review passed for RC2 source manifest
  `c01bc11020794956f8939aa9fd83cb620899767be4d21b3a8bbc16800969ffd7`.
  This closed the anonymous restored-ready transport issue and added a default
  cap of three INFO-consuming attempts per model turn. The reviewer did not
  rerun all 43 Node tests and 43 local workerd simulations; those remain Builder
  evidence with synthetic data, not hosted recovery or real model evidence.
- The separately reviewed one-line ASSETS binding is integrated. An isolated
  Node 22.18.0 format/lint/TypeScript/build pass produced all 312 Worker/client
  payload files byte-for-byte equal to the accepted validation deployment.
  The final generated config includes `assets.binding: ASSETS`. Original dirty
  work and historical sealed sources were not overwritten.
- A separately authorized independent Worker was uploaded once and read back
  at 100%: version `3f9ef4e3-9c3d-41b4-9853-7f27d869b142`. Its entry and two
  Worker chunks matched frozen hashes. It has its own SQLite Durable Object
  and static assets, no production routes, Cron, shared D1, or cross-Worker
  bindings. Agentel's production web/MCP and Central were unchanged.
- Real hosted HTTP checks rejected anonymous/wrong operator credentials with
  401 and foreign-Origin/cross-site requests with 403. Authenticated HTML and
  the entry JavaScript returned 200 with their frozen hashes.
- A native WebSocket protocol client, separately from the browser, discovered
  six guarded read-only tools and made one `get_connection` request with an
  ordinary STARTER Reader. Its current rule cost was 0 INFO and its identity
  was read-only. No application, assignment, delivery, reward, or SDK write was
  performed. This phase initiated zero metered business reads and zero model
  calls; one balance snapshot does not establish a global allowance delta.
- Actual Chrome, with the existing operator Basic credential, rendered the
  page and displayed Connected, Agentel ready, and six tools. Operator refresh
  and a frontend page reload reconnected. No chat message or attachment was
  sent. The password-manager save prompt was declined. The in-app browser's
  Basic challenge was client-blocked; native Chrome succeeded without changing
  the access policy. Opening the Worker by a cross-site link remains rejected.
- Browser reload/reconnection does not prove that Cloudflare evicted and
  reconstructed the Durable Object. Hosted reconstruction remains UNVERIFIED:
  no platform lifecycle evidence was captured. No debug restart endpoint was
  added. The existing local workerd reconstruction tests retain their scope.
- Credentials were excluded from stored public evidence. The operator remains
  responsible for the protected validation Worker's later teardown; that was
  not performed by this acceptance run.
- At this hosted phase-1 cutoff, Linux GitHub Actions, current model autonomous
  Mission/Skill selection, an actual model cap experience, and template or
  directory submission were not run. Check the exact Git commit's PR/Actions
  result for later Linux CI; do not transfer a PASS between source versions.
