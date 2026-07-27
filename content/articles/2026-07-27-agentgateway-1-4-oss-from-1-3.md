---
title: "agentgateway 1.4.0 OSS: what changed since 1.3.0"
date: 2026-07-27
description: "A readable walkthrough of open-source agentgateway from 1.3.0 to 1.4.0 — new model API, unified gateways, UI Settings, modern MCP, token exchange, and what to do when you upgrade."
tags: ["agentgateway", "oss", "release", "mcp", "llm", "kubernetes", "oauth", "gateway-api", "a2a", "observability", "helm", "ui"]
categories: ["AI Gateway"]
---

[agentgateway](https://github.com/agentgateway/agentgateway) **v1.4.0** landed on July 27, 2026.

If you are still on 1.3.x, this post is the upgrade map. Not every PR — just what actually changed, why it matters, and what to touch first.

**Sources**

- [v1.3.0](https://github.com/agentgateway/agentgateway/releases/tag/v1.3.0) (June 18)
- [v1.3.1](https://github.com/agentgateway/agentgateway/releases/tag/v1.3.1) (June 22)
- [v1.4.0](https://github.com/agentgateway/agentgateway/releases/tag/v1.4.0) (July 27)
- Full diff: [v1.3.1...v1.4.0](https://github.com/agentgateway/agentgateway/compare/v1.3.1...v1.4.0)

**Install bits for 1.4.0**

- Images: `cr.agentgateway.dev/agentgateway:v1.4.0` and `cr.agentgateway.dev/controller:v1.4.0`
- Charts: `agentgateway`, `agentgateway-crds`, and `agentgateway-standalone`
- Binaries: proxy + `agctl`

---

## The short version

Four things matter most:

1. **`AgentgatewayModel`** — models become real Kubernetes objects
2. **Standalone `gateways`** — UI, LLM, and MCP can share one port
3. **UI Settings** — protect the console like any other route
4. **Auth + MCP** — token exchange, Cross App Access, modern MCP, more IdPs

Everything else is important, but those four change how you run the thing day to day.

| Area | What moved |
|------|------------|
| Kubernetes LLM API | New `AgentgatewayModel` CRD |
| Standalone | `gateways` replace `binds`; one-port default; DB/hybrid storage |
| UI | Settings page to bind and lock down the console |
| MCP | Spec 2026-07-28, Apps, better federation and traces |
| Auth | Token exchange, Cross App Access, Entra for MCP |
| Platform | DaemonSet gateways, Gateway API bumps, better monitors |
| Packaging | Real standalone Helm chart; musl images dropped |

---

## What 1.3 already gave you

v1.3.0 was a big release on its own. New UI (LLM / MCP / Traffic), cost tracking, virtual models, reusable providers and guardrails, more LLM providers, body buffering, better CEL/`agctl`, better traces.

v1.3.1 was a small polish release four days later: controller `podLabels`, Vertex multi-region, same port with different protocols across gateways, UI fixes for wildcards and virtual models.

Nothing in 1.3.1 forces a redesign. **1.4 does** — especially if you run standalone, or you want models to look like first-class Kubernetes objects.

---

## 1. Models as Kubernetes objects

This is the Kubernetes headline.

In 1.3, putting an LLM on the cluster usually meant wiring listeners, `HTTPRoute` body matches, AI policies, and backends by hand. Standalone already felt model-centric. Kubernetes did not.

In 1.4, **`AgentgatewayModel`** closes that gap.

Each object is one client-facing model (or a wildcard). It attaches to a Gateway listener with `parentRefs`. Mentally, it maps to standalone’s model / virtual-model idea. The listener has to opt into LLM behavior on purpose.

Short names: `agmodel` / `agentgatewaymodels.agentgateway.dev`.

Why that helps:

- One GitOps object per model (or family)
- Less copy-paste HTTPRoute matching for every model name
- Kubernetes and standalone stop teaching two different stories

Your old HTTPRoute-based LLM routes still work. You do not have to rewrite everything on day one. Start with one non-prod model on the new API and grow from there.

---

## 2. Standalone gets real “gateways”

Standalone config adds a **`gateways`** concept — with routes, TCP routes, and UI — as the replacement for **`binds`**.

What that unlocks in plain terms:

- LLM, MCP, the UI, and normal HTTP routes can live on the **same listener**
- One route can show up on HTTP and HTTPS without awkward duplication
- The default path is now **UI + LLM + MCP on one port** (Helm leans on **4000**)
- You can put different policies on different surfaces: OIDC for the UI, API keys for the LLM, same gateway

Around that, standalone also got:

- Hybrid mode: file config plus database-backed writes
- Postgres storage and multi-replica notify
- Simpler host + TLS setup
- Admin interface **off** by default
- Migration helpers in the CLI
- A **revamped standalone Helm chart** for 1.4

That last point matters. Upstream basically treats the 1.4 standalone chart as the first real baseline. Do not assume a soft bump from early 1.3 chart experiments.

If your demos still say `binds:`, convert them before you call the upgrade finished.

---

## 3. UI Settings: lock down the console

This is the part you can see.

1.4 adds **UI Settings** under **Tools → Settings**. The subtitle says it clearly: expose the UI on a traffic gateway, then attach policies that protect it.

![agentgateway 1.4 UI Settings — Public UI gateway set to ui, with policy cards for OIDC, JWT, Authorization, External authz, Basic auth, API keys, CSRF, and CORS](/images/articles/2026-07-27-agentgateway-1-4-oss-from-1-3/ui-settings.png)

On that page you can:

- Pick the **Public UI gateway** (screenshot uses a gateway named `ui`)
- Save it, or view the diff first
- Turn on access policies with the same toolkit you already use elsewhere:
  - OIDC
  - JWT
  - Authorization rules
  - External authz
  - Basic auth
  - API keys
  - CSRF
  - CORS
- Expand the **current top-level policy YAML** if you want to see exactly what got written

This is the UI half of the gateway story. The console stops being “that admin port we hope nobody finds.” It becomes a normal route with real auth.

After you upgrade: open Settings, bind the UI to the gateway you actually expose, and turn on OIDC or JWT before that listener hits a real network.

---

## 4. MCP grows up

MCP is not a side feature in 1.4. It got a real protocol bump and a lot of federation polish.

**Protocol and federation**

- Early support for **MCP 2026-07-28** (`server/discover`, `subscriptions/listen`, richer headers and `_meta`)
- Basic **MCP Apps** support (capability ads, `ui://` rewrite, RBAC stripping denied UI resources)
- Multi-target subscriptions
- Lots of multiplex fixes (opaque URIs, app-originated tool calls, SSE events, fail-open fanout)
- Sessions pinned to a backend
- **SEP-414** trace context through `_meta.traceparent`, not only HTTP headers — so stdio backends stay on the same trace

**More identity providers**

Native MCP auth now includes:

- **Microsoft Entra ID** (handles Entra’s awkward `resource` / v2 behavior)
- **Descope**
- **Authentik**

Okta was already first-class in 1.3. The theme is the same: stop needing a custom adapter proxy just to log into MCP.

**Small but painful policy fixes**

- Request-phase guardrail rejections can return **200** like the response phase (some clients hate non-JSON-RPC error statuses)
- Target-level MCP policies are less likely to get dropped
- Passthrough checks are tighter

If you put several MCP servers behind one agentgateway front door, 1.4 is the first release where modern clients, multi-server federation, Apps, and traces feel designed together.

---

## 5. Auth for how agents really call APIs

Checking “is this JWT valid?” is not enough. Agents need **downstream tokens that still represent the user**.

### Token exchange (RFC 8693)

With `backendAuth.tokenExchange`, the gateway takes the inbound user token, exchanges it at your token endpoint, and sends the result upstream.

One front door. Many upstreams. Each gets its own per-user credential. You do not have to invent ext_proc glue for every API.

### Cross App Access / ID-JAG (XAA)

`backendAuth.crossAppAccess` is the delegated path: exchange the user’s ID token for an ID-JAG, then use that to get a **user-scoped** access token for the backend.

That is the “agent calls Microsoft Graph or an internal API **as the user**, without a second login” story.

Examples landed for Keycloak, xaa.dev, token exchange, and JWT bearer.

### Other security bits worth knowing

- SHA-256 encoded API keys
- Admin IP allowlist
- Timing-attack fixes in auth
- `private_key_jwt` with certificates
- Multiple secret-sourced backend headers
- AWS session tags, dynamic `RoleSessionName`, per-backend SigV4 region
- Frontend TLS with multiple client CAs
- Bundled certs
- Sensible cloud-auth timeouts

If your 1.3 setup is “gateway checks JWT, upstream still sees a shared service account,” 1.4 is when you can fix that for real.

---

## 6. LLM path fixes you will actually feel

Not every LLM change needs a banner. These ones show up in production:

- **Vertex / Gemini** — native `generateContent` paths, multi-region (from 1.3.1), better embeddings behavior
- **Azure / Foundry** — Anthropic on Foundry, Responses path fixes, Azure auth types on the Kubernetes side
- **Bedrock** — image translation on the Responses path, tool-name length sanitizing, cache-write tokens in logs
- Reasoning fixes when translating messages → completions
- Decompress / decode request bodies before JSON parse
- Webhook guardrail headers and path via CEL
- Optional logging of user **and** prompt together
- Cost: Prometheus counter, better catalog loading, virtual keys from ConfigMaps
- Telemetry: tool calls, richer debug traces, A2A response metrics
- Safer handling when bodies exceed buffer limits

There was also a fair amount of internal cleanup (provider capabilities, LLM crate split). That is why weird translation edge cases keep getting fixed in this train.

---

## 7. Traffic and platform

A few platform pieces matter if you run this beyond a laptop:

- **DaemonSet** gateways in addition to Deployments
- Internal bind mode for east-west / mesh-style frontends
- Frontend policies aimed at a specific Gateway port
- TCPRoute v1 and a move toward Gateway API 1.6
- ext_proc `failureMode` and richer attributes
- Stream large bodies instead of hard-failing every oversized payload
- CEL replace mode for headers
- Delay policy for fault injection
- Better ext_authz HTTP caching
- Helm extras: `extraContainers`, PodMonitor, standalone ServiceMonitor

Use DaemonSet only if you have a node-local reason. Deployment is still the default.

---

## 8. Upgrade gotchas

Read these before you roll:

1. **Standalone config and chart changed.** `binds` → `gateways`, new storage modes, different defaults, admin off by default. Read the 1.4 values file. Do not blind-upgrade.
2. **Musl images are gone.** Update pins and SBOMs if you depended on them.
3. **Embedded UI builds by default** (`UI=0` to skip).
4. **`agctl` is much smaller**, and some commands moved under a proxy subcommand. Check scripts.
5. Nightly `agctl` builds exist if you live on main.
6. **s390x** support is there for the platforms that need it.
7. There is a test-only rmcp security note — do not drag test deps into prod images.

---

## How I would upgrade

### Kubernetes

1. Bump **CRDs first** (`agentgateway-crds` → v1.4.0). Confirm `AgentgatewayModel` exists.
2. Bump controller and proxy together.
3. Smoke test existing HTTPRoutes, MCP, JWT/ext_authz, and traces.
4. Move **one** non-prod model onto `AgentgatewayModel`.
5. If you use client mTLS / FrontendTLS, retest multi-CA.
6. Keep Deployment unless you truly need DaemonSet.

### Standalone

1. Snapshot config. Use migration helpers if you have local state.
2. Convert `binds` → `gateways`.
3. Put UI / LLM / MCP on the unified port on purpose.
4. Open **UI Settings**, bind the console, turn on OIDC or JWT.
5. If you expected local PVC writes from early charts, move writable mode to **Postgres**.
6. Install from `cr.agentgateway.dev/charts/agentgateway-standalone:v1.4.0`.

### If auth is the pain

1. List upstreams still using shared service accounts.
2. Try token exchange on one API.
3. Try Cross App Access on one user-delegated SaaS API.
4. For MCP + Entra, use the native provider instead of a home-grown proxy.

---

## What I would turn on first

If you already run 1.3 in anger, this is my order:

1. **One gateway port + UI Settings** — fewer listeners, console actually protected
2. **Token exchange or XAA** — stop handing agents god-mode upstream tokens
3. **`AgentgatewayModel`** — cleaner GitOps model catalog
4. **Modern MCP + SEP-414 traces** — federation you can debug
5. **DaemonSet** — only if node-local ingress is a real requirement

---

## Links

- [agentgateway v1.4.0](https://github.com/agentgateway/agentgateway/releases/tag/v1.4.0)
- [v1.3.1...v1.4.0](https://github.com/agentgateway/agentgateway/compare/v1.3.1...v1.4.0)
- [v1.3.0](https://github.com/agentgateway/agentgateway/releases/tag/v1.3.0) · [v1.3.1](https://github.com/agentgateway/agentgateway/releases/tag/v1.3.1)
- Docs: [Kubernetes quick start](https://agentgateway.dev/docs/kubernetes/latest/quickstart/) · [Standalone quick start](https://agentgateway.dev/docs/standalone/latest/quickstart/)
- On this site: [Eight principles of an AI gateway](/articles/2026-07-12-eight-principles-of-an-ai-gateway-agentgateway/) · [Why the v1 line matters](/articles/2026-03-12-agentgateway-v1-why-it-matters/)

---

**Bottom line:** 1.3 made agentgateway a serious LLM/MCP gateway with UI and cost. **1.4 makes day-2 operations less weird.** Models become API objects. UI, LLM, and MCP can share a gateway. The console gets real policy. Auth can finally carry the user downstream.

Bump the charts. Convert standalone `binds`. Bind and lock down the UI. Put one real delegated-auth path in front of an upstream. Then call the upgrade done.
