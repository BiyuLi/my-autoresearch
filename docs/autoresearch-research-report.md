# AutoResearch 仓库研究报告

> 报告生成时间：2026-03-13  
> 研究状态：原理分析完成，环境未就绪  
> 原文仓库：https://github.com/karpathy/autoresearch

---

## 一、项目概述

**AutoResearch** 是 Andrej Karpathy 设计的一个极简 AI 研究自动化框架，核心理念是：

```
传统研究：人写代码 → 人调参数 → 人跑实验 → 人分析（耗时数周）
AutoResearch：人写指令 → AI自动迭代 → 自动实验 → 自动优化（overnight数百实验）
```

---

## 二、核心架构

### 2.1 项目结构

```
autoresearch/
├── prepare.py      # 准备脚本（固定，不修改）
│   ├── 下载训练数据
│   ├── 训练BPE tokenizer
│   └── 数据加载器、评估工具
│
├── train.py        # 训练脚本（Agent修改）⭐
│   ├── GPT模型架构
│   ├── 优化器（Muon + AdamW）
│   ├── 训练循环
│   └── 超参数（学习率、batch size等）
│
└── program.md      # Agent指令（人编写）⭐
    ├── 任务描述
    ├── 约束条件
    └── 实验目标
```

### 2.2 关键设计约束

| 约束 | 目的 | 效果 |
|------|------|------|
| **只修改train.py** | 限制Agent范围 | 避免Agent搞崩整个系统 |
| **固定5分钟实验** | 标准化比较 | 实验结果可横向对比，overnight可跑100次 |
| **单文件编辑** | 简化diff review | 人可轻松查看修改 |
| **固定评估指标** | 客观优化目标 | Agent明确知道好坏（val_bpb越低越好） |

---

## 三、为什么能成功？（核心原理）

### 3.1 约束即力量

**关键洞察：** 约束不是限制，而是让Agent更高效地搜索解空间。

- 只改一个文件 → Agent聚焦核心
- 固定时间 → 快速反馈
- 明确指标 → 目标清晰

### 3.2 人机分离

```
人类（CEO/Researcher）：
- 定义研究目标（program.md）
- 设定约束和评估标准
- 审查实验结果

AI Agent（Worker）：
- 修改代码（train.py）
- 执行实验
- 决策保留/回滚
```

**分工逻辑：**
- 人擅长：高层次决策、创造性思考、目标设定
- AI擅长：枯燥的重复实验、大规模搜索、不知疲倦

### 3.3 快速反馈循环

```
修改代码（1分钟）→ 训练（5分钟）→ 评估（秒级）→ 决策（秒级）
= 每6-7分钟完成一轮迭代

overnight（8小时）≈ 80-100次实验
```

**对比传统：**
- 传统：1天可能只能跑3-5组实验
- AutoResearch：1晚可跑100组实验

### 3.4 自举进化

```
第1轮：baseline（随机初始化）
第2轮：找到较好方向（如学习率1e-4）
第3轮：在该方向上优化（如batch size）
第4轮：组合优化
...
第N轮：收敛到最优配置
```

**关键：** 每一轮的best model成为下一轮的baseline。

---

## 四、创新点分析

### 4.1 可编程研究（Programmable Research）

**传统：**
- 研究是"艺术"，依赖经验
- 每次研究从零开始

**AutoResearch：**
- 研究是"工程"，可程序化
- 通过program.md定义研究流程
- 可复用、可迭代、可积累

**类比：**
```
传统编程：写代码解决问题
AutoResearch：写program.md让AI解决问题
```

### 4.2 Human-in-the-Loop but Not in-the-Details

**不是完全自动化：**
- 人设定方向和约束
- AI在约束内自动执行

**不是完全人工：**
- 人不修改代码
- AI自动修改和优化

**最佳状态：** 人把控"做什么"，AI决定"怎么做"。

### 4.3 Meta-Learning through Code Evolution

**不仅是超参数搜索：**
- 可以修改网络架构
- 可以修改训练流程
- 可以修改优化器

**本质：** 代码层面的元学习（Meta-Learning），让AI学会如何写代码。

### 4.4 极简主义

**核心只有3个文件：**
```
prepare.py  - 固定，不修改
train.py    - 唯一可修改的文件
program.md  - 人写的指令
```

**为什么极简能成功？**
- 减少复杂度 → 减少出错可能
- 聚焦核心 → Agent更容易优化
- 易于理解 → 人更容易掌控

---

## 五、工作流程详解

### Step 1: 准备（Prepare）

**内容：**
```python
# prepare.py - 一次性准备
- 下载/生成数据
- 数据预处理
- 初始化tokenizer（NLP任务）
```

**为什么固定？**
- 数据准备是"基础设施"，不应随实验变化
- 避免Agent浪费时间在数据上

### Step 2: 训练（Train）

**内容：**
```python
# train.py - Agent唯一可修改的文件
- 模型定义
- 优化器配置
- 训练循环
- 超参数设置
```

**Agent如何修改？**
- 读取当前train.py
- 根据program.md的指令修改
- 保存新版本
- 运行5分钟

### Step 3: 评估（Evaluate）

**自动完成：**
- 训练结束后自动计算val_bpb
- 与历史最佳比较
- 记录到experiments/exp_XXX/

### Step 4: 决策（Decide）

**Agent决策逻辑：**
```python
if current_val_bpb < best_val_bpb:
    # 改善 → 保留
    save_checkpoint()
    log("improved", current_val_bpb)
    # 继续在此方向优化
else:
    # 恶化 → 回滚
    rollback_to_best()
    log("worse", current_val_bpb)
    # 尝试其他方向
```

---

## 六、实际应用：FastBEV + nuScenes 实验

### 6.1 任务设定

- **模型**：FastBEV-tiny
- **数据**：伪造nuScenes mini（10帧）
- **目标**：优化BEV分割性能
- **约束**：单GPU，5分钟/实验

### 6.2 program.md 示例

```markdown
# FastBEV nuScenes 实验

## 任务
在伪造的nuScenes数据上优化FastBEV模型。

## 当前baseline
- mIoU: 0.15
- 训练时间: 5min

## 可调参数
1. 学习率 [1e-5, 1e-3]
2. Batch size [2, 8]
3. 数据增强 [0.0, 1.0]
4. Backbone冻结 [True, False]

## 评估
- 主要指标：mIoU
- 约束：训练不崩溃

## 目标
mIoU > 0.25
```

### 6.3 已产出成果

| 文档/代码 | 内容 | 位置 |
|-----------|------|------|
| 中文指导手册 | AutoResearch完整中文教程 | projects/autoresearch-chinese-guide.md |
| 框架深度分析 | 原理、创新点、实施步骤 | projects/autoresearch-framework-analysis.md |
| FastBEV实验配置 | program.md + 代码 | projects/fastbev-nuscenes-experiment/ |
| 伪造数据生成器 | 生成10帧nusc数据 | fake_data_generator.py |
| FastBEV模型 | 简化版模型定义 | fastbev_model.py |

---

## 七、局限性与挑战

### 7.1 适用场景有限

**适合：**
- 标准监督学习任务
- 有明确评估指标
- 单GPU可运行
- 快速可验证的实验

**不适合：**
- 需要多GPU分布式训练
- 实验周期很长（>1小时）
- 评估指标不明确
- 需要人工标注/验证

### 7.2 环境依赖

**需要：**
- 稳定的Python环境
- PyTorch等框架
- GPU计算资源

**当前状态：**
- 本机环境检查未通过（PyTorch未安装）
- 建议云端部署后运行

### 7.3 Agent能力限制

**当前Agent能力：**
- 能调整超参数
- 能修改简单架构
- 能理解明确指令

**还不能：**
- 创新性架构设计
- 理解复杂约束
- 处理模糊目标

---

## 八、实施建议

### 方案1：云端部署（推荐）

**架构：**
```
OpenClaw (控制)  →  云端GPU服务器 (训练)
      ↑                      ↓
   program.md            experiments/
```

**优点：**
- 环境可控
- GPU资源充足
- 24/7运行

**步骤：**
1. 配置云端GPU环境（AWS/GCP/阿里云）
2. 安装PyTorch和依赖
3. 上传代码
4. 启动Agent自动实验

### 方案2：本地运行（简化版）

**调整：**
- 使用更小模型
- 使用更少数据
- 缩短实验时间（如2分钟）

**步骤：**
1. 安装PyTorch等依赖
2. 配置GPU支持
3. 本地运行实验

### 方案3：纯分析模式（当前）

**重点：**
- 理解框架原理 ✅ 已完成
- 设计实验方案 ✅ 已完成
- 准备环境后运行 ⏳ 等待

---

## 九、核心发现总结

### 9.1 AutoResearch为什么能成功？

1. **约束设计**
   - 只修改train.py → Agent范围可控
   - 固定5分钟 → 实验可比
   - 单文件 → 易于review

2. **人机分离**
   - 人：定义目标、设定约束
   - AI：自动修改、执行、优化

3. **快速迭代**
   - 5分钟/实验
   - overnight可跑100次

4. **极简架构**
   - 仅3个核心文件
   - 降低复杂度

### 9.2 创新价值

- **研究范式转变**：从"人驱动"到"AI驱动"
- **效率提升**：从"天级"到"夜级"
- **可复制性**：program.md可分享、复用

---

## 十、下一步行动

### 立即可做
- [x] 原理分析 ✅ 已完成
- [x] 中文手册 ✅ 已完成
- [x] FastBEV实验设计 ✅ 已完成

### 需要环境
- [ ] 配置云端GPU环境
- [ ] 安装PyTorch和依赖
- [ ] 启动Agent自动实验

### 预期产出
-  overnight跑100次实验
-  找到最优超参数配置
-  生成性能优化报告

---

## 附录：相关文档索引

| 文档 | 路径 | 内容 |
|------|------|------|
| 本报告 | projects/autoresearch-research-report.md | 综合研究报告 |
| 中文指导手册 | projects/autoresearch-chinese-guide.md | 完整中文教程 |
| 框架深度分析 | projects/autoresearch-framework-analysis.md | 原理与创新点分析 |
| 项目总结 | projects/autoresearch-summary.md | 执行结果总结 |
| FastBEV实验 | projects/fastbev-nuscenes-experiment/ | 实验配置与代码 |

---

*报告完成：2026-03-13*  
*状态：原理分析完成，等待环境就绪后运行实验*
