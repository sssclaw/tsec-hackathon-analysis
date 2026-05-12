# BreachWeave Code & Architecture Analysis

## Overview

BreachWeave is the open‑source implementation of the winning team (**ai 小分队**) in the second Tencent Cloud Intelligent Penetration Hackathon.  It is published as a Bun monorepo with several packages and applications built on top of the **pi‑coding‑agent** SDK.  The repository emphasises a layered architecture and a minimal core:  

* **Apps**: `apps/cli` provides a command‑line interface and routes into either a web UI (`ui-web`) or a terminal UI (`ui-tui`).  This acts as the entry point for running penetration challenges.  
* **Packages**: `packages/core` contains the bulk of the harness logic, including configuration management, challenge orchestration, solver tasks and observers.  `packages/ui-web` implements a web UI and REST API; `packages/ui-tui` provides an interactive terminal UI.  The code is written in TypeScript and compiled via [Bun](https://bun.sh).  
* **Dependency graph**: a strict one‑directional dependency graph ensures that UI packages depend on core, which depends on configuration and the underlying pi‑coding‑agent.  This encourages modularity and prevents circular dependencies【768107471180543†L4-L21】.  
* **Configuration**: user preferences, prompts, skills and tool definitions are stored under `~/.tch-agent/config/` and loaded by the `ConfigManager` class.  The manager can list available prompts, tools or provider settings【768107471180543†L54-L67】.

## Manager (`manager.ts`)

The **manager** is responsible for global orchestration.  Key elements of its implementation include:

### Planning tools

The manager registers a set of **planner tools** (e.g., `planner_get_state`, `planner_start_challenge`, `planner_launch_solver`, `planner_stop_solver`, `planner_stop_challenge`).  These functions expose high‑level operations that the planner LLM may invoke.  For example, `planner_get_state` returns a structured snapshot of all active challenges, solvers, hints, memory summaries and timeouts; `planner_launch_solver` instantiates a new solver agent for a given task; and `planner_stop_solver` gracefully terminates a solver.  Exposing only these functions constrains the planner and prevents arbitrary code execution.

### State snapshot & planning loop

To decide the next action, the manager constructs a snapshot of the current world state.  This includes:

* A digest summarising the content of each solver’s memory and ideas (the digest is hashed to detect staleness).
* Timing information (e.g., how long a solver has been running, time since last hint, timeouts).
* Hints from the competition API and the current challenge status.

This snapshot is passed into a planner agent session (via pi‑coding‑agent).  The planner, guided by a prompt, analyses the state and uses the planner tools to issue instructions.  The manager executes these instructions synchronously or asynchronously.  This **Ralph loop** continues until the planner indicates that no further actions are needed or a time limit is reached.  Externalising the stop condition prevents the LLM from halting prematurely.

### Memory management & rewriting

The manager enforces limits on solver memory size and implements multi‑pass rewrites to compress context.  When memory grows too large, it invokes a summarisation routine that rewrites the history into shorter, high‑quality summaries (known as **RTK rewrite**).  Separate stores for **memory** (technical logs) and **ideas** (high‑level reasoning) allow the manager to produce concise state snapshots for the planner while preserving details for solver queries【738928078656857†L28-L99】.

### Concurrency and fairness

Constants defined at the top of `manager.ts` control maximum concurrent solvers, solver stale timeouts and fairness scheduling.  The manager ensures that solvers do not hog resources and will stop idle or redundant solvers if necessary.  It also monitors API rate limits and sets delays between hint requests to avoid being blocked by the challenge API.

### API interactions

The manager wraps the official challenge API through functions like `startChallenge`, `stopChallenge`, `submitFlag` and `getHints`.  These functions operate in both **mock** mode (for local testing) and **real** mode when connected to the competition server.  They handle retries, error logging and update the local state accordingly.

### Observability & logging

Extensive logging functions are defined to format solver tables, challenge summaries and hint histories.  This helps users inspect the internal state during runs.  The manager also writes logs to disk for later analysis.  Observers (discussed below) feed additional warnings into these logs.

## Solver

Each **solver** agent is responsible for executing a penetration task assigned by the manager.  The solver pipeline comprises the following stages:

1. **Task intake:** the solver receives a task that includes the challenge URL, current memory summary, previous ideas and any hints.  It uses these to plan its next actions.  
2. **Tool execution:** solvers can call a suite of tools—HTTP client, terminal commands, vulnerability scanners, payload encoders, etc.—provided by the harness.  Tool outputs are appended to the memory store.  When new vulnerabilities are discovered, the solver generates payloads and calls the flag submission tool.  
3. **Reasoning & ideation:** after each tool call, the solver records high‑level thoughts in the ideas store.  This separation ensures that the planner receives concise reasoning without being overwhelmed by raw output.  
4. **Summarisation:** periodically, the solver summarises both memory and ideas, applying multiple passes of rewriting to stay within the context window.  These summaries are then used by the manager when constructing the planning snapshot.  
5. **Termination:** when the solver believes it has extracted a flag or made no progress for some time, it informs the manager through the submission tool or via its logs.  The manager then decides whether to stop or restart the solver.

## Observer

The **observer** is a lightweight monitor that inspects solver trajectories.  It looks for indications of drift (e.g., repeating the same commands without progress), context bloat or suspicious behaviour.  The observer does not modify solver actions directly but logs warnings that the manager can use to decide whether to intervene.  Observers help maintain safety and prevent runaway loops.

## Data flow summary

The high‑level data flow within BreachWeave is as follows:

1. **Configuration** defines available prompts, tools and provider settings.
2. **User or API** triggers `manager.startChallenge`, which requests a challenge from the API and registers it in the manager’s state.
3. The **manager** enters a **planning loop**, constructing a snapshot of the world and invoking the planner LLM via pi‑coding‑agent.  
4. The **planner** issues high‑level instructions using planner tools.  Most commonly, it instructs the manager to launch a solver with a specified task.
5. A **solver** instance executes the task, calling harness tools to interact with the target application.  It records outputs in memory and ideas and periodically summarises its state.
6. The **observer** monitors the solver’s actions and logs any anomalies.  
7. The **manager** updates the world snapshot with solver results and continues the planning loop, possibly launching new solvers, stopping idle ones or submitting flags.  
8. The loop continues until the challenge is solved or the time limit is reached, after which `manager.stopChallenge` cleans up state and logs.

## Strengths & Potential Improvements

**Strengths**:

* **Decoupling of roles** allows independent development and debugging of manager, solver and observer components.
* **Planner tools** encapsulate operations in a safe, structured interface, reducing the risk of prompt injection or arbitrary code execution.
* **Context compression** through RTK rewrite enables long runs despite limited context windows.
* **Layered memory** (ideas vs. memory) provides rich information for decision making while keeping the LLM input concise.
* **Externalised termination** via the Ralph loop prevents premature halting and ensures the planner continually reevaluates state until completion.

**Potential Improvements**:

* **Dynamic tool selection:** currently, available tools are mostly fixed.  Integrating dynamic tool discovery or user‑provided extensions could enhance versatility.
* **Adaptive model selection:** although multiple models are run in parallel, the mechanism for choosing which model’s suggestion to follow is simple.  A more principled voting or ranking method could improve robustness.
* **Improved observer feedback:** the observer only logs anomalies.  Incorporating a feedback mechanism where the observer’s warnings can directly influence planner prompts may accelerate recovery from drift.
* **Fine‑grained monitoring of resource usage:** implementing metrics for CPU, memory and network usage per solver could improve fairness and prevent resource exhaustion.

In conclusion, BreachWeave provides a comprehensive, well‑engineered harness that balances autonomy with safety.  Its architecture offers valuable insights for anyone seeking to build agents capable of tackling long‑range tasks such as penetration testing or complex software development.
