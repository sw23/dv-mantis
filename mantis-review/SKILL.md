---
name: mantis-review
description: >-
  Independently reviews RTL findings and filters out false positives.
  Use when consolidated findings need validation against the actual HDL source.
  Don't use for reproducing bugs in simulation or patching RTL.
---

# Reviewer (/mantis-review)

## System Goal

Independent Validator. Reviews consolidated findings against the active HDL
source to verify validity and filter out noise and false positives.

## Command Definition

- **Command:** `/mantis-review`
- **Description:** Independently reviews RTL findings and filters out false
  positives.

## Input/Output Contract

- **Reads**:
  - `workspace/findings/` (finding JSON files).
  - `workspace/.mantis_state.json` (to track current loop pass).
  - Target HDL source files (at paths/lines in `code_paths`).
- **Writes**:
  - Updates findings on disk in-place (sets `"status"`, `"reasoning"`,
    `"repro_hints"`, `"triage_checklist"`, and appends history).
  - Writes helper script `workspace/helpers/append_review.py`.
- **Preconditions**:
  - `workspace/findings/` exists with finding files.
  - Target HDL source files exist.
- **Idempotency Guarantee**:
  - Modifies finding files in-place using the helper script `append_review.py`.
    It must check if a review for the current pass is already recorded in the
    finding's history array, skipping the review update if the last history
    entry is already `"stage": "reviewer"` for the current pass to prevent
    duplicate history records.

## Instructions

Read and evaluate the deduplicated findings against the actual HDL source of the
design. **Assume every finding is a false positive by default. Your job is to
disprove the finding using an adversarial stance. Evaluate the claim based ONLY
on the RTL and the raw claim itself. Explicitly ignore the original finder's
prose reasoning and justification, as they may be hallucinated.**

Execute your validation as follows:

1. **Load Clustered Findings:** Read the JSON files in the `workspace/findings/`
   directory. If the directory is empty or missing, emit a result footer with
   "status": "skipped" and stop (see schema.json, "Harness Result Contract").

2. **Source Code Inspection:** For each finding, read the exact files and line
   numbers listed in `code_paths` to ensure the finding is grounded in the
   actual design state. Do not make assumptions about the validity of a path
   without inspecting the HDL first.

3. **Strict Validation Filtering (Apply the 13 Negative Constraints):** Evaluate
   each finding against these strict criteria. Mark a finding as
   **FALSE_POSITIVE** if it violates any of the following rules:

   01. **Ignore Hypothetical Misuse:** Do not flag bugs that rely on an
       instantiating (parent) module hypothetically violating a documented
       interface contract — e.g., tying a synchronous input to the wrong clock,
       driving an illegal parameter, or de-asserting reset out of spec — if the
       module itself behaves correctly for all legal, in-contract inputs.
   02. **Ignore Missing Hygiene / Defense-In-Depth:** Do not report pure lint
       style (unused signals, missing `default_nettype`, magic numbers), a
       missing `default` arm on an already fully-enumerated `case`, or missing
       SVA assertions / functional coverage as design bugs on their own.
   03. **Require Strict Reproducibility:** Only mark a finding as VALID if a
       direct, unambiguous, and triggerable defect exists within the RTL logic.
       If the finding is extremely fragile (e.g., relies on a state that is
       provably unreachable, or an input combination the interface makes
       impossible), mark it as FALSE_POSITIVE. *Note on Metastability, Glitches,
       and Races:* Do NOT dismiss CDC metastability, reset-domain-crossing, or
       glitch-based bugs simply because any single event has a low probability.
       Real silicon runs for billions of cycles, so a hazard that is
       mathematically possible and not synchronized/constrained will eventually
       manifest and must be treated as reproducible.
   04. **Avoid Pedantic Linting:** If the RTL uses standard safe constructs
       (proper multi-flop synchronizers on async signals, non-blocking (`<=`)
       assignments in sequential `always_ff`, gray-coded CDC FIFO pointers,
       fully-specified `case`/`unique case`) but simply lacks extreme paranoia,
       mark it as FALSE_POSITIVE.
   05. **No Bug Stretching on Mitigations:** If you are reviewing a mitigation or
       a corrected variant of a block that successfully blocks the original bug
       class (e.g., a two-flop synchronizer that resolves the reported CDC), do
       NOT invent adjacent bug classes (e.g., claiming a reset glitch when
       reviewing a CDC fix). If the primary bug is successfully blocked, mark it
       as FALSE_POSITIVE.
   06. **Evaluate Questionable File Paths:** Do NOT instantly dismiss a finding
       simply because its path contains `/tb`, `/sim`, `/model`, or `/bench`.
       Some behavioral models, reference blocks, or "example" RTL are actually
       synthesized into the design or reused as production IP. Do not blindly
       assume it is out of the synthesized design; instead, take reasonable
       measures to trace whether the module is actually instantiated in the
       synthesizable hierarchy.
   07. **Ignore Capacity / Backpressure Nitpicks:** Do not flag FIFOs, queues, or
       buffers for finite depth, or a datapath for lacking infinite buffering / a
       full-flag, unless correctly handling overflow, underflow, or backpressure
       is the primary stated purpose of that module.
   08. **Intrinsic Design Bugs:** If a module contains a fundamentally broken
       construct — an unintended inferred latch on a datapath, a combinational
       loop, a reset that never actually reaches a critical register, an
       asynchronous signal sampled with no synchronizer, a hardcoded key/debug
       backdoor, or a multiply-driven net — mark it as VALID even if the module
       is not currently instantiated anywhere in the design.
   09. **Verify Mitigations Pragmatically:** Do not hallucinate flaws in active
       mitigations. If the code instantiates a correct multi-flop synchronizer,
       gray-codes a CDC pointer, or gates a register write behind a lock bit,
       accept that the mitigation works.
   10. **Refine `code_paths` Strictly:** The `code_paths` field should only
       include the exact `filename:line_number` of the flawed RTL block. Strip
       out any testbench files, verification IP, or correct instantiating parent
       modules from `code_paths`.
   11. **Ignore Provably-Masked or Tied-Off Bits:** If a finding is an
       out-of-range array/memory index, an out-of-range address, or extra signal
       bits, but the design provably masks, ties off, decodes to a defined
       error/default response, or otherwise ignores those bits/addresses
       downstream under all execution paths (e.g., unused MSBs are constant, or
       an illegal address is caught by the decoder and returns a bus error), mark
       the finding as a FALSE_POSITIVE (By Design).
   12. **Ensure Source Code Coherence (Anti-Hallucination):** Verify that every
       file path listed in `code_paths` exists in the repository, and that module
       names, signal names, or line numbers actually exist at those locations. If
       references are missing or incorrect, immediately mark the finding as a
       FALSE_POSITIVE to prevent downstream agents from wasting resources on
       hallucinated bugs.
   13. **Verify Controllability of the Trigger (Threat-Boundary Tracing):**
       Before marking a finding VALID, confirm that the signals, registers, or
       state that trigger the bug are actually controllable through the design's
       real interface and threat model (bus channels, register writes, DMA
       descriptor fields, cross-clock inputs) — not merely reachable by
       artificially forcing an internal net to an illegal value in simulation.
       Identify and cite the `filename:line_number` where the triggering value
       enters the analyzed RTL (the "Ingress Point") from an untrusted source
       and flows to the buggy logic, OR where that state is driven by an
       untrusted writer.
       - If you have access to the driving RTL (e.g. an untrusted bus master or
         agent in a multi-module design), cite that writer.
       - If you only have access to the block under review, cite the interface
         ingress point on the path (e.g., an AXI/AHB/APB write-data or address
         channel, a register-file write, a DMA descriptor field, or a cross-clock
         input).
       - If the triggering state is proven to originate solely from trusted
         on-chip origins (tied-off constants, hardwired straps, internal
         FSM-only state with no reachable interface path), mark FALSE_POSITIVE.
         Keep the `access_position` field consistent with the cited ingress.
       - Exception: Do not apply this rule to Intrinsic Design Bugs (Rule 08)
         where the defect exists in the RTL independent of any active driver.

   - **Status Resolution:**

     - Mark as **FALSE_POSITIVE** if it violates any of the 13 rules above.
     - Mark as **VALID** if it passes all rules and has a clear, triggerable
       defect.
     - Mark as **PROVISIONALLY_VALID** if it passes the rules, but you are
       uncertain of its feasibility without dynamic verification (e.g. requires
       simulation/formal proof or precise clock/reset timing).
     - Mark as **NEEDS_RESEARCH** if the review is inconclusive due to high
       complexity, unresolved external/black-box IP, or massive design
       hierarchies.

   - **Checklist Construction:**

     - Construct the `triage_checklist` object evaluating all 13 negative
       constraints. For each rule, set `outcome` to:
       - `"PASS"`: if the finding satisfies the constraint (does not violate the
         rule, meaning the bug remains potentially valid).
       - `"FAIL"`: if the finding violates the rule (which requires the finding
         status to be marked as `FALSE_POSITIVE`).
       - `"UNKNOWN"`: if the rule applicability is unresolved/needs research.
       - `"NOT_APPLICABLE"`: if this rule is entirely irrelevant to this class of
         bug.

4. **Construct Reproduction Hints:** For every finding marked as **VALID** or
   **PROVISIONALLY_VALID**, provide high-signal `"repro_hints"` explaining how a
   reproducer agent can trigger the bug: what stimulus, register-write sequence,
   clock relationship, reset timing, or state precondition is required, and what
   result confirms it (e.g., an SVA assertion failure, an `X` propagating onto a
   live signal, a mismatch against a reference model, a formal counterexample
   (CEX), or a hang/timeout with no forward progress).

5. **Token-Optimized File Updates:** To minimize LLM output tokens, **do not
   re-emit or manually rewrite the entire JSON object in your output.** Instead,
   write a reusable helper script (e.g., `workspace/helpers/append_review.py`)
   during your first finding update. For all subsequent findings, do not
   regenerate the script; simply execute the existing helper script with the new
   parameters to append the required fields.

   You must append the following to the existing object:

   - A `"status"` field (one of `"VALID"`, `"FALSE_POSITIVE"`,
     `"PROVISIONALLY_VALID"`, or `"NEEDS_RESEARCH"`).

   - A `"reasoning"` field.

   - A `"repro_hints"` field (optional for `"NEEDS_RESEARCH"` or
     `"FALSE_POSITIVE"`).

   - A `"triage_checklist"` object containing evaluations for all 13 negative
     constraints (each key in the object maps to the constraint of the matching
     name from Section 3 above):

     ```json
     {
       "ignore_hypothetical_misuse": { "outcome": "PASS" },
       "ignore_missing_hygiene": { "outcome": "PASS" },
       "require_strict_reproducibility": { "outcome": "FAIL", "reason": "Relies on an FSM state that is provably unreachable given the interface constraints." },
       "avoid_pedantic_linting": { "outcome": "PASS" },
       "no_bug_stretching_on_mitigations": { "outcome": "PASS" },
       "evaluate_questionable_file_paths": { "outcome": "PASS" },
       "ignore_capacity_backpressure_nitpicks": { "outcome": "PASS" },
       "intrinsic_design_bugs": { "outcome": "PASS" },
       "verify_mitigations_pragmatically": { "outcome": "PASS" },
       "refine_code_paths_strictly": { "outcome": "PASS" },
       "ignore_masked_or_tied_off_bits": { "outcome": "PASS" },
       "ensure_source_code_coherence": { "outcome": "PASS" },
       "verify_controllability_of_trigger": { "outcome": "PASS" }
     }
     ```

     For backward compatibility, the schema also permits `"passes": <bool>` as an
     alternative to `"outcome"`, but `"outcome"` is preferred. The `reason` must
     be provided if the outcome is `FAIL`, `UNKNOWN`, or `NOT_APPLICABLE` (or if
     `passes` is `false`) to explain the evaluation; it should be omitted for
     `PASS` / `true` to save tokens.

   - An entry to the `"history"` array:

   ```json
   {
     "stage": "reviewer",
     "action": "reviewed",
     "details": "Determined status as [VALID/FALSE_POSITIVE/PROVISIONALLY_VALID/NEEDS_RESEARCH] because [reason]",
     "pass_number": <current_pass_number>,
     "timestamp": "<current_iso8601_timestamp>"
   }
   ```

When complete, emit the Harness Result Contract footer as the final part of your response (see schema.json, "Harness Result Contract").
