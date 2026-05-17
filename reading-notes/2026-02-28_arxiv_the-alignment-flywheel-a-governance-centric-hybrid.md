# The Alignment Flywheel: A Governance-Centric Hybrid MAS for Architecture-Agnostic Safety

- **Source**: https://arxiv.org/abs/2603.02259
- **Date**: 2026-02-28
- **Authors / Org**: Elias Malomgré, Pieter Simoens
- **Type**: arxiv

## Core Problem

Current autonomous systems, especially those using learned or generative models, have safety behaviors that are opaque, difficult to audit, and costly to update after deployment because safety is entangled with training. There is a lack of architecture-agnostic frameworks that allow runtime safety enforcement and iterative refinement without retracting or retraining the decision-making component.

## Core Approach

1. **Proposer-Safety Oracle Separation**: The Proposer (any autonomous decision component) generates candidate trajectories, while the Safety Oracle returns raw safety signals through a stable interface, decoupling generation from governance.

2. **Enforcement Layer with Explicit Risk Policy**: A runtime enforcement layer applies explicit, version-controlled risk policies to gate or modify actions based on the Safety Oracle's signals.

3. **Governance MAS for Oracle Supervision**: A governance multi-agent system audits the Safety Oracle, performs uncertainty-driven verification, and manages versioned refinement of the oracle artifact.

4. **Patch Locality Principle**: Safety failures are mitigated by updating the governed oracle artifact and its release pipeline, rather than retracting or retraining the Proposer, enabling rapid, localized fixes.

5. **Implementation-Agnostic Protocols**: The architecture specifies roles, artifacts, protocols, and release semantics (e.g., runtime gating, audit intake, signed patching, staged rollout) without depending on specific Proposer or Oracle implementations.

## Evidence

No quantitative empirical results are provided in the abstract. The paper presents a formal architecture and engineering framework without benchmarks or numerical claims.

## Assumptions & Open Questions

- ⚠️ The abstract assumes that a Safety Oracle can provide reliable raw safety signals through a stable interface, but does not address how to construct such an oracle for complex, open-ended tasks or how to guarantee its stability across updates.
- ⚠️ The patch locality principle assumes that most safety failures originate from the oracle or its policy, not from the Proposer's inherent capabilities—this may not hold for failures caused by emergent Proposer behaviors that the oracle cannot detect.
- ⚠️ The architecture relies on a governance MAS to audit and refine the oracle, but the abstract does not discuss how to prevent the governance system itself from being compromised or how to ensure its own safety and correctness.
- ⚠️ No discussion of computational overhead, latency, or scalability of the runtime enforcement layer, which could be critical for real-time or resource-constrained deployments.
- ⚠️ The approach assumes that version-controlled, signed patching and staged rollout are sufficient for safety, but does not address how to handle zero-day vulnerabilities or adversarial attacks that bypass the oracle entirely.
