# Lightweight Distillation of SAM 3 and DINOv3 for Edge-Deployable Individual-Level Livestock Monitoring and Longitudinal Visual Analytics

- **Source**: https://arxiv.org/abs/2604.27128
- **Date**: 2026-04-29
- **Authors / Org**: Haiyu Yang, Miel Hostens
- **Type**: arxiv

## Core Problem

Foundation-model pipelines for precision livestock farming achieve high accuracy but require GPU memory budgets that exceed commodity edge accelerators, preventing deployment on low-power devices for real-time on-farm monitoring.

## Core Approach

1. **Feature Pyramid Network (FPN) student encoder**: A multi-scale student encoder built on TinyViT-21M-512 replaces the 446M-parameter SAM 3 backbone (PE-ViT-L+), reducing parameters while preserving multi-scale feature extraction for segmentation.

2. **Four-term direction-then-scale distillation loss**: A distillation loss that first aligns the direction (cosine similarity) and then the scale (L2 norm) of student and teacher features, applied across four terms to guide the student's learning from the teacher.

3. **Backbone-substitution inference with sliding-window session pruning**: During inference, the teacher backbone is replaced by the student, and a sliding-window mechanism prunes past video frames to bound streaming GPU memory growth, enabling long-duration video processing on edge hardware.

## Evidence

On the Edinburgh Pig dataset, the compressed pipeline achieves 92.29% MOTA and 96.15% IDF1 (1.68 and 0.84 percentage points below the SAM 3 teacher), a 7.77-fold reduction in system-level parameters, a 3.01-fold reduction in peak VRAM (19.52GB to 6.49GB), and 97.34% top-1 accuracy with 91.67% macro-F1 on nine-class pig behaviour classification. The pipeline fits within an NVIDIA Jetson Orin NX 16GB with 4.9GB headroom.

## Assumptions & Open Questions

- ⚠️ The proposed on-device embedding-pool re-identification mechanism is not empirically validated; its per-individual footprint of ~94MB per animal per year is estimated but its real-world accuracy and scalability are unknown.
- ⚠️ The distillation loss assumes that direction-then-scale alignment is sufficient to transfer all relevant knowledge from the teacher; it may miss higher-order relational or contextual information critical for rare behaviors or occluded animals.
- ⚠️ The sliding-window session pruning assumes that past frames can be safely discarded without affecting re-identification accuracy; this may fail for long-term identity tracking across discontinuous sessions.
- ⚠️ The evaluation is limited to a single dataset (Edinburgh Pig); generalizability to other livestock species, farm environments, or camera setups is not demonstrated.
- ⚠️ The 4.9GB headroom on the Jetson Orin NX may be insufficient when accounting for real-time video capture, preprocessing, and other system overheads not modeled in the paper.
