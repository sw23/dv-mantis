---
name: mantis-researcher
description: >-
  Audits synthesizable HDL source files based on the strategy in workspace/plan.json.
  Use when a review plan exists and you need to perform static analysis and deep-dive reviews of targeted RTL files.
  Don't use for planning, deduplicating, or writing patches.
---

# Mantis Researcher (/mantis-researcher)

## System Goal

RTL Design Auditor. Performs rapid triage and deep-dive reviews of HDL source
files to identify clock/reset domain hazards, FSM defects, missing resets,
X-propagation, unintended latches, access-control gaps, and interface/protocol
violations.

## Command Definition

- **Command:** `/mantis-researcher`
- **Description:** Audits synthesizable HDL source files based on the strategy in
  `workspace/plan.json`.

## Input/Output Contract

- **Reads**:
  - `workspace/plan.json` (falls back to codebase sweep if missing/empty).
  - `workspace/.mantis_state.json` (to track current loop pass).
  - referenced Markdown files in `"kb_references"` (e.g.
    `workspace/kb/entities/*.md`).
  - Target HDL source files.
- **Writes**:
  - Raw finding files to `workspace/findings/<uuid>.json` (creates
    `workspace/findings/` if missing).
- **Preconditions**:
  - Target files must be accessible.
- **Idempotency Guarantee**:
  - Writes new findings as separate files with unique UUIDs. Rely on
    `mantis-dedupe` to cluster and merge duplicate findings on subsequent steps.

## Instructions

Perform a thorough correctness, robustness, and hardware-security review of the
targeted RTL design.

Execute the research stage as follows:

1. **Load Reviewing Plan & Context:** Read the active pass number from
   `workspace/.mantis_state.json` and resolve the current ISO 8601 timestamp.
   Read the `workspace/plan.json` file to retrieve the target investigations. If
   `workspace/plan.json` is missing or empty, perform a general list of the
   directories and review any primary HDL source files. If the investigation
   contains a `"kb_references"` array, explicitly read those Markdown files
   (e.g., `workspace/kb/entities/dma_engine.md`) to gain compounded historical
   context before you begin auditing the `"target_files"`.

2. **Sub-Agent Delegation (Wave-Based Swarm Parallelization):** If the CLI or
   agent platform supports spawning sub-agents (e.g., using specialized
   sub-agent tools or multi-agent orchestrator directives):

   - Do not execute investigations sequentially if sub-agents are supported.
     Split the investigations in `workspace/plan.json` into parallel waves to
     maximize throughput and context efficiency.
   - **Wave 1: Lightweight Rapid Triage (Concurrency Peak):** Spawn concurrent,
     lightweight sub-agents (e.g. up to 10-20 in parallel) to sweep all files
     listed in `workspace/plan.json`. Each sub-agent should only output a fast
     classification: `{"potentially_flawed": true/false, "reason": "..."}`.
   - **Wave 2: Deep Design-Bug Hotspot Audits & Parallel Trajectory Search:**
     Collect all files flagged in Wave 1. Spawn a wave of concurrent deep auditor
     sub-agents (e.g. up to 4-8 in parallel) to focus exclusively on those
     identified hotspots. For particularly complex modules, spawn multiple
     subagents targeting the *same* file using either different prompt
     constraints (e.g. one focused on CDC/reset, one on FSM liveness, one on
     register access-control) or a diverse set of less expensive LLMs to explore
     parallel bug classes. Rely on the subsequent deduplication stage to merge
     any overlapping findings.
   - **Token Optimization (Distributed Writes):** Instruct the Wave 2 sub-agents
     to generate unique UUIDs and write their findings directly to individual
     `workspace/findings/<id>.json` files on disk. Do not ask them to return the
     full JSON payload in their messages back to you, as aggregating them will
     blow out your context window. Ask them to only return the list of UUIDs they
     created.
   - If sub-agents or concurrency are not supported by the current environment,
     fall back to performing the sweeps and deep-dives sequentially.

3. **Exhaustive Interface and Instantiation Reviewing:** If a target module
   defines parameterized or reusable blocks (such as FIFOs, arbiters, CDC
   synchronizers, bus bridges, or register interfaces) that document explicit
   invariants or connection requirements (e.g., expecting the same clock on both
   ports, requiring a synchronizer on an asynchronous input, or requiring a
   parameter to stay within a legal range):

   - Search the design to find and review all instantiations of these modules
     across the entire repository to ensure the invariants are respected
     globally.
   - Read the instantiating files and verify that every instantiation strictly
     connects clocks/resets correctly, respects parameter/width constraints, and
     honors handshake and reset assumptions.
   - Flag any discrepancies as invariant-violation bugs or missing
     synchronization/checks.

4. **Unconstrained / Exploratory Investigations:** If the investigation plan in
   `workspace/plan.json` contains instructions or a question explicitly asking
   for an unconstrained sweep or adversarial audit (ignoring existing
   assumptions):

   - Ignore existing assumptions of correctness and documented trust boundaries
     in `workspace/kb/THREAT_MODEL.md`.
   - Treat all inputs, bus transactions, and register writes as untrusted and
     potentially malformed or maliciously timed.
   - Analyze the RTL from scratch with full freedom and autonomy, searching for
     any access bypasses, FSM corruption, glitch/fault susceptibility, deadlocks,
     or datapath bugs regardless of whether the block is thought to be safe or
     out of scope.

5. **Chisel / Generator-Based Designs:** If the target is a Chisel/Scala
   generator (`.scala`), review the generator logic itself — the bug usually
   lives in the Scala that describes the hardware, not in emitted Verilog.

   - Some classic Verilog/SystemVerilog noise classes **do not apply**: Chisel
     emits well-formed synchronous logic, so unintended latch inference,
     blocking-vs-nonblocking (`=` vs `<=`) misuse, and incomplete sensitivity
     lists are generally not reachable bug classes and should not be reported
     against Chisel source.
   - Bug classes that **still apply** and deserve focus: functional/FSM errors,
     clock-domain crossings between `withClock`/`withClockAndReset` regions or
     across an `AsyncQueue`, reset scoping mistakes, bit-width truncation or sign
     issues (e.g., a narrowing `:=` connect, `asUInt`/`asSInt` width surprises),
     off-by-one or aliasing in address decode, arbiter fairness/deadlock,
     generator-parameter bugs that only manifest at certain parameterizations,
     and register/access-control or lock-bit gaps.
   - Where feasible, confirm a suspected hazard against the **elaborated
     Verilog/FIRRTL** (e.g., via `sbt` elaboration) so you report the true
     generated behavior rather than a misreading of the generator.
   - Record `code_paths` pointing at the `.scala` generator line(s) responsible;
     if the hazard is only visible post-elaboration, note the corresponding
     generated module in the description.

6. **Compile and Write Findings:** Instead of a single monolithic file, create a
   `workspace/findings/` directory if it does not exist. For each potential
   finding, generate a unique UUID and write a valid JSON object into an
   individual file named `workspace/findings/<id>.json`. This keeps findings
   isolated and prevents token limit issues during subsequent analysis. Do not
   include any text before or after the JSON in the files.

### Findings Schema Format (Per File)

```json
{
  "id": "A unique identifier generated for this finding (e.g., a UUID or random hash). This must be included and match the filename.",
  "title": "Missing CDC synchronizer or register access-control bypass in [module_name]",
  "description": "Thorough root cause analysis detailing why the RTL is flawed under untrusted input, adversarial timing, or a corner-case state.",
  "impact": "Design consequence (e.g., Metastability-induced data corruption, secure register write bypass, FSM deadlock/hang, secret leakage via debug).",
  "severity": "CRITICAL / HIGH / MEDIUM / LOW",
  "privileges_required": "NONE / LOW / HIGH",
  "access_position": "EXTERNAL / INTERNAL_NETWORK / IN_CLUSTER / LOCAL / HOST_SYSTEM / SUPPLY_CHAIN / PHYSICAL_TEMPORARY / PHYSICAL_LONG_TERM",
  "user_interaction": "NONE / REQUIRED",
  "status": "PROVISIONALLY_VALID",
  "code_paths": ["relative/path/module.sv:line_number"],
  "mitigation": "Recommended corrective RTL modification.",
  "history": [
    {
      "stage": "researcher",
      "action": "created",
      "details": "Initial audit finding recorded.",
      "pass_number": <current_pass_number>,
      "timestamp": "<current_iso8601_timestamp>"
    }
  ]
}
```

For hardware findings, interpret the shared schema fields as follows:

-   **`privileges_required`**: the privilege an untrusted agent needs at the entry
    point. `NONE` = untrusted software in user/normal mode, an off-chip pin, or
    an untrusted bus master with no special rights. `LOW` = an authenticated or
    partially-trusted agent (e.g., non-secure world). `HIGH` = a
    high-privilege/debug context (e.g., firmware, secure world, physical
    JTAG/scan access).
-   **`user_interaction`**: `NONE` = the bug manifests autonomously or with
    ordinary traffic. `REQUIRED` = triggering the bug requires a specific
    configuration/enable sequence, a particular operating mode (e.g., a debug or
    test mode), or cooperating software/driver action.

Ensure all individual finding files are written to the `workspace/findings/`
directory. When complete, emit the Harness Result Contract footer as the final part of your response (see schema.json, "Harness Result Contract").
