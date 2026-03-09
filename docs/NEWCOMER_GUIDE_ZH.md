# Auto-GPT 新人入门指南（中文）

## 1. 这个代码库整体是怎么组织的？

这是一个以 `scripts/main.py` 为入口的 Python CLI 项目，核心流程是：

1. 读取配置（API Key、模型、模式开关等）
2. 构造系统提示词和目标
3. 调用 LLM 生成下一步动作（JSON 格式）
4. 解析并执行命令（搜索、读写文件、执行代码、记忆写入等）
5. 将结果写回历史上下文，进入下一轮循环

从目录上看，建议按下面分层理解：

- **入口与运行**
  - `main.py` / `scripts/main.py`：程序启动与主循环。
- **LLM 与会话上下文**
  - `scripts/chat.py`：拼接上下文、控制 token、调用聊天接口。
  - `scripts/llm_utils.py`：对 OpenAI/Azure ChatCompletion 的薄封装。
  - `scripts/token_counter.py`：消息 token 计算（用于上下文截断）。
- **配置与 AI 身份**
  - `scripts/config.py`：全局配置单例，读取 `.env` 与 Azure 配置。
  - `scripts/ai_config.py`：AI 名称/角色/目标的持久化与提示词构造。
- **命令系统（Agent 可执行动作）**
  - `scripts/commands.py`：命令路由中心（google、browse、文件操作、子 Agent、执行 shell 等）。
  - `scripts/file_operations.py`、`scripts/execute_code.py`：具体动作实现。
- **记忆系统**
  - `scripts/memory/`：`local` / `redis` / `pinecone` 三类后端抽象与实现。
- **提示词与数据**
  - `scripts/data/prompt.txt`：核心系统提示词模板。
  - `scripts/data.py`：读取提示词等数据。
- **测试**
  - `tests/`：包含 config、json 解析、网页抓取、memory 等测试。

## 2. 新人必须先理解的关键点

### A. “主循环 + JSON 命令”是第一原理

这个仓库最核心不是 UI，而是 “LLM 产出结构化 JSON → 程序执行命令” 的闭环。
优先读：

- `scripts/main.py`：看如何收集 assistant reply、打印 thoughts、调用 `get_command/execute_command`。
- `scripts/commands.py`：看支持哪些命令，以及每个命令如何映射到实际函数。

### B. 上下文与 token 管理决定稳定性

Auto-GPT 的行为很大程度受上下文窗口影响。`scripts/chat.py` 会：

- 注入系统消息（身份、时间、记忆）
- 动态截断历史消息
- 为模型回复保留 token

如果后续要做“更聪明/更稳定”，这里是优先改造点。

### C. 配置来源与运行模式

`Config` 是单例，读取 `.env`，控制：

- 模型选择（`FAST_LLM_MODEL` / `SMART_LLM_MODEL`）
- 是否 Azure
- 是否允许执行本地 shell（高风险）
- 记忆后端（local/redis/pinecone）

排查问题时先看配置值是否符合预期。

### D. 记忆后端是可插拔的

`memory/__init__.py` 按配置决定使用本地、Redis 或 Pinecone。对新人来说：

- 本地后端最容易调试
- Redis/Pinecone 更贴近生产场景
- 任何“记忆效果异常”都要先确认后端与初始化逻辑

## 3. 推荐学习路径（按 1~2 周）

### 第 1 天：跑通与观察

1. 按 README 准备环境并启动。
2. 用最简单目标跑 1~2 轮。
3. 打开 debug，观察日志与命令执行链。

### 第 2~3 天：读核心代码（建议顺序）

1. `scripts/main.py`
2. `scripts/chat.py`
3. `scripts/commands.py`
4. `scripts/config.py` + `scripts/ai_config.py`
5. `scripts/memory/*`

### 第 4~5 天：做一个小改动（低风险）

可选任务：

- 新增一个只读命令（如系统信息查询）
- 给某个命令补参数校验
- 给 `chat.py` 增加更清晰的 debug 输出

### 第 2 周：做一个中等改动（可验收）

可选任务：

- 新增记忆后端适配层
- 为命令执行增加更清晰的错误分类
- 优化 prompt 结构并做 A/B 对比（成功率、步数、token 消耗）

## 4. 新人常见坑（提前规避）

1. **JSON 解析失败**：模型输出不完全合规时，先看 `json_parser` 和主循环的修复逻辑。
2. **token 超限/上下文丢失**：先看 `chat.py` 的上下文裁剪与预留回复 token。
3. **命令无权限执行**：`EXECUTE_LOCAL_COMMANDS=False` 时 shell 命令会被拒绝。
4. **记忆“没生效”**：检查 `MEMORY_BACKEND`、后端服务连通性、是否被清空。
5. **环境变量遗漏**：先核对 `.env` 与 README 的必填项。

## 5. 对后续成长最有价值的实践建议

- **先做可观测性，再做能力增强**：先把日志、错误分类、指标（成功率/token）做清楚。
- **每次只改一个闭环**：比如“只改命令路由”或“只改上下文构建”，便于定位回归。
- **优先补测试再重构**：该仓库已有测试基础，改核心逻辑前先加用例保护。
- **把安全开关当作默认约束**：尤其是本地命令执行、文件写入等高风险能力。
- **记录实验结论**：Prompt 和参数调优要沉淀实验记录，避免反复踩坑。

---

如果你是第一次参与此项目，建议把“**跑通一次完整任务 + 读懂 main/chat/commands/config**”作为第一阶段目标；能做到这一步，就已经超过多数只停留在“会启动项目”的新人了。
