---
title: "agentgateway 1.4.0 OSS: what changed since 1.3.0"
date: 2026-07-27
description: "Readable upgrade map for open-source agentgateway 1.3.0 → 1.4.0 — MCP 2026-07-28, Cross App Access, token exchange, gateways + UI Settings, experimental AgentgatewayModel, breaking changes, and the High severity MCP session advisory."
tags: ["agentgateway", "oss", "release", "mcp", "llm", "kubernetes", "oauth", "gateway-api", "a2a", "observability", "helm", "ui", "security"]
categories: ["AI Gateway"]
---

[agentgateway](https://github.com/agentgateway/agentgateway) **v1.4.0** landed on July 27, 2026.

If you are still on 1.3.x, this is the upgrade map. It tracks the [official v1.4.0 release notes](https://github.com/agentgateway/agentgateway/releases/tag/v1.4.0) — highlights, breaking changes, security, and the day-2 bits that actually matter — plus a bit of operator commentary.

**Also useful**

- [v1.3.0](https://github.com/agentgateway/agentgateway/releases/tag/v1.3.0) · [v1.3.1](https://github.com/agentgateway/agentgateway/releases/tag/v1.3.1)
- Diff: [v1.3.1...v1.4.0](https://github.com/agentgateway/agentgateway/compare/v1.3.1...v1.4.0)

**Install bits**

- Images: `cr.agentgateway.dev/agentgateway:v1.4.0`, `cr.agentgateway.dev/controller:v1.4.0` (glibc — **no more musl image tags**)
- Charts: `agentgateway`, `agentgateway-crds`, `agentgateway-standalone`
- Binaries: still built with musl; proxy + `agctl`

---

## The short version

Upstream’s own highlight reel:

1. **Full MCP `2026-07-28` support** (and a year of community work behind it)
2. **Cross App Access / Enterprise-Managed Authorization for MCP** (ID-JAG)
3. **OAuth token exchange** backend auth (RFC 8693 + JWT bearer)
4. **Standalone `gateways`** + DB-backed UI config (sqlite / postgres) + **standalone Helm chart**
5. **Experimental `AgentgatewayModel`** on Kubernetes (off by default)
6. Gateway API **v1.6**, fault injection, richer guardrails, lots of fixes

My add-on for operators: **read the breaking + security sections before you helm upgrade.** There is a High severity MCP auth advisory in this release.

| Area | What moved |
|------|------------|
| MCP | Protocol 2026-07-28, Apps, traces via `_meta`, Entra/Descope/authentik |
| Auth | Token exchange, Cross App Access (incl. MCP EMA), hashed API keys |
| Standalone | `gateways` supersede `binds`, UI on same port, sqlite/postgres storage |
| UI | Settings to bind console + attach OIDC/JWT/API key/CSRF/CORS |
| Kubernetes | Experimental `AgentgatewayModel` (off by default) |
| Platform | DaemonSet, Gateway API 1.6 / TCPRoute v1, monitors, delay fault injection |
| Security | [GHSA-mvgg-jvj2-4frq](https://github.com/agentgateway/agentgateway/security/advisories/GHSA-mvgg-jvj2-4frq) + CEL body guidance |

---

## Read this first: breaking changes

Straight from upstream. Do these before you call the upgrade done.

### 1. Gateway API v1.6 and TCPRoute v1

agentgateway builds against **Gateway API v1.6**. The controller uses **`TCPRoute` v1**, not `v1alpha2`.

**Action:** re-apply the Gateway API CRDs that match this release **before** you upgrade the controller/proxy.

### 2. MCP request-phase guardrail rejections return HTTP 200

If an MCP guardrail rejects in the **request** phase, the gateway now returns **HTTP 200** with a JSON-RPC error body — same as the response phase.

**Action:** fix clients/tests that expected a non-200 on request-phase rejects.

Docs: [k8s MCP guardrails](https://agentgateway.dev/docs/kubernetes/main/mcp/guardrails/) · [standalone](https://agentgateway.dev/docs/standalone/main/mcp/guardrails/)

### 3. Standalone `auth.location` no longer nests `expression`

Custom token location config dropped the double-nested `expression` field. Use the flattened form.

**Action:** grep your standalone policies for nested `auth.location` expressions and update them.

### 4. musl container images removed

Musl **image** variants are gone. Use standard glibc images. **Binary** releases are still musl builds.

**Action:** update image pins, digests, and SBOMs.

---

## Read this second: security

### High: MCP sessions crossing routes (GHSA-mvgg-jvj2-4frq)

This release fixes [GHSA-mvgg-jvj2-4frq](https://github.com/agentgateway/agentgateway/security/advisories/GHSA-mvgg-jvj2-4frq) — **High (8.1)** — where stateful MCP sessions could cross routes and overwrite authorization policy.

Read the advisory for impact and mitigation. Credit: [Aonan Guan](https://github.com/0dd).

Related hardening in the same train includes pinning MCP sessions to a backend and tightening passthrough checks. Still: if you run MCP with authz on 1.3, treat 1.4 as a **security upgrade**, not just a feature bump.

### CEL `request.body` / `response.body` in auth policies

Upstream flags a footgun (recommendation + behavior change), also from the same reporter.

| Attribute | 1.3.x and earlier | 1.4 |
|-----------|-------------------|-----|
| `request.body` | Truncated to `http.maxBufferSize` (default 2MB) | **Not available if truncated** |
| `request.truncatedBody` | Not available | Truncated to max buffer size |

Why it matters: a policy like `string(request.truncatedBody).contains("attacker-payload")` can miss a match when the body is over 2MB, or when encoding/compression changes what you thought you were matching. Auth on raw bodies is easy to get wrong.

**Action:** audit CEL auth that touches bodies. Prefer not using body contents for hard authorization when you can avoid it. If you must, handle the “body missing / truncated” case explicitly.

---

## What 1.3 already gave you

v1.3.0 brought the rebuilt UI (LLM / MCP / Traffic), cost tracking, virtual models, reusable providers/guardrails, more LLM providers, body buffering, better CEL/`agctl`, better traces.

v1.3.1 was polish: controller `podLabels`, Vertex multi-region, same port with different protocols across gateways, UI wildcard/virtual-model fixes.

1.3.1 does not force a redesign. **1.4 does** — config shape, CRDs, MCP auth behavior, and at least one security-fixing upgrade path.

---

## 1. MCP protocol 2026-07-28

This is highlight #1 in the official notes.

agentgateway adds support for the upcoming [MCP `2026-07-28` protocol](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/):

- Stateful **and** stateless servers (SEP-2575 server-stateless gap closed; skip synthetic `initialize` for modern requests)
- Trace context through MCP **`_meta`** (SEP-414 / `traceparent`) — stdio backends stay on the trace
- Basic **MCP Apps** (plus multiplexing fixes for app-originated tool calls)
- Multi-target subscriptions/listen, opaque URI multiplexing, preserved multi-resource tool-result capabilities

Most of this is new enough that dedicated guides are still thin. Use the [Kubernetes API reference](https://agentgateway.dev/docs/kubernetes/main/reference/api/) and [Standalone config reference](https://agentgateway.dev/docs/standalone/main/reference/configuration/) for fields available today.

### New MCP identity providers

- **Microsoft Entra ID** (k8s + standalone) — bridges Entra vs MCP-auth mismatches (OIDC discovery metadata, strip RFC 8707 `resource`, short-circuit DCR with your pre-registered app id)
- **Descope** and **authentik** on standalone
- Okta was already first-class in 1.3

Docs: [k8s MCP auth](https://agentgateway.dev/docs/kubernetes/main/mcp/auth/) · [standalone MCP auth](https://agentgateway.dev/docs/standalone/main/mcp/mcp-authn/) · [Descope guide](https://agentgateway.dev/docs/standalone/main/integrations/auth/descope/)

---

## 2. Cross App Access (for MCP and backends)

Official name in the MCP world: [Enterprise-Managed Authorization](https://modelcontextprotocol.io/extensions/auth/enterprise-managed-authorization). Implementation path: OAuth Identity Assertion Authorization Grant — **Cross App Access / ID-JAG**.

An enterprise IdP can broker access between a client app and an MCP server (or other backend) without a separate interactive OAuth dance for every downstream app.

Docs: [k8s Cross App Access](https://agentgateway.dev/docs/kubernetes/main/security/backend-authn-cross-app-access/) · [standalone](https://agentgateway.dev/docs/standalone/main/configuration/security/backend-authn/cross-app-access/)

Examples landed for Keycloak, xaa.dev, token exchange, and JWT bearer. Subject token source is configurable.

---

## 3. OAuth token exchange backend auth

The gateway can exchange an incoming token for a backend credential using [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) token exchange and [RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523) JWT bearer.

Also in this release:

- Kubernetes controller support
- Custom token types + OAuth 2.1-ish exchange defaults
- Inject **multiple** secret-sourced headers
- Override the resolved secret key for OAuth/GCP-style refs

Docs: [k8s OAuth token exchange](https://agentgateway.dev/docs/kubernetes/main/security/backend-authn-oauth/) · [standalone](https://agentgateway.dev/docs/standalone/main/configuration/security/backend-authn/oauth-token-exchange/)

If your 1.3 setup is “edge JWT ok, upstream still sees a shared service account,” this is the fix.

### Related auth extras

- Virtual keys from **ConfigMaps** (not only Secrets)
- API keys stored as **SHA-256 hashes** (raw key need not live in plaintext config)
- Admin IP allowlist + timing-attack fixes
- `private_key_jwt` with certificate credentials
- AWS assume-role: session tags + `RoleSessionName` / `sessionNameExpression` via CEL (e.g. propagate `jwt.sub`)
- Per-backend SigV4 region
- Frontend TLS with **multiple client CAs**
- Bundled certs
- Cloud auth fetch timeouts

Virtual keys docs: [k8s](https://agentgateway.dev/docs/kubernetes/main/llm/cost-controls/virtual-keys/) · [standalone](https://agentgateway.dev/docs/standalone/main/llm/cost-controls/virtual-keys/)

---

## 4. Standalone: `gateways`, storage, Helm

### `gateways` replace `binds`

New top-level **`gateways`** unifies UI, LLM, MCP, and ordinary routes on one listener/port. That is what makes **OIDC on the UI** a first-class story.

Important nuance from upstream:

- `gateways` **supersedes** `binds`
- Existing **`binds` still work**
- The UI offers a **one-click migration** from binds → gateways

Also: simpler host/TLS config, LLM+MCP on the same port, internal bind mode with wildcard fallback.

Config reference: [standalone configuration](https://agentgateway.dev/docs/standalone/main/reference/configuration/)

### Database-backed config (no PVC required)

UI-created config can live in a database:

- **sqlite** for local
- **postgres** for remote / HA

That avoids needing a persistent disk for writable UI workflows. Hybrid mode, policy writes, multi-replica notify, and “log errors to SQL too” landed in the same train. Storage naming moved away from the old configStore wording.

Admin interface is **not** exposed by default.

### Standalone Helm chart

`cr.agentgateway.dev/charts/agentgateway-standalone:v1.4.0` is real: modes for file vs database, gateways-oriented defaults, OIDC secret support, metrics service + ServiceMonitor.

Docs: [Standalone Helm](https://agentgateway.dev/docs/standalone/main/deployment/helm/)

Treat 1.4 as the first serious baseline for that chart, not a soft bump from early experiments.

---

## 5. UI Settings: protect the console

In the product UI: **Tools → Settings → UI Settings**.

Bind the console to a traffic gateway, then attach the same policy toolkit you use elsewhere.

![agentgateway 1.4 UI Settings — Public UI gateway set to ui, with policy cards for OIDC, JWT, Authorization, External authz, Basic auth, API keys, CSRF, and CORS](/images/articles/2026-07-27-agentgateway-1-4-oss-from-1-3/ui-settings.png)

You get:

- **Public UI gateway** picker + view diff / save
- Policy cards: OIDC, JWT, Authorization, External authz, Basic auth, API keys, CSRF, CORS
- Expandable top-level policy YAML for the GitOps-minded

After upgrade: bind the UI to the gateway you actually expose, turn on OIDC or JWT, then put that listener on a real network.

---

## 6. Experimental `AgentgatewayModel` (off by default)

Kubernetes headline — with the official caveats.

**`AgentgatewayModel`** brings the standalone model-centric LLM experience to Kubernetes: serve multiple models on one gateway, route by model name in the request. Higher level than assembling `AgentgatewayBackend` + HTTPRoute body matchers by hand.

- Short names: `agmodel` / `agentgatewaymodels.agentgateway.dev`
- **Experimental**
- **Off by default**
- Existing APIs remain available

Why try it: one GitOps object per model/family, less HTTPRoute glue, k8s and standalone stop telling different stories.

Why not force it day one: experimental. Pilot one non-prod model after the cluster is stable on 1.4.

---

## 7. Guardrails, ext_proc, CEL, fault injection

### Guardrails

- **`BackendConnectionPolicy`** for guardrail callouts (TCP/TLS/HTTP/tunnel) on OpenAI moderation, Bedrock guardrails, Google Model Armor
- Default callout timeouts
- Clearer guardrail decisions in logs/UI
- ext_proc **`failureMode`** (fail-open / fail-closed)

### External processing

Controller support for `metadataContext`, `requestAttributes`, `responseAttributes`, plus `failureMode`.

### CEL

- **Custom CEL functions** you can register for policies
- Opt-in **CEL filters for OTel spans**
- CEL filters that decouple OTLP log fields from stdout
- `source.connectHeaders` for inbound CONNECT headers
- Header transformation **replace** mode via CEL
- Body handling split: `body` vs `truncatedBody` (see security section)

CEL refs: [k8s](https://agentgateway.dev/docs/kubernetes/main/reference/cel/) · [standalone](https://agentgateway.dev/docs/standalone/main/reference/cel/)

### Fault injection: `delay`

New **delay** traffic policy injects latency before forward. Duration string, CEL duration, or number-as-milliseconds. Injected delay counts against the request timeout.

Docs: [k8s fault injection](https://agentgateway.dev/docs/kubernetes/main/resiliency/fault-injection/) · [standalone](https://agentgateway.dev/docs/standalone/main/configuration/resiliency/fault-injection/)

---

## 8. LLM, A2A, and provider path fixes

From the official “LLM gateway enhancements” plus the long PR train:

- **Frontend TLS** multi-CA client cert validation
- **Bedrock:** Responses→Bedrock images, 64-char tool name sanitizing, cache-write tokens in access logs
- **Gemini / Vertex:** embeddings fix, native generateContent paths, detect-mode model/usage extraction
- **Azure AI Foundry:** Anthropic endpoints; Responses/date-based API path fixes
- **A2A v1.0** agent card format in URL rewriting
- Reasoning fixes on messages→completions; body decode before JSON parse
- Cost: Prometheus counter; catalog ConfigMap loading; virtual keys from ConfigMaps
- Telemetry: tool calls, A2A response metrics, richer dtrace
- Semantic routing examples (including Responses)

Providers: [k8s](https://agentgateway.dev/docs/kubernetes/main/llm/providers/) · [standalone](https://agentgateway.dev/docs/standalone/main/llm/providers/)

---

## 9. Deployment and ops

- **DaemonSet** data plane (still defaults to Deployment)
- Helm **`extraContainers`** on control plane pods
- Proxy scrape via **PodMonitor**; standalone metrics Service + **ServiceMonitor**
- `agentgateway_controller_build_info` metric
- Standalone SQL logging for success **and** error, with faster writes
- Internal bind mode (annotation + wildcard fallback)
- Frontend policies targetable to a Gateway port
- Buffer: stream when body exceeds limit instead of always hard-failing
- ext_authz HTTP cache + custom HTTP bodies
- `agctl` much smaller; some trace/config commands under proxy subcommand; nightly artifacts
- **s390x** support
- Embedded UI build by default (`UI=0` to opt out)

---

## How I would upgrade

### Kubernetes

1. Re-apply **Gateway API CRDs** for the v1.6 / TCPRoute v1 set this release expects
2. Bump **`agentgateway-crds`** → v1.4.0
3. Bump controller + proxy **together**
4. If you run authenticated MCP: read [GHSA-mvgg-jvj2-4frq](https://github.com/agentgateway/agentgateway/security/advisories/GHSA-mvgg-jvj2-4frq) and validate session/route authz behavior
5. Smoke HTTPRoutes, MCP, JWT/ext_authz, traces, guardrails
6. Audit CEL auth that uses `request.body` / `response.body`
7. Only then pilot **one** non-prod `AgentgatewayModel`
8. Keep Deployment unless you truly need DaemonSet

### Standalone

1. Snapshot config
2. Note: `binds` still work; migrate with UI one-click or convert to `gateways` deliberately
3. Flatten any nested `auth.location` expressions
4. Pick storage: **sqlite** local vs **postgres** HA (drop PVC assumptions from early charts)
5. Open **UI Settings**, bind console, enable OIDC/JWT
6. Install/upgrade `agentgateway-standalone` v1.4.0 after reading values

### Auth-heavy estates

1. Inventory upstreams still on shared service accounts
2. Prototype token exchange on one API
3. Prototype Cross App Access for one MCP or SaaS API
4. Prefer native Entra MCP provider over home-grown resource-parameter proxies
5. Prefer hashed virtual keys / ConfigMap sources where you can

---

## What I would turn on first

1. **Security fix path** — upgrade MCP authz estates for the High advisory
2. **Gateway API CRD re-apply + clean 1.4 roll**
3. **One gateway + UI Settings** — console actually protected
4. **Token exchange or XAA** — user-scoped upstream tokens
5. **MCP 2026-07-28 + `_meta` traces** — federation you can debug
6. **Experimental AgentgatewayModel** — one non-prod pilot, not a big-bang rewrite

---

## Links

- [Official v1.4.0 release](https://github.com/agentgateway/agentgateway/releases/tag/v1.4.0)
- [Security advisory GHSA-mvgg-jvj2-4frq](https://github.com/agentgateway/agentgateway/security/advisories/GHSA-mvgg-jvj2-4frq)
- [v1.3.1...v1.4.0](https://github.com/agentgateway/agentgateway/compare/v1.3.1...v1.4.0)
- Docs: [Kubernetes quick start](https://agentgateway.dev/docs/kubernetes/latest/quickstart/) · [Standalone quick start](https://agentgateway.dev/docs/standalone/latest/quickstart/)
- On this site: [Eight principles of an AI gateway](/articles/2026-07-12-eight-principles-of-an-ai-gateway-agentgateway/) · [Why the v1 line matters](/articles/2026-03-12-agentgateway-v1-why-it-matters/)

---

**Bottom line:** 1.4 is not only features. It is **MCP protocol catch-up**, **real delegated auth** (token exchange + Cross App Access for MCP), **standalone that can run UI/LLM/MCP on one port with DB-backed config**, an **experimental model API** on Kubernetes, and a set of **breaking + security fixes you should not skip**.

Re-apply Gateway API CRDs. Bump charts. Read the MCP session advisory. Bind and lock down the UI. Then play with AgentgatewayModel — in that order.
