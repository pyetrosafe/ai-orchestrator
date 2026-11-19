---
description: Starts the orchestration cycle (Cloud plans and audits; Local executes microtasks)
agent: orchestrator
---
Analyze the request below and execute it using the Cloud + Local orchestration workflow:

$ARGUMENTS

Remember:
1. Briefly present the plan divided into tasks, subtasks, and microtasks (`T1.1.1`, `T1.1.2`, `T2.1.1`, ...) before or when starting the first delegation.
2. A task represents a larger group of actions, a subtask is a specific action within a task, and a microtask is the smallest unit of work that can be delegated to `local-worker`, for example, "write a paragraph", "generate an item list", "create a simple diagram", "fix a syntax error", etc.
3. Delegate **only one microtask at a time** to `local-worker`, requiring the standardized response contract and a record in `.opencode/state/last-result.md`.
4. Audit every response: if it fails, send a surgical correction; if it passes, advance to the next microtask until everything is complete.
5. Instructions:
   5.1. Subagent Invocation Protocol - Communication with `local-worker` must always follow the tool's explicit invocation format. Never invoke `subagent` without the `description` argument. The correct technical format is: `subagent(agent: <name>, description: <summary>, prompt: <instructions>)`. The `description` field is required agent metadata, even if the Orchestrator must obtain this value from the configuration file.