# No Test Cases, No Problem: Distillation-Driven Code Generation for Scientific Workflows

- **Source**: https://arxiv.org/abs/2604.23106
- **Date**: 2026-04-25
- **Authors / Org**: Siddeshwar Raghavan, Tanwi Mallick
- **Type**: arxiv

## Core Problem

Existing multi-agent LLM frameworks for code generation rely on execution feedback from I/O test cases, which are unavailable in scientific workflows because generating such test cases requires solving the original problem. This gap prevents effective iterative improvement for scientific code generation.

## Core Approach

1. **Student-Teacher Knowledge Distillation**: Instead of execution feedback, MOSAIC uses a student-teacher framework where the teacher provides domain-specific examples and structured problem decomposition to guide the student's code generation, grounding the process without I/O test cases.

2. **Consolidated Context Window (CCW)**: To maintain consistent reasoning across multiple agents handling chained subproblems, CCW consolidates context from previous steps, reducing hallucinations and ensuring coherence in the generated code.

## Evidence

Experiments on the SciCode benchmark show that MOSAIC improves accuracy, executability, and numerical precision over existing approaches while relying on lightweight models. Specific numerical results are not provided in the abstract.

## Assumptions & Open Questions

- ⚠️ The approach assumes that domain-specific examples and structured problem decomposition are sufficient to replace execution feedback, but this may not hold for highly novel or underspecified scientific problems.
- ⚠️ The paper does not quantify the overhead or scalability of the Consolidated Context Window, leaving open questions about its effectiveness with very long or complex workflows.
- ⚠️ The reliance on lightweight models may trade off performance for efficiency; it is unclear how MOSAIC compares to larger, more capable models in terms of code quality.
- ⚠️ The abstract does not discuss failure cases or scenarios where distillation fails to ground generation, such as when domain examples are scarce or ambiguous.
