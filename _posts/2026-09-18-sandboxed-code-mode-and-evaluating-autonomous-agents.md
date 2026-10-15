---
layout: post
title: Sandboxed code mode, recursive LLM calls, and judges that read reasoning
description: Building Cortex's persistent-agent runtime with bubblewrap isolation, oneshot external_llm calls, and evaluation judges that grade how an agent thinks.
summary:
tags: [Agents, Sandboxing, Evals, TypeScript]
---
Standard tool-calling loops fall apart when an agent has to fan out across a fleet.

If you ask an agent to inspect 10 drones, take a photo on each, classify the anomaly, and clear the alarm, a standard ReAct loop takes dozens of serial turns. Every intermediate JSON payload lands in the conversation history, bloats the context window, and burns tokens re-reading old tool outputs.

Over the past year on Cortex, I built our persistent-agent runtime around a different model: **sandboxed code mode**, **recursive oneshot LLM sub-calls**, **unified wake signals**, and **evaluation judges that inspect operator reasoning**.

### Hardening code mode with bubblewrap and stdio JSON-RPC

Instead of calling one tool per turn, a code-mode agent writes a TypeScript or Python program (`execute_typescript` / `execute_python`) that loops, branches, and fans out concurrently.

Early on, local sandboxes ran as child processes under the pod user with scrubbed environment variables. That's not real isolation — `/proc/*/environ`, local sockets, and outbound network egress are still reachable if prompt injection lands in a tool response.

I hardened the runner using platform-native sandboxing (`bubblewrap` on Linux pods, `seatbelt` on macOS) with network access completely disabled (`allowNetwork: false`). To let sandboxed code still call MCP tools without opening any network sockets, the `external_*` tool bridge runs over **stdio JSON-RPC** pipes between the sandbox and the host process:

<figure class="diagram-figure">
  <a href="/assets/diagrams/cortex-agent-runtime.html" target="_blank">
    <img class="diagram-light" src="/assets/diagrams/cortex-agent-runtime.light.png" alt="Cortex Persistent-Agent and Code-Mode Runtime">
    <img class="diagram-dark" src="/assets/diagrams/cortex-agent-runtime.dark.png" alt="Cortex Persistent-Agent and Code-Mode Runtime">
  </a>
  <figcaption>
    <span>Cortex persistent-agent host with bubblewrap network isolation, stdio JSON-RPC tool mediation, MinIO image spill, and reasoning-aware strategy judge.</span>
    <a href="/assets/diagrams/cortex-agent-runtime.html" target="_blank">Open interactive diagram ↗</a>
  </figcaption>
</figure>

Inside the sandbox, code has pure compute and zero network exfiltration paths. And when an MCP tool returns a 5 MB camera frame, the host intercepts the base64 image before `JSON.stringify`, spills it to MinIO with a 7-day TTL, and rehydrates it just-in-time for a single model look-message so the agent's transcript never OOMs the host.

### `external_llm`: recursive judgment inside deterministic loops

Code mode is great at deterministic orchestration (`Promise.all`, retries, filtering), but terrible when step 2 of a loop needs fuzzy judgment — like reading a thermal anomaly report and deciding whether it's a false alarm. Without a bridge, the script has to exit and hand control back to the outer agent turn.

I added `external_llm`: a stateless, oneshot completion (no history, no tools, optional JSON schema) exposed directly inside `execute_typescript`:

```typescript
const photos = await parallel(events, 4, e =>
  external_take_photo({ event_id: e.id })
);

const verdicts = await parallel(photos, 3, p =>
  external_llm({
    prompt: `Thermal anomaly report: ${JSON.stringify(p.value.metadata)} — verdict?`,
    schema: {
      type: "object",
      properties: {
        verdict: { enum: ["confirmed", "false_alarm", "inconclusive"] }
      },
      required: ["verdict"]
    }
  })
);

await parallel(verdicts, 5, v =>
  external_override_resolution({
    event_id: v.id,
    verdict: v.value.object.verdict
  })
);
```

Deterministic TypeScript handles the fan-out and concurrency limits; the LLM is called only for the 50-token classification in the middle. Across our task-flow and persistent-agent workloads, keeping intermediate payloads out of the main loop cut median prompt tokens by **60–86%** and wall-clock time by over **80%**.

### One signal registry for waking sleeping agents

Persistent agents shouldn't spin in `while (true)` loops waiting for the world to change. Originally we had seven different wake mechanisms scattered across the codebase — cron heartbeats, MCP pollers, regex watchdogs, BullMQ wake queues, and peer messages — three of which were nearly identical polling loops with different schemas.

I consolidated them into a single `PersistentAgentSignal` registry (`trigger` doorbells, `drive` scorecards, and `timer` crons) backed by an append-only fire log. A single `signal-tick` worker claims due signals with `SELECT ... FOR UPDATE SKIP LOCKED`, runs the shared MCP invoker or `wake_when` regex matcher, and pushes wake events onto the agent's command channel. We also added `run_tool_when` — host-side reflex actions that fire a tool immediately when a watchdog matches, without waiting for an LLM turn to wake up.

### Letting the strategy judge read the operator's mind

To benchmark these agents, we run them against a 10-drone delivery-fleet simulator and grade every run with polymorphic `JudgeBinding` evaluators (ground-truth node scorers, LLM rubric judges, and classical greedy/Hungarian/auction solver baselines).

When I looked at why our **strategy judge** was producing shallow feedback, the bug was obvious: we asked the judge to evaluate the agent's *strategy*, but `buildRunDigest` only fed it the simulator's action log. Given only timestamps and tool calls, the judge just recited telemetry back to us:

> *"Recognized that thermal_anomaly events require drone_15 and routed all 9 events to it (t+334s, t+378s, t+452s...)"*

That's a list of clicks, not a critique of reasoning.

I updated the benchmark harness to extract the agent's persisted reasoning chunks and working-memory updates (`standing_rules`, `world_state`, `open_commitments`) and pass them to the judge alongside the action log, plus a `suggest_next_attempt` step. Once the judge could compare *what the agent thought was happening* against *what the simulator actually did*, it started catching real failure modes — like an agent formulating a solid dispatch rule in turn 3 and forgetting it after a context compaction in turn 12.
