---
name: mantis-summarize
description: >-
  Pre-processes the RTL design by generating design-focused summaries (mantis-summary.md) for each directory to make planning and research more efficient.
  Use when starting a review campaign to map the design hierarchy before threat modeling and planning.
  Don't use for executing RTL reviews, writing testbenches, or patching HDL.
---

# Summarizer (/mantis-summarize)

## System Goal

Design Mapper. Automates the generation of design-focused, deterministic
summaries of directory contents to reduce token overhead for downstream planning
and research stages.

## Command Definition

- **Command:** `/mantis-summarize`
- **Description:** Pre-processes the RTL design by generating design-focused
  summaries (`mantis-summary.md`) for each directory to make planning and
  research more efficient.

## Input/Output Contract

- **Reads**:
  - `workspace/.mantis_state.json` (to track current loop pass).
  - Design directories and HDL source files (excluding generated/build outputs,
    simulation logs, IP caches, `.git`, and third-party vendor IP).
  - Child directory summaries (`mantis-summary.md` files from subdirectories).
  - `workspace/historical_learnings.jsonl` (optional, to enrich summaries).
- **Writes**:
  - Traversal script to workspace.
  - `mantis-summary.md` file in each directory containing HDL source.
- **Preconditions**:
  - HDL source files and directory structure must be present.
- **Idempotency Guarantee**:
  - Deterministically overwrites existing `mantis-summary.md` files in-place
    with updated rollups.

## Instructions

Your task is to write and execute a script that will traverse the design
directory tree and create a `mantis-summary.md` file in each directory
containing HDL source.

This is an **optional pre-processing phase** designed to drastically reduce the
context window size required for the strategist (`/mantis-plan`), and provide a
quick reference map for researchers (`/mantis-researcher`) and threat model
generator (`/mantis-threat-model`).

Execute the summarize stage as follows:

1. **Write the Traversal Script (Bottom-Up Hierarchical):** Write a script
   (e.g., Python or bash) in your workspace that walks the design directory tree
   using a **bottom-up (post-order) traversal**.

   - The script must ignore non-design directories such as generated/build
     outputs, simulation logs, IP caches, `.git`, and third-party vendor IP that
     is out of scope.
   - By traversing bottom-up, the script ensures that submodule directories are
     summarized *before* their parent directories.
   - When analyzing a directory, the script should pass the LLM the local HDL
     source files in that directory (e.g., `.v`, `.sv`, `.vh`, `.svh`, `.vhd`, or
     Chisel generator sources `.scala`) **PLUS** the `mantis-summary.md` files of
     its immediate subdirectories. Do not pass the raw source files of
     subdirectories to the parent.
   - When analyzing very large directories or auto-generated RTL, context window
     size might become a problem. Instead of passing files and directory
     summaries in bulk, generate per-module summaries or operate in more
     efficient chunks to avoid passing too many tokens for the LLM to handle.

2. **Generate the Design Summary (Map-Reduce):** The script should read
   `workspace/historical_learnings.jsonl` (if it exists) to check for past design
   bugs and fixes associated with modules in the current directory, and pass them
   in context. The script should instruct the LLM or agent tool to generate a
   concise, design-focused summary of the directory. To keep token lengths
   reasonable at higher levels of the hierarchy, the LLM should abstract away
   lower-level details, focusing on the rolled-up microarchitecture. The prompt
   used by your script should ask for:

   - **Core Modules:** What are the primary modules and submodules, and what do
     they do?
   - **Interfaces & Ports:** What top-level ports, buses (e.g., AXI, AHB, APB,
     Wishbone, TileLink), or interfaces are exposed to other blocks?
   - **Trust Boundaries & External Inputs:** Does this block handle
     untrusted-agent-influenceable inputs, e.g., software-writable registers, an
     untrusted bus master/DMA, off-chip pins, or JTAG/debug/scan access?
   - **Sensitive Logic:** Are there clock-domain crossings, reset domains, finite
     state machines (FSMs), arbiters, key/secret storage, fuse/OTP or lock-bit
     registers, or memory/address-decode logic?
   - **Historical Bugs & Fixes:** What modules or logic in this directory have
     historical design bugs or fixes recorded in
     `workspace/historical_learnings.jsonl`? Summarize the past fixes, blocks
     affected, and bug classes to highlight past regressions or recurring
     weaknesses.

   The summary must be a reasonable size to incorporate into work on larger
   problems, so aim for several thousand words or fewer.

3. **Output to `mantis-summary.md`:** The script must write the resulting summary
   into a file named `mantis-summary.md` inside the corresponding directory.
   Overwrite it if it already exists.

4. **Execute the Script:** Run the script you just wrote to generate all the
   summaries across the design. Wait for it to finish successfully.

When complete, emit the Harness Result Contract footer as the final part of your response (see schema.json, "Harness Result Contract").
