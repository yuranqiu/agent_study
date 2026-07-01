# Interview QA · Coding Agent 进阶

> 来源：vibe-agentic-interview-notes (SihaoLiu)
> 覆盖：Claude Code / Codex 深度面试题

---

## 一、背景知识篇

### Q: `/command` 和 Skill 的区别是什么？什么时候用哪个？

**Answer:**

| 维度 | Slash Command | Skill |
|------|--------------|-------|
| 调用方式 | 用户显式 `/command` | 可被模型自主调用（YAML description） |
| 上下文 | 注入文本，无状态 | 共享主上下文，分级加载 |
| 附属文件 | 不支持 | 支持脚本/模板/示例 |
| 生命周期 | 无状态 | 有状态，compact 安全 |
| 当前状态 | **deprecated** | **推荐使用** |

**决策：优先 Skill**（几乎所有新场景）
Command 仅用于：完全确定性的、必须用户显式触发的极简 prompt 包装

---

### Q: Sub-agent 和 Skill 的本质区别？

**Answer:**

| 维度 | Skill | Sub-agent |
|------|-------|-----------|
| 上下文 | 共享主上下文 | 独立上下文 |
| 调用代价 | 低（只是文本注入） | 高（完整推理回合） |
| 适合场景 | "怎么做"的指导、规范 | 大量工具调用、可能产生大量噪声 |
| 结果 | 改变主对话后续行为 | 返回一段总结 |
| 内部子 Agent | 不开新上下文 | 无法再开 sub-agent（max_depth=1） |

---

### Q: Sub-agent 之间可以互相通信吗？

**Answer:**
- **默认不能**。Claude Code 的 sub-agent 是星型：所有子 Agent 只与主对话通信
- **Agent Team 实验特性**：开启后可用 `SendMessage(to=<agent_id>, message=...)` 直接互通
- 设计原因：强制信息汇聚、保证状态可审计、防止循环对话

---

### Q: Agent Team 和 Sub-agent 的最大区别？

**Answer:**

| 维度 | Sub-agent | Agent Team |
|------|-----------|------------|
| 拓扑 | 主→子树形 | 成员互通图 |
| 生命周期 | fire-and-forget | 长期存在 |
| 状态共享 | 只返回最终 summary | 持续讨论维护上下文 |
| 适用场景 | 一次性任务 | 多角色并行推进 |

---

### Q: MCP 是什么？和 API 接口的区别？

**Answer:**

| 维度 | 裸 API | MCP |
|------|-------|-----|
| 描述方式 | 自然语言文档 | 标准 JSON Schema |
| 工具发现 | 需预先告知模型 | 连接时自动 list |
| 鉴权 | 各家自己定义 | 标准 OAuth 2.0 |
| 传输 | HTTP 各异 | stdio/HTTP/SSE |
| 复用性 | 每个平台重做 | 一份 server 全网通用 |

**类比**：API = USB 线（各形状）；MCP = USB-C（统一接口）

---

### Q: Sub-agent 可以再衍生 sub-agent 吗？

**Answer:**
- **不可以**。Claude Code 明确禁止嵌套（max_depth=1）
- 原因：防止无限递归、保持信息汇聚、避免上下文爆炸
- 解法：主对话串行/并行派发扁平化 sub-agent，或用 Skill 组织内部步骤

---

## 二、细节梳理篇

### Q: CLAUDE.md / AGENTS.md 加载顺序和优先级？

**Answer:**

Claude Code 优先级（高→低）：
1. **Managed policy** — 企业 IT 管控，不可覆盖
2. **User** — `~/.claude/CLAUDE.md`
3. **Project** — 仓库根 `./CLAUDE.md`
4. **Local** — `CLAUDE.local.md`（通常 gitignore）

**实践原则**：
- 每个文件 < 200 行
- 写"agent 必须遵守的规则"，不写"项目介绍"
- 避免信息腐烂 — 宁缺毋滥

---

### Q: Hooks 有哪些类型？列举常用场景

**Answer:**

**类型**：`command`（shell）/ `http`（POST）/ `prompt`（单轮 LLM）/ `agent`（多轮）/ `mcp_tool`

**常用场景**：
1. `PostToolUse` 命中 Edit → 自动跑 prettier/eslint
2. `PreToolUse` 命中 Edit(/.env*) → 阻止误改密钥（exit 2）
3. `Stop` → 发桌面通知
4. `SessionStart` → 输出 git 状态、环境变量
5. `PreToolUse` 命中 Bash(rm -rf *) → ask 确认

---

### Q: Permission Mode 有哪些？

**Answer:**

| 模式 | 行为 | 适合场景 |
|------|------|---------|
| default | 每工具首次询问 | 不熟悉新项目 |
| acceptEdits | 自动放行文件编辑/fs | 熟悉项目加速迭代 |
| plan | 只读，禁止写操作 | code review / 方案设计 |
| auto | 后台分类器判断 | 半自动信任模式 |
| dontAsk | 白名单外一律拒绝 | 严格生产场景 |
| bypassPermissions | 全部放行（保留熔断） | 沙箱/容器内 |

---

### Q: `/compact` 时哪些保留，哪些丢失？

**Answer:**

**保留（重新注入）**：
- System prompt / 项目根 CLAUDE.md / 未限制 path 的 rules
- Auto memory（从磁盘重读）/ 对话摘要
- 前 N 个高优先级 skill 描述（25K token 预算内）

**丢失**：
- 历史 tool call 详细输出（读过的文件内容等）
- 路径范围限定的嵌套 CLAUDE.md / rules（等下次访问时重新触发）
- MCP tool 完整 schema（只保留名字，等下次调用时 fetch）

---

### Q: Plugin 和 Skill 的关系？

**Answer:**
- **Skill**：单个能力单元（SKILL.md + 附属资源）
- **Plugin**：打包发布单位，可包含多个 skill + sub-agent + hooks + MCP server + 二进制
- 调用方式：`/<plugin-name>:<skill-name>`（命名空间避免冲突）
- 关系：**Skill 是零件，Plugin 是装好的整车**

---

### Q: ToolSearch / Deferred Tools 是什么？为什么要这样设计？

**Answer:**
- **现象**：MCP 工具和部分内置工具的完整 JSON schema 不立刻加载，只把名字告知模型
- **原因**：
  1. 节省 token（大型 MCP server 可能几十上百工具，全部 schema 上万 token）
  2. 降低注意力噪声
  3. 按需加载
- **代价**：第一次使用某工具多一次 ToolSearch round trip

---

## 三、工作流篇

### Q: 接到中等复杂度新功能任务，agentic 工具如何拆解？

**Answer:**

1. **Plan mode** (Shift+Tab) 只读探索
2. **Explore sub-agent** 并行搜索相关文件（找入口、数据流、测试）
3. **Plan sub-agent** 或主对话产出 TDD 方案（先写验收标准/测试用例）
4. **ask-codex** 找第二意见（cross-model review）
5. **AskUserQuestion** 让用户在关键岔路拍板
6. ExitPlanMode → acceptEdits 模式落地
7. 先写测试再实现，实时跑测试
8. 完成后自审 + 提交前过 verification skill

---

### Q: 什么时候应该开 sub-agent，什么时候不该？

**Answer:**

**应该开**：
- 读 10+ 文件做调研（Explore）
- 大量 grep/find/日志分析，输出会很噪
- 独立 code review / 安全 review
- 并行做多个无依赖子任务

**不该开**：
- 只需要 1-3 个工具调用 → 主对话直接做更快
- 需要主对话保留思考过程用于后续决策
- 需要多次交互式确认（sub-agent 不能问用户）
- 任务就是"读一个已知路径" → 直接 Read

---

### Q: 如何避免主对话上下文被污染？

**Answer:**
1. 大量读取/调研 → Explore sub-agent，只回传 summary
2. 长输出命令 → 用 head/grep 收敛，或写文件再让 agent 选读
3. 重复性任务封装成 skill
4. 及时 `/compact` 在里程碑后压缩
5. 不要让 agent 反复读同一文件（harness 维护文件状态）
6. 把"长期记住的事"写进 CLAUDE.md / auto memory
7. MCP 工具按需加载，不要一次性全连上

---

### Q: 代码出 bug，如何用 agentic 工具调试？

**Answer:**
1. 用 systematic-debugging skill 进入系统化排查
2. **不要急着改代码** — 先复现，写能稳定触发 bug 的最小用例/测试
3. 让测试先失败，确认理解了 bug
4. Explore 找出 bug 路径上的相关文件
5. 假设→验证→修复（不是猜测式打补丁）
6. 修完后跑全部相关测试 + verification
7. 复杂 bug 用 codex:rescue 做独立诊断

---

## 四、系统设计篇（开放思辨）

### Q: 设计一个"agentic 低代码平台"，优先考虑哪些设计原则？

**讨论方向：**
1. **Source of truth**：Agent 生成的代码是 source，还是低代码 DSL 是 source？二选一
2. **可逆性与版本控制**：必须能 diff/revert/review agent 的修改
3. **沙箱与权限分级**：平台用户的 agent 不能直接访问数据库
4. **上下文供给**：业务知识怎么进入 agent？
5. **失败可观测**：tool call 失败必须能回放
6. **不要替代用户思考**：关键岔路必须显式确认
7. **成本可见**：每个操作的 token/时延都要前置可见

---

### Q: 设计企业"MCP 网关"，如何处理权限/审计/限流？

**讨论方向：**
- **权限**：读写角色看不到 delete 工具的 schema
- **审计**：每次 tool call 落日志（user / agent_id / 参数 / 耗时）
- **限流**：同一 user 的 QPS/并发限制
- **数据脱敏**：tool 返回值经中间层脱敏再回 agent
- **降级策略**：后端挂了返回 structured error，不超时
- **版本管理**：tool schema 加版本号，backward compatible
- **隔离**：不同租户 MCP server 进程隔离

---

### Q: Claude Code / Codex 多 agent 协作选星型还是网状？

**讨论方向：**
- **星型优点**：易调试、上下文可控、责任清晰；缺点：主对话成瓶颈
- **网状优点**：接近真实团队，并行高；缺点：循环对话、上下文爆炸、调试地狱
- **现实答案**：大多数任务星型够用，极少数复杂多角色才上网状
- **进一步思辨**：多 agent 的 ROI 是否真的高于"更强的单 agent"？

---

### Q: 设计"长期记忆"机制，决定记什么、不记什么？

**讨论方向：**
- **应该记**：用户反复纠正的偏好、项目硬约束、易遗忘的 trivia
- **不应该记**：一次性临时对话、含敏感数据内容、过时事实
- **如何更新**：写时合并、定期 GC（太老降权/删除）、用户可审查/编辑
- **容量控制**：限定大小（25KB），超出自定义权重取舍
- **跨项目共享**：偏好类跨项目共享；项目知识类不共享

---

## 五、概念哲学篇（开放思辨）

### Q: 为什么应该"保持 context 简单"？

**讨论方向：**
- **注意力是有限资源**：过多无关信息稀释关键指令权重（"lost in the middle"）
- **Token 成本**：每次推理按全部 context 计费/计算
- **可调试性**：短 context 容易定位冲突；长 context 几乎不可能复盘
- **可演化性**：context 越简单越容易迭代/替换
- **哲学**："简单"不是"少信息"，而是"高信噪比"

---

### Q: Agent 应该"主动"还是"被动"？什么时候问用户？

**讨论方向：**
- **可逆性**：可逆的（本地改文件）自决；不可逆的（push/delete/send）必问
- **影响半径**：只影响本机自决；影响共享系统必问
- **不确定度**：心里没底时应该问，不赌
- **用户可达性**：在线多问；后台/离线任务保守自决
- **设计哲学**：问对问题，关键岔路问，鸡毛蒜皮不打扰

---

### Q: TDD 在 agentic 工作流里为什么更重要？

**讨论方向：**
- **Agent 容易"自信地写错"**：会编 API/字段/返回值，测试是唯一客观裁判
- **测试是和 agent 沟通的契约**：不需要详细描述"怎么做"，只要说"做完后这个测试必须过"
- **TDD 给 agent 提供了反馈循环**：没有失败的测试，agent 不知道自己错没错
- **风险**：agent 自己写测试又自己实现可能"为过测试而过测试"；关键测试应人写或独立 review

---

### Q: Agentic 工具最终会让"程序员"消失吗？

**讨论方向：**
- **不会消失，会重塑**：写代码外包，但理解需求/拆分系统/判断对错/承担责任永远需要人
- **新角色会出现**：Agent 架构师 / AI 审稿员 / Context 工程师
- **历史类比**：编译器没让汇编程序员消失，但"手写 x86"的人变少了
- **关键追问**：5 年后你日常工作的什么比例还是"自己打字"？30%？5%？

---

### Q: 一个 agent 反复改不对一个 bug，应该继续还是关掉自己上？

**讨论方向：**
- **信号识别**：agent 在循环尝试同一类错误解 → 上下文已被污染，继续多半无效
- **沉没成本**：已经烧了多少 token 不重要，重要的是"再烧 X token 能否解决"
- **元问题**：Agent 卡住通常意味着信息缺失而非推理失败 — 应人工补信息（贴日志/文档），不是让它继续猜
- **健康习惯**：给自己定上限（"agent 改 3 次还不对就接手"）

---

*最后更新：2026-05-27*