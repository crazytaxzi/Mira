# MIRA 2.0

Fresh clean slate for a persistent, voice-accessible cognitive agent.

MIRA is being rebuilt around `gpt-5.6-luna`, the OpenAI Responses API, a generic model-directed reason–act–observe loop, durable context and memory, scoped permissions, verified side effects, and the existing STT/TTS systems.

## Core rule

**Luna owns interpretation, planning, tool choice, recovery, and synthesis. Runtime code owns schemas, permissions, budgets, execution, persistence, verification, and audit.**

MIRA will not be implemented as an intent tree, keyword router, or collection of scripted domain workflows. The first milestone includes repeatable evaluations for ambiguous tasks, tool restraint, failure recovery, budget discipline, coherent continuation, and truthful completion.

## Current artifacts

- [Cognitive Agent Architecture and Delivery Plan](docs/MIRA_AGENTIC_CORE_PLAN.md)
- [MIRA Cognitive Kernel v0.1 implementation issue](https://github.com/crazytaxzi/Mira/issues/1)

## Current phase

The next bounded target is **MIRA Cognitive Kernel v0.1**:

- establish a measured Luna reasoning baseline;
- build one coherent Luna agent with persisted reasoning context;
- implement a compact situation packet and task ledger;
- add the generic reason–act–observe loop;
- provide strict tools and structured observations;
- enforce token, time, cost, permission, and side-effect boundaries;
- prove competence through repeated evals rather than one scripted demo.
