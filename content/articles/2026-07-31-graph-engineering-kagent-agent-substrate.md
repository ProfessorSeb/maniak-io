---
title: "Graph Engineering on Kubernetes: kagent + Agent Substrate"
date: 2026-07-31T10:00:00-04:00
description: "Multi-agent demos fall apart when topology is hidden in prompts and every idle agent holds a pod. Here's a human take on graph engineering — and how to run it on kagent + Agent Substrate."
draft: false
categories:
  - AI
  - Agents
  - Kubernetes
tags:
  - kagent
  - Agent Substrate
  - graph engineering
  - multi-agent
  - SandboxAgent
  - GitOps
  - gVisor
  - MCP
  - Kubernetes
---

# Graph Engineering on Kubernetes: kagent + Agent Substrate

**By Sebastian Maniak**

Here's a familiar pattern. You stand up a few agents, each in its own pod. You give one a supervisor prompt that vaguely says "delegate when needed." Tools get wired in application code. The demo works.

Then real traffic shows up. Half the sessions sit idle. Nobody can draw who called whom after an incident. And you're paying for a fleet of pods that spend most of their lives doing nothing.

That is not a model problem. It is an architecture problem — specifically a **topology** problem.

[A practical guide from beamnxw](https://x.com/beamnxw/status/2081022966645535079) cuts through a lot of the fuzzy language. The short version is worth memorizing:

> **Harness engineering** builds the machinery around the model.  
> **Loop engineering** designs the repeated work-and-feedback cycle.  
> **Graph engineering** makes the workflow topology explicit: nodes, branches, joins, state transitions, and controlled cycles.

Even shorter: **environment → feedback → flow**.

Those three layers all sit around the same model. They all affect reliability. They can all involve "loops." They are still not the same thing. Confusing them is how teams draw twelve-node diagrams on day one, never define what "done" means, and dump every tool they can find into one agent.

This post takes that framing seriously — especially the **graph** layer — and maps it onto open source you can actually run:

- **[kagent](https://kagent.dev)** as the control plane (agents, tools, and edges as Kubernetes CRDs)
- **[Agent Substrate](https://github.com/agent-substrate/substrate)** as the execution layer (sandboxed actors that suspend when idle and wake fast)

The goal is not more jargon. It is a multi-agent system you can review in Git, operate on a cluster, and not overpay for while it waits.

---

## Why the three layers suddenly matter

A raw language model cannot create files, keep project state, run a test suite, open a browser, enforce an approval rule, or restart a failed job. Those capabilities come from the environment it sits in.

As agentic software matures, a stack is coming into focus:

1. At the foundation, the **harness** — the code and runtime that actually run the model  
2. Next, **loops** — repeating execution and quality checks  
3. Finally, **graphs** — structured paths that guide the whole process  

Labels are still messy in the wild. "Agent harness" is starting to mean something specific. "Loop engineering" is newer practitioner language. **Graph engineering** here is practical, not academic: you are building agent workflows as explicit directed graphs or state machines — control flow and state transitions, *not* knowledge-graph entity modeling.

That distinction matters the moment an agent leaves a demo notebook and starts touching files, APIs, customers, or production clusters.

---

## Layer 1 — Harness: the environment

The harness is everything outside the weights. In practice that usually includes:

- **Context injection** — instructions, retrieved facts, conversation state, task policies  
- **Action surfaces** — APIs, shells, browsers, databases, MCP tools  
- **Persistence** — files, checkpoints, sessions, progress logs, memory  
- **Execution control** — timeouts, retries, budgets, model routing, sub-agent spawning, approval gates  
- **Safety** — permissions, isolation, allow lists, secret handling  
- **Observability** — traces, tool I/O, cost, latency, eval results  

A useful trick from the same guide: *remove the model from your architecture diagram*. What is left is probably the harness.

Two teams can use the same foundation model and get very different outcomes because one gives the model clean tools, a stable workspace, constrained permissions, and observable state — and the other gives it a vague prompt and an unreliable API wrapper. The intelligence may be similar. The working conditions are not.

On our stack, a large part of the harness is **Agent Substrate**: gVisor actors, WorkerPools, snapshot/restore to object storage, network isolation, and the lifecycle that makes idle sessions cheap. Tools and memory still come through kagent and MCP. The point is the same as the article's: **the harness is the working system**, not a footnote under the model card.

---

## Layer 2 — Loop: evidence and stop rules

Every tool-using agent already has a small loop:

call the model → look at the result → run tools → feed observations back → repeat until something terminal happens.

**Loop engineering** starts when you deliberately design cycles *around* that behavior: verification loops, event-driven wakeups, improvement loops that learn from traces. LangChain's newer framing treats these as a *stack* of loops, not one magic `while`.

A well-engineered loop has anatomy:

- **Trigger** — user request, schedule, failed test, new data, grader feedback  
- **Goal** — a specific state to reach, not "keep improving"  
- **State and memory** — what the next cycle needs without replaying everything  
- **Action policy** — what the agent may change, call, delegate, or spend  
- **Evidence** — tests, schema validation, citations, diffs, metrics, human review  
- **Feedback** — compact, actionable description of why evidence failed  
- **Stopping rule** — success, budget, timeout, irrecoverable error, human escalation  

The key line from beamnxw, which I keep repeating to teams:

> Do not loop on confidence. Loop on evidence.

"The agent says it is done" is not a stopping condition. "The tests pass, the links resolve, the schema validates, and the reviewer approves" is.

A prompt tells the model what to do during a call. A **loop** specifies what the system does *after* the call: how it observes results, chooses feedback, decides whether to continue, persists progress, and terminates. Prompt quality still matters. The loop turns a one-shot instruction into a managed process.

Tradeoff: every grader, reviewer, or retry costs latency and money. Prefer the simplest architecture that works. Add loops where the cost of failure is higher than the cost of verification.

On kagent, those loops live **inside** each graph node — the research agent, the writer, the reviewer — not as a vague global "try harder" policy.

---

## Layer 3 — Graph engineering (the deep dive)

This is the heart of the post. Graph engineering asks a different question than harness or loop:

> Not only *what* the agent does — *what component is permitted to run next?*

Steps are **nodes**. Allowed next steps are **edges**. Edges can express sequence, conditional branching, parallel fan-out, joins, loops, and human interrupts. **State traverses the graph.** The topology is what makes control flow checkable — by you, by a reviewer, by Git.

LangGraph-style systems emphasize durable execution, shared state, and human-in-the-loop control. AutoGen's docs put it bluntly: use a graph when you need exact control over agent order, different next steps for different outcomes, deterministic branching, or complex multi-step processes with cycles.

That is not "draw boxes because multi-agent is trendy." That is **control over agents**, not another abstraction for the sake of it.

### What graph engineers actually decide

beamnxw lists the real design work. This is the checklist I use when someone says "we should make it multi-agent":

**1. Node boundaries**  
Which work belongs in a deterministic function, an LLM call, a specialist agent, or a human review step?  
If everything is "another LLM agent," you have not drawn boundaries — you have multiplied prompts.

**2. State schema**  
What may each node read or update? How do parallel updates merge?  
If state is "whatever is in the chat log," joins and recovery will hurt.

**3. Routing conditions**  
Which *evidence* sends work forward, backward, sideways, or to escalation?  
Routing on vibes ("looks good") is how you reintroduce invisible control flow.

**4. Concurrency**  
What can run in parallel? What must join? What shared resources need coordination?  
Fan-out without a join plan is just concurrent chaos.

**5. Cycles and exits**  
Where are retries legal? How many are allowed? What makes the cycle *safe*?  
A cycle without an exit is a cost leak with extra arrows.

**6. Durability**  
Where do checkpoints occur? How does execution resume after interruption?  
Long-running multi-agent work without durability is a demo, not a system.

If you cannot answer those six, you do not have a graph design yet. You have a slide.

### When a graph is worth the ceremony

Graphs earn their keep when the process has:

- meaningful **branches**  
- **parallel** work that must rejoin  
- **approvals**  
- **recovery** paths  
- multiple **specialist** agents with different tools and prompts  

They are *less* useful when the job is simply "give one agent three tools and let it work."

There is also a real risk the article calls out: **a graph can freeze assumptions too early**. If the model must dynamically invent the plan, forcing every possible path into a diagram can make the system more brittle, not less. Formalize paths you have observed. Do not invent a twenty-node topology on day one because the architecture blog had pretty boxes.

### How the three layers nest in one system

Notice the nesting carefully:

- The **graph** runs with support from the harness  
- One or more **loops** live *inside* the graph (usually inside nodes)  
- The **harness** supplies the state, tools, sandboxes, and evaluators those loops need  

Categories overlap because software layers overlap. Each still gives you a different lever when the system fails:

| If you see… | Fix this layer first |
|-------------|----------------------|
| Cannot resume, loses workspace, tool surface is a mess, permissions are wrong | **Harness** |
| Runs forever, no proof of success, retries have no budget | **Loop** |
| Wrong specialist runs, no approval path, unreadable flow, no recovery route | **Graph** |

Do not blame the model for orchestration failures. A model cannot compensate reliably for stale state, ambiguous tool schemas, broken APIs, or missing exit conditions. Improve the layer that owns the failure.

### Expensive mistakes (steal these warnings)

These come almost straight from the same guide, and I have watched all of them in real projects:

**Building a graph before understanding the work.**  
Teams translate a business process into dozens of nodes before they have watched a capable agent solve it. Start with traces from a simpler harness. Formalize the stable paths.

**Letting the same model write and grade without safeguards.**  
Self-review can help, but it shares blind spots. Prefer deterministic checks where possible, separate reviewer context, and require humans for high-impact actions.

**Using "keep trying" as a loop specification.**  
Unbounded retry is a cost leak. Every loop needs a measurable objective, fresh evidence, max attempts, and a named escalation path.

**Treating the harness as a dumping ground.**  
More tools and memory are not automatically better. Crowded toolsets raise selection errors. Noisy context raises confusion. Broad permissions raise risk.

**Blaming the model for orchestration failures.**  
If the graph has no exits and the harness loses state, a better model will not save you.

---

## Mapping graph engineering onto kagent + substrate

Now the practical part: how do those abstract graph decisions show up as Kubernetes resources?

### The split of ownership

```
  You / UI / Git / CronJob
            │
            ▼
     kagent  — graph control plane
     (nodes, edges, models, MCP, HITL)
            │
            │  create / resume actor
            ▼
     Agent Substrate  — harness density
     (WorkerPool, gVisor, snapshots)
            │
            ▼
     Running actors
     (loops inside each node session)
```

- **kagent** owns topology: which agents exist, what they may call, which model they use, when a human must approve  
- **Substrate** owns dense, isolated execution: suspend idle sessions, restore sub-second, sandbox with gVisor  
- **Kubernetes** owns platform primitives: CRDs, CronJobs, GitOps, RBAC  

Put simply:

> **kagent makes the graph declarative. Substrate makes idle nodes cheap and isolated. Loops still live inside the nodes.**

### Graph decisions → concrete objects

Here is beamnxw's design list, translated into this stack:

**Node boundaries**  
A specialist is a `SandboxAgent` (declarative Go ADK on substrate) or an `AgentHarness` (coding backends like OpenClaw/Hermes). A deterministic step might stay outside the LLM entirely. A human step is `requireApproval` on a tool — or a separate operator-facing path. Do not make every box an LLM if a function or a human is the right node type.

**State schema**  
Prefer explicit memory, prompt templates, and session/ACP state over "the whole chat forever." Each node should know what it is allowed to read and what it is allowed to write. Parallel specialists need a merge story (orchestrator collects structured outputs; do not hope free-form prose merges cleanly).

**Routing conditions**  
In a declarative multi-agent setup on kagent, routing often lives in the orchestrator's system message *plus* the hard constraint of which agents appear as tools. That is weaker than a pure state-machine framework for complex branching — and that is fine if your edges are few and stable. For high-stakes branching, make the condition evidence-based in the prompt *and* keep the legal next agents limited to the tool list. Illegal edges simply do not exist.

**Concurrency**  
If two specialists can run without shared mutable state, you can fan out (sequentially via orchestrator today, or with explicit parallel patterns as your stack grows). Size the **WorkerPool** for peak concurrent *running* actors, not for total nodes in the graph.

**Cycles and exits**  
A research ↔ revise cycle is an edge back to writer/reviewer with a max iteration count in the orchestrator policy. Without a max and an escalation edge (human), you reinvent unbounded "keep trying."

**Durability**  
This is where substrate shines. Idle graph nodes do not need to hold pods. Actors checkpoint RAM + filesystem to object storage and resume when the next edge fires. Graph durability is "the session can survive interruption"; Kubernetes CronJobs handle "start this path on a schedule."

### Edges you can review in Git

In kagent, an edge is usually one of:

1. **Agent-as-tool** — orchestrator may call `research-node`  
2. **MCP tool binding** — node may call a named tool on a ToolServer  
3. **HITL gate** — tool requires human approval before it runs  

That is the difference between "the supervisor will figure it out" and "the allowed topology is in the CRD."

---

## Building the graph on this stack

### Install order (harness under the graph)

Install **Agent Substrate** first (`ate-system`), then **kagent** with substrate enabled, then a **WorkerPool**, then `ModelConfig`, then the agents that form your graph.

I will not re-paste every Helm flag — see the [kind guide](/articles/2026-06-25-kagent-agent-substrate-suspend-resume-kind/) and [code share](/articles/2026-07-13-kagent-oss-agent-substrate-kind-guide/). Constraint worth knowing early: substrate-backed declarative agents want the **Go** runtime today.

### Declare nodes as SandboxAgents

A node is an honest job description: system message, model, tools, pool.

```yaml
apiVersion: kagent.dev/v1alpha2
kind: SandboxAgent
metadata:
  name: research-node
  namespace: kagent
spec:
  type: Declarative
  description: Research specialist — structured facts only
  declarative:
    runtime: go
    modelConfig: default-model-config
    systemMessage: |
      You research a topic and return structured facts only.
      Prefer tools over guesswork. Cite sources when you can.
      Stop when the evidence is enough — do not invent certainty.
    tools:
      - type: McpServer
        mcpServer:
          name: kagent-tool-server
          kind: RemoteMCPServer
          toolNames:
            - k8s_get_resources
  platform: substrate
  substrate:
    workerPoolRef:
      name: kagent-default
```

Writer and reviewer nodes are the same shape with different prompts and tool allowlists. That *is* node boundary design.

### Draw edges with agent-as-tool

```yaml
apiVersion: kagent.dev/v1alpha2
kind: SandboxAgent
metadata:
  name: briefing-orchestrator
  namespace: kagent
spec:
  type: Declarative
  description: Research → draft → review for an industry briefing
  declarative:
    runtime: go
    modelConfig: default-model-config
    systemMessage: |
      You own the briefing pipeline.
      1) Call research-node. Require structured facts with sources.
      2) Call writer-node with those facts only.
      3) Call reviewer-node. If evidence fails, send feedback
         to writer-node at most twice, then escalate to a human.
      Never invent a specialist that is not in your tool list.
    tools:
      - type: Agent
        agent:
          name: research-node
          namespace: kagent
      - type: Agent
        agent:
          name: writer-node
          namespace: kagent
      - type: Agent
        agent:
          name: reviewer-node
          namespace: kagent
  platform: substrate
  substrate:
    workerPoolRef:
      name: kagent-default
```

Read that system message again as **graph policy**: sequence, evidence-based cycle with a max, escalation exit, and a hard tool list so illegal edges do not exist. That is loop design living inside a graph node — exactly the nesting the three-layer model describes.

### Human interrupts as real nodes

High-impact tools should not be free edges. With kagent, `requireApproval` pauses execution until a person decides. That is a human interrupt in graph terms — see [HITL on kagent](/articles/2026-03-11-human-in-the-loop-kagent/). Prefer it for publish, delete, deploy, and spend. Leave pure reads free so the graph does not drown in clicks.

### Schedules: CronJob as the missing trigger

Neither kagent nor substrate is primarily a calendar product. Kubernetes already is.

Keep nodes as CRDs (mostly idle, cheap on substrate). Use a `CronJob` to fire the orchestrator on a schedule. Substrate resumes the actor, the graph runs, the actor suspends again.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-briefing-trigger
  namespace: kagent
spec:
  schedule: "0 8 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: trigger
              image: curlimages/curl:8.5.0
              args:
                - -sS
                - -X
                - POST
                - http://kagent-controller.kagent.svc:8083/api/a2a/kagent/briefing-orchestrator/
                # Adjust path, body, and auth for your environment
```

Trigger from CronJob. Topology from CRDs. Density from substrate. No fourth control plane for "08:00."

### Why sparse graphs stay cheap

Most multi-agent graphs are sparse in *time*. Research runs, then sleeps. Writer runs, then sleeps. Reviewer runs for a moment. Orchestrator waits on a human.

Pod-per-agent pays for the worst case continuously. Substrate pays for concurrent **running** work:

1. Edge fires / request arrives  
2. Actor restores onto a free WorkerPool pod  
3. Node loop runs inside gVisor  
4. Idle → checkpoint to object storage → worker returns to the pool  

```bash
kubectl scale workerpool kagent-default -n kagent --replicas=5
```

Scale concurrency, not "number of boxes in the architecture diagram."

---

## Worked example: research-and-publish briefing

beamnxw uses a research-and-publishing agent as the running example of how layers nest. Here is the same story on kagent + substrate.

```
CronJob or user
      │
      ▼
[briefing-orchestrator]
      │
      ├─▶ [research-node]  ── MCP tools, structured facts
      │
      ├─▶ [writer-node]    ── draft from facts only
      │
      └─▶ [reviewer-node]  ── evidence checks
                │
         pass? ─┼─ yes → publish (optional HITL)
                │
                └─ no  → feedback to writer (max N)
                         then escalate to human
```

**Graph decisions in this design:**

- **Nodes:** orchestrator, research, writer, reviewer, human (on publish)  
- **Edges:** agent-as-tool refs; optional cycle writer ↔ reviewer; escalate edge  
- **State:** structured facts object, draft, review findings — not one blob of chat  
- **Routing:** reviewer evidence decides pass / revise / escalate  
- **Cycles:** max two revise loops, then human  
- **Durability:** each specialist session is a substrate actor; idle does not hold a pod  

**Loop decisions:** mostly inside reviewer — grade against sources, return compact feedback, stop on pass, budget, or escalate.

**Harness decisions:** gVisor isolation, tight MCP tool lists per node, OTel traces, snapshots so a long briefing can pause on HITL without burning a warm Deployment overnight.

If you only remember one sequencing rule from the article: **do not draw this whole graph on day one.** Start with a single agent and traces. Promote stable handoffs into CRDs when you see them repeating.

---

## Questions worth asking before you scale this

Before you grow the graph, sit with a few questions. They are adapted from the production checklist in the beamnxw guide — translated into this stack.

**On the graph:** Which paths must be deterministic? Where can work run in parallel? What state is shared, and who merges it? Where are the human gates and recovery routes? Could a teammate see the legal topology from the YAML alone?

**On the loops:** What evidence proves success for each node? What feedback comes back on failure? How many retries? What happens when the budget is gone?

**On the harness:** Are tools few, clear, and observable? Does state survive suspend/resume? Are permissions tight? Can someone pause, inspect, and resume a run without redeploying the world?

**On ops:** Can you replay a bad run and point at a CRD change that fixed it? Are you watching cost, latency, failure rate, and how often a human has to step in?

If you cannot answer the graph questions, a bigger model will not save you. If you cannot answer the harness questions, more nodes will not either.

---

## When you should *not* build a graph

If the job is "one agent, three tools, let it work," you do not need twelve nodes. Build a solid harness and a tight loop first.

Add a multi-node graph when you keep reimplementing the same branches, specialist handoffs, recovery paths, or human gates in glue code — and when those paths are stable enough that freezing them is a feature, not a trap.

Graph engineering is for control you can inspect. It is not a fashion statement.

---

## Bottom line

Steal the three-layer framing and use it on purpose:

- **Harness** — the environment that makes the model operable  
- **Loop** — the iterative process with evidence and stop rules  
- **Graph** — the explicit map of what may run next  

None of them replaces the others. A pretty graph with a broken harness still loses state. A great harness with no stop rules still burns money. Careful loops still turn to spaghetti when branching and approvals live only in ad-hoc code.

On Kubernetes, the split is clean enough to ship:

- **kagent** makes the graph (and a lot of the policy) declarative  
- **Agent Substrate** makes idle nodes cheap and isolated  
- **You** still design node boundaries, evidence, and exits  
- **CronJobs** handle "run this path at 8am" without inventing a new product  

Keep the topology in Git. Scale the WorkerPool for concurrent work, not for diagram vanity. Put humans on tools that can hurt.

---

## Where to start (call to action)

You do not need the full briefing pipeline on day one. Do this instead:

1. **Stand up the harness.** Install Agent Substrate, then kagent with substrate enabled, then a small WorkerPool. Follow the hands-on path:  
   [Suspend & resume on kind](/articles/2026-06-25-kagent-agent-substrate-suspend-resume-kind/) · [OSS substrate code share](/articles/2026-07-13-kagent-oss-agent-substrate-kind-guide/)

2. **Ship one node you trust.** One `SandboxAgent`, one model, a short tool list. Chat with it. Watch it suspend and resume. Get comfortable before you draw edges.

3. **Add edges only when traces demand them.** Promote repeated handoffs into agent-as-tool refs. Keep illegal specialists off the tool list so they cannot run by accident.

4. **Put a human on the dangerous tools.** Start with [HITL on kagent](/articles/2026-03-11-human-in-the-loop-kagent/). Approve deletes and deploys. Leave reads free.

5. **Put the YAML in Git.** Treat prompt, model, and tool changes like production config. [GitOps for agents](/articles/2026-04-02-gitops-for-agents-deployment-and-management/) is the longer version of that argument.

6. **Schedule later.** When you need calendar wakeups, add a CronJob that triggers the orchestrator. Do not wait for a native "agent cron" product.

Runnable demos live in [`sebbycorp/kagent-demos`](https://github.com/sebbycorp/kagent-demos/tree/main/01-kagent-agent-substrate). Upstream docs: [kagent.dev](https://kagent.dev) · [learn.agentsubstrate.dev](https://learn.agentsubstrate.dev/) · [github.com/kagent-dev/kagent](https://github.com/kagent-dev/kagent). The conceptual spine for this post is still [beamnxw's three-layer guide](https://x.com/beamnxw/status/2081022966645535079).

**Your move:** clone the demo, run one substrate-backed agent on kind this week, and only then decide which edges belong in the graph. Topology you have not earned from traces is just decoration.
