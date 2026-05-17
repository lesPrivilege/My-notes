# Edge AI for Automotive Vulnerable Road User Safety: Deployable Detection via Knowledge Distillation

- **Source**: https://arxiv.org/abs/2604.26857
- **Date**: 2026-04-29
- **Authors / Org**: Akshay Karjol, Darrin M. Hanna
- **Type**: arxiv

## Core Problem

Large object detection models suffer catastrophic accuracy loss under INT8 quantization required for edge deployment, while small models trained directly have poor detection performance. This gap prevents accurate, safety-critical VRU detection on resource-constrained hardware.

## Core Approach

1. **Knowledge Distillation (KD) Framework**: A compact YOLOv8-S student (11.2M parameters) is trained to mimic the outputs of a larger YOLOv8-L teacher (43.7M parameters), transferring precision calibration rather than raw detection capacity.

2. **Post-Training Quantization (PTQ) to INT8**: Both teacher and student models are quantized to INT8 after training, simulating edge deployment constraints. The KD student is designed to retain accuracy under this quantization.

3. **Full-Scale BDD100K Evaluation**: The method is evaluated on the full BDD100K dataset (70K training images) to ensure realistic, large-scale validation of VRU detection performance.

## Evidence

The teacher suffers -23% mAP under INT8 quantization, while the KD student retains accuracy with only -5.6% mAP. At INT8, the KD student achieves 0.748 precision vs. 0.653 for direct training (14.5% gain), reduces false alarms by 44% vs. the collapsed teacher, and exceeds the teacher's FP32 precision (0.748 vs. 0.718) in a model 3.9x smaller.

## Assumptions & Open Questions

- ⚠️ The paper assumes that the teacher model's FP32 precision is a reliable upper bound for distillation, but does not explore whether a different teacher architecture or ensemble could yield even better student performance.
- ⚠️ The evaluation is limited to BDD100K; generalizability to other VRU datasets (e.g., Cityscapes, EuroCity Persons) or different edge hardware (e.g., NVIDIA Jetson, Google Coral) is not demonstrated.
- ⚠️ The claim that KD transfers 'precision calibration' rather than raw detection capacity is inferred but not directly proven via ablation or feature-level analysis.
- ⚠️ The 44% false alarm reduction is relative to the collapsed teacher; comparison to a non-collapsed baseline (e.g., a larger student or alternative quantization-aware training) is missing.
- ⚠️ The paper does not address latency or throughput measurements on actual edge devices, only parameter count and quantization robustness.
