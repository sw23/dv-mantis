---
name: mantis-chain
description: >-
  Analyzes individual RTL design findings to identify and construct complex multi-step bug chains.
  Use after validation stages to see if multiple low-severity design bugs can be combined into a higher impact compromise.
  Don't use for initial RTL auditing or writing patch code.
---

# Bug Chainer (/mantis-chain)

## System Goal

Bug Chain Architect. Analyzes isolated, individually-validated design
findings and historical knowledge base primitives to identify and construct
complex, multi-step design-bug chains — zooming out from module-level bugs to
see whether they can be exposed through some combination of inputs at the
top-level Design Under Test (DUT).

## Command Definition

- **Command:** `/mantis-chain`
- **Description:** Analyzes individual RTL design findings to identify and
  construct complex bug chains.

## Input/Output Contract

- **Reads**:
  - `workspace/findings/` (validated finding JSON files where status is
    `"VALID"`, and viability is `"VIABLE"`, `"CONDITIONAL_VIABLE"`, or
    `"SAMPLE_OR_TEST"`).
  - `workspace/kb/entities/*.md` and `workspace/kb/vulnerabilities/*.md`
    (knowledge base primitives).
  - `workspace/.mantis_state.json` (to track current loop pass).
- **Writes**:
  - Net-new bug chain finding JSON files to
    `workspace/findings/<new_uuid>.json`. Original findings are left unmodified.
- **Preconditions**:
  - Validated or viable findings must exist in `workspace/findings/`.
- **Idempotency Guarantee**:
  - Before writing a new bug chain finding, the skill must check existing bug
    chain findings in `workspace/findings/` by comparing the constituent finding
    UUIDs. If a chain finding with the exact same constituent UUID sequence
    already exists, it must skip creating a duplicate.

## Instructions

Read the current batch of validated findings and explore whether multiple
seemingly low-severity or disparate design bugs can be sequentially combined to
achieve a higher-impact compromise or failure at the top-level DUT.

This stage is **best-effort and non-blocking**. Chaining means zooming out from
an isolated module-level bug to see whether it can be exposed through some
combination of inputs at the top-level DUT — typically via an existing top-level
testbench (cocotb, UVM, FireSim, or similar). Constructing and exercising such a
harness usually requires design-specific domain knowledge, so treat chaining as
an opportunistic enrichment: if no viable chain can be built, that is an
acceptable outcome. Never halt the pipeline, downgrade, or discard the
underlying individual findings because a chain could not be constructed.

Execute the chaining stage as follows:

1. **Load Primitives & Validated Findings:**

   - Read the JSON files in the `workspace/findings/` directory. Filter for
     findings that have passed validation (e.g., status is `"VALID"` and
     viability is `"VIABLE"`, `"CONDITIONAL_VIABLE"`, or `"SAMPLE_OR_TEST"`).
   - Read the Markdown Knowledge Base (`workspace/kb/entities/` and
     `workspace/kb/vulnerabilities/`) to identify architectural primitives that
     might not be bugs on their own, but could serve as stepping stones (e.g.,
     "An untrusted bus master can write this descriptor pointer", "This register
     is not protected by the global lock bit", "Debug mode exposes this internal
     bus").

2. **Cross-Finding Analysis (The Chaining Matrix):**

   - Analyze the preconditions and postconditions of each validated finding.
   - Ask: *Can the output or side-effect of Finding A satisfy the strict
     precondition required to trigger Finding B?*
   - Example Chains to look for:
     - **Missing Lock Bit + Debug Exposure = Secret Extraction:** A register
       that fails to lock a key path (Finding A) combined with a debug/JTAG
       interface that can read that path in a production mode (Finding B) leaks a
       secret.
     - **Address-Decode Off-by-One + Missing Write-Protect = Secure Config
       Overwrite:** An address decoder that aliases a protected region (Finding
       A) plus a control register that lacks a write-protect check (Finding B)
       lets an untrusted master overwrite secure configuration.
     - **CDC Glitch + Illegal FSM State = System Hang:** An unsynchronized
       control crossing (Finding A) drives a one-hot FSM into an illegal
       multi-hot state (Finding B), producing an unrecoverable deadlock.
     - **Untrusted DMA Descriptor + Unchecked Dereference = Out-of-Range
       Access:** An engine that accepts an untrusted-master-controlled
       descriptor pointer (Finding A) and dereferences it without a bounds check
       (Finding B) enables cross-boundary memory access.

3. **Construct "Super Findings":**

   - If a viable bug chain is discovered, do **NOT** modify or delete the
     original isolated findings. They still need to be fixed individually.
   - Instead, generate a **net-new UUID** and create a new finding JSON file in
     `workspace/findings/<new_uuid>.json`.
   - **Constituent Findings**: You must record the array of constituent finding
     UUIDs in the structured `"constituent_findings"` property (e.g.,
     `["UUID_A", "UUID_B"]`). This clearly documents the sequence of execution
     and the links of the bug chain.
   - **Determine Entry Point Privileges**: The `privileges_required` field for
     the chain must represent the privilege level required to initiate the
     *first* step of the chain (the entry point). For example, if the chain
     starts with an unprivileged register write (NONE) that unlocks a protected
     path, which is then used to extract a key, the chain's `privileges_required`
     must be set to `NONE`.
   - **Determine Attacker Position**: The `attacker_position` field for the
     chain must inherit the attacker position from the entry point / first
     constituent finding of the chain.
   - **Determine User Interaction Requirement**: The `user_interaction` field for
     the chain must be set to `REQUIRED` if the entry point or any step in the
     chain requires a specific configuration/enable sequence, a debug/test mode,
     or cooperating software. It should only be set to `NONE` if the entire chain
     triggers autonomously or with ordinary traffic.
   - **Determine Status**: Set `status` to `"VALID"`.
   - **Determine Production Viability**: Inherit from constituent findings. If
     any constituent is `"SAMPLE_OR_TEST"`, set to `"SAMPLE_OR_TEST"`. Else if
     any constituent is `"CONDITIONAL_VIABLE"`, set to `"CONDITIONAL_VIABLE"`.
     Otherwise, set to `"VIABLE"`.
   - **Determine Reproduction Status**: Inherit from constituent findings:
     - If any constituent has a `repro_status` of `"failed_to_reproduce"` or
       `"not_attempted"`, set to `"not_attempted"`.
     - Otherwise, if all constituents have a `repro_status` of `"reproduced"` or
       `"statically_confirmed"`, set to `"statically_confirmed"`.
     - A bug chain must **never** inherit `"reproduced"` (as reproduction of
       constituents does not prove the end-to-end chain works).

   ### Chain Findings Schema Format (Per File)

   ```json
   {
     "id": "A unique identifier generated for this chain finding. Must match filename.",
     "title": "Bug Chain: [Impact] via [Finding A] and [Finding B]",
     "description": "Step-by-step documentation of the bug chain. Start with Step 1 (Triggering Finding A) and explain how its outcome feeds into Step N (Triggering Finding Z).",
     "impact": "The combined, escalated impact of the chain (e.g., Secure key extraction, arbitrary secure-config overwrite, unrecoverable SoC hang). This should be higher than the individual findings.",
     "severity": "CRITICAL / HIGH",
     "privileges_required": "NONE / LOW / HIGH",
     "user_interaction": "NONE / REQUIRED",
     "code_paths": [
       "relative/path/module_A.sv:line_number",
       "relative/path/module_B.sv:line_number"
     ],
     "attacker_position": "EXTERNAL / INTERNAL_NETWORK / IN_CLUSTER / LOCAL / HOST_SYSTEM / SUPPLY_CHAIN / PHYSICAL_TEMPORARY / PHYSICAL_LONG_TERM (inherited from entry point)",
     "mitigation": "Recommended strategy to break the chain. Usually involves fixing at least one, if not all, of the underlying links.",
     "status": "VALID",
     "production_viability": "VIABLE / SAMPLE_OR_TEST / CONDITIONAL_VIABLE",
     "repro_status": "statically_confirmed / not_attempted",
     "constituent_findings": ["UUID_A", "UUID_B"],
     "history": [
       {
         "stage": "chainer",
         "action": "created",
         "details": "Constructed by chaining findings [UUID_A] and [UUID_B].",
         "pass_number": <current_pass_number>,
         "timestamp": "<current_iso8601_timestamp>"
       }
     ]
   }
   ```

4. **Chain Deduplication Tagging:**

   - To ensure `/mantis-dedupe` treats these chains differently than raw
     findings, ensure the word "Chain" is prominently featured in the `"title"`
     and `"history"` fields as shown in the schema.

Ensure any newly constructed chain files are written to the
`workspace/findings/` directory. When complete, emit the Harness Result Contract footer as the final part of your response (see schema.json, "Harness Result Contract").
