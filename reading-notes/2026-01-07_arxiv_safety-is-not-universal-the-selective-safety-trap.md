# Safety Is Not Universal: The Selective Safety Trap in LLM Alignment

- **Source**: https://arxiv.org/abs/2601.04389
- **Date**: 2026-01-07
- **Authors / Org**: Iago Alves Brito, Walcy Santos Rezende Rios, Julia Soares Dollis, Diogo Fernandes Costa Silva, Arlindo Rodrigues Galvão Filho
- **Type**: arxiv

## Core Problem

Current safety evaluations aggregate harms under generic categories, creating an illusion of universal protection while obscuring vulnerabilities specific to underrepresented populations. This work addresses the gap in understanding and measuring how safety alignment varies across demographic groups.

## Core Approach

1. 1. MiJaBench construction: A bilingual (English-Portuguese) adversarial benchmark with 43,961 controlled jailbreaking prompts targeting 16 minority groups, designed to systematically audit demographic-specific safety failures.

2. 2. Large-scale evaluation: Testing 14 state-of-the-art LLMs on MiJaBench to curate 615,454 prompt-response pairs (MiJaBench-Align), revealing defense rate fluctuations up to 42% within the same model based on target group.

3. 3. Targeted direct preference optimization (DPO): Applying DPO on a 1B-parameter baseline to achieve zero-shot safety generalization to unseen demographics and attack strategies, demonstrating a pathway to equitable alignment.

## Evidence

Evaluation of 14 state-of-the-art LLMs on MiJaBench shows defense rates fluctuate by up to 42% within the same model solely based on the target group. The disparity persists across architectures and languages and is amplified by scaling. Targeted DPO on a 1B-parameter baseline achieves strong zero-shot safety generalizations to entirely unseen demographics and complex attack strategies.

## Assumptions & Open Questions

- ⚠️ The paper assumes that the 16 minority groups and 43,961 prompts in MiJaBench are representative of all relevant demographic vulnerabilities, but this may not cover intersectional or culturally specific harms.
- ⚠️ The DPO-based mitigation is demonstrated only on a 1B-parameter model; it is unclear if the approach scales effectively to larger models without performance degradation or new biases.
- ⚠️ The definition of 'safety' and 'harm' is taken as given from existing taxonomies; the paper does not question whether these categories themselves embed cultural or linguistic biases.
- ⚠️ The study focuses on English and Portuguese; the selective safety trap may manifest differently in other languages or cultural contexts, and the benchmark may not generalize.
- ⚠️ The paper does not explore potential negative side effects of targeted DPO, such as over-generalization that could overly restrict harmless content for certain groups.
