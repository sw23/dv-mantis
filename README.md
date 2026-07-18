# Mantis Skills (Hardware Edition): Portable Toolkit for Building RTL Design-Bug Review Harnesses

> [!NOTE] **This is a specialized fork.** This repository is a clone of Google's
> [Mantis](https://github.com/google/mantis) project. The upstream Mantis is a
> toolkit for building **software** security review harnesses. This fork has been
> re-purposed to hunt for **hardware Register-Transfer Level (RTL) design bugs**
> (Verilog, SystemVerilog, VHDL, and Chisel/Scala generators) — both
> functional-correctness defects and hardware-security weaknesses — instead of
> software vulnerabilities. The Chisel support makes it practical to point the
> suite at the many open-source hardware designs (e.g., Rocket Chip, BOOM,
> Chipyard) that are written as Chisel generators. The overall
> architecture, sequential flow, and state contracts are inherited from upstream;
> the wording, validation heuristics, noise filters, and risk calibration have
> been retargeted for the hardware design domain. For the original software-focused
> project, its documentation, and its authors, see
> [github.com/google/mantis](https://github.com/google/mantis).

> [!CAUTION] **USE AT YOUR OWN RISK. BE EXTREMELY CAREFUL.** This suite is
> designed to generate and execute autonomously generated code (testbenches,
> assertions, and simulation/formal harnesses) that may be unstable or perform
> unexpected actions. **USE THIS ONLY IN ISOLATED, RESTRICTED ENVIRONMENTS.**
> Never run this suite on a machine with access to production systems, sensitive
> design IP, or internal networks. See the "Unattended Cloud Deployment" section
> for mandatory hardening requirements.

> [!IMPORTANT] **RESPONSIBLE USE** AI models are non-deterministic and can
> hallucinate findings or generate incorrect patches. **All findings must be
> manually verified by a hardware design or verification engineer before being
> acted upon.** Do not mass-file unverified, AI-generated reports to IP vendors
> or open-source hardware maintainers. A failure to automatically reproduce a
> design bug in simulation does not definitively mean it is a false positive, nor
> does a successful reproducer guarantee the bug is silicon-affecting in all
> configurations. Use these skills responsibly.

Mantis Skills (Hardware Edition) is a decoupled, sequential, and design-focused
set of **Skills** designed for use with Coding Agents. It is intended to be a
**flexible foundation and starting point** rather than a rigid set of
instructions. You should adapt, tune, and extend these skills to fit your
organization's specific hardware stack, HDLs, and verification flow.

For example, while the default skills will look for generic RTL design bugs
(clock-domain crossing hazards, reset issues, FSM defects, X-propagation,
unintended latches, register access-control flaws, arithmetic/width bugs, and
hardware-security weaknesses), they can be adapted for:

*   **SoC Integration Reviews**: Auditing bus fabrics (AXI, AHB, APB, Wishbone,
    TileLink), interconnect, and inter-IP trust boundaries for protocol and
    isolation bugs.
*   **Security IP Reviews**: Auditing key managers, crypto blocks, secure-boot
    ROMs, fuse/OTP and lock-bit logic, and memory-protection/IOMMU blocks for
    access-control and isolation weaknesses (see MITRE's Hardware Design CWE view,
    CWE-1194).
*   **Gate-Level / Netlist (Gray-Box) Auditing**: Pointing the suite at a
    post-synthesis or gate-level netlist (or an encrypted/third-party IP block)
    without RTL source, to emulate what an adversary who only has the delivered
    netlist can discover.
*   **Custom Verification Environments**: Replacing the default simulation stage
    with formal property checking, FPGA/emulation prototypes, or custom
    directed/constrained-random testbench flows.

We strongly recommend using AI to iterate on these skills and using your internal
design specs, coding standards (lint rules, CDC/RDC methodology), and build/EDA
flow to augment the threat model. We also strongly recommend adapting risk
calibration to your environment and risk tolerance.

This suite enables anyone with an agentic coding tool to systematically review,
deduplicate, validate, criticize, reproduce, and patch RTL designs of any scale.
It also features a continuous learning loop that allows the suite to adapt across
iterative runs and avoid redundant analysis.

Above all, while orchestrated design-bug discovery is incredibly powerful and
useful, it is even more important to use this in a suitably isolated environment
to prevent impacting production systems or lab equipment. See the notes on
unattended cloud deployment later in this guide.

______________________________________________________________________

## Architecture and Sequential Flow

The pipeline is composed of fifteen distinct components (one supervisor and
fourteen execution stages), maintaining state across a directory of finding files
(`workspace/findings/*.json`). This entire process can be supervised autonomously
by the overarching **`/mantis-meta-agent`**.

```mermaid
graph TD
    Meta["/mantis-meta-agent (Supervisor)"]

    subgraph "Continuous Review Loop"
        Hist["/mantis-history (Optional)"]
        Sum["/mantis-summarize (Optional)"]
        Arch["/mantis-architecture"]
        TM["/mantis-threat-model"]
        Plan["/mantis-plan"]
        Res["/mantis-researcher"]
        Ded["/mantis-dedupe"]
        Rev["/mantis-review"]
        Cri["/mantis-critic"]
        Rep["/mantis-reproduce"]
        Cha["/mantis-chain"]
        Pat["/mantis-patch"]
        Cal["/mantis-calibrate"]
        Ref["/mantis-reflect"]
    end

    FileHist[("historical_learnings.jsonl")]
    FileSum[("mantis-summary.md")]
    FileKB[/"workspace/kb/ (Markdown KB)"/]
    FilePlan[("plan.json")]
    FileFind[("workspace/findings/*.json")]
    FileLearn[("learnings.jsonl")]

    Meta --> Hist
    Hist --> Sum
    Sum --> Arch
    Arch --> TM
    TM --> Plan
    Plan --> Res
    Res --> Ded
    Ded --> Rev
    Rev --> Cri
    Cri --> Rep
    Rep --> Cha
    Cha --> Pat
    Pat -.->|Patch Re-verify Loop| Rep
    Pat --> Cal
    Cal --> Ref
    Ref -.->|Next Loop Iteration| Arch

    Hist -.->|Generates| FileHist
    Hist -.->|Reads| FileSum
    Sum -.->|Reads| FileHist
    Sum -.->|Generates| FileSum
    Arch -.->|Reads| FileHist
    Arch -.->|Generates| FileKB
    Arch -.->|Reads/Clears| FileLearn
    TM -.->|Reads/Updates| FileKB
    Plan -.->|Reads| FileKB
    Plan -.->|Reads| FileSum
    Plan -.->|Generates| FilePlan

    Res -.->|Reads| FilePlan
    Res -.->|Reads| FileKB
    Res -.->|Creates| FileFind
    Ded -.->|Reads| FileLearn
    Ded -.->|Merges| FileFind
    Rev -.->|Updates| FileFind
    Cri -.->|Updates| FileFind
    Rep -.->|Updates| FileFind
    Cha -.->|Reads| FileKB
    Cha -.->|Creates| FileFind
    Pat -.->|Updates| FileFind
    Cal -.->|Updates| FileFind

    Ref -.->|Parses Trajectories & Appends| FileLearn
    Cri -.->|Appends| FileLearn
    Pat -.->|Appends| FileLearn
```

1.  **`/mantis-meta-agent` (Supervisor):** A persistent, overarching agent that
    launches the continuous loop, monitors execution, handles errors, reports
    findings, and archives the `workspace/findings/` directory between loops.
2.  **`/mantis-history` (History Extractor):** An optional pre-processing step
    that analyzes the design's version control system (VCS) history to extract
    past design bugs, RTL fixes, and recurring failure patterns, saving findings
    to `historical_learnings.jsonl`.
3.  **`/mantis-summarize` (Summarizer):** An optional pre-processing step that
    generates a `mantis-summary.md` for each directory, reading past design bugs
    from `historical_learnings.jsonl` to enrich summaries and provide a quick
    reference map to optimize downstream planning and research.
4.  **`/mantis-architecture` (Knowledge Base Architect):** Analyzes the design
    and clears the `learnings.jsonl` inbox to synthesize a permanent,
    interlinked Markdown Knowledge Base (`workspace/kb/`) detailing modules,
    clock/reset/power domains, dataflows, and historical bug classes.
5.  **`/mantis-threat-model` (Threat Modeler):** Evaluates the module entities
    and architecture defined in the KB to establish or refine a living
    `workspace/kb/THREAT_MODEL.md`, focusing on trust boundaries and untrusted-agent
    profiles (untrusted software, bus masters, JTAG/scan, adjacent IP).
6.  **`/mantis-plan` (Strategist):** Scans design boundaries and reads the KB
    indices to output a targeted review strategy into `plan.json`, injecting
    specific `kb_references` file paths for context.
7.  **`/mantis-researcher` (Mantis Researcher):** Executes file-by-file triage
    and deep RTL bug reviews, outputting hotspots as individual JSON files in
    `workspace/findings/`.
8.  **`/mantis-dedupe` (Deduplicator):** Groups index-based duplicate findings,
    merging records and deleting redundancies within `workspace/findings/`.
9.  **`/mantis-review` (Validator):** Filters out false positives using strict
    pragmatic constraints tuned for RTL noise, updating the status in
    `workspace/findings/<id>.json`.
10. **`/mantis-critic` (Critic):** Verifies that a bug survives synthesis into
    real silicon (ignoring simulation-only artifacts and debug/DFT-gated logic),
    updates silicon viability in `workspace/findings/<id>.json`, and appends false
    positives/non-viable paths to `learnings.jsonl`.
11. **`/mantis-reproduce` (Proof-of-Concept Developer):** Writes testbenches,
    SystemVerilog Assertions (SVA), or formal properties, executes them in
    isolated simulation/formal environments, and updates reproduction status in
    `workspace/findings/<id>.json`.
12. **`/mantis-chain` (Bug Chainer):** Analyzes individual validated findings and
    knowledge base primitives to identify and construct complex multi-step bug
    chains, creating new "Super Findings" in `workspace/findings/`.
13. **`/mantis-patch` (Patcher):** Generates and applies RTL fixes, runs
    post-patch verification in the simulation/formal environment, updates patch
    status in `workspace/findings/<id>.json`, and appends logs to
    `learnings.jsonl`.
14. **`/mantis-calibrate` (Risk Calibrator):** Calculates a final numerical
    Mantis Risk Score (1-10) for each finding in the workspace directory based on
    impact, evidence, and viability, appending the results directly to each
    `workspace/findings/<id>.json` file.
15. **`/mantis-reflect` (Reflector):** Parses the execution trajectories of the
    agents from the current round, extracting false assumptions, tool failures,
    and successes, and appends these structured insights to the `learnings.jsonl`
    inbox.

______________________________________________________________________

## Prerequisites and Setup

Before executing any skills, ensure your local CLI environment is fully
configured. Because Mantis is intended to be platform agnostic we do not
recommend any specific software. We have used it with Gemini CLI and Antigravity
CLI, among others. Any coding agent framework should work. We have used these
skills successfully with both the Google ADK and Antigravity SDK. You might also
consider:

1. **An HDL simulator** (required for the `/mantis-reproduce` stage).
   Open-source options include **Verilator** and **Icarus Verilog**
   (`iverilog`); commercial options include Questa/ModelSim, VCS, and Xcelium.

   - **For Chisel designs**, also install a JDK and **`sbt`** so the generator
     can be elaborated. Chisel designs are typically exercised with
     **chiseltest** (ScalaTest driving Treadle or a Verilator backend) and can
     be elaborated to Verilog/FIRRTL, which you can then simulate with the tools
     above. Many open-source SoCs (Rocket Chip, BOOM, Chipyard) ship an
     sbt-based build for exactly this.
   - **A formal verification tool (recommended)**: For proving properties and
     generating counterexamples on hard-to-simulate bugs, install
     **SymbiYosys** (`sby`) with a backend solver, or use a commercial tool such
     as JasperGold.
   - **Docker (recommended for isolation)**: The reproducer executes
     AI-generated testbenches, which may contain file I/O, `$system` calls, or
     DPI-C/PLI code. Running your simulator inside a container (ideally with
     networking disabled, e.g. `--network none`) provides a reasonable boundary.
     For extra hardening when executing untrusted AI-generated harnesses,
     consider registering the `runsc` (gVisor) runtime in your Docker daemon
     configuration (`/etc/docker/daemon.json`):

2. **gVisor (runsc)**: For enhanced security when executing untrusted
   AI-generated crash reproducer code, register the `runsc` runtime in your
   Docker daemon configuration (`/etc/docker/daemon.json`):

   ```json
   {
     "runtimes": {
       "runsc": {
         "path": "runsc"
       }
     }
   }
   ```

   Restart Docker to apply: `sudo systemctl restart docker`.

3. **Relevant Cloud SDKs**: If running remote cloud sandboxes instead of local
   containers.

### Installing the Skills

You can install these skills either globally (available across all projects) or
locally to a specific workspace. You can also ask your coding agent for help.
Clone this repo then tell your coding agent you want to use these skills to
review your codebase!

To install the skills via CLI:

```shell
npx skills add google/mantis
```

______________________________________________________________________

## Beginner's Guide & Best Practices

If you are new to automated AI-assisted RTL design reviews, keep these
recommendations in mind:

### 1. "Interactive Mode" (Human-in-the-Loop)

*   **What it is:** For users or organizations not yet ready to deploy fully
    unattended, long-running pipelines, you should run the pipeline in an
    "Interactive Mode".
*   **How to use it:** Launch your CLI normally (e.g., type `agy` or `gemini` in
    your terminal). Then, from *inside* the interactive chat UI, type the slash
    commands (e.g., `/mantis-plan`) individually. Do not use `--yolo` or
    `--dangerously-skip-permissions` flags when launching the CLI.
*   **Why it matters:** The CLI will pause and prompt you for human approval
    before executing any sensitive command (especially when the
    `/mantis-reproduce` or `/mantis-patch` agents attempt to run simulator/formal
    invocations or write to files). This allows you to inspect what the AI intends
    to run. To run without human approval you will require stronger boundaries to
    keep the agents contained.

### 2. Hardened Isolation & The "No Host-Run" Rule

*   **Why it matters:** AI models can sometimes generate testbench or harness code
    that breaks the host (e.g., a SystemVerilog `$system()` call, aggressive file
    I/O, or a runaway DPI-C routine). Run generated harnesses *only* inside a
    contained simulation environment if you aren't reading them.
*   **Mantis Protection:** The `/mantis-reproduce` and `/mantis-patch` skills are
    explicitly instructed to execute reproducers inside isolated environments
    (e.g., a container with networking disabled) and to never point stimulus at
    real production silicon or shared lab equipment.
*   **Disclaimer:** While these instructions are designed to maintain isolation,
    **AI agents are non-deterministic**. They may occasionally attempt unsafe
    actions or bypass intended constraints if the local environment allows it.
    These instructions do NOT provide an absolute guarantee of safety. Always
    prioritize running this suite in a dedicated, isolated VM (see GCE section
    below) to provide a reasonable boundary that the AI cannot escape.

### 3. Model Choice & Tiered Efficiency

To maximize the speed and efficiency of your automated pipeline, you should
strategically pair the right AI model class with the specific task. You do not
need to use the heaviest, most advanced frontier models for every stage:

*   **Tier 1 (Triage & Deduplication):** For rapid classification sweeps (e.g.,
    Wave 1 of `/mantis-researcher`) or clustering similar text patterns
    (`/mantis-dedupe`), choose fast "flash" or "lite" tier models. These tasks do
    not require immense logic depth, just rapid text parsing, allowing you to
    parallelize massive file sweeps with zero bottleneck. Avoid models that are so
    low-powered they struggle with basic instructions, but don't slow your
    pipeline down by over-allocating intelligence here. Consider allowing the
    planner to specify a difficulty level for a given research task to allow
    targeting simpler questions at faster models, while allowing for some more
    complex design-bug discovery tasks to benefit from the most advanced frontier
    models.
*   **Tier 2 (Deep Reasoning):** Save your most powerful, heavy-reasoning flagship
    models for the highly complex stages that demand deep context and zero-shot
    problem solving: `/mantis-reproduce` (writing functional testbenches and
    formal properties) and `/mantis-patch` (writing side-effect-free RTL fixes).
*   **Tip:** For very large designs, configure your plan `/mantis-plan` to focus
    on specific high-risk blocks (e.g. `rtl/crypto/` or `rtl/dma/`) to keep the
    scan focused and efficient.

Try different tiers of models in different parts of your pipeline to see what
works well and what does not.

### 4. Understanding False Positives (The "Negative Filter" Rule)

AI-based scanning will produce
[false positives](README_AGENTS.md#understanding-false-positives-the-negative-filter-rule)
(the `/mantis-review` stage applies negative rules tuned for RTL noise to filter
them).

- **Expect Noise**: AI scanners can be overly enthusiastic. The `/mantis-review`
  stage runs a strict validator applying 12 negative rules tuned for RTL noise
  (e.g., ignoring pure lint style, provably tied-off bits, or hypothetical
  parent-module misuse). The 12 are by no means set in stone; adapt, reframe, or
  even split them out into a different stage of their own if it suits your use
  case.
- **Low/Hardening Risks Are NOT False Positives**: Effective risk calibration is
  critical as a first stage of triage. Take care when tuning your pipeline to
  ensure the difference between a false positive and something that is currently
  below the risk tolerance bar (e.g., a defense-in-depth hardening opportunity)
  does not negatively impact your ability to detect real design bugs.
- **Iterate Small**: Start with narrow-scope scans to tune the pipeline rather
  than running a repository-wide sweep on Day 1. Like the lint and CDC tools of
  old, AI-based design-bug scanning can be noisy; unlike rule-based tools, there
  are ways to tune this without creating highly complex waivers. It is far more
  efficient to run a small scan, triage a few items, and use that to feed back
  into constructing your scanning pipeline than to run a scan over an entire SoC
  and report every potential bug at once.

______________________________________________________________________

## Running the Pipeline (Manual Mode)

You can execute the reviewing stages sequentially from **inside** your active CLI
terminal.

1. Start your CLI from your terminal.

2. Inside the interactive UI prompt, type the skills sequentially:

   ```text
   # 0. (Optional) Analyze the design's version control system (VCS) history and extract past design bugs
   /mantis-history

   # 1. (Optional) Generate mantis-summary.md directory maps
   /mantis-summarize

   # 2. Synthesize design structure and historical learnings into the Markdown Knowledge Base
   /mantis-architecture

   # 3. Iteratively develop the design's living threat model based on the KB
   /mantis-threat-model

   # 4. Map target interface boundary and build scanning roadmap, injecting KB references
   /mantis-plan

   # 5. Run multi-threaded/sequential RTL design-bug sweep using injected context
   /mantis-researcher

   # 6. Consolidate overlapping files and duplicate bugs
   /mantis-dedupe

   # 7. Verify RTL validity & filter false positives
   /mantis-review

   # 8. Eliminate non-viable (simulation-only / debug-gated) issues
   /mantis-critic

   # 9. Generate testbenches/assertions/formal properties and run them in simulation/formal
   /mantis-reproduce

   # 10. Combine validated individual findings into multi-step bug chains
   /mantis-chain

   # 11. Apply minimal RTL fixes and verify they block the reproducer
   /mantis-patch

   # 12. Calculate final matrix risk ratings and append to individual findings
   /mantis-calibrate

   # 13. Extract insights from execution trajectories and append to the learnings inbox
   /mantis-reflect

   # 14. Generate human-readable security review packet report
   /mantis-report

   # 15. (Manual Step) Review the report. Optionally, you can apply & commit approved patches to your codebase. To continue analysis, archive workspace/findings/, and loop back to Step 2 to start the next pass.
   ```

______________________________________________________________________

## Building Deterministic Pipelines & Non-Determinism

While the `/mantis-meta-agent` provides dynamic steering for exploratory design
research, we highly recommend wrapping the Mantis Skills in a **deterministic
programmatic pipeline** (e.g., Python, Bash, Rust, or CI/CD workflow) for use in
enterprise or production settings.

By treating the individual skills (like `/mantis-researcher`, `/mantis-review`,
and `/mantis-reproduce`) as microservices that read and write JSON state in the
`workspace/findings/` directory, you can build a rigid orchestrator that provides
absolute reliability and strict isolation guarantees. Better yet, you should use
more durable and resilient databases instead of json files on a single machine.

**Before building your harness, strictly adhere to the inter-stage data contracts
defined in [schema.json](schema.json).** Of course, you can also modify the schema
based on what works for you. There are few hard rules here.

### The Pipeline Adapter Skill (/mantis-pipeline-adapter)

To get started on brainstorming your custom pipeline for high reliability, token
efficiency (such as using UUID-based referencing), and adaptability to custom
environments (via MCP), see the
[Pipeline Adapter Guide](mantis-pipeline-adapter/SKILL.md).

> **Note on Standalone vs. Harness Mode:** When using Mantis Skills directly from
> the CLI in standalone mode, skills like `/mantis-review` or `/mantis-patch`
> will instruct the LLM to write temporary reusable Python scripts to update the
> JSON state files. However, in a true programmatic harness, your orchestrator
> should override these instructions and provide native tool calls or functions
> for state management to avoid forcing the LLM to write one-off scripts.

### Why Build a Programmatic Harness?

*   **Determinism:** Some stages such as the reproduction agent or patch agent
    include recommendations to have subagents criticize the reproducer or patch.
    While it is reasonable to demonstrate the overall workflow, a more
    deterministic critic stage that the agent cannot bypass by "forgetting" to
    call the critic subagent will likely produce better results.
*   **Mitigates Prompt Injection Risk:** An LLM orchestrating shell/EDA commands
    is susceptible to host-level prompt injection if it ingests malicious content
    (e.g., a comment planted in a third-party IP file). Moving the orchestration
    to a hardened deterministic pipeline removes the LLM's control over the host
    environment.
*   **Enforces Strict Sandboxing:** Rather than relying on the LLM to remember to
    contain a simulation run, your deterministic harness can programmatically
    enforce that untrusted AI-generated testbenches are executed exclusively
    within a locked-down VM, container, or gVisor sandbox.
*   **CI/CD Integration:** A deterministic script executing the static analysis
    and deduplication stages is predictable and easily integrated into standard
    automated workflows like GitHub Actions or Jenkins, alongside your existing
    lint/CDC/regression jobs.
*   **Scale:** The pipeline can be decomposed into several pieces, allowing you to
    scale horizontally across a suitably sized fleet during periods of low
    utilization.
*   **Deterministic Reporting:** While the pipeline relies on machine-readable
    JSON files (`workspace/findings/*.json`) to safely maintain internal state, a
    programmatic harness can deterministically translate these JSON findings into
    human-readable Markdown reports or automatically file them into bug-tracking
    systems without risking LLM hallucination or state corruption. Only use an LLM
    for deterministic subsets of this reporting process, such as providing an
    executive summary if necessary.

### The Hybrid Approach

To maintain the dynamic, adaptive nature of the suite while ensuring
deterministic execution, you can build a pipeline that:

1.  **Iterates Programmatically:** A harness loops over the workspace, invoking
    the static and dynamic skills via the CLI.
2.  **Feeds Learnings Back:** The harness takes the resulting `learnings.jsonl`
    file and invokes `/mantis-plan` to generate a newly updated `plan.json`,
    effectively allowing the AI to guide the deterministic runner on what to
    analyze next.
3.  **Hardcodes the Execution Sandbox:** You can optionally configure the
    deterministic versions of `/mantis-reproduce` and `/mantis-patch` to *only
    generate* the patch or testbench file, leaving the actual simulation/formal
    execution and grading to your harness in a strictly controlled sandbox.

--------------------------------------------------------------------------------

## The Reality of Non-Determinism

A critical concept to understand when using AI for design research is
**Non-Determinism**.

*   **Coverage is not an absolute guarantee:** Even though Stage 2
    (`/mantis-plan`) attempts to use programmatic shell scripts to map your entire
    design, the agent running those scripts is fundamentally non-deterministic. It
    might occasionally fail to run the script correctly, hallucinate parameters,
    or skip modules.
*   **Trajectory/Conversation analysis:** One way to mitigate the lack of
    determinism is to programmatically review all the tool calls made by the
    agents to see what they've done. This can be used to calculate coverage and
    efficiency metrics, although what those numbers mean exactly we will leave to
    your imagination.
*   **Reasoning shifts across loops:** Because the LLM's analysis is
    non-deterministic, it may miss a subtle CDC hazard or access-control bypass on
    Pass 1 but identify it clearly on Pass 5 as its internal "attention" shifts or
    as it gains context from other findings. This is why we generally recommend
    running this scanning pipeline many times.
*   **Diminishing Returns:** You might expect the pipeline to eventually "finish"
    and stop reporting bugs. In reality, the discovery of findings often does not
    stop completely; rather, the *quality and severity* of the findings will
    eventually degrade as the LLM starts hallucinating or reaching for pedantic
    non-issues.

The continuous loop is designed to leverage this non-determinism allowing the AI
multiple passes to catch things it missed. However, **it is up to each user to
experiment with the suite, review the Risk Calibrator scores on the findings, and
determine for themselves when the quality of findings has dropped enough to pause
the loop.** In the long term you will also have to determine how often to rescan,
such as when new models with greater capabilities are made available or when a
design has received sufficiently large changes to warrant a complete rescan
instead of just an analysis of a given diff or changelist.

______________________________________________________________________

## Advanced / Unattended Cloud Deployment (GCE)

Running the continuous review loop 24/7 in a fully autonomous, unattended state
presents unique security risks, particularly **host-level prompt injection**.
Beyond this, agents might simply make mistakes and perform actions you did not
intend.

**As a result, deploying to a hardened VM such as an isolated Google Compute
Engine (GCE) instance is a STRICT REQUIREMENT for unattended mode.**
(Alternatively you can build a more structured deterministic pipeline where
individual risky actions are sandboxed, although this will require more up front
effort).

### 1. Hardened GCE Environment

To provide a security boundary that an AI agent cannot easily escape, you MUST
configure your environment as follows:

*   **Network Isolation:** Provision the GCE VM with **no external internet
    access**, or at least use a secure web proxy with a trusted allowlist and good
    rate limiting and egress controls. This is especially important when the
    design under review includes proprietary or third-party IP.
*   **VPC Service Controls (VPC-SC):** Place the VM inside a VPC-SC perimeter.
    This is an important defense against exfiltration of design IP if an agent is
    compromised.
*   **Least-Privilege Service Account:** Attach a dedicated IAM Service Account to
    the VM with strictly limited roles. Do *not* use broad roles like
    `roles/aiplatform.user` or `roles/storage.objectAdmin`. Instead:
    *   **Custom AI Role:** Create a custom IAM role that *only* grants
        `aiplatform.endpoints.predict` and
        `aiplatform.endpoints.generateContent`. This restricts the agent to only
        query models and prevents modifying AI infrastructure.
    *   **Append-Only GCS Storage:** To store intermediate results or backups,
        grant the service account `roles/storage.objectCreator` and
        `roles/storage.objectViewer` to a specific GCS bucket. **Crucially, do not
        grant delete permissions (`storage.objects.delete`).** Also consider other
        append-only storage mechanisms.
    *   **GCS Versioning:** Enable Object Versioning on the GCS bucket. This
        provides a mechanism so that even if the AI or an untrusted testbench
        payload overwrites a file (like `learnings.jsonl`), previous states are
        preserved as non-current versions, preventing the AI from permanently
        deleting the history.

### 2. Bypassing Interactive Prompts (Unattended Mode)

**Warning:** Only use these flags if the **Hardened GCE Environment** (above) is
fully implemented. By default, the CLI tools require manual confirmation before
executing system commands. To run the pipeline entirely unattended, you must pass
the appropriate auto-approve flag when starting the CLI, such as
`--dangerously-skip-permissions` or `--yolo`.

### 3. Automated Design-Bug Alerting (Cloud Pub/Sub)

When running unattended, you might desire an isolated way to be notified when the
pipeline discovers a high-confidence design bug. There are numerous ways to do
this, including connecting the pipeline to **Google Cloud Pub/Sub**.

1.  **Setup:** Create a Pub/Sub topic (e.g., `mantis-verified-bugs`) and grant
    your GCE VM's Service Account the `roles/pubsub.publisher` role.
2.  **Hooking it up:** The `/mantis-meta-agent` skill can be instructed to trigger
    notifications natively. You can instruct the meta-agent to run `gcloud pubsub
    topics publish mantis-verified-bugs --message="$(cat
    workspace/findings/<id>.json)"` whenever a design bug is successfully
    reproduced.
3.  **Routing:** Subscribe a Google Cloud Function or Cloud Run service to that
    Pub/Sub topic to route the alert payload directly into your team's chat, issue
    tracker, or paging system. This cleanly decouples the isolated scanning
    environment from your internal alerting infrastructure.

______________________________________________________________________

## Meta-Agent Orchestration & Evaluation

For a truly autonomous and persistent design-review operation, you can employ the
**Meta-Agent Orchestration** pattern by invoking the `/mantis-meta-agent` skill.
In this setup, a high-level "Meta-Agent" (a long-lived Gemini or Antigravity CLI
session) is responsible for driving the entire reviewing pipeline.

### The Meta-Agent's Role:

*   **Orchestration:** The Meta-Agent manages the execution of each stage natively
    using CLI subagent delegation.
*   **Persistence:** It operates in a single, long-lived conversation that spans
    days or weeks, ensuring that the review continues working towards the goal of
    design-bug discovery, patching, and reporting even while you are in meetings,
    away for the evening, or over the weekend.
*   **Supervision:** It keeps an eye on the task, handles minor environmental
    hiccups, reads logs, and ensures the pipeline remains operational.
*   **Interactive Steering:** A major advantage of this pattern is that you can
    chat with the Meta-Agent while subagents are working. You can ask for status
    updates, collaboratively debug environment issues, or provide high-level
    strategic guidance (e.g., "Deep dive on the CDC boundaries in the DMA engine")
    to influence the swarm's focus in real-time or in the next loop.
*   **Security Boundaries:** While you can run the Meta-Agent with auto-approve
    flags (`--dangerously-skip-permissions`), you must strictly confine it within
    the hardened security boundaries previously described (VPC-SC, no external
    internet, and restricted IAM roles).

This pattern transforms the suite from a set of disjointed tools into a
continuous, self-driving design-review operation.

--------------------------------------------------------------------------------

## Evaluating and Optimizing Mantis Skills

Evaluating an autonomous, multi-agent pipeline like Mantis is notoriously
difficult. Running full end-to-end evaluations for every prompt tweak is
cost-prohibitive in both time and API tokens. To safely modify these skills or
optimize model costs, you should adopt a **Tiered Evaluation Strategy** and
measure proxy metrics rather than just binary success.

### The Tiered Evaluation Strategy

Do not evaluate the entire loop unless necessary. Split your evaluations into
three tiers:

1.  **Tier 1: Static Checks**

    *   **What it is:** Fast, programmatic linting of the skill files.
    *   **What to measure:** Do the `SKILL.md` files parse? Are the YAML
        frontmatters correct? Do they define the required tools? Are the system
        prompts within the context window limits?

2.  **Tier 2: Isolated "Unit" Evals**

    *   **What it is:** Evaluating a single skill (e.g., `/mantis-patch`) in a
        vacuum, entirely decoupled from the rest of the pipeline.
    *   **The Setup:** Feed a static, hardcoded input (a mocked `findings.json`
        and a target RTL file) to a single skill and observe its output.
    *   **What to measure:**
        *   **Format:** Did it output the expected JSON schema or valid diff?
        *   **Tool Use:** Did it attempt to call the correct tools (`run_command`
            vs `view_file`)?
        *   **LLM-as-a-Judge:** Use a cheaper, faster model to grade the
            qualitative output with a strict rubric (e.g., "Did the patch add a
            correct two-flop synchronizer on the CDC path? Yes/No.").

3.  **Tier 3: The "Golden Dataset" End-to-End Eval**

    *   **What it is:** A full run of the entire pipeline. Only run this when doing
        a major release or swapping base model classes (e.g., upgrading to a newer
        flagship model).
    *   **The Setup:** Curate a tiny dataset of 3-5 real-world, representative
        buggy RTL designs (e.g., designs with known CDC, FSM, or access-control
        bugs).
    *   **What to measure:** Binary outcomes. Did the final regression pass? Did
        `/mantis-reproduce` generate a working testbench/property that triggers
        the bug? You could also perform human evaluation to see if there were
        novel design bugs discovered.

### Measuring the "Unmeasurable"

When evaluating intermediate stages (like `/mantis-researcher`), binary success is
difficult to define. Instead, track these proxy metrics to gauge skill
degradation:

*   **Tool Error Rate:** Count how many times the agent's tool calls fail (e.g.,
    bad bash syntax, invalid file paths, un-elaboratable testbenches). A spike in
    tool errors after a prompt change indicates the skill's instruction set has
    degraded or that the prompts might need to be adapted to a new model or coding
    agent harness.
*   **Trajectory Efficiency (Turns/Tokens):** If `/mantis-reproduce` used to write
    a testbench in 5 turns, and after a prompt tweak it takes 150 turns or loops
    repeatedly, that is a measurable regression in efficiency.
*   **The "Give Up" Rate:** How often does the agent explicitly output phrases
    like "I cannot determine", "I am stuck", or enter an infinite loop before
    hitting a token limit?

### The "Shadow Eval" Method

Do not build a massive evaluation harness on day one. Instead, build your dataset
organically:

1.  When running the pipeline manually, wait for the agents to fail at a specific
    task.
2.  Save that exact starting state (the user prompt, the workspace files, the JSON
    state).
3.  Fix the skill prompts until the agent succeeds.
4.  Turn that specific, isolated state into your first automated test.

By building your eval dataset exclusively from real-world failures, you ensure you
are only spending tokens testing regressions that actually matter.

#### Optimizing Parallelism and Model Selection

When tweaking the pipeline or introducing features like Parallel Trajectory
Search, you should run experiments to ensure you are getting a return on your
token investment:

*   **Try Different Models:** For any given stage, experiment with swapping the
    flagship model for a cheaper, faster model or a specialized coding model. Use
    the Tier 2 "Unit" Evals to verify if the cheaper model degrades the success
    rate before rolling it out.
*   **Evaluate Parallel Trajectories:** If you implement parallel trajectory
    search (e.g., spawning multiple `Researchers` or `Patchers`), test different
    numbers of concurrent agents (e.g., 2, 3, or 5). If running parallel
    researchers always results in them finding the exact same design bugs, then
    the parallelization is not yielding unique value and is just burning tokens.
    Conversely, if parallel patchers consistently produce a much cleaner, more
    idiomatic fix than a single agent, the compute cost can be justified.

______________________________________________________________________

## Roadmap / Future Work

- **Continuous Pipeline:** The current pipeline is designed to be run as a
  point-in-time review of a design, and not as something that is intended to
  regularly sync with upstream RTL changes mid-run. It should be straightforward
  (if a little tricky) to tweak the pipeline to better support this, but it
  probably will not work today.
- **Skill Self-Improvement (Meta-Learning):** The current
  `workspace/learnings.jsonl` and Knowledge Base (KB) architecture tracks
  design-specific empirical outcomes to adapt the `THREAT_MODEL.md` and context
  pointers. Future iterations of the pipeline could take this a step further and
  use this historical data to reflect on and automatically rewrite its own
  `SKILL.md` prompts. For example, if a certain type of hallucination is
  repeatedly caught by the Critic, a self-improvement meta-agent could update the
  Researcher's `SKILL.md` instructions to explicitly filter out that specific
  pattern before it even reaches the Review stage. **Security Note:** Committing
  automated changes to `SKILL.md` files must always be human-gated to prevent an
  untrusted agent from using prompt injection (e.g., via a malicious payload in a target
  IP file) to trick the meta-agent into ignoring a bug class globally.
- **Silicon Dark Factory:** Integrate this pipeline into an entirely AI-driven
  hardware development flow. Instead of design-bug discovery for action by
  humans, Mantis would become the autonomous design-verification and tapeout
  gating component. Before the dark factory can sign off on a design, it must
  have had N hours of adversarial design-bug research or "red teaming" by a
  pipeline like Mantis.

______________________________________________________________________

## Troubleshooting Guide

### 1. Loop Iterations are Re-Evaluating the Same RTL

- **Symptom:** The loop keeps reviewing the same files and reporting identical
  bugs.
- **Solution:** Ensure `/mantis-architecture` completes successfully and writes
  its synthesized knowledge to the `workspace/kb/` directory. The `/mantis-plan`
  strategist checks this Knowledge Base to dynamically skip already analyzed
  areas. Check that file permissions allow writing to `workspace/kb/`.

### 2. Other Issues

- **Symptom:** Something isn't working.
- **Solution:** Ask an AI coding tool to review your pipeline and the
  conversations or trajectories that are leading to the unexpected behavior.
  They will often give you useful insights.

______________________________________________________________________

## Attribution & Disclaimer

This project is a fork of Google's
[Mantis](https://github.com/google/mantis), retargeted from software security
review to hardware RTL design-bug review. All credit for the original
architecture, pipeline design, and skill structure belongs to the upstream Mantis
authors.

This fork is not affiliated with or endorsed by Google, and (like the upstream
project) is not an officially supported Google product. This project is intended
for demonstration purposes only. It is not intended for use in a production
environment.
