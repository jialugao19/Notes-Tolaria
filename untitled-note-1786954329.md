---
type: Note
---
# Skills

skill 写作原则：

1. **从真实经验中提炼 skill，不要让模型凭通用知识凭空生成：**最好的来源是实际任务、review comments、事故记录、runbook、历史修复。
2. **只写 agent 缺少的东西。**只告诉模型它容易做错、且必须知道的东西，不重新教授它本来就知道的一整套通用知识。没有这条 instruction，agent 会不会做错？
3. **控制 scope。**太窄会导致一次任务加载很多 skill；太宽又会难以精准触发，并让无关规则干扰执行。可以把 skill 当成设计 function。
4. **Less is more。**过于完整的 skill 反而可能让 agent 执行不适用的规则。简洁、逐步、带少量关键例子。
5. **使用 reference。**`SKILL.md` 只放每次都需要的核心规则；详细知识放到 `references/`，而且要明确写出**什么时候读取哪个 reference**。
6. **方案和目标。**多种方案都合理时，给 agent 判断空间，并说明目标；容易出错、顺序敏感、破坏性强的任务才应该使用严格步骤。
7. **迭代 skill。**看最终结果，也看 execution trace：false positive、漏触发、绕路、无意义步骤、过多选择，都是需要修改 skill 的信号。

description: Focus on user intent, not implementation.

- capability “它实际能干什么？”
  positive routing “用户说什么/想做什么时应该触发？”
  semantic expansion “用户即使没说 Skill 名或关键词，什么意图仍应该触发？”
  boundary “哪些很像，但其实不该触发？”

- 模板例如：
  <verb + concrete capabilities>.\\
  Use when [user intent / task / artifact / context].\\
  Also use when [important implicit cases].\\
  Do not use for [confusing adjacent cases].
  


## 检查当前状态

### inspect-compute

检查 cpu, gpu, memory

If Prefect, 检查实验进程


### gpush

需要重新设计 commit 相关 skill。目前已删除




## 写作规范：代码+设计

### python-code-quality

1. Google：一般不要使用 `staticmethod`，写 module-level function。
2. function - module - class：如无必要，勿增实体。

** **`SKILL.md`负责：

- 什么时候触发？
- 要不要读 reference？
- 修改范围应该多大？

`references/design.md`负责回答一些具体问题


### correct-then-fast

补充：NVDIA 性能指南 [https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html)

1. 如果存在明显的算法复杂度问题，例如 O(n²) 可以直接变成 O(n log n)，没有必要为了遵守形式先跑 profiler。
2. **correctness before accepting performance results**，如 reference comparison、unit testing、numerical accuracy
3. **representative workload**，要能代表实际使用场景
4. **profile / measure rather than guess；且重复测量**
5. **最终应该看 end-to-end performance**。 microbenchmark 和 end-to-end benchmark 的地位大家存在分歧。
6. **GPU benchmark 必须注意异步执行**。例如 CUDA kernel launch 是异步的，直接用 CPU wall clock 很容易量错，需要正确同步或使用 CUDA event。

后续需要补充 references: 例如只有检测到 CUDA/PyTorch workload 时才读对应 reference。

例如，PyTorch 自己提供 profiler 来查看 CPU/CUDA operator、input shapes、stack traces 和 device kernel activity


### maximize-gpu-utilization

补充：[https://docs.nvidia.com/deeplearning/performance/dl-performance-getting-started/index.html?utm\_source=chatgpt.com](https://docs.nvidia.com/deeplearning/performance/dl-performance-getting-started/index.html?utm_source=chatgpt.com)

1. **GPU utilization 是诊断信号，不应脱离 throughput/latency 使用。**
2. **先 measurement/profile，再判断 GPU 为什么闲。**
3. **数据供给、数据搬运和 CPU orchestration 与 GPU kernel 本身同样重要。** NVIDIA 的 deep-learning performance 指南明确指出 data movement 本身会限制 performance；PyTorch 也专门提供 DataLoader/H2D optimization guidance。
4. **batch size 是重要杠杆，但不是唯一答案。**

### audit-nn-exp

review exp-nn 系列实验是否建议启动

1. checkpoint 保存
2. mlflow
3. 检查模型参数是否符合预期

未来可能补充：prefect

### nn-exp-reuse 待修改

再探再报


### frontend-design

来源于 Claude Code

[https://github.com/anthropics/claude-code/blob/main/plugins/frontend-design/skills/frontend-design/SKILL.md](https://github.com/anthropics/claude-code/blob/main/plugins/frontend-design/skills/frontend-design/SKILL.md)


## 报告

### exp-report

1. self-constrained html
2. 按照项目中的 report 模板生成
3. 配置、结果、图表和结论必须能追溯到真实实验 artifact；缺失事实不能凭空补全。

内容正确性由项目规范 + agent review 判断；机械可验证项交给 validate\_html\_report.py，包括外部资源依赖、内嵌图片有效性，以及可选的离线浏览器渲染和整页截图。


## 思考+规划

### grill-me

grilling

来源网络。

Resolve consequential uncertainty relentlessly; don't ask questions relentlessly.

[https://github.com/mattpocock/skills/blob/main/skills%2Fproductivity%2Fgrilling%2FSKILL.md?utm\_source=chatgpt.co](https://github.com/mattpocock/skills/blob/main/skills%2Fproductivity%2Fgrilling%2FSKILL.md?utm_source=chatgpt.com)
