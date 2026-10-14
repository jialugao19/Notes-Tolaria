---
type: Note
---
# 分为 4 个 repo

将现有研究流程分为：

- Data
- Feature
- Model
- Evaluation


### 文档建设

AGENTS 是 map，repo docs 是 system of record


四个 repo 的公共结构:

- README.md：项目入口和文档导航
- AGENTS.md：Agent 必须遵守的 invariant
- CONTRACT.md：对上下游公开的稳定接口
- docs/：内部设计、规范和历史记录
  - docs/PROGRESS.md：当前状态、验证证据、已知问题及维护格式
- configs/：人工参数和配置模板
- artifact 内的 manifest.yaml：某份实际产物的 identity、fingerprint 和 inventory


### exp-nn-data

- exp-repo/exp-nn-data/CONTRACT.md: 数据输入输出、reader API、时间语义.
- exp-repo/exp-nn-data/docs/DESIGN.md: 数据质量、生命周期、发布与验证设计.
- exp-repo/exp-nn-data/docs/review-report.md: 历史 review.
- exp-repo/exp-nn-data/configs/market.yaml: 数据生产配置模板.

### exp-nn-feature

- exp-repo/exp-nn-feature/CONTRACT.md: Dataset 构建、读取、样本和存储接口.
- exp-repo/exp-nn-feature/docs/datasets/v002_1min.md: 当前 Dataset 设计.
- exp-repo/exp-nn-feature/docs/datasets/v001_1min.md: 冻结历史设计.
- configs/{v001_1min,v002_1min}/config.yaml: 各版本参数.
- configs/{v001_1min,v002_1min}/features.yaml: 各版本有序 feature catalog.

### exp-nn-model

- exp-repo/exp-nn-model/CONTRACT.md: 全系统 identity vocabulary, model/prediction artifact 分层及公开 API.
- exp-repo/exp-nn-model/docs/TRAINING.md: 训练、checkpoint、恢复和 selection 设计.
- exp-repo/exp-nn-model/EXPERIMENTS.md: 当前实验设计、运行证据与下一步.
- exp-repo/exp-nn-model/docs/templates/EXPERIMENTS.md: 实验记录模板.
- exp-repo/exp-nn-model/configs/baseline.yaml: baseline 参数.

### exp-nn-evaluation

- exp-repo/exp-nn-evaluation/CONTRACT.md: prediction 输入、evaluation 输出与身份接口.
- exp-repo/exp-nn-evaluation/docs/PIPELINE.md: 计算与呈现流程.
- exp-repo/exp-nn-evaluation/docs/METRICS.md: 公共指标的唯一语义定义.
- exp-repo/exp-nn-evaluation/docs/OUTLINE.md: 报告类型导航.
- exp-repo/exp-nn-evaluation/docs/OUTLINE_EXPERIMENT.md: 模型实验报告内容.
- exp-repo/exp-nn-evaluation/docs/OUTLINE_FACTOR.md: 单因子报告内容.
- exp-repo/exp-nn-evaluation/docs/DESIGN.md: 报告视觉、排版和交互规范.
- exp-repo/exp-nn-evaluation/configs/baseline.yaml: evaluation 参数.
