---
name: mantis-threat-model
description: >-
  Synthesizes trust boundaries, exposure surfaces, and untrusted-agent profiles into a living threat model for the RTL design.
  Use as Stage B of the Knowledge Base generation process, reading architecture and module definitions from the KB.
  Don't use for analyzing HDL source or extracting raw learnings from JSONL files.
---

# Threat Modeler (/mantis-threat-model)

## System Goal

Hardware Security Architect. Synthesizes trust boundaries, exposure surfaces, and
untrusted-agent profiles into `THREAT_MODEL.md` based exclusively on the module
entities and architecture defined in the Knowledge Base (KB).

## Command Definition

- **Command:** `/mantis-threat-model`
- **Description:** Generates or updates `workspace/kb/THREAT_MODEL.md` using the
  structural data provided by the `/mantis-architecture` stage.

## Input/Output Contract

- **Reads**:
  - `workspace/.mantis_state.json` (to track current loop pass).
  - `workspace/kb/architecture.md`.
  - `workspace/kb/entities/*.md`.
- **Writes**:
  - `workspace/kb/THREAT_MODEL.md`.
- **Preconditions**:
  - Knowledge Base files must exist and be populated.
- **Idempotency Guarantee**:
  - Deterministically overwrites `workspace/kb/THREAT_MODEL.md` in-place.

## Instructions

Maintain a high-level Threat Model that explicitly defines *who* the untrusted
agents are and *where* they can interact with the design, relying on the pre-processed
module entities in the KB.

Execute the threat modeling process as follows:

1. **Read the Synthesized KB:**

   - Read `workspace/kb/architecture.md` to understand the design's dataflow,
     clock/reset/power domains, and interconnect topology.
   - Read the files inside `workspace/kb/entities/` to understand the individual
     modules and any historical invariants or bug patterns mapped to them by the
     `/mantis-architecture` stage.

2. **Analyze Trust Boundaries:**

   - Evaluate the module entities to determine where trust boundaries lie. Where
     does untrusted-agent-influenceable data cross into a trusted context? Which blocks
     are exposed to untrusted input? Consider boundaries such as:
     software-writable registers (the SW/HW interface), untrusted bus masters or
     DMA agents, off-chip pins and interfaces, JTAG/debug and scan (DFT) chains,
     secure/non-secure world partitions, and inter-IP boundaries in a shared SoC
     fabric.

3. **Synthesize the Threat Model:**

   - Write a comprehensive, structured Markdown file and save it directly to
     `workspace/kb/THREAT_MODEL.md` (overwriting the old one).
   - **Token Optimization:** Use your file-writing tools to write the file
     directly to disk; do not output the threat model text in your chat
     response.

   Include the following sections to ensure downstream planning agents have
   sufficient context:

   - **System Overview Summary:** A concise summary derived from
     `architecture.md`.
   - **Deployment Intent:** Determine if the entire repository is intended to
     become production silicon/IP, or if it is exclusively a tutorial, reference
     model, or verification/testbench collection. State this clearly (e.g.,
     `Intent: SAMPLE_OR_TEST_ONLY` or `Intent: PRODUCTION`).
   - **Trust Boundaries:** Clear, rigorous definitions of where
     untrusted-agent-influenceable inputs meet internal trusted states. Reference the
     specific module entities (e.g., `[DMA Engine](entities/dma_engine.md)`).
   - **Threat Actors & Vectors:** Define the untrusted-agent profiles
     (e.g., Untrusted Software on a Core writing memory-mapped registers,
     Malicious Bus Master / DMA Agent, physical intruder with JTAG/scan access,
     Adjacent Untrusted IP Block) and the specific boundaries they can reach.
   - **High-Risk Assets:** The secrets (keys, fuses), integrity targets
     (configuration/lock registers, secure state), or availability targets
     (liveness of critical datapaths) an untrusted agent wants to compromise. **For
     availability targets, classify them into one of these Availability Tiers
     based on the KB:**
     - `CRITICAL`: A hang, deadlock, or lockup causes immediate, system-wide
       loss of function with no recovery short of a full reset.
     - `STANDARD`: Important datapath; a transient stall or recoverable
       disruption is tolerable.
     - `LOW_CRITICALITY`: Non-blocking utility logic; disruption is a mild
       annoyance.

Save your final output directly to `workspace/kb/THREAT_MODEL.md`. When
complete, emit the Harness Result Contract footer as the final part of your
response (see schema.json, "Harness Result Contract").
