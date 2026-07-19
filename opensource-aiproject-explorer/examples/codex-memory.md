# 示例：Codex Memory

用户追问：

> Codex 有哪些记忆？如何存储、检索？这个设计能否用于其他 Agent？

## 组织方式

### 1. 先解释起源

从“只依赖当前 Prompt”到“携带全部聊天历史”，再解释容量、成本和噪声问题。

一句话本质：

> 过去交互经过筛选和提炼，保存在当前上下文之外，并在未来相关任务中按需取回。

### 2. 建立通用分类

使用同一个例子：

- 工作记忆：当前测试输出；
- 经历记忆：上次修复完整过程；
- 知识记忆：仓库稳定验证规则；
- 程序性记忆：格式检查到全量测试的步骤；
- 偏好：用户不希望一开始执行昂贵的全量测试。

此时不出现 Rust 文件。

### 3. 映射 Codex

- 工作记忆 → `ContextManager.items`
- 经历证据 → rollout JSONL
- 单会话提取 → `stage1_outputs`
- 长期知识 → `MEMORY.md`
- 导航摘要 → `memory_summary.md`
- 程序性知识 → `skills/`

说明这是教学映射，不是 Codex 原生 enum。

### 4. Phase 1 环节卡

**定位**：单 rollout 提取器。

**问题**：原始会话完整但噪声大。

**输入**：过滤后的 rollout、工作目录、提取规则。

**处理**：领取、过滤、模型提取、最小信号判断、脱敏。

**输出**：`raw_memory`、`rollout_summary`、`rollout_slug`。

**边界**：不做跨会话去重，所以可以并行。

### 5. Phase 2 环节卡

**定位**：全局知识编辑器。

**问题**：Phase 1 素材重复、冲突和过期。

**输入**：Stage 1 输出、使用统计、旧手册、ad-hoc notes。

**处理**：加锁、选择、同步、diff、合并。

**输出**：`memory_summary.md`、`MEMORY.md`、summaries、Skills。

**边界**：维护共享文件，所以必须串行。

### 6. 分层读取

1. `memory_summary.md`：导航层，最低 token 成本；
2. `MEMORY.md`：知识层，提供主题结论；
3. rollout / raw evidence：证据层，高风险或精确问题才读取；
4. Skill：程序层，提供可执行步骤；
5. citation：反馈层，记录真实使用。

越往后成本越高、覆盖越窄，但证据精度和可执行性越强。

### 7. 迁移

可迁移的是：

```text
Capture → Select → Store → Retrieve → Inject → Feedback
```

两阶段 LLM、Markdown、git diff 和关键词检索只是 Codex 的实现选择。小型 Agent 可以从用户确认的结构化偏好、JSON/Markdown、标签检索和显式删除开始。

## 后续追问

用户继续询问 citation 时，先读索引和 Memory topic，再读取 citation parser 与 usage store；不重新扫描整个 Memory 模块。
