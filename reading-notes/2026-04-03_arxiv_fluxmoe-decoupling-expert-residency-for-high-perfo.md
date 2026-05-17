# FluxMoE: Decoupling Expert Residency for High-Performance MoE Serving

- **Source**: https://arxiv.org/abs/2604.02715
- **Date**: 2026-04-03
- **Authors / Org**: Qingxiu Liu, Cyril Y. He, Hanser Jiang, Zion Wang, Alan Zhao, Patrick P. C. Lee
- **Type**: arxiv

## Core Problem

In MoE models, expert weights occupy GPU memory even when idle, competing with the KV cache for space. Since KV cache capacity directly limits serving throughput, this memory contention leads to underutilization and degraded performance, especially under severe memory constraints.

## Core Approach

1. **Expert Paging Abstraction**: Treats expert weights as transient, streamable resources rather than persistent residents. Experts are materialized on demand from host memory and evicted immediately after use, freeing GPU memory for the KV cache.

2. **On-Demand Materialization and Eviction**: Expert weights are loaded into GPU memory only when needed for a forward pass and are evicted right after computation, minimizing their residency time and maximizing available space for throughput-critical runtime state.

3. **Integration with vLLM**: FluxMoE is implemented atop the vLLM serving framework, leveraging its existing infrastructure for efficient memory management and scheduling while adding the expert paging mechanism.

## Evidence

FluxMoE achieves up to 3.0× throughput gains over vLLM in memory-intensive regimes, without compromising model fidelity.

## Assumptions & Open Questions

- ⚠️ The approach assumes that expert weights can be fetched from host memory fast enough to avoid pipeline stalls; the paper does not quantify the latency overhead of on-demand loading or its impact on tail latency.
- ⚠️ It assumes that the KV cache is the primary memory bottleneck; other runtime state (e.g., activations, optimizer states) may also compete for memory but are not discussed.
- ⚠️ The throughput gains are reported only for memory-intensive regimes; performance under less constrained memory (e.g., with ample GPU memory) is not characterized, leaving open whether the paging overhead could degrade performance in those cases.
- ⚠️ The system is evaluated only on vLLM; generalizability to other serving frameworks or hardware architectures (e.g., with different memory hierarchies) is not addressed.
- ⚠️ The paper does not discuss the impact of expert paging on batch scheduling or request-level latency, which could be critical for interactive serving scenarios.
