---
name: mantis-reproduce
description: >-
  Generates and runs testbenches, assertions, or formal properties to verify RTL design bugs.
  Use when viable findings exist and you need to write and execute a simulation or formal proof to confirm the bug.
  Don't use for RTL auditing or patching.
---

# Reproducer (/mantis-reproduce)

## System Goal

Verification Engineer. Designs testbenches, SystemVerilog Assertions (SVA), or
formal properties and executes them inside isolated simulation/formal
environments to empirically verify design bugs.

## Command Definition

- **Command:**
  `/mantis-reproduce [--reverify] [--finding_id=<uuid>] [--force] [--target_root=<path>] [--state_root=<path>]`
- **Description:** Generates and runs testbenches, SVA properties, or formal
  proofs to recreate RTL design bugs. Recreation is **best-effort**:
  constructing a module-level testbench or formal property often requires
  design-specific domain knowledge, so a failure to recreate the bug must never
  block the finding from advancing to later stages.
- **Parameters:**
  - `--reverify`: When executing as part of patch verification to isolate
    re-verification outcomes.
  - `--finding_id`: The specific finding UUID to reproduce. **Must** be provided
    and is required when `--reverify` is specified.
  - `--force`: Override/bypass eligibility checks for targeted normal runs.
  - `--target_root`: Path to the root of the target RTL codebase under test
    (defaults to `.`).
  - `--state_root`: Path to the root of the Mantis state directory containing
    `workspace/` (defaults to `.`).

## Input/Output Contract

- **Reads**:
  - `state_root/workspace/findings/` (viable/conditional findings).
  - `target_root/` (Repository HDL source files to analyze trigger paths).
  - `state_root/workspace/archive/.repro_attempts.json`.
  - `state_root/workspace/.mantis_state.json` (to track current loop pass).
- **Writes**:
  - PoC reproduction files (e.g. `tb_[uuid].sv` or a formal property file inside
    `state_root/workspace/reproducers/`).
  - If run normally: updates findings in-place under
    `state_root/workspace/findings/` (sets `"repro_status"`,
    `"repro_file_path"`, `"run_command"`, `"repro_output"`, and appends
    history). Updates status to `"VALID"` if provisionally valid.
  - If run with `--reverify`: updates findings in-place under
    `state_root/workspace/findings/` (sets `"reverify_status"`,
    `"reverify_file_path"`, `"reverify_run_command"`, `"reverify_output"`, and
    appends history with stage `"reverify"`). Does not modify `"repro_*"` fields
    or `"status"`.
  - Updates `state_root/workspace/archive/.repro_attempts.json` atomically.
- **Preconditions**:
  - Findings must exist in `state_root/workspace/findings/`.
  - Simulation/formal runtime environment must be available.
- **Idempotency Guarantee**:
  - Updates findings in place. Uses
    `state_root/workspace/archive/.repro_attempts.lock` file locking and atomic
    temporary file swaps (`os.replace` on
    `state_root/workspace/archive/.repro_attempts.json.tmp`) to guarantee
    concurrency safety and retry stability.

## Instructions

Write a reproducer — a self-contained module-level testbench, an assertion, or a
formal property — that demonstrates a confirmed RTL design bug.

This stage is **best-effort and non-blocking**: a module-level testbench (e.g.
cocotb or chiseltest), an SVA property, or a formal proof usually requires
design-specific domain knowledge to construct and exercise. Make a genuine
attempt, but if you cannot build a triggering harness, classify the outcome
honestly (e.g. `failed_to_reproduce` or `not_attempted`), record what you tried,
and let the finding proceed. Never halt the pipeline or discard a finding solely
because it could not be recreated here.

Execute the reproduction stage under these constraints:

1. **Load Viable Findings:**

   - If `--finding_id` is supplied:
     - Load only that finding's file
       (`state_root/workspace/findings/<uuid>.json`). Exit if it does not exist.
     - If `--reverify` is specified: Enforce the **expected patch workflow
       state** for the loaded finding:
       - The finding's `"status"` must be `"VALID"` or `"PROVISIONALLY_VALID"`.
       - The finding's `"repro_status"` must be `"reproduced"`.
       - The finding's `"patch_status"` must NOT be `"VERIFIED_SECURE"` or
         `"MITIGATION_PROPOSED"`.
       - Exit with an error if these conditions are not met, explaining the
         invalid state.
     - If `--reverify` is NOT specified (Targeted Normal Run):
       - If `--force` is NOT specified, enforce standard eligibility filters:
         - The finding's `"status"` must be `"VALID"` or
           `"PROVISIONALLY_VALID"`.
         - The finding's `"production_viability"` must be `"VIABLE"`,
           `"SAMPLE_OR_TEST"`, or `"CONDITIONAL_VIABLE"`.
         - Exit with an error if these conditions are not met, explaining the
           invalid state.
       - If `--force` is specified, bypass these eligibility checks.
   - If `--finding_id` is not supplied:
     - **Constraint:** Exit if `--reverify` is specified (it requires
       `--finding_id`).
     - Read the JSON files in the `state_root/workspace/findings/` directory.
     - **Strict Eligibility Filter (Normal Runs):** Include only findings where:
       - `"status"` is `"VALID"` or `"PROVISIONALLY_VALID"`.
       - `"production_viability"` is `"VIABLE"`, `"SAMPLE_OR_TEST"`, or
         `"CONDITIONAL_VIABLE"` (or skip this viability filter if not checking
         viability, but always check status).
     - If no applicable findings exist, emit a result footer with
       `"status": "skipped"` and stop (see schema.json, "Harness Result
       Contract").

2. **Strict Host Isolation Constraint:**

   - Host command execution outside your controlled tooling is strictly
     prohibited. Do not run arbitrary commands on your parent host terminal.
   - All reproducer executions must run in an isolated simulation or formal
     environment (e.g., a container or scratch working directory). Never point a
     generated stimulus at real production silicon, a shared lab bench, or
     networked test equipment without explicit authorization. Simulations and
     formal runs are self-contained; keep them that way.

3. **Writing and Launching the Reproducer:** Write a self-contained testbench
   (e.g., a SystemVerilog/Verilog `tb_[uuid].sv`, a Verilator C++ harness, a
   cocotb Python test, or — for a Chisel DUT — a **chiseltest**/ScalaTest test
   driving Treadle or a Verilator backend), an SVA property bound to the DUT, or
   a formal property file that triggers the target bug. **All generated
   reproducer and re-verification harnesses MUST be written inside the
   `state_root/workspace/reproducers/` directory (never in the `target_root`
   directory).** You must ensure the parent directory
   `state_root/workspace/reproducers/` exists (e.g. using `mkdir -p`) before
   writing any files. Analyze the RTL path and its preconditions carefully. If
   your initial reproduction attempt fails, evaluate whether the finding details
   (such as signal names, register sequences, clock/reset relationships, or
   state assumptions) are slightly incorrect based on your observations, and
   adjust the finding details dynamically to attempt a fix. If you cannot find a
   triggerable path after trying multiple approaches and adjustments, abandon the
   attempt and mark it as `failed_to_reproduce`.

   To run your reproducer, use the simulation or formal tools available in your
   environment (e.g., Verilator, Icarus Verilog (`iverilog`), Questa/ModelSim
   (`vsim`), VCS, Xcelium, SymbiYosys/`sby`, or JasperGold; for Chisel, `sbt
   test` with chiseltest/Treadle). Select the most appropriate tool and flags for
   the target. **All elaboration, compilation, and simulation/formal commands
   MUST be run with current working directory (Cwd) set to `target_root`.**
   Resolve and use the absolute path of the generated reproducer (e.g. using
   Python's `os.path.abspath` on
   `state_root/workspace/reproducers/tb_[uuid].sv`) when generating and storing
   the `"run_command"` or `"reverify_run_command"`. **Match the tool to the
   artifact:** for a Chisel generator, either write a chiseltest test against the
   Chisel module directly, or elaborate it to Verilog with `sbt` and drive the
   generated RTL with a Verilog simulator; if only a gate-level or post-synthesis
   netlist is available, run a gate-level simulation; if an FPGA/emulation
   prototype is available, you may exercise it directly. Use your best judgment to
   construct a working harness for the artifact.

   - *Optional Parallel Trajectory Search:* If your environment or agent
     framework supports spawning subagents, you can deploy multiple concurrent
     agents to attempt the reproducer via different approaches (e.g., a directed
     testbench, a constrained-random test, and a formal property). If any
     trajectory succeeds, immediately adopt its stimulus and discard the others
     to escape potential "give up" loops.

   - **Reproduction Status Classification:**

     - **`reproduced`**: The testbench/property successfully demonstrated the bug
       (an assertion fired, an `X` reached a live output, a mismatch vs.
       reference occurred, or formal returned a counterexample).
     - **`failed_to_reproduce`**: The reproducer was executed but did not trigger
       the bug.
     - **`statically_confirmed`**: Simulation/formal was impossible due to
       environmental constraints (e.g., missing library/technology cells, an
       unavailable reference model or vendor tool) but the flaw is statically
       obvious (e.g., an asynchronous signal sampled with no synchronizer, a
       hardcoded key). This is strongly discouraged and should only be used as a
       last resort.
     - **`not_attempted`**: The reproduction stage was skipped entirely (e.g.,
       due to tool setup failure, timeouts, or explicit skip configuration).

4. **Strict Interface & Internal-Invariant Constraints:**

   - Your reproducer should drive the DUT through its real top-level ports and
     bus interfaces (e.g., AXI/AHB/APB transactions, legal register writes)
     wherever possible, or strictly respect the design's global execution
     invariants, to avoid generating artificial, non-viable failures.
   - Do not declare a finding as "reproduced" if the failure can only be achieved
     by using `force`/`release` or direct hierarchical pokes to set an internal
     signal to a value the real interface can never produce (e.g., forcing a
     one-hot FSM into an illegal multi-hot encoding that no legal input sequence
     can reach), or by violating a documented timing/reset contract that the
     surrounding logic guarantees.
   - If a bug cannot be triggered through the real interface or under legal
     operating conditions, classify the finding as `"failed_to_reproduce"` due to
     "Internal Invariant Protection."

5. **Failure-Aware Validation:** Analyze the simulator/formal output (logs,
   assertion reports, waveforms, exit status) to classify reproduction success
   depending on the bug class:

   - **Control / Access / Protocol Bugs:** A successful reproducer is a directed
     test or property that explicitly demonstrates the failure (e.g., a write to
     a locked register succeeds, an FSM enters an illegal state, a bus
     transaction violates the protocol handshake, or a secure asset is read from
     a non-secure context).
   - **Datapath / Timing / Structural Failures:** Treat the reproduction as
     `"reproduced"` if the run exhibits a concrete failure signal. Check for:
     - SVA assertion failures or `$error`/`$fatal` messages.
     - `X` (unknown) propagation onto a live output or control signal that a real
       reset would not clear.
     - A functional mismatch against a golden/reference model.
     - A formal counterexample (CEX) trace.
     - A deadlock/livelock or watchdog timeout with no forward progress.

6. **Token-Optimized File Updates:** To minimize LLM output tokens, **do not
   re-emit or manually rewrite the entire JSON object in your output.** Instead,
   use in-place editing tools (like a short script in your preferred language, or
   `jq`) to programmatically append the new fields to the existing
   `state_root/workspace/findings/<id>.json` file.

   Additionally, you must **Update the Reproduction Attempt Cache** to help the
   planner track attempts efficiently:

   - Maintain a JSON cache file at
     `state_root/workspace/archive/.repro_attempts.json`. Ensure the parent
     directory `state_root/workspace/archive/` exists (e.g.,
     `mkdir -p state_root/workspace/archive/`) before creating, reading, or
     locking the cache file.
   - Key the cache by a stable identifier that persists across loop runs even if
     UUIDs are regenerated. Use a combination of the finding's normalized title
     and its primary file path:
     `stable_key = normalized_title + "@" + primary_file_path`.
     - Compute `normalized_title` by converting the title to lowercase and
       removing all non-alphanumeric characters.
     - Compute `primary_file_path` by taking the first entry in `code_paths` and
       stripping any line number suffixes (e.g., converting
       `rtl/dma_engine.sv:120` to `rtl/dma_engine.sv`).
   - To prevent race conditions during concurrent executions (including locking
     bypasses caused by atomic file replacement) and protect lockless readers:
     - Use a separate dedicated lock file
       `state_root/workspace/archive/.repro_attempts.lock` which is never deleted
       or replaced.
     - Perform updates atomically using Python's `fcntl.flock` on this lock file:
       1. Open the lock file
          `state_root/workspace/archive/.repro_attempts.lock` (creating it if
          missing) and acquire an exclusive lock (`fcntl.flock` with
          `fcntl.LOCK_EX`) inside a context manager (`with` statement).
       1. Read the current contents of the cache file
          `state_root/workspace/archive/.repro_attempts.json` (treating it as
          `{}` if missing or empty).
       1. Increment the integer value for this finding's `stable_key` by 1.
       1. Write the updated JSON to a temporary file in the same directory (e.g.,
          `state_root/workspace/archive/.repro_attempts.json.tmp`).
       1. Atomically replace the target cache file with the temporary file (e.g.,
          `os.replace` in Python) to ensure readers never see a truncated or
          incomplete file.
       1. Close the lock file descriptor to release the lock (automatically
          handled by exiting the `with` context manager).

   Depending on whether the `--reverify` flag is provided:

   - **If run normally (no `--reverify` flag):** You must append or update the
     following on the existing object:

     - `"repro_status"` (`"reproduced"`, `"statically_confirmed"`,
       `"not_attempted"`, or `"failed_to_reproduce"`).

     - `"repro_file_path"` (path to the testbench/assertion/property).

     - `"run_command"` (the exact simulator/formal invocation).

     - `"repro_output"` (the relevant log/assertion/CEX output).

     - If reproduction succeeds (`repro_status` is evaluated as `"reproduced"` or
       `"statically_confirmed"`) and the finding's current `"status"` is
       `"PROVISIONALLY_VALID"`, you **must** update `"status"` to `"VALID"`.

     - An entry to the `"history"` array:

     ```json
     {
       "stage": "reproduce",
       "action": "reproduced",
       "details": "Reproduction status evaluated as [reproduced/failed_to_reproduce] using command: [run_command]",
       "pass_number": <current_pass_number>,
       "timestamp": "<current_iso8601_timestamp>"
     }
     ```

   - **If run with `--reverify`:** You must append or update the following on the
     existing object (do not touch `repro_*` or `status`):

     - `"reverify_status"` (`"bug_persists"`, `"bug_resolved"`).

     - `"bug_persists"`: The new/modified reproducer still triggers the bug on
       the patched RTL (the fix did not fully resolve it).

     - `"bug_resolved"`: The reproducer was run but the patched RTL no longer
       triggers the bug.

     - `"reverify_file_path"`

     - `"reverify_run_command"`

     - `"reverify_output"`

     - An entry to the `"history"` array:

     ```json
     {
       "stage": "reverify",
       "action": "reproduced",
       "details": "Re-verify status evaluated as [bug_persists/bug_resolved] using command: [reverify_run_command]",
       "pass_number": <current_pass_number>,
       "timestamp": "<current_iso8601_timestamp>"
     }
     ```

7. **Criticism of Reproduction Validity:** To ensure the reproduction is a valid
   demonstration of the reported design bug, have a subagent with a fresh context
   window review and criticize the generated reproducer — checking that it drives
   the DUT legally and does not manufacture the failure via illegal forces or
   contract violations. Seek genuine criticism to ensure false reports are never
   surfaced later.

When complete, emit the Harness Result Contract footer as the final part of your response (see schema.json, "Harness Result Contract").
