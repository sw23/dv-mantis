# Calibration Rules Catalogue

This document defines the 27 calibration sanity triage rules (caps and
downgrades) used to calculate the final severity and priority of findings.

## Table of Contents

- [Core Principle: Marginal Capability](#core-principle-marginal-capability)
- [Category A: Force-Downgrade to LOW (Cap at 2.0 / LOW Priority)](#category-a-force-downgrade-to-low-cap-at-20--low-priority)
- [Category B: Force-Cap to HIGH (Cap at 7.9 / Maximum HIGH Priority)](#category-b-force-cap-to-high-cap-at-79--maximum-high-priority)
- [Category C: Force-Cap to MEDIUM (Cap at 5.9 / Maximum MEDIUM Priority)](#category-c-force-cap-to-medium-cap-at-59--maximum-medium-priority)

## Core Principle: Marginal Capability

The final severity and priority of a finding are strictly bounded by the
**marginal capability** gained by the untrusted master/agent over its
prerequisite position on the design's real interfaces. If triggering the bug
does not grant that agent significant new control, access, or state-corruption
capability beyond what is already inherent to its starting position (or already
reachable through the design's legitimate bus/pin/register interfaces), the
finding must be capped or downgraded.

______________________________________________________________________

### Category A: Force-Downgrade to LOW (Cap at 2.0 / LOW Priority)

01. **`repro_failure` (Reproduction Failure or Not Attempted)** The reproduction
    failed (`repro_status: "failed_to_reproduce"`), was not attempted
    (`repro_status: "not_attempted"`), or the `repro_status` field was missing
    (treated as `"not_attempted"`), regardless of theoretical production
    viability.

02. **`unreachable_inputs` (Unreachable / Uncontrolled Inputs)** The finding
    relies on stimulus that is documented as highly unlikely to be drivable onto
    the module's inputs, and no path from a real bus/pin interface (a trust
    boundary) is proven.

03. **`third_party_reachability` (Third-Party / Supply Chain Reachability)** Bugs
    in third-party IP blocks (vendor IP with known errata) where a reachable
    path from a real design input to the buggy logic has not been actively
    demonstrated in simulation or formal.

04. **`minor_config_hygiene` (Minor Configuration Hygiene)** Minor deviations
    from best practice (e.g., slightly loose default register values, an
    optional-but-missing guard on a low-value internal bus) without a clear
    trigger path.

05. **`non_security_critical` (Non-Security Critical Components)** The finding
    affects a block or state with no functional-correctness or security
    sensitivity (e.g., debug status registers, cosmetic status/LED outputs,
    non-functional observability logic).

06. **`vague_code_paths` (Vague Code Paths / Fragile Assumptions)** Relying on
    unverified assumptions about neighboring block behavior or adjacent RTL that
    have not been proven along a concrete path.

07. **`unreliable_triggers` (Unreliable/Noisy Triggers)** Triggers that are
    unlikely to occur in practice or indistinguishable from normal operation
    (e.g., a glitch that requires an input combination that never appears in
    legal bus traffic).

08. **`prerequisite_shell` (Prerequisite Shell Access / Equivalent Primitives)**
    The untrusted master already possesses the **same or higher** privilege
    access to the target block through a legitimate interface than triggering
    the bug provides, rendering the gained access redundant under the Principle
    of Marginal Capability (e.g., using a bug to obtain register read/write it
    can already perform via a legitimate CSR access, or using an FSM glitch to
    reach a state it can already command from its authoritative bus). This does
    NOT apply to low-to-high privilege escalation (e.g., a low-privilege agent
    reaching a higher-privilege domain), which should cap at MEDIUM.

09. **`physical_long_term` (Physical Long-Term / Laboratory Access)** If the
    attack requires long-term physical access to the device or specialized
    laboratory equipment (e.g., fault injection, side-channel analysis, chip
    decapping). Force-downgrade to **LOW (2.0)** due to the extreme execution
    barrier and requirement for physical possession.

10. **`trusted_controller_zero_delta` (Trusted-Controller-Mediated Interface -
    Zero Delta)** If the vulnerable interface is reachable only from a block
    that holds designed-in authoritative control over the target (e.g.,
    configuration master->peripheral, debug/config master->block, protocol
    master->slave, secure controller->power domain, firmware->IP block), and
    triggering the bug grants **zero marginal capability** (i.e., the controller
    could already achieve the identical effect via its standard, legitimate
    interface), force-downgrade to **LOW (2.0)**. (This generalizes the
    *Standard Host-to-Guest Attacks* rule below).

11. **`standard_host_attacks` (Standard Host-to-Guest Attacks)** If the
    `attacker_position` is `HOST_SYSTEM` (the SoC host/firmware acting on a peer
    block) on standard, non-isolated deployments. Force-downgrade to **LOW
    (2.0)** as the host already possesses total designed-in control over the
    peer block, meaning the bug offers zero marginal capability over the
    prerequisite position (equivalent primitives). **Default assumption:** treat
    as non-isolated (this rule fires) UNLESS the Threat Model, RTL path, or
    finding description explicitly names a secure/isolated domain, secure
    enclave, TrustZone secure world, root-of-trust, or attestation (in which
    case apply the Confidential Computing Host Attacks cap-HIGH rule instead).

______________________________________________________________________

### Category B: Force-Cap to HIGH (Cap at 7.9 / Maximum HIGH Priority)

1. **`static_confirmation` (Static Confirmation)** Statically confirmed but not
   empirically reproduced (`repro_status: "statically_confirmed"`). Cap
   `likelihood_score` at **3**, apply **0.8** multiplier to Hazard, and MUST NOT
   be CRITICAL. *Exception:* If the finding details (description, history, or
   reproduction output) include a valid simulation waveform/log, an assertion
   (SVA) failure trace, a formal counterexample (CEX), or an X-propagation
   report proving the bug was actually triggered in execution (e.g., in a prior
   simulation run or by an external verification tool), treat it as empirically
   reproduced (Likelihood 5) and do not apply this static cap.

2. **`strict_xss` (Strict XSS Caps)** A transient, easily-contained functional
   glitch class (e.g., a single-cycle spurious output or a benign X-glitch that
   is normally masked or does not reach a live output). Default to MEDIUM or
   LOW; cap at HIGH (7.9) only when the glitch is latched into a critical
   control register on a security- or safety-critical block with no masking or
   downstream correction.

3. **`internal_nested` (Internal / Nested Components)** Any finding with a
   Network/Trust Exposure multiplier less than 1.0 (i.e., an Internal Block or
   Privileged Zone). If the calculated score lands in the CRITICAL range, cap
   the score at **7.9** and downgrade the priority to HIGH. *Exception:* Do NOT
   cap at HIGH if the block is core interconnect/infrastructure IP (e.g.,
   interconnect fabric, memory controller, secure bridge, arbiter) AND the
   impact escapes to the host domain (e.g., host-domain memory R/W) or allows
   cross-domain/cross-master escalation. These remain eligible for CRITICAL.
   **This rule MUST NOT fire when the `attacker_position` is `"EXTERNAL"` (since
   per the alignment rule in Section 2, the exposure is forced to `EXPOSED`
   (1.0), which precludes this cap).**

4. **`probabilistic_llm` (Probabilistic LLM Vectors)** Bugs that depend on
   non-deterministic or probabilistic timing conditions (e.g., a metastability
   window or a race that only manifests probabilistically) to trigger. Cap at
   **HIGH** (7.9) and default to **MEDIUM** or **LOW**. *Exception:* If the
   untrusted agent can drive the offending input repeatedly without rate,
   concurrency, or protection limits that would impede the attempt (allowing it
   to reliably hit the timing/metastability window and effectively eliminate the
   non-determinism), this cap may be lifted.

5. **`supply_chain_prerequisites` (Supply-Chain / Build-Time Prerequisites)** If
   triggering the bug requires the agent to already possess a supply-chain
   position (e.g., the ability to modify upstream RTL source or the IP netlist)
   or write access to the synthesis/build (bitstream) pipeline. Cap at **HIGH
   (7.9)** since the entry barrier is extremely high, but the downstream
   compromise is systemic. (Force-downgrade to LOW/2.0 only if it already
   possesses equivalent legitimate access to the target, as per the Prerequisite
   Shell Access rule).

6. **`non_default_config` (Non-Default Configurations)** Findings that are only
   exploitable under non-default configurations (e.g., non-default register,
   strap, or parameter settings). Cap at **HIGH (7.9)** to reflect the
   additional configuration barrier.

7. **`confidential_computing_host` (Confidential Computing Host Attacks)** If
   the `attacker_position` is `HOST_SYSTEM` (the SoC host/firmware acting on an
   isolated secure domain — a secure enclave, TrustZone secure world, or
   root-of-trust) in an isolated-domain deployment. Cap at **HIGH (7.9)**
   because while the host has full control of the platform, the isolated secure
   domain is specifically designed to protect against host-level compromise.
   (If not an isolated-domain deployment, see the Standard Host-to-Guest Attacks
   rule under LOW).

8. **`trusted_controller_critical_bypass` (Trusted-Controller-Mediated Interface
   \- Critical Bypass)** If the vulnerable interface is reachable only from a
   designed-in authoritative controller, and triggering the bug allows that
   controller to bypass target-side **documented security controls** (e.g.,
   lock-bits, write-protect, secure/non-secure isolation) or **safety-of-life
   limits** (e.g., hard clamp/threshold limits) it was designed to respect, cap
   at **HIGH (7.9)**. (If the bug allows lateral reach into a different trust
   domain or achieves persistence surviving controller re-provisioning, do not
   cap).

______________________________________________________________________

### Category C: Force-Cap to MEDIUM (Cap at 5.9 / Maximum MEDIUM Priority)

1. **`local_attack_vector` (Local Attack Vector)** Bugs reachable only from a
   same-domain local agent already inside the block or domain (e.g., an on-die
   privileged agent performing local privilege escalation) without crossing an
   isolation boundary into another domain (no domain escape). (Downgrade to
   LOW/2.0 if it only affects a single agent's own isolated registers or data).

2. **`self_contained_blast` (Self-Contained Blast Radius)** If the maximum
   impact of the bug is confined to registers, memory, or state that the
   triggering agent already owns or has full designed-in authority over — its
   own domain, block, register bank, private buffer, or single-agent context —
   and does not cross any isolation boundary between mutually-distrusting
   agents, cap at **MEDIUM (5.9)**.

   - The bug may grant genuinely new capability within that domain (e.g., a
     config agent reaching a state within its own block), but the design's core
     isolation guarantees to other parties still hold.
   - Do **NOT** apply this cap if the bug:
     - reaches another agent's resources (cross-domain, cross-master,
       cross-context),
     - touches shared or multi-party infrastructure (shared cache, shared bus,
       interconnect fabric, arbiter, co-domain side-channel),
     - places the agent's domain upstream of others (configuration master, scan
       chain, build/bitstream flow — i.e., a supply-chain position), or
     - persists in a way that survives the agent's own resource lifecycle and
       could later affect a different agent reusing that slot.

3. **`rarely_exposed` (Rarely Exposed Components)** Findings in blocks
   documented as 'rarely exposed' or 'unlikely to be drivable through a real
   interface'.

4. **`equivalent_primitives` (Equivalent Primitives - No Boundary Breach)** The
   agent profile capable of triggering the bug already possesses equivalent
   access, privileges, or capabilities (primitives) through standard design
   features (e.g., a configuration master reading a register through a bug that
   it can already read via a legitimate CSR access). Because this offers low
   marginal capability over its prerequisite position, cap at **MEDIUM (5.9)**
   to maintain visibility for defense-in-depth cleanup.

5. **`documented_insecure_config` (Documented Insecure Configurations)**
   Non-default configurations that are explicitly documented in public manuals
   as insecure, diagnostic-only, or strictly non-production (e.g.,
   debug/test-mode straps, scan-enable, factory bring-up modes). Cap at **MEDIUM
   (5.9)**.

6. **`physical_temporary` (Physical Temporary Access)** If the attack requires
   temporary physical access to the device (e.g., toggling a debug/JTAG jumper,
   brief scan/probe access, evil-maid tampering) without long-term laboratory
   analysis. Cap at **MEDIUM (5.9)**.

7. **`high_privilege_external` (High-Privilege External Access)** Bugs with
   `attacker_position: "EXTERNAL"` that require `privileges_required: "HIGH"`
   (e.g., an authenticated JTAG/debug port or a privileged bus master reachable
   through an external interface). Cap at **MEDIUM (5.9)**, unless triggering the
   bug results in escaping the block boundary (into the host domain) or
   cross-domain escalation.

8. **`trusted_controller_standard_bypass` (Trusted-Controller-Mediated Interface
   \- Standard Bypass)** If the vulnerable interface is reachable only from a
   designed-in authoritative controller, and triggering the bug allows that
   controller to bypass target-side **standard safety or sanity limits** (but
   not critical safety-of-life limits or documented security controls) it was
   expected to respect, cap at **MEDIUM (5.9)**.
