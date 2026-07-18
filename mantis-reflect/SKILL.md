---
name: mantis-reflect
description: >-
  Extracts learnings from execution trajectories at the end of a Mantis loop.
  Use to parse agent conversations, extract successes, failures, and false assumptions, and append them to workspace/learnings.jsonl.
  Don't use for analyzing HDL source or writing patches.
---

# Reflector (/mantis-reflect)

## System Goal

Execution Trajectory Analyst. Analyzes the sequence of thoughts, tool calls, and
observations (the "trajectory" or "conversation") of the other Mantis agents.
Extracts valuable insights to prevent future agents from making the same
mistakes.

## Command Definition

- **Command:** `/mantis-reflect`
- **Description:** Parses execution trajectories from the current loop and
  appends structured insights to `workspace/learnings.jsonl`.

## Input/Output Contract

- **Reads**:
  - `workspace/.mantis_state.json` (to track current loop pass).
  - Subagent execution logs (`transcript.jsonl` files). The schema
    `execution_log_entry` defined in `schema.json` is the normalized
    representation. The orchestrator/adapter must normalize raw logs from
    unsupported frameworks before passing them, or the reflector must parse
    unsupported formats on a best-effort basis.
  - **Locating Logs:** The orchestrator must pass the list of absolute file
    paths to the execution log files (e.g. `transcript.jsonl` files) for the
    subagents executed during this round. Use these paths directly to read the
    logs.
- **Writes**:
  - Appends structured trajectory insights to `workspace/learnings.jsonl`.
- **Preconditions**:
  - Execution logs for the current round must exist and contain entries.
- **Idempotency Guarantee**:
  - Parses logs and filters already-recorded learnings to prevent duplicate
    entries in `workspace/learnings.jsonl`. It should check existing lines in
    `workspace/learnings.jsonl` to ensure it doesn't duplicate the same insight
    if retried.

## Instructions

Analyze the execution trajectories of all successfully executed subagents in the
round (every stage that ran: history, summarize, architecture, threat-model,
plan, researcher, dedupe, review, critic, reproduce, chain, patch, calibrate) to
distill what went right and what went wrong.

Execute the reflection stage as follows:

1. **Extract Trajectories (Token Optimization):**

   - Do not attempt to read the entire, raw `transcript.jsonl` or
     `conversation.jsonl` files natively with `read_file`, as they can be
     massive and blow out your context window.
   - Use the absolute log file paths passed to you by the orchestrator to access
     the log files.
   - Instead of reading the full files, use your bash/command execution tools to
     parse and filter the logs. For example, write a short Python script or use
     `jq`/`grep` to extract key events (which should conform to the
     `execution_log_entry` schema; if raw logs from different frameworks are
     provided, parse them on a best-effort basis): tool error messages, final
     agent summaries, instances where an agent "gave up", or messages indicating
     a design invariant or trust boundary assumption was incorrect.

2. **Synthesize Insights:** Review the extracted events. Look for:

   - **False Assumptions:** Did a researcher spend turns trying to trigger a bug
     on a signal, only to realize it was already synchronized or gated by a lock
     bit in another module?
   - **Tool Failures:** Did the reproducer fail consistently because of a missing
     technology library, an unelaboratable testbench, or a simulator flag
     mismatch?
   - **Successful Strategies:** Did a patcher successfully close a bug using a
     specific idiomatic RTL pattern (e.g., a standard two-flop synchronizer) that
     should be reused?

3. **Append to the Inbox (`workspace/learnings.jsonl`):** For each distinct
   insight, append a structured JSON object to `workspace/learnings.jsonl`.

   ### Reflection Schema Format (`workspace/learnings.jsonl`)

   ```json
   {"type": "trajectory_insight", "action": "add | update | remove", "target_entity": "[e.g., dma_engine.sv or sim_env]", "insight": "The researcher assumed the config signal crossed clock domains unsynchronized, but it is actually synchronized by a two-flop synchronizer in the parent. Do not report a CDC bug on this signal.", "source_stage": "mantis-researcher"}
   ```

   Ensure the file is appended to, not overwritten. When complete, emit the
   Harness Result Contract footer as the final part of your response (see
   schema.json, "Harness Result Contract").
