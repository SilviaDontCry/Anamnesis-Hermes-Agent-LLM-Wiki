# Anamnesis — Hermes Agent Wiki记忆架构

> 让 Agent 跨 session 持续记忆。不靠提示词靠机制，不靠人脑靠代码。 
> 
> 根据Andrej Kaparthy大神的LLM Wiki架构思路带来的灵感，针对Hermes Agent的记忆优化方案。
> 
> by SilviaDontCry 曾靖

---

## 一、问题

LLM Agent 每次 session 醒来都是白板。通用方案（RAG、向量数据库）要么太重要么不可控。我们需要一个轻量、可追溯、可回滚的持久记忆系统。

### 1.1 核心矛盾

- **容量 vs 可见性**：MEMORY块（每轮注入的 system prompt）有字符上限，不能塞太多。但关键行为规则必须每轮可见，不能等按需加载
- **稳定性 vs 时效性**：项目信息会频繁变化，写在 MEMORY块 里要反复编辑，每次编辑都消耗 token
- **可靠性 vs 复杂度**：写日志靠提示词，新 session 可能遗忘。加代码 hook 要改上游仓库，升级会被覆盖

### 1.2 为什么不用 RAG？

RAG 每次查询都要：嵌入 → 检索 → 重排 → 注入。延迟高、成本高、结果不稳定。Anamnesis 的思路是**编译一次，持久存储**：知识写入 log 文件后，直接读文件获取，不需要每次检索。

### 1.3 为什么不用向量数据库？

向量数据库适合"我不知道知识在哪"的场景。但 Agent 的记忆是结构化的——我知道"用户偏好在 preferences.log.md"，不需要语义搜索。直接 read_file 比嵌入检索快 100 倍，且结果确定。

---

## 二、核心原理

### 2.1 三层架构

```
┌─────────────────────────────────────────────────────────┐
│ L0: 热缓存 (MEMORY块)                                    │
│ - 每轮自动注入 system prompt，零额外调用                 │
│ - 混合模式：行为规则保留文字 + 项目/技术转为索引         │
│ - 上限 1500 字符                                        │
├─────────────────────────────────────────────────────────┤
│ L1: 明细层 (~/.hermes/memory/*.log.md)                   │
│ - 按需读取，search_files 检索                           │
│ - 追加式记录，永不编辑旧条目                            │
│ - 统一 ID 体系 (D/L/P/J)                                │
│ - Code Guardian 保护：快照 + git commit                 │
├─────────────────────────────────────────────────────────┤
│ L2: 知识库 (wiki/ + skills/)                             │
│ - 技术文档、设计方案、编译后知识                        │
│ - search_files 检索                                     │
└─────────────────────────────────────────────────────────┘
```

**为什么分三层？**

- **L0 解决"每轮都要知道"的问题**：安全规则、用户身份、行为铁律——这些如果放在 L1，每轮都要额外 read_file，浪费 token 和延迟
- **L1 解决"容量上限"的问题**：MEMORY块只能放 ~1500 字符，但项目的详细信息（API 配置、架构细节、踩坑教训）远超这个限制。Log 文件无容量限制
- **L2 解决"知识沉淀"的问题**：Log 是流水账，wiki 是编译后的知识。定期整理 log → wiki，形成可复用的知识库

### 2.2 数据流

```
事件发生（用户偏好、决策、踩坑、项目变更）
  │
  ├→ L1: 立即追加到 ~/.hermes/memory/*.log.md
  │    （原始记录，Code Guardian 快照保护）
  │
  └→ L0: 如需每轮记忆 → 更新 MEMORY块（压缩为索引）
       │
       └→ L2: 定期编译到 wiki/references（交叉引用、去重）
```

**类比银行系统：**

- L1 = 银行流水（每笔交易都记，原始记录）
- L0 = 账户余额（压缩后的摘要，一眼看到关键信息）
- L2 = 财务报表（编译后的分析，支持决策）

### 2.3 路径规则

| 类型                    | 放哪里          | 原因                        |
| --------------------- | ------------ | ------------------------- |
| 系统文件（log、SOUL、MEMORY） | `~/.hermes/` | 服务 agent 的基础设施，跟着 agent 走 |
| 用户项目                  | 用户自定义路径      | 用户自己的软件，agent 不该控制        |
| MEMORY块索引用户项目         | 用项目实际路径      | 索引只是引用，不挪动用户文件            |

**为什么要分开？** agent 更新时只拉上游代码，不碰 `~/.hermes/` 下的用户文件。如果 log 文件放在用户项目目录里，项目目录结构变了就找不到。放在 `~/.hermes/` 下，和 agent 绑定，路径永远稳定。

---

## 三、MEMORY块：混合模式

### 3.1 为什么是混合模式？

有两种极端方案：

- **纯索引**：所有内容转成 `模块名:路径|状态`。问题是安全规则、行为铁律这些也要按需读 log，每次多一次 read_file 调用
- **纯文字**：所有内容都塞在 MEMORY块 里。问题是 1500 字符根本不够，项目信息频繁变化要反复编辑

混合模式按"是否每轮需要"分类：

| 类型       | 处理方式     | 原因               |
| -------- | -------- | ---------------- |
| 安全/权限规则  | 保留原文     | 每轮必须读，不能等按需加载    |
| 关键用户偏好   | 保留原文（压缩） | 影响每轮行为，忘了就犯错     |
| 行为铁律     | 保留原文（压缩） | 每轮都可能触发，需即时记忆    |
| 项目/架构/流程 | 转为索引     | 不是每轮都用，按需读 log   |
| 技术细节/配置  | 转为索引     | 具体执行时再查          |
| 非关键偏好    | 转为索引     | 需要时 search_files |

### 3.2 示例 MEMORY块（~400 字符）

```
Anamnesis:~/.hermes/memory/*.log.md|三层运行中
§
安全区~/workspace | 免批:读取+工作目录 | 需批:区外写删
§
User | 用户时区 | 沟通语言 | 直接答复杂先计划
- 自主学习 | 偏好简洁 | ≤3call/轮 | token效率
§
文件发送: MEDIA:/绝对路径 | 不确定先试不反驳
踩坑: 没验证别下结论 | 用户纠正先信后查
§
project-alpha:~/workspace/project-alpha/|活跃
project-beta:~/workspace/project-beta/|开发中
code_guardian:~/.hermes/extensions/code_guardian/|已上线
```

### 3.3 段落语义（§ 分隔）

MEMORY块用 § 分隔逻辑段落，SCHEMA.md 定义了每个段落的语义：

| 段落  | 语义                 | 可变性      |
| --- | ------------------ | -------- |
| §1  | 系统标记（Anamnesis 状态） | 低        |
| §2  | 安全规则               | 低        |
| §3  | 用户身份 + 核心偏好        | 中        |
| §4  | 行为铁律               | 低        |
| §5+ | 项目/技术索引            | 高（可增删合并） |

**为什么用 § 而不是 YAML/JSON？** MEMORY块是 system prompt 的一部分，会被 LLM 读取。§ 是自然语言友好的分隔符，LLM 能直接理解。YAML/JSON 需要解析，增加复杂度且 LLM 不一定严格遵守格式。

### 3.4 为什么这些保留文字？

| 内容     | 字符  | 为什么不能转索引                                             |
| ------ | --- | ---------------------------------------------------- |
| 安全区规则  | ~72 | 每轮判断操作权限，读 log 来不及。如果 agent 先读 log 再判断安全区，多一次 API 调用 |
| 用户身份   | ~37 | 定义沟通语言和方式，37 字符成本忽略不计                                |
| 核心偏好   | ~55 | 防止反复问、防止犯错、控制成本——这些错误的代价远超 55 字符                     |
| 文件发送铁律 | ~50 | 多次踩坑的血泪教训。如果转索引，agent 可能忘记读就直接发文件                    |
| 踩坑核心   | ~56 | "没验证别下结论"和"用户纠正先信后查"是行为铁律，每轮都可能触发                    |

### 3.5 压缩策略

当 MEMORY块 接近 1500 字符时，按优先级处理：

1. **先砍索引行** — 合并相似条目，删除不活跃项目
2. **再压缩文字** — 精简措辞，去掉冗余修饰
3. **最后砍条目** — 移动非核心条目到纯索引
4. **绝对不砍** — 安全规则、用户身份、行为铁律

---

## 四、Log 文件：统一格式

### 4.1 为什么用追加式？

追加式（append-only）的核心优势是**不可篡改的历史**：

- **可追溯**：每个条目有日期和 ID，按时间线排列
- **可回滚**：配合 Code Guardian 快照，可以恢复到任意历史状态
- **无冲突**：多个 session 并发写入不会覆盖彼此的内容
- **简单可靠**：不需要复杂的合并逻辑，追加就是追加

**为什么不编辑旧条目？** 编辑会丢失历史。如果一个决策后来被推翻，编辑旧条目意味着你不知道之前是什么。追加新条目标注 `[superseded: D005]` 保留了完整的决策演变历史。

### 4.2 格式规范

所有 log 文件遵循 SCHEMA.md 定义的通用结构：

```markdown
# [文件名] Log

追加式记录，永不编辑旧条目。
通用规则见 SCHEMA.md。
格式: ## [YYYY-MM-DD] ID | 类别 | 标题

---

## [2026-04-16] D001 | 架构 | code_guardian v1 落地
决策内容
原因: 为什么这么决定
状态: active
```

每个文件在头部保留自己的格式说明，但引用 SCHEMA.md 的通用规则。这样 agent 读任何一个文件都能知道格式，读 SCHEMA.md 能知道完整规范。

### 4.3 ID 体系

| 文件                 | 前缀  | 内容        | 示例         |
| ------------------ | --- | --------- | ---------- |
| decisions.log.md   | D   | 架构决策、技术选型 | D001, D002 |
| lessons.log.md     | L   | 踩坑教训、错误修正 | L001, L002 |
| preferences.log.md | P   | 用户偏好、工作习惯 | P001, P002 |
| projects.md        | J   | 项目状态      | J001, J002 |

**为什么需要 ID？**

- 引用：`[superseded: D005]` 需要唯一标识符
- 追踪：知道总共有多少条记录
- 搜索：`search_files "D001"` 能精确定位

**ID 追踪机制：** `.id-counter.json` 文件记录每个前缀的当前最大 ID：

```json
{"D": 1, "L": 8, "P": 11, "J": 3}
```

写入时：读取 → +1 → 使用新 ID → 更新文件。避免手动计数出错。

### 4.4 各文件职责和扩展字段

**decisions.log.md** — 决策记录

- 扩展字段：`原因`、`状态`（active/superseded）
- 为什么需要状态字段：决策会被推翻，需要标记哪些是当前有效的

**lessons.log.md** — 踩坑教训

- 扩展字段：`问题`、`解决`、`教训`、`参考`
- 为什么需要参考字段：教训关联具体代码位置，下次遇到类似问题能快速定位

**preferences.log.md** — 用户偏好

- 扩展字段：`来源`（用户说/观察到）
- 为什么需要来源字段：区分"用户明确要求"和"我观察到的偏好"，前者权重更高

**projects.md** — 项目状态

- 扩展字段：`位置`、`说明`、`约束`
- 为什么追加不编辑：项目状态会变（开发中→已上线→已暂停），保留历史状态变化记录

### 4.5 触发规则

| 事件        | 写入                 | 格式                           |
| --------- | ------------------ | ---------------------------- |
| 用户表达偏好/习惯 | preferences.log.md | `## [date] P-ID \| 类别 \| 内容` |
| 做出项目决策    | decisions.log.md   | `## [date] D-ID \| 类别 \| 内容` |
| 遇到坑/错误教训  | lessons.log.md     | `## [date] L-ID \| 类别 \| 内容` |
| 项目状态变化    | projects.md        | `## [date] J-ID \| 类别 \| 内容` |

**写入方式：** 必须走 Code Guardian `write_memory_log()`，不直接 write_file。原因见第五章。

---

## 五、Code Guardian 集成

### 5.1 为什么要走 Code Guardian？

直接 write_file 有两个问题：

1. **无历史**：写入后旧内容丢失，无法回滚
2. **无记录**：不知道什么时候写的、为什么改的

Code Guardian 提供三层保护：

1. **快照**：写入前备份旧版到 `.snapshots/`
2. **写入**：追加新条目（不覆盖）
3. **changelog + git**：自动 commit，记录变更原因

### 5.2 write_memory_log() 执行流程

```python
def write_memory_log(log_file: str, entry: str, reason: str = "memory update") -> dict:
    """追加写入记忆日志，走三层保护：快照 → 写入 → changelog + git"""
```

```
write_memory_log("decisions.log.md", "## [2026-04-17] D002 | ...", "decision: xxx")
  │
  ├→ 1. 读取 ~/.hermes/memory/decisions.log.md 现有内容
  ├→ 2. 快照旧版 → .snapshots/ 目录（带时间戳）
  ├→ 3. 追加新条目（不覆盖旧内容）
  ├→ 4. 写入文件
  └→ 5. git add + commit + 追加 .changelog.yaml
```

### 5.3 设计决策：绕过 is_code_file()

Code Guardian 的 `is_code_file()` 根据扩展名判断是否为代码文件（.py, .js 等）。`.log.md` 不在列表里，会被跳过。

但 log 文件仍然需要快照和 git 保护。方案：直接调 snapshot 和 changelog 模块，跳过 header 校验。

为什么跳过 header 校验？Log 文件不需要 `# purpose/tags/status` 头——那是代码文件的规范，不是日志文件的规范。

### 5.4 为什么不用 hermes_tools hook？

原方案计划在 hermes 的 `memory_tool()` 函数里加 hook，memory add 成功后自动分类写入 log 文件。但：

- `memory_tool.py` 在 hermes-agent 上游仓库（`github.com/NousResearch/hermes-agent`）
- 按"agent 不 fork"原则，不直接修改上游代码
- hermes update 时上游文件会被覆盖，改动丢失

**替代方案：SOUL.md 指令驱动 + write_memory_log()**

- SOUL.md 写明"写 log 必须走 write_memory_log"
- SOUL.md 每轮注入 system prompt，指令始终可见
- Agent 每次写 log 时读到指令，调用 code_guardian 的函数
- 比直接 write_file 强（有快照/git 保护）
- 比代码 hook 弱（可被遗忘），但 SOUL.md 每轮注入，遗忘概率很低

**升级路径：** 如果未来 hermes 支持 observer/hook 机制（如事件回调），可以升级为代码级强制，不依赖提示词。

### 5.5 初始化要求

`~/.hermes/memory/` 目录必须初始化 git 仓库。changelog.py 的 `_find_git_root()` 从文件路径向上查找 `.git` 目录，找不到就跳过 git commit 和 changelog.yaml。

```bash
cd ~/.hermes/memory
git init
git config user.email "agent@hermes.local"
git config user.name "Hermes Agent"
git add -A
git commit -m "[anamnesis] 初始化 memory git 仓库"
```

**没有 git 仓库时：** 快照正常创建（`.snapshots/`），但 git commit 和 changelog.yaml 会静默跳过。agent 不会报错，只会在 hermes 日志里留下 warning。所以初始化是必须的，不能省略。

**为什么 memory 目录独立一个 git？** 不和 workspace 或 extensions 共用。原因：
- memory 是 agent 基础设施，不随用户项目变化
- 独立 git 避免 commit 历史污染用户项目
- 独立 git 避免 hermes update 影响

---

## 六、规范化（SCHEMA.md）

### 6.1 为什么需要 SCHEMA.md？

在没有 SCHEMA.md 之前：

- Log 文件格式靠 agent 记忆，新 session 可能写错
- MEMORY块段落语义是隐式的，agent 可能误读
- ID 没有追踪机制，可能冲突
- 路径引用不一致（有的用相对路径，有的用绝对路径）

SCHEMA.md 是单一事实来源（Single Source of Truth），定义所有格式、路径、ID 规则。Agent 首次 session 读一次，后续都遵守。

### 6.2 SCHEMA.md 定义的内容

| 章节        | 内容                   |
| --------- | -------------------- |
| MEMORY块格式 | § 段落语义、索引行格式、压缩规则    |
| Log 文件格式  | 通用字段、扩展字段、ID 体系      |
| 写入方式      | 必须走 write_memory_log |
| 路径规范      | 绝对路径规则、文件位置          |
| 安全规则      | 不存密钥、安全区检查           |

### 6.3 ID 追踪（.id-counter.json）

```json
{"D": 1, "L": 8, "P": 11, "J": 3}
```

- 值为已使用的最大 ID
- 写入时读取 → +1 → 使用 → 更新
- 避免手动计数和 ID 冲突
- 隐藏文件（`.` 前缀），不污染目录列表

### 6.4 段落语义化

MEMORY块用 § 分隔，SCHEMA.md 定义每个段落的含义：

```
§1  系统标记（Anamnesis 状态）     ← 低频变更
§2  安全规则                      ← 低频变更
§3  用户身份 + 核心偏好            ← 中频变更
§4  行为铁律                      ← 低频变更
§5+ 项目/技术索引                  ← 高频变更（可增删合并）
```

**为什么这样分？** §1-4 是"不变的底座"，agent 压缩时只压缩 §5+ 的索引行，不动 §1-4。这样保证安全规则和行为铁律永远不会被意外删除。

### 6.5 SOUL.md 段落顺序锁定

SOUL.md 用 HTML 注释锁定段落顺序：

```html
<!-- 段落顺序锁定：1.Core Truths 2.Boundaries 3.记忆架构 4.写代码标准 5.自动整理 6.Continuity -->
```

新增段落必须加在指定位置，不能随意插入。防止结构被破坏。

---

## 七、文件位置

### 7.1 安全（hermes update 不影响）

```
~/.hermes/extensions/code_guardian/     ← 独立 git，自定义扩展
~/.hermes/SOUL.md                       ← 用户配置
~/.hermes/memories/MEMORY.md            ← 用户数据
~/.hermes/memory/                       ← Anamnesis 全部文件
~/.hermes/memory/SCHEMA.md              ← 格式规范
~/.hermes/memory/.id-counter.json       ← ID 追踪
~/.hermes/memory/*.log.md               ← 四个日志文件
~/workspace/wiki/                       ← 工作区文档
~/workspace/项目目录/                    ← 用户项目
```

### 7.2 危险（hermes update 会覆盖）

```
~/.hermes/hermes-agent/                 ← 上游仓库，不要改
```

### 7.3 为什么系统文件在 ~/.hermes/？

- **路径稳定性**：`~/.hermes/` 是 hermes 的 home 目录，不随用户项目变化
- **更新安全**：hermes update 只拉 `hermes-agent/` 子目录，不碰其他文件
- **跟随 agent**：如果用户换机器，`~/.hermes/` 整个目录迁移即可
- **与 hermes 生态一致**：MEMORY.md、USER.md、SOUL.md 都在这里

### 7.4 完整文件清单

```
~/.hermes/memory/
├── SCHEMA.md              ← 格式规范总定义
├── .id-counter.json       ← ID 追踪
├── .git/                  ← 独立 git 仓库（Code Guardian 依赖）
├── .changelog.yaml        ← 变更日志（自动创建）
├── .snapshots/            ← 快照备份（自动创建）
├── decisions.log.md       ← D-ID 决策记录
├── lessons.log.md         ← L-ID 踩坑教训
├── preferences.log.md     ← P-ID 用户偏好
└── projects.md            ← J-ID 项目状态
```

---

## 八、概念澄清

### 8.1 两个 MEMORY

hermes 系统有两个同名概念：

| 名称            | 位置                  | 作用               | 谁管        |
| ------------- | ------------------- | ---------------- | --------- |
| **MEMORY块**   | SOUL.md 顶部注入        | Anamnesis 自定义索引层 | 本方案       |
| **MEMORY.md** | ~/.hermes/memories/ | hermes 内存工具存储    | hermes 原生 |

两者独立，没有联动。Anamnesis 管的是前者。MEMORY.md 是 hermes 内置的记忆工具，有自己的容量限制和注入机制。MEMORY块是我们定义的、嵌入在 SOUL.md 中的文本块。

### 8.2 与 memory-index skill 的关系

Anamnesis 和 memory-index（Karpathy 模式）是同一系统的两个视角：

| 维度   | Anamnesis       | memory-index    |
| ---- | --------------- | --------------- |
| 核心文件 | memory/*.log.md | references/*.md |
| 热缓存  | MEMORY块         | MEMORY.md       |
| 职责   | 运行时日志           | 编译后知识           |
| 写入   | 事件驱动追加          | 定期整理编译          |
| 数据流  | raw → log       | log → compiled  |

**Anamnesis 是源头，memory-index 是编译结果。** 事件先写入 Anamnesis log，然后定期整理到 memory-index 的 references 页面。

---

## 九、搜索指南

| 找什么     | 怎么找                                        | 哪层   | 延迟                 |
| ------- | ------------------------------------------ | ---- | ------------------ |
| 即时规则/偏好 | MEMORY块已注入                                 | L0   | 0（已注入）             |
| 用户偏好详情  | read `~/.hermes/memory/preferences.log.md` | L1   | 1 次 read_file      |
| 项目决策    | read `~/.hermes/memory/decisions.log.md`   | L1   | 1 次 read_file      |
| 踩坑教训    | read `~/.hermes/memory/lessons.log.md`     | L1   | 1 次 read_file      |
| 项目状态    | read `~/.hermes/memory/projects.md`        | L1   | 1 次 read_file      |
| 格式规范    | read `~/.hermes/memory/SCHEMA.md`          | L1   | 1 次 read_file      |
| 编译后知识   | skill_view('memory-index')                 | L1.5 | 1 次 skill_view     |
| 技能/方法   | search_files 在 `skills/`                   | L2   | 1 次 search_files   |
| 技术文档    | search_files 在 `wiki/`                     | L2   | 1 次 search_files   |
| 历史对话    | session_search                             | L3   | 1 次 session_search |

**为什么 SCHEMA.md 在搜索表里？** Agent 写 log 前需要知道格式。如果格式不确定，先 read SCHEMA.md，再按格式写入。比猜格式然后写错要高效。

---

## 十、风险与回退

| 风险                         | 缓解                    | 回退           |
| -------------------------- | --------------------- | ------------ |
| Agent 忘记走 write_memory_log | SOUL.md 每轮注入，指令始终可见   | 手动补写         |
| write_memory_log 静默失败      | logger.warning 记录     | 检查 hermes 日志 |
| MEMORY块压缩丢失信息              | 先备份到 log 文件           | 从 git 回滚     |
| hermes update 覆盖上游代码       | 所有修改在 extensions/     | 不受影响         |
| ID 冲突                      | .id-counter.json 自动递增 | 手动修正计数器      |
| Log 文件格式错误                 | SCHEMA.md 定义规范        | 按规范修正文件      |
| 路径不一致                      | 统一绝对路径规则              | 搜索替换修正       |

---

## 十一、收益

- **不忘写** — SOUL.md 指令驱动 + write_memory_log 保护
- **可追溯** — 每条 log 有 ID、日期、分类
- **可回滚** — Code Guardian 快照 + git
- **可审计** — changelog.yaml + git log
- **两系统统一** — Anamnesis(raw) + memory-index(compiled) 分工明确
- **MEMORY块瘦身** — 压缩率 ~77%，余量充足
- **格式统一** — 所有 log 文件同一格式，同一 ID 体系，SCHEMA.md 单一事实来源
- **hermes 更新安全** — 所有修改在上游仓库之外
- **路径清晰** — 系统文件在 hermes，用户项目自定义
- **可复用** — SCHEMA.md 是标准化规范，其他 agent/项目可以直接采用
