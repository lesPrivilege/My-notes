# CheXthought: A global multimodal dataset of clinical chain-of-thought reasoning and visual attention for chest X-ray interpretation

- **Source**: https://arxiv.org/abs/2604.26288
- **Date**: 2026-04-29
- **Authors / Org**: Sonali Sharma, Jin Long, George Shih, Sarah Eid, Christian Bluethgen, Francine L. Jacobson, Emily B. Tsai, Global Radiology Consortium, Ahmed M. Alaa, Curtis P. Langlotz
- **Type**: arxiv

## Core Problem

Current vision-language models for chest X-ray interpretation are trained on paired images and reports, lacking the cognitive processes (chain-of-thought reasoning) and visual attention patterns that underlie expert clinical reasoning. This gap limits model transparency, factual accuracy, and the ability to communicate uncertainty or disagreement.

## Core Approach

1. **Global multi-reader annotation pipeline**: Collected 103,592 chain-of-thought reasoning traces and 6,609,082 visual attention annotations from 501 radiologists across 71 countries, covering 50,312 multi-read chest X-rays, ensuring diverse clinical perspectives.

2. **Chain-of-thought reasoning traces**: Radiologists provided step-by-step verbal reasoning during interpretation, capturing distinct visual search strategies, integration of clinical context, and communication of uncertainty.

3. **Synchronized visual attention annotations**: Eye-tracking or equivalent spatial annotations were synchronized with reasoning traces, linking where radiologists looked to what they said, enabling spatial grounding of clinical reasoning.

4. **Multi-reader disagreement modeling**: Leveraged multiple annotations per image to predict human-human and human-AI disagreement directly from the image, facilitating transparent communication of case difficulty and model reliability.

## Evidence

['CheXthought reasoning outperforms state-of-the-art vision-language model chain-of-thought in factual accuracy and spatial grounding.', 'Visual attention data used as an inference-time hint recovers missed findings and significantly reduces hallucinations.', 'Vision-language models trained on CheXthought achieve significantly stronger pathology classification, visual faithfulness, temporal reasoning, and uncertainty communication.', 'Multi-reader annotations enable prediction of human-human and human-AI disagreement directly from an image.']

## Assumptions & Open Questions

- ⚠️ The dataset assumes that radiologists' verbalized reasoning and eye-tracking data fully capture the cognitive processes underlying clinical interpretation, but unspoken heuristics or subconscious pattern recognition may be missed.
- ⚠️ The global diversity of 71 countries is claimed, but the distribution of radiologists per country and potential biases in training or experience are not detailed, which may affect generalizability.
- ⚠️ The reduction in hallucinations and improvement in factual accuracy are reported without specifying the baseline models or datasets used for comparison, making it hard to assess the magnitude of improvement.
- ⚠️ The method for predicting human-human and human-AI disagreement is described only at a high level; the specific model architecture, evaluation metrics, and performance numbers are not provided.
- ⚠️ The dataset's size (50,312 images) is large but may still underrepresent rare pathologies or atypical presentations, limiting the model's ability to handle edge cases.
