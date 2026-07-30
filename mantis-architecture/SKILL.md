---
name: mantis-architecture
description: >-
  Synthesizes raw learnings and RTL design analysis into an interlinked Markdown Knowledge Base (KB).
  Use at the beginning of a loop to build or update architecture.md, module entities, and bug classes.
  Don't use for generating threat models or formulating execution plans.
---

# Architect (/mantis-architecture)

## System Goal

Knowledge Base Synthesizer. Translates ephemeral insights from the learnings
queue (`workspace/learnings.jsonl`) and structural analysis of the RTL design
into a canonical, interlinked Markdown Knowledge Base (`workspace/kb/`).

## Command Definition

- **Command:** `/mantis-architecture`
- **Description:** Builds the foundation of the KB by defining the design's
  microarchitecture, mapping specific module entities (IP blocks), and
  categorizing recurring bug classes.

## Input/Output Contract

- **Reads**:
  - `workspace/learnings.jsonl` (raw insights from the current round).
  - `workspace/historical_learnings.jsonl` (optional, past bug-class metadata).
  - RTL design directory structure and key HDL source files.
  - Existing Markdown files in `workspace/kb/` (to validate/decay check).
  - `workspace/.mantis_state.json` (to retrieve pass count).
- **Writes**:
  - Markdown files under `workspace/kb/` (`architecture.md`,
    `entities/[module_name].md`, `vulnerabilities/[CWE-ID].md`, `index.md`).
  - Archives `workspace/learnings.jsonl` to
    `workspace/archive/learnings/learnings_pass_${N}_${X}.jsonl`.
- **Preconditions**:
  - `workspace/learnings.jsonl` must exist.
- **Idempotency Guarantee**:
  - Transactional: moves `workspace/learnings.jsonl` to archive only after
    programmatically verifying all KB Markdown updates were written
    successfully. KB files are overwritten in-place.

## Instructions

Analyze the RTL design and pending learnings to construct a permanent,
Markdown-based memory for future agents.

> **No-Internet Directive:** Perform this task using **only** the local
> workspace, the target design files, and offline tools available in your
> environment. Do **not** access the internet or use any web-fetching capability
> (e.g., `curl`, `wget`, web search, or browser/`fetch` tools) to look up
> documentation, specifications, protocol standards, CWE/CVE databases,
> datasheets, or prior art. Rely solely on your own knowledge and the provided
> local context.

Execute the architecture stage as follows:

1. **Read the Inbox (`workspace/learnings.jsonl` and
   `workspace/historical_learnings.jsonl`):**

   - Parse the contents of `workspace/learnings.jsonl` (and
     `workspace/historical_learnings.jsonl` if it exists). Extract all
     trajectory insights, discovered design bugs, confirmed failure paths, and
     verified fixes.

2. **Analyze Design Boundaries:**

   - Examine the directory structure and key HDL source files. Dynamically
     identify the core modules, interfaces, clock/reset/power domains, and trust
     boundaries of the design based on the repository's contents. This applies
     broadly: identify IP blocks, bus fabrics and interconnect (AXI, AHB, APB,
     Wishbone, TileLink), register files and CSRs, JTAG/debug and scan (DFT)
     interfaces, memory and cache controllers, cryptographic blocks, fuse/OTP
     and lock-bit logic, arbiters, and clock-domain-crossing (CDC) or
     reset-domain-crossing (RDC) boundaries. For a Chisel/Scala design, identify
     these entities in the generator sources (modules, `Bundle` interfaces,
     `withClock` regions, `AsyncQueue` crossings, and `Diplomacy`/parameter-negotiated
     blocks). If only a gate-level or post-synthesis netlist is available,
     identify the equivalent structural primitives rather than forcing an
     RTL-source workflow.

3. **Build or Update the Knowledge Base (KB):**

   - Create or update files in the `workspace/kb/` directory using standard
     Markdown. Follow these strict paths:

     - `workspace/kb/architecture.md`: High-level dataflow, clock/reset/power
       domain definitions, interconnect topology, the software-visible register
       map, and overall functional/availability requirements (if documented or
       inferable from specs, IP-XACT, or constraints/SDC files).
     - `workspace/kb/entities/[module_name].md`: Specific definitions for
       modules (e.g., `dma_engine.md`). Must include links to associated bug
       classes and document known invariants (e.g., "This FSM is
       one-hot-encoded", "This register is locked by `cfg_lock`"). Document the
       block's criticality and availability requirements (classify as CRITICAL,
       STANDARD, or LOW_CRITICALITY if applicable). Incorporate trajectory
       insights here.
     - `workspace/kb/vulnerabilities/[CWE-ID_or_BugClass].md`: Descriptions of
       bug classes (e.g., `CWE-1245-Improper-FSM.md`, `CDC-Metastability.md`, or
       `Latch-Inference.md`) that have been historically relevant to this
       design, including examples of what *not* to do. Hardware CWEs (MITRE view
       CWE-1194) are a useful reference here.
     - `workspace/kb/index.md`: A root catalog containing links and 1-line
       summaries to every file created above. This is the map the Planner will
       read.

   - **Important Formatting Rules:** Use relative links to cross-reference
     entities and bug classes (e.g., `[DMA Engine](entities/dma_engine.md)`).
     Ensure all markdown files are concise and focused on actionable design
     context.

4. **Validate and Decay Knowledge (Drift Prevention):**

   - Knowledge becomes stale when RTL is patched or refactored. Before
     finalizing the KB updates, spot-check the assertions in the existing
     `workspace/kb/entities/` against the live HDL source.
   - If an entity file claims a signal crosses clock domains without a
     synchronizer (based on an old learning) but the live code now instantiates
     a proper two-flop synchronizer (because a fix landed), **delete or correct
     that outdated learning** in the KB.
   - If a learning is repeatedly proven wrong by the current trajectory
     insights, actively correct it to prevent the "wrong learning" from
     persisting and blinding future agents.

5. **Transactional Inbox Clearing & Archiving:**

   - To prevent infinite loops and token bloat, you must clear the queue and
     archive the learnings.
   - **Verify and Finalize:** Programmatically verify that all Markdown KB
     updates were successfully written to disk and that cross-references are
     valid.
   - **Commit by Moving:** Only after verifying synthesis success, move
     `workspace/learnings.jsonl` to the archive directory:
     - Ensure the target directory exists (e.g.,
       `mkdir -p workspace/archive/learnings/`).
     - Determine the loop pass number `N` by reading `"pass_number"` from
       `workspace/.mantis_state.json`. If missing or invalid, scan
       `workspace/archive/` for folders matching `findings_pass_N` or
       `loopN_findings` and resolve `N` to `max_found + 1`, defaulting to 1 if
       no archives exist.
     - Determine the sub-index `X` by counting existing files matching
       `learnings_pass_${N}_*.jsonl` in `workspace/archive/learnings/` and
       adding 1.
     - Move the file:
       `mv workspace/learnings.jsonl workspace/archive/learnings/learnings_pass_${N}_${X}.jsonl`.
   - If synthesis fails or is interrupted, leave `workspace/learnings.jsonl`
     intact in its original location to ensure no data is lost.

When complete, emit the Harness Result Contract footer as the final part of your response (see schema.json, "Harness Result Contract").
