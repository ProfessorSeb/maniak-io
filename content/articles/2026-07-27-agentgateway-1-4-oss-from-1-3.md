---
title: "agentgateway 1.4.0 OSS: everything that changed since 1.3.0"
date: 2026-07-27
description: "A practical walkthrough of open-source agentgateway from v1.3.0 to v1.4.0 — AgentgatewayModel, unified standalone Gateways, modern MCP, token exchange and Cross App Access, DaemonSet gateways, and the ops surface that actually matters when you upgrade."
tags: ["agentgateway", "oss", "release", "mcp", "llm", "kubernetes", "oauth", "gateway-api", "a2a", "observability", "helm"]
categories: ["AI Gateway"]
---

[agentgateway](https://github.com/agentgateway/agentgateway) **v1.4.0** shipped on July 27, 2026. If you are still on **v1.3.x**, this is the upgrade map: what actually changed in OSS between **1.3.0** and **1.4.0**, what is net-new vs patch noise, and what to plan for before you bump the chart.

Sources for this post:

- [v1.3.0](https://github.com/agentgateway/agentgateway/releases/tag/v1.3.0) (2026-06-18)
- [v1.3.1](https://github.com/agentgateway/agentgateway/releases/tag/v1.3.1) (2026-06-22)
- [v1.4.0](https://github.com/agentgateway/agentgateway/releases/tag/v1.4.0) (2026-07-27)
- Full diff: [v1.3.1...v1.4.0](https://github.com/agentgateway/agentgateway/compare/v1.3.1...v1.4.0)

Artifacts for 1.4.0:

- Images: `cr.agentgateway.dev/agentgateway:v1.4.0`, `cr.agentgateway.dev/controller:v1.4.0`
- Charts: `agentgateway`, `agentgateway-crds`, and **`agentgateway-standalone`**
- Binaries: proxy + `agctl`

---

## Short version

| Area | What moved 1.3 → 1.4 |
|------|----------------------|
| Kubernetes LLM model API | New **`AgentgatewayModel`** CRD — model-centric config instead of hand-wired routes |
| Standalone | **`gateways`** replaces **`binds`**; UI + LLM + MCP on one port by default; hybrid/DB storage |
| MCP | Spec **2026-07-28**, Apps, multi-target subscriptions, SEP-414 `_meta` tracing, more IdPs |
| Auth | RFC 8693 **token exchange**, **Cross App Access / ID-JAG**, Entra-native MCP, admin IP allowlist |
| Workloads / platform | **DaemonSet** gateways, Gateway API / TCPRoute bumps, s390x, PodMonitor/ServiceMonitor |
| LLM path | Vertex native Gemini endpoints, Responses↔Bedrock images, cost Prometheus counter, tool-call telemetry |
| Packaging | Standalone Helm chart is real in 1.4; musl images dropped |

If you only remember three things: **AgentgatewayModel** on Kubernetes, **unified Gateways** on standalone, and a much more serious **auth + MCP** story.

---

## Where 1.3 left you

v1.3.0 was already a big release: rebuilt UI (LLM / MCP / Traffic), cost and token analysis, virtual models, reusable providers and guardrails, a pile of new LLM providers, body buffering, richer CEL/`agctl`, and better traces.

v1.3.1 (four days later) was small and useful:

- Controller `podLabels`
- Vertex multi-region endpoints
- Same port with different protocols across gateways
- UI fixes for wildcards and virtual models
- Tighter JSON schema / CEL registration behavior

Nothing in 1.3.1 forces a redesign. **1.4 does**, especially if you run standalone or you want model config to look like first-class Kubernetes objects.

---

## 1. `AgentgatewayModel` — model-centric Kubernetes API

This is the Kubernetes headline for 1.4.

Before: expose an LLM by assembling listeners, `HTTPRoute` body matching, AI route policies, and provider backends by hand. Standalone already had a model-centric path (match model name → provider → virtual model / failover). Kubernetes did not.

Now: each **`AgentgatewayModel`** defines one client-facing model (or wildcard), attaches to a Gateway listener via `parentRefs`, and maps to the same mental model as standalone `llm.model` / virtual models. The listener has to opt into LLM behavior explicitly.

Short names: `agmodel` / `agentgatewaymodels.agentgateway.dev`.

Why you care:

- GitOps-friendly model inventory (one object per model or family)
- Less copy-paste of HTTPRoute matchers for every model name
- Aligns k8s and standalone so demos and production stop diverging

Upgrade note: existing HTTPRoute-based LLM routes still work. Treat `AgentgatewayModel` as the preferred path for new models, not a forced rewrite on day one.

---

## 2. Standalone: `gateways` replace `binds`

Standalone config grew a **`gateways`** concept (with `routes`, `tcpRoutes`, `ui`) as the replacement for **`binds`**.

What that unlocks:

- **LLM, MCP, UI, and ordinary HTTP routes on the same listener**
- One route exposed on many listeners more cleanly (HTTP + HTTPS is the common case)
- Default guided path: **UI + LLM + MCP on one port** (Helm default leans on **4000**)
- Policies on the UI itself (OIDC on UI, API key on LLM in the same gateway is a supported pattern)

Related standalone work in the same window:

- LLM and MCP on the same port even before the full Gateways cutover
- **Hybrid mode**: file config plus database-backed writes (policies, config store renamed toward **storage**)
- Config / policy writes to Postgres, multi-replica notify support
- Simpler host + TLS configuration
- Admin interface **not** exposed by default
- CLI helpers for **config migrations**
- Standalone Helm chart **revamped for 1.4** (modes: read-only file vs database; `gateways` not `binds`; OIDC secret support). Note from upstream: the standalone chart never really “shipped finished” in 1.3, so treat the 1.4 chart as the first real baseline, not a soft bump.

If you still have `binds:` YAML from 1.3 demos, plan a conversion pass before you call the upgrade done.

---

## 3. MCP: modern protocol, federation, and enterprise IdPs

MCP is not a side feature in 1.4. It got a real protocol bump and ops hardening.

### Protocol and federation

- Initial support for **MCP 2026-07-28** (`server/discover`, `subscriptions/listen`, `Mcp-Method` / `Mcp-Name`, richer `_meta`)
- **MCP Apps** basics (capability advertisement, `ui://` rewrite for federation, RBAC stripping of denied UI resources)
- **Multi-target subscriptions / listen**
- Multiplex fixes: opaque resource URIs, `prefix: never` for app-originated tool calls, SSE message events, fail-open fanout skipping upstream JSON-RPC errors
- Session pinning to backend
- SEP-2575 server-stateless conformance gap closed
- **SEP-414**: propagate trace context via `_meta.traceparent` (not only HTTP headers) so stdio backends stay on the same trace

### Auth providers for MCP

Native MCP IdP coverage expanded hard:

- **Microsoft Entra ID** (handles Entra’s rejection of RFC 8707 `resource` and related v2 quirks)
- **Descope**
- **Authentik**
- (1.3 already had Okta as first-class; 1.4 keeps pushing “MCP OAuth without a custom adapter proxy”)

### Guardrails / policy behavior

- Request-phase MCP guardrail rejections returned as **200** to match response-phase behavior (clients that choke on non-JSON-RPC HTTP errors will thank you)
- Target-level MCP policies no longer dropped in edge cases
- Passthrough route checks tightened

If you federate multiple MCP servers behind one agentgateway front door, 1.4 is the first release where “modern client + multi-server + traces + Apps” feels intentional instead of heroic.

---

## 4. Auth that matches how agents actually call APIs

Agents do not only need “is this JWT valid?”. They need **downstream tokens that represent the user**.

### RFC 8693 token exchange

`backendAuth.tokenExchange` lets the gateway take the inbound user bearer token, exchange it at a configured token endpoint, and send the result upstream. One front door, many upstreams, each with their own per-user credential — without inventing ext_proc glue for every API.

Controller-side support and builders landed in the same train; OAuth 2.1-ish defaults and custom token types followed.

### Cross App Access / ID-JAG (XAA)

`backendAuth.crossAppAccess` implements the Identity Assertion Authorization Grant flow: exchange the user’s ID token for an ID-JAG at the IdP, then use `jwt-bearer` against the resource AS to get a **user-scoped** backend access token. That is the “agent calls Microsoft Graph / internal API **as the user** without a second login” path.

Examples landed for Keycloak, xaa.dev, token exchange, and JWT bearer. Subject token source is configurable.

### Other auth / security

- SHA-256 encoded API keys
- Admin **IP allowlist** + timing-attack fixes in auth
- OAuth **private_key_jwt** with certificate credentials
- Backend auth: inject **multiple** secret-sourced headers; override resolved secret keys for OAuth/GCP refs
- AWS: session tags, `RoleSessionName` (including CEL), per-backend SigV4 region
- Frontend TLS: **multiple CAs** for client cert validation
- Bundled certs support
- Cloud auth fetch timeout defaults

If your 1.3 setup is “gateway validates JWT, upstream still sees a static service account,” 1.4 is when you can fix that properly.

---

## 5. LLM path improvements (beyond the new CRD)

Not every LLM change needs a press release, but several will hit you in production:

- **Vertex**: native Gemini `generateContent` / `streamGenerateContent`; multi-region was already in 1.3.1; rawPredict rewrite for any publisher; Gemini embeddings no longer fail the path
- **Azure / Foundry**: Anthropic endpoints on Foundry; date-based Azure Responses API path fixes; Azure auth types on the k8s side
- **Bedrock**: Responses → Bedrock **image** translation; tool name sanitization for Converse’s 64-char limit; cache-write token propagation into access logs
- Reasoning fixes on messages → completions (including non-GPT models with tools)
- Request body **Content-Encoding** decoded before JSON parse; decompress on model router path
- Webhook guardrails: headers and path configurable via **CEL**
- Log **user and prompt** together when you opt in
- **Cost**: Prometheus counter metric; cost catalog ConfigMap loading updates; virtual keys from ConfigMaps
- **Telemetry**: tool calls in telemetry; A2A response telemetry; richer dtrace (streaming bodies, frontend policy selection, sampling decision, final status/error)
- CEL custom functions; CONNECT headers as `source.connectHeaders`; safer body exposure when buffer limits are hit
- Guardrail callout default timeouts; better guardrail logs/UI

Centralizing provider format capabilities and splitting the LLM crate are mostly internal, but they explain why translation edge cases (Responses, Messages, Completions, detect mode) keep getting fixed in this train.

---

## 6. Traffic, Gateway API, and extensibility

- **DaemonSet** gateway workloads (in addition to Deployment) — node-local proxy patterns without fighting the deployer
- **Internal bind mode** (annotation + wildcard fallback) for mesh-style / east-west frontends
- Frontend policies targetable to a **Gateway port**
- **TCPRoute v1**; bump toward Gateway API **1.6**; Gateway-TCP conformance profile
- ext_proc: `failureMode`, richer metadataContext / requestAttributes / responseAttributes
- Body buffer: stream when body exceeds limit (instead of hard-failing every large payload)
- Transformation: CEL **replace** mode for headers
- **Delay** policy for fault injection
- ext_authz: HTTP cache path + custom body for HTTP payloads (building on 1.3 gRPC cache work)
- Match repeated HTTP headers individually
- Helm: `extraContainers` on control plane; PodMonitor for proxy scrape; standalone ServiceMonitor / metrics service

---

## 7. Ops, packaging, and breaking edges

Worth calling out before you roll clusters:

1. **Standalone chart and config model changed.** `binds` → `gateways`, storage modes, default ports, admin exposure. Read the 1.4 chart values before a blind `helm upgrade`.
2. **Musl images dropped.** If your digests or SBOMs assumed musl tags, update pin lists.
3. **Embedded UI build** is part of the default build (`UI=0` to opt out).
4. **agctl** lost ~100MB (good), and some trace/config logic moved under a proxy subcommand — scripts may need path updates.
5. **Nightly agctl** artifacts exist if you live on main.
6. **s390x** support landed for the platforms that need it.
7. Security docs include a test-only rmcp advisory — do not copy test deps into prod images casually.

Controller build info metric and JWKS fetch concurrency improvements are quiet reliability wins.

---

## Suggested upgrade path

### Kubernetes (controller + proxy)

1. Bump **CRDs chart first** (`agentgateway-crds` → v1.4.0). Confirm `AgentgatewayModel` CRD is present.
2. Bump controller + proxy to v1.4.0 together (avoid skew for new model API and MCP bits).
3. Smoke: existing HTTPRoutes, MCP backends, JWT/ext_authz, traces.
4. Pilot one non-prod model on **`AgentgatewayModel`** instead of converting everything.
5. If you terminate mTLS or use FrontendTLS, test multi-CA client auth.
6. Decide Deployment vs **DaemonSet** only if you have a node-local reason; default stays Deployment.

### Standalone

1. Snapshot config. Run migration CLI helpers if you have local state.
2. Convert `binds` → `gateways`. Put UI/LLM/MCP on the unified port deliberately.
3. If you used PVC-ish local write assumptions from early standalone charts, move to **Postgres storage** for writable hybrid mode.
4. Re-verify OIDC on UI vs API keys on LLM if you collapse ports — CORS and cookie paths get simpler, but policy attachment moves with the gateway object.
5. Install/upgrade via `cr.agentgateway.dev/charts/agentgateway-standalone:v1.4.0`.

### Auth-heavy estates

1. Inventory upstreams that still use shared service accounts.
2. Prototype **token exchange** for one API, **crossAppAccess** for one user-delegated SaaS API.
3. For MCP + Entra, prefer the **native Entra provider** over homegrown resource-parameter stripping proxies.

---

## What I would actually use from 1.4 first

Personal priority order if you already run 1.3 in anger:

1. **Unified standalone gateway port** — fewer listeners, fewer ingress rules, UI protected with real auth.
2. **Token exchange / XAA** — stop minting god-mode upstream tokens for agents.
3. **AgentgatewayModel** — clean GitOps model catalog on Kubernetes.
4. **MCP 2026-07-28 + SEP-414 traces** — federation that debuggable in Jaeger/Langfuse, not folklore.
5. **DaemonSet mode** — only if you are doing per-node LLM/MCP ingress or host-network adjacent designs.

---

## Links

- Release: [agentgateway v1.4.0](https://github.com/agentgateway/agentgateway/releases/tag/v1.4.0)
- Compare: [v1.3.1...v1.4.0](https://github.com/agentgateway/agentgateway/compare/v1.3.1...v1.4.0)
- Prior baseline: [v1.3.0](https://github.com/agentgateway/agentgateway/releases/tag/v1.3.0) · [v1.3.1](https://github.com/agentgateway/agentgateway/releases/tag/v1.3.1)
- Docs quick starts: [Kubernetes](https://agentgateway.dev/docs/kubernetes/latest/quickstart/) · [Standalone](https://agentgateway.dev/docs/standalone/latest/quickstart/)
- Related on this site: [Eight principles of an AI gateway](/articles/2026-07-12-eight-principles-of-an-ai-gateway-agentgateway/) · [agentgateway v1 line](/articles/2026-03-12-agentgateway-v1-why-it-matters/)

---

**Bottom line:** 1.3 made agentgateway a serious LLM/MCP gateway with UI and cost. **1.4 makes the config model and identity model match how people actually run agents** — models as API objects, one gateway port for humans and machines, and backend auth that can carry the user. Upgrade the charts, convert standalone `binds`, and put one real delegated-auth path in front of an upstream before you call it done.
