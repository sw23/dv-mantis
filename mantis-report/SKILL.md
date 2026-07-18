---
name: mantis-report
description: >-
  Generates a human-readable design-bug review packet compiled from reproduced findings and bug chains, including best-effort findings that were not empirically reproduced.
  Use at the end of a review cycle to produce stakeholder-facing documentation.
  Don't use for auditing HDL or verifying patches directly.
---

# Reporter (/mantis-report)

## System Goal

Hardware Design-Bug Reporting Expert. Synthesizes complex, technical finding logs into a
high-quality, human-readable review packet for hardware designers and stakeholders.

## Command Definition

- **Command:** `/mantis-report`
- **Description:** Generates a human-readable design-bug review packet containing
  reproduced findings and bug chains, plus best-effort validated findings that
  could not be empirically recreated (clearly labeled as such).

## Input/Output Contract

- **Reads**:
  - `workspace/findings/*.json` (all finding files).
  - `workspace/.mantis_state.json` (to track current loop pass).
- **Writes**:
  - `workspace/report/review_packet_pass_<N>.md` (pass-numbered markdown
    report).
  - Updates copy/symlink at `workspace/report/review_packet-latest.md`.
- **Preconditions**:
  - Calibrated and reproduced findings exist in `workspace/findings/`.
- **Idempotency Guarantee**:
  - Writes to pass-numbered files. In-place overwrite of
    `review_packet-latest.md`. Since it reads the pass number from the state,
    re-running the same pass updates the same pass report rather than generating
    a new pass number.

## Instructions

Compile a professional Markdown report detailing the verified and reproduced
design bugs and bug chains, plus best-effort validated findings that could not
be empirically recreated (clearly labeled as such).

Execute the reporting stage as follows:

1. **Load and Filter Findings:**

   - Read all JSON files from the `workspace/findings/` directory.
   - **Filter:** Include only actionable findings. You MUST include:
     1. Findings where `repro_status` is `"reproduced"`.
     1. Findings where `repro_status` is `"statically_confirmed"` BUT contain
        valid simulator/formal logs, assertion/SVA failures, or
        counterexample/waveform traces proving empirical triggering impact
        (treated as empirically reproduced by calibration).
     1. Valid Bug Chains (constructed by the chainer, identified by "Bug Chain"
        in the title, or history, or if the "constituent_findings" property is
        present and non-empty) even if they inherited a status of
        `"statically_confirmed"`.
     1. Best-effort findings that are `"VALID"` and viable but were not
        empirically recreated (`repro_status` is `"failed_to_reproduce"`,
        `"not_attempted"`, or `"statically_confirmed"` without empirical
        traces). Recreation and chaining are best-effort and non-blocking, so
        these must **not** be dropped solely for lacking a reproducer — include
        them but **clearly label them "Statically identified (not empirically
        reproduced)"** so reviewers can distinguish confirmed reproductions from
        static-only findings.
   - Do **not** include false positives or non-viable findings.
   - **Severity Filtering:** Exclude findings with a priority of `"LOW"` from the
     main report body. You must place these lower-priority issues into a
     separate, dedicated "Appendix: Low Priority Findings" section at the very
     end of the report, keeping the main report focused on high-risk issues.

2. **Extract Key Artifacts:** For each included finding, extract and format:

   - **Header Metadata:** Title, ID (UUID), Inferred Exposure, Final Risk Score,
     and Qualitative Priority.
   - **Design-Bug Description & Impact:** A clear explanation of the bug and the
     concrete impact on the system.
   - **Reproduction Evidence:**
     - The reproducer path (`repro_file_path`) — the generated testbench, SVA
       property, or formal property file — and the execution command
       (`run_command`).
     - A clean snippet of the simulator/formal output showing the successful bug
       trigger (`repro_output`).
     - If the finding was included as best-effort (not empirically reproduced),
       these fields may be absent; in that case state "Statically identified —
       not empirically reproduced" in place of the reproduction evidence rather
       than omitting the finding.
   - **Risk Rationale:** The independent validation reasoning (`reasoning`),
     production viability reasoning (`critic_reasoning`), and outrage factor
     analysis (`outrage_commentary`).
   - **Remediation & Patch:**
     - The recommended mitigation strategy.
     - The verified patch diff (`patch_diff`) and re-verification status to prove
       the fix is resilient.
   - **Secrets & Sensitive-IP Redaction:** Before writing any finding data
     (including description, reproducer script/command, and reproducer logs) to
     the report, you **must** scan and redact any hardcoded keys (e.g.
     efuse/test/debug keys), tokens, credentials, PII (names, emails, phone
     numbers), internal hostnames/codenames, and sensitive internal IP details,
     replacing them with standard placeholders like `<REDACTED_SECRET>`,
     `<REDACTED_PII>`, `<REDACTED_INTERNAL_HOST>`, or `<REDACTED_IP_DETAIL>` to
     ensure the report is safe for broader distribution.

3. **Generate Review Packet:**

   - **Report Header & Disclaimer:** At the very top of the report (before the
     Executive Summary):

     1. Display the target RTL codebase version information read from
        `"vcs_info"` in `workspace/.mantis_state.json`:

        - If `"vcs_type"` is `"git"`, show:
          `Target Version: Git branch [branch] at commit [commit_hash] [(dirty) if dirty is true]`.
        - If `"vcs_type"` is `"hg"`, show:
          `Target Version: Mercurial branch [branch] at revision [commit_hash] [(dirty) if dirty is true]`.
        - If `"vcs_type"` is `"multi-vcs"`, show:
          `Target Version: Multi-VCS (repo) manifest [revision] [(dirty) if dirty is true]`.
        - If `"vcs_type"` is `"none"`, show:
          `Target Version: None (No version control detected)`.
        - If `"vcs_type"` is `"unknown"`, or if `vcs_info` is missing, show:
          `Target Version: Unknown (VCS detection failed/error)`.

     1. Include a prominent disclaimer note stating: *"This report was
        automatically generated by Mantis AI. All findings and patches are
        AI-generated and must be manually verified by a hardware design or
        security subject matter expert before deployment or disclosure."*

   - **Grouping by Patch Status (Exclusivity):** Organize the Executive Summary
     table and the main body of the report by grouping findings. **Bug chains
     MUST be excluded from these main groups and reported ONLY in their dedicated
     "Bug Chains (Not End-to-End Reproduced)" section.** For standard (non-chain)
     findings, group them into three distinct categories based on their
     remediation status (strictly mutually exclusive):

     1. **Category 1: Patch Independently Verified**: Findings where
        `patch_status` is `"VERIFIED_SECURE"`.
     1. **Category 2: Patch Proposed / Mitigation Identified**: Findings where
        `patch_status` is in
        `["MITIGATION_PROPOSED", "VERIFICATION_INCOMPLETE"]` OR (`patch_diff` is
        present AND `patch_status` is unset/empty).
     1. **Category 3: Unpatched / Verification Failed**: Findings where
        `patch_status` is in `["VERIFICATION_FAILED", "ERROR"]` OR (`patch_diff`
        is not present AND `patch_status` is unset/empty).

   - **Dedicated Bug Chains Section:** Create a dedicated section titled
     `"Bug Chains (Not End-to-End Reproduced)"` specifically for bug chains.
     Document each chain finding here, listing its title, qualitative priority,
     risk score, and detailing its constituent findings (their IDs and
     individual status). Do not mix bug chains with standard findings in
     Categories 1, 2, or 3.

   - **Pass-Numbered Output:** Do not overwrite the same `review_packet.md` file
     on every execution. Instead, determine the current run/pass number `N` of
     the pipeline (resolved from `"pass_number"` in
     `workspace/.mantis_state.json`. If missing or invalid, scan
     `workspace/archive/` for folders matching `findings_pass_N` or
     `loopN_findings` and resolve `N` to `max_found + 1`, defaulting to 1 if no
     archives exist). Write the report to a pass-numbered file:
     `workspace/report/review_packet_pass_<N>.md` (where `<N>` is the sequential
     pass number, e.g., `review_packet_pass_1.md`).

   - **Latest Copy/Symlink:** After writing the pass-numbered report, update a
     symlink or write a copy of the file to
     `workspace/report/review_packet-latest.md` pointing/copying to the newest
     pass report, so that the latest version is always reachable.

   - Use clean, professional Markdown formatting with clear headers, tables for
     metadata, and syntax-highlighted code blocks for logs and diffs.

   - Include a high-level **Executive Summary** table at the top listing all
     included findings, their priority, and their risk scores.

When complete, emit the Harness Result Contract footer as the final part of your response (see schema.json, "Harness Result Contract").
