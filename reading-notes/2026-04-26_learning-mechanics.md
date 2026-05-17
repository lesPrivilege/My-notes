# There Will Be a Scientific Theory of Deep Learning

- **来源**: arXiv:2604.21691
- **日期**: 2026-04
- **作者/机构**: Jamie Simon, Daniel Kunin, Alexander Atanasov, Enric Boix-Adserà, Blake Bordelon, Jeremy Cohen, Arthur Jacot, Eric J. Michaud, Nikhil Ghosh, Mason Kamb, Berkan Ottlik, Florentin Guth, Dhruva Karkada, Joseph Turnbull — UC Berkeley, Harvard, UPenn, Flatiron Institute, NYU, Astera Institute, Stanford
- **类型**: paper (position paper / 综述 / 宣言)
- **置信度**: high
- **模式**: deep-review

## 核心问题

深度学习作为最强、最不可解的机器学习方法，其成功高度依赖试错和经验调参，缺乏统一的科学框架来解释「为什么会 work」以及「如何 work」。古典学习理论（统计学习、PAC、凸优化）在非凸、过参数化、高度非线性的神经网络面前解释力不足。这篇论文的核心主张是：**这样的理论不仅可能，而且正在浮现**，其形态将类似物理学中的「力学」（mechanics）。

## 核心方案

论文以五条证据线索构建「learning mechanics」（学习力学）的论证框架：

1. **可解析求解的理想化设定** — 如 deep linear networks、NTK 核回归、multi-index models，这些简化模型保留了真实网络的关键行为（低秩贪婪学习、相位转变、边缘稳定振荡），同时可以精确求解。作用类似物理中的简谐振子和氢原子。

2. **可揭示本质行为的极限** — 无限宽度极限下的 lazy vs. rich dichotomy（类比材料的弹性 vs. 塑性变形）；无限深度极限；batch size、learning rate 的各种极限。核心信念是 Discretization Hypothesis：有限神经网络只是连续极限的离散近似。

3. **捕捉宏观可观测量的简单经验律** — neural scaling laws（loss 关于 compute/data/parameters 的幂律）、edge of stability（Hessian 最大特征值稳定在 2/η）、neural collapse（类内聚拢、类间正则单纯形）、neural feature ansatz、gradient flow conservation laws（Noether 定理推广）。

4. **超参数的去耦与理解** — SGD 的 learning rate / batch size 线性缩放规则（SDE 视角）、critical batch size 的权衡理论、Tensor Programs / µP 实现跨宽度的 learning rate transfer（超参数从小模型零样本迁移到大模型）。

5. **跨设定与跨任务的普适现象** — 不同架构的归纳偏置趋同（CNN vs Transformer、UNet vs U-ViT）、表征趋同（Platonic representation hypothesis）、数据的普适统计结构（Zipf's law、power-law spectrum）。

论文将这五条线索与古典力学、连续介质力学、统计力学、量子力学中的工具和思想逐类比（见 Table 1）。

## 学习力学的七大 desiderata

1. **基础性**（fundamental）— 从第一性原理逻辑推导
2. **数学性**（mathematical）— 定量而非定性
3. **预测性**（predictive）— 可被简单实验重复验证
4. **全面性**（comprehensive）— 同一个框架描述训练过程、隐藏表征、最终权重
5. **直觉性**（intuitive）— 简单、启发性高
6. **实用性**（useful）— 减少调参、指导数据设计、为 AI safety 提供基础
7. **谦逊性**（humble）— 明确其适用边界

## 学习力学 ⇄ 机械可解释性

论文的 Section 3.1 尤其精彩——提出了 symbiotic relationship 的愿景：

- **学习力学 → 机械可解释性**：为其核心假设（linear representability、locality、sparsity、compositionality）提供严格数学基础；解释机制如何通过训练动态形成（「nothing in biology makes sense except in the light of evolution」）。
- **学习力学 ← 机械可解释性**：提供丰富的具体现象作为理论建模的靶标（induction heads、Fourier features in modular arithmetic、data-driven feature geometry）。

> 学习力学是深度学习的物理学，机械可解释性是深度学习的生物学。

## 十大开放方向

1. 什么是真实深层非线性学习的简单可解模型？
2. 能捕捉自然数据结构的理论长什么样？
3. 深度学习是否隐式最小化某种函数复杂度？
4. 如何形式化定义 neural features？
5. 有限神经网络能否被理解为无限极限的近似（Discretization Hypothesis 的验证）？
6. 能否理解并消除所有超参数？
7. 能否先验预测 scaling law 指数？
8. Loss curvature 与架构、特征、泛化的交互机制？
9. 什么造就了一个好的 optimizer？
10. 什么意义上不同模型学到相似表征？如何定义 similarity metric？

## 证据

论文的「证据」不是新实验结果，而是对现有研究成果的系统性综述与重新框架化：

- Deep linear networks 的精确动力学解（Saxe et al. 2014）以及 NTK 对测试误差的精确预测（Simon et al. 2023a）
- Lazy vs. rich 训练的实验验证（Chizat et al. 2019, Figure 2）
- Neural scaling laws 在 GPT-3/BERT/LM 上的经验观测（Kaplan et al. 2020, Figure 3）
- Edge of stability 跨架构（FC、VGG、ResNet）的复现（Cohen et al. 2021a, Figure 4）
- µP 跨宽度 learning rate transfer 的实验验证（Yang et al. 2022, Figure 5）
- 扩散模型不同架构（DDPM、consistency model、U-ViT）输出趋同（Zhang et al. 2024, Figure 6a）
- 语言模型与视觉模型表征相似度随性能增加而增加（Huh et al. 2024, Figure 6b）

## 风险与弱点

- ⚠️ **论证强度不均衡**：五条证据中 solvable settings 和 hyperparameter 部分最充实，universal phenomena 部分较薄弱，Platonic representation 存在争议（论文自己也引用 Gröger et al. [2026] 质疑 similarity metric 选择）。
- ⚠️ **物理类比的局限性**：结论反复强调学习力学与物理力学的亲缘关系，但 analogy is not derivation。物理系统有 Hamiltonian/Lagrangian 形式主义的统一数学框架，深度学习目前缺乏这种等级的统一动力学描述。
- ⚠️ **Discretization Hypothesis 未经严格验证**：这是 infinite limits 框架的核心假设，但论文承认尚未被严格证明。如果有限神经网络的某些行为在无限极限中消失（而非被近似保留），该框架会受到根本挑战。
- ⚠️ **适用的 regime 不清晰**：coarse-graining 的尺度在哪里？loss 层面的 scaling laws 是 coarse，特征层面的细节（机械可解释性关心的）则很 fine。不同尺度的理论如何衔接未说明。
- ⚠️ **论点的 position 本质上是 optimistic**：无新结果，而是对已有结果的叙事重构。是优秀的 advocacy piece，但读者需意识到这点。
- ⚠️ **团队观点同质性的潜在偏差**：15 位作者多来自或关联 UC Berkeley、Flatiron Institute、Harvard 的 Pehlevan 组和 Ganguli 组，代表用统计物理方法做 DL theory 的特定 tradition。其他路径（singular learning theory、algorithmic information theory 等）着墨较少。
- ⚠️ **对 skepticism 的回应较弱**：Section 4 对「理论不可能 / 不重要」反驳的深度不够，尤其未正面回答「如果 LLM 行为在 fundamental 层面不可约（即不存在优于 empiricism 的简化），learning mechanics 的 falsification condition 是什么？」

## 待验证问题

- Discretization Hypothesis 能否在非 residual networks 或 non-asymptotic regime 中被严格表述和验证？
- Learning mechanics 的七个 desiderata 之间是否存在 tradeoffs？（全面 vs 简单，谦逊 vs 实用）
- 学习力学与机械可解释性的 symbiosis 在多大程度上是真实的协同，而非「同一个房间的两只刺猬」——各自在自己的方法论轨道上并行运行？
- 论文提出的 framework 能否在下一个 LLM 核心突破（如新的训练范式、新的架构）面前保持解释力，或者会被迫大幅 revision？
