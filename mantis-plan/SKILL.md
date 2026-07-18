---
name: mantis-plan
description: >-
  Formulates a targeted RTL review plan based on the active threat model and historical learnings.
  Use when starting a design review campaign to map the design boundaries and generate a roadmap (workspace/plan.json).
  Don't use for executing RTL reviews, writing testbenches, or patching HDL.
---

# Strategist (/mantis-plan)

## System Goal

Verification Architect. Analyzes design structure, directory metadata, and
historical records to map the exposure/interface surface and formulate an adaptive
review roadmap.

## Command Definition

- **Command:** `/mantis-plan`
- **Description:** Formulates a targeted RTL design review plan based on the
  active threat model and historical learnings.

## Input/Output Contract

- **Reads**:
  - `workspace/.mantis_state.json` (to track current loop pass).
  - `workspace/kb/THREAT_MODEL.md` (if exists).
  - `workspace/kb/index.md` (checks existence to determine Mode A vs B).
  - Mode A: traverses design directories and RTL source files, reads
    `mantis-summary.md` (if available).
  - Mode B: reads `workspace/kb/index.md`, `workspace/kb/THREAT_MODEL.md`,
    `workspace/archive/.repro_attempts.json` (if exists), VCS diffs or file
    timestamps/hashes.
- **Writes**:
  - `workspace/plan.json`.
  - Copies retry-eligible finding JSON files from
    `workspace/archive/findings_pass_K/` or `workspace/archive/loopK_findings/`
    (where K is the pass it was archived in) to `workspace/findings/`
    (preserving their original UUID filenames).
- **Preconditions**:
  - Design must be accessible.
- **Idempotency Guarantee**:
  - Overwrites `workspace/plan.json` directly. In Mode B, consults
    `.repro_attempts.json` and skips rescheduling already processed findings
    unless the target files have been modified in the current loop.

## Instructions

Analyze the design structure and create a detailed RTL review plan that avoids
duplication of prior efforts while digging deep into complex cross-module paths,
clock/reset domain boundaries, and un-scanned design areas.

> **Target Agnosticism Directive:** The target you are evaluating may be RTL
> source (Verilog, SystemVerilog, VHDL), a Chisel/Scala generator (`.scala` that
> elaborates to Verilog/FIRRTL, common in open-source designs), a
> gate-level/post-synthesis netlist, an encrypted or third-party IP block, or a
> live FPGA/emulation prototype. Ground your planning in whatever format the
> target is currently in. You are authorized and encouraged to use whatever
> suitable tools are at your disposal (e.g., standard Unix tools, HDL linters like
> Verilator `--lint-only` or Verible, `sbt`/chiseltest to elaborate Chisel,
> `yosys` for structural exploration, or elaboration/hierarchy dumps) to explore
> the design structure. For a Chisel project, plan investigations over the Scala
> generator sources (and, where available, the elaborated Verilog). If RTL source
> is not available, do not attempt to force an RTL-source workflow (e.g. searching
> only for `.sv` files); adapt and 'do what works' for the artifact at hand (e.g.,
> a synthesized netlist).

Execute the planning stage as follows:

1. **Check for Threat Model Context:** Check the knowledge base directory for a
   `workspace/kb/THREAT_MODEL.md` file. If it exists, read it completely to
   understand the design's official security boundaries, threat actors, assets,
   high-risk interfaces, and trusted inputs.

2. **Determine Mode & Retrieve Learnings:** Check if the knowledge base index
   `workspace/kb/index.md` exists.

   - **MODE A: First-Pass Exhaustive Mode (No `workspace/kb/index.md` found):**
     If this is the first run, guarantee complete coverage of the design. To
     avoid hitting output token limits on large designs, do not generate the
     `workspace/plan.json` manually in your text response. Instead, execute a
     shell command to run a short script in your preferred language that:

     1. Uses `find` or `os.walk` to crawl all synthesizable design directories.
        If a `mantis-summary.md` file exists in a directory, use its contents to
        understand the directory structure instead of reading every individual
        HDL file. Otherwise, crawl all RTL source files (e.g., `.v`, `.sv`,
        `.vh`, `.svh`, `.vhd`, `.vhdl`, and Chisel generator sources `.scala`).
     2. Ignores testbench/verification folders, build/simulation artifacts, and
        out-of-scope vendor IP (e.g., `sim/`, `tb/`, generated netlist dumps,
        `.git`).
     3. Programmatically formats the list into the `workspace/plan.json` schema
        and writes it directly to disk. Because this is an automated script,
        instruct it to use a generic, overarching baseline question for the
        `"question"` field (e.g., "Conduct a baseline audit for CDC/RDC issues,
        reset correctness, FSM liveness, latch inference, X-prop, and register
        access-control flaws"), reserving highly contextual custom questions for
        Mode B.

   - **MODE B: Strategic Learning Mode (`workspace/kb/index.md` exists):** Read
     `workspace/kb/index.md` and `workspace/kb/THREAT_MODEL.md` to review the
     compounded historical knowledge of the design, including trust boundaries,
     bug classes, and microarchitectural components. Adapt your focus to design
     new, targeted deep dives and regression reviews for modules and files that
     have histories of design bugs. You may generate the `workspace/plan.json`
     manually using your file-writing tools for this mode, as the scope will be
     much narrower.

     - **Targeted Re-Evaluation & Retries**: Review the KB index, entity files,
       and the reproduction attempt cache file
       (`workspace/archive/.repro_attempts.json` if it exists). You must
       identify findings that need re-evaluation or retries:

       1. **Schedule for Research:** For findings in the archive marked
          `"NEEDS_RESEARCH"`, schedule a targeted investigation in
          `workspace/plan.json` (to gather missing context and resolve them to
          `"VALID"` or `"FALSE_POSITIVE"`).

       2. **Copy for Retry (Bypass Research):** For findings in the archive
          marked `"VALID"` or `"PROVISIONALLY_VALID"` that are eligible for
          retry, do **not** schedule them in `workspace/plan.json`. Instead,
          **copy the finding JSON files directly from the archive (e.g.,
          `workspace/archive/findings_pass_K/<id>.json` or
          `workspace/archive/loopK_findings/<id>.json`, where K is the pass it
          was archived in) back to `workspace/findings/<id>.json`** (preserving
          their UUIDs). A finding is eligible for retry if:

          - The reproduction status (`repro_status`) is `"not_attempted"` (or
            missing), OR
          - Reproduction previously failed (`"failed_to_reproduce"`) but has had
            **fewer than 2 total reproduction attempts** (check the attempt
            count in `workspace/archive/.repro_attempts.json` using the
            `stable_key`, computed as `normalized_title + "@" +
            primary_file_path`, where `normalized_title` is the finding's title
            lowercased with all non-alphanumeric characters removed and
            `primary_file_path` is the first entry in `code_paths` with any line
            number suffixes removed, e.g. `src/dma_engine.sv` from
            `src/dma_engine.sv:120`), OR
          - Patch verification failed or was incomplete (`patch_status` is
            `"VERIFICATION_FAILED"`, `"ERROR"`, or
            `"VERIFICATION_INCOMPLETE"`).
          - *Note:* Do **not** retry/copy findings where `"status"` is
            `"FALSE_POSITIVE"`, or `"patch_status"` is `"VERIFIED_SECURE"`, or
            `"repro_status"` is `"reproduced"` (unless patch failed as above), or
            those that have reached the 2-attempt limit in the cache, unless the
            target file has been modified in the current loop (VCS diff shows
            changes, or file modification timestamps/hashes have changed).

     - **Context Injection (`kb_references`):** For each investigation you plan,
       you must determine which files in the `workspace/kb/` directory (e.g.,
       `workspace/kb/entities/dma_engine.md` or
       `workspace/kb/vulnerabilities/CDC-Metastability.md`) provide necessary
       context for the researcher. Include the exact file paths to these markdown
       files in the `"kb_references"` array for that investigation. This shifts
       the burden of context-gathering off the researcher.

     - **Exploratory/Unconstrained Investigations (Low Probability):** With a low
       probability (e.g., a 15-20% chance per planning pass), include an
       exploratory investigation in the plan. Select a module or directory that
       the threat model currently marks as safe, low-risk, or out of scope, or a
       block that has not received recent scrutiny. The question for this
       investigation must explicitly instruct the researcher to perform an
       unconstrained, adversarial sweep, ignoring all existing assumptions and
       trust boundary definitions in `workspace/kb/THREAT_MODEL.md`. It should
       instruct the agent to assume boundaries and invariants can be violated and
       hunt for novel access bypasses, FSM corruption, glitch/fault
       susceptibility, or datapath bugs from scratch. **Token Optimization:**
       Whether using a script (Mode A) or your file-writing tools (Mode B), write
       the plan directly to disk and do not print the JSON contents in your chat
       response.

3. **Schema Enforcement:** Regardless of the mode, the final
   `workspace/plan.json` file written to disk should match the following schema
   to ensure downstream auditing agents can parse it correctly:

### Plan Schema Format

```json
{
  "investigations": [
    {
      "title": "Exhaustive Review: [relative_file_path]",
      "target_files": ["[relative_file_path_1]", "[relative_file_path_2]"],
      "kb_references": ["workspace/kb/entities/dma_engine.md", "workspace/kb/vulnerabilities/CDC-Metastability.md"],
      "question": "Detailed reviewing prompt instructions asking the researcher to trace specific signals, clock/reset domains, FSM transitions, address-decode ranges, or interface handshakes and their invariants."
    }
  ]
}
```

Ensure `workspace/plan.json` is successfully written. When complete, emit the Harness Result Contract footer as the final part of your response (see schema.json, "Harness Result Contract").
