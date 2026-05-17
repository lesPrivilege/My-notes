# Topology-Aware Representation Alignment for Semi-Supervised Vision-Language Learning

- **Source**: https://arxiv.org/abs/2604.26370
- **Date**: 2026-04-29
- **Authors / Org**: Junwon You, Mihyun Jang, Sangwoo Mo, Jae-Hun Jung
- **Type**: arxiv

## Core Problem

Existing semi-supervised vision-language learning methods rely on pairwise alignment and fail to model the global structure of multimodal manifolds. Prior topology-based approaches use persistence diagram matching, which does not guarantee geometric alignment or utilize cross-modal pairing information.

## Core Approach

1. **Topologically salient edge selection via persistent homology**: ToMA uses persistent homology to identify edges that are topologically important (e.g., H_0-death edges and H_1-birth edges) from the data manifold, rather than relying on all pairwise relationships.

2. **Cross-modal alignment of selected edges**: The identified topologically salient edges are aligned across modalities using available image-text correspondences, ensuring that the global structure of each modality's representation manifold is preserved in the shared space.

3. **Incorporation of both H_0 and H_1 topological features**: ToMA leverages H_0-death edges (capturing connectivity) and lightweight H_1-birth edges (capturing cycle structure) without constructing 2-simplices, providing higher-order structural signals efficiently.

## Evidence

Experiments show stable gains on remote sensing (clear improvements) and modest but consistent benefits on fashion retrieval. Additional analysis indicates ToMA is more stable than alternative topology-based objectives, and lightweight H_1-birth edges provide useful higher-order structural signals.

## Assumptions & Open Questions

- ⚠️ The paper assumes that persistent homology features (H_0-death and H_1-birth edges) are sufficient to capture meaningful global structure for alignment, but does not prove that other topological features (e.g., higher-dimensional homology) are irrelevant.
- ⚠️ The method's reliance on available cross-modal correspondences assumes that these pairings are accurate and representative, which may not hold in noisy or weakly supervised settings.
- ⚠️ The experiments are limited to remote sensing and fashion retrieval; it is unclear how ToMA generalizes to other specialized domains (e.g., medical imaging, autonomous driving).
- ⚠️ The paper does not quantify the computational overhead of persistent homology computation relative to simpler pairwise methods, leaving open questions about scalability to large datasets.
