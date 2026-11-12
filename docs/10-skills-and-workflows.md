# 10 · 工具、Skills 与工作流（Codex 复现指南）

> **本文定位**：回答用户要求的「继续开发要用到的工具、skills、下载方法，以及我们是怎么干活的」。
> **两条最重要的前置声明**：
> 1. **skills 是 WorkBuddy 侧的助手能力，不是 foul-play 的依赖**。Codex 不需要、也不应该在 `E:\Tools\foul-play\.venv` 里安装任何 skill。它们是「工作方法」，不是运行时组件。
> 2. **本项目没有一份既有的、成文的工作流规范文档**。证据标签体系【实证】/【源码】/【推测】/【待核】/【更正】/【待确认】是**整理本知识库时新建的约定**，定义写在 `README.md` 与 `AGENTS.md`，工作区 memory 与 `reports/` 里**没有**这套标签的书面定义。纪律条文散落在工作区日志中（本文 §5 给出可核验的具体文件:行号）。
>
> ⚠️ **可移植性警告**：用户级 `C:\Users\abc\.workbuddy\MEMORY.md` **不在本仓库里**，Codex 在别的机器上看不到。本知识库已把其中与工程相关的硬纪律转写进 `AGENTS.md` 与本文；**不要以那份外部记忆为准，也不要假设它存在**。

---

## 1. 两套运行环境，别混

| | 汇编本库时 | Codex 接手后 |
|---|---|---|
| 宿主 | WorkBuddy（多智能体团队 + skills） | Codex 或任意工程师 |
| 只需要什么 | 本地源码 + 报告 + 日志 + skills | **本仓库文档 + `E:\Tools\foul-play` 等本地路径** |
| 需要的工具 | 只读 grep / Read / 并行子代理 | 同一套只读工具即可 |
| 不需要什么 | —— | skills、WorkBuddy 账号、用户级 MEMORY.md、团队子代理 |

**Codex 的最小充分输入**：`AGENTS.md`（红线 + 命令）→ `docs/00`（本体）→ `docs/03`（开关状态）→ `docs/11`（悬案）→ 对应专题文档。工具链与重建方法在 `docs/01`。

---

## 2. 用户级 skills 全表（**19 个，逐个实测 frontmatter**）

【实测】目录 `C:\Users\abc\.workbuddy\skills\`：`ls -1a` 去掉 `.` / `..` 后共 **22 项** = **19 个目录（各有 `SKILL.md`，即 19 个有效 skill）** + **3 个 json 迁移文件**（均非 skill）：

| 非目录条目 | 属性 |
|---|---|
| `_bm_skillid_migration.json` | 可见，49 字节 |
| `.disable_to_model_invocation_migration.json` | **隐藏**，36 字节 |
| `.model_invocation_to_override_migration.json` | **隐藏**，35 字节 |

> 【更正】早期用 `ls -1`（不显示点文件）只看到 20 项 / 1 个 json，据此写的「20 项含 1 个 json」是**漏计**；实际 22 项 / 3 个 json。**19 个有效 skill 的结论不变**。

下表 `目录` / `name` / `description 节选` 全部来自各自 `SKILL.md` 的 frontmatter 原文。

> ⚠️ 两处**目录名 ≠ frontmatter `name`**：`QQ浏览器自动化` → `qqbrowser-skill`；`dev-expert__skillhub` → `dev-expert`。按 `name` 调用，不要按目录名。
> ⚠️ 「与本项目关系」是**本次整理时按 description 语义做的判断**，不是既有书面记录；**本项目没有保存 skill 调用日志**，所以「实际是否被调用过」一律【无法判断】。

| # | 目录 | frontmatter `name` | description 节选 | 关系 |
|---|---|---|---|---|
| 01 | `QQ浏览器自动化` | `qqbrowser-skill` | Browser automation CLI for AI agents；打开/导航网页、填表 | 【可迁移·无调用记录】 |
| 02 | `delivery-no-pseudoblock` | `delivery-no-pseudoblock` | 交付前伪阻断 / 缓干闸门（通用行为级纪律）；交付结果前先自检 | 【直接相关】 |
| 03 | `dev-expert__skillhub` | `dev-expert` | 编程专家，含工程纪律层（根因调试硬闭环、事故复盘、威胁建模、AI 编码治理）；七步闭环 Step 0–6，**禁跳过验证·清理·复盘** | 【直接相关】 |
| 04 | `engineering-rigor` | `engineering-rigor` | 改动前必须先定位根因（文件路径+行号+触发条件+影响面），禁止凭现象猜着改 | 【直接相关】 |
| 05 | `expression-and-understanding` | `expression-and-understanding` | 理解侧：任何需求动手前先呈现自己的理解并经用户确认才算确信 | 【直接相关】 |
| 06 | `game-lifecycle-expansion` | `game-lifecycle-expansion` | 游戏生命周期与玩法扩展优化（留存/付费诊断） | 【可迁移·无调用记录】 |
| 07 | `general-problem-solving` | `general-problem-solving` | 通用问题解决总纲：信息查证、文档产出、数据整理、小工具、环境排障 | 【直接相关】 |
| 08 | `hard-problem-solving` | `hard-problem-solving` | 机理不明的疑难问题攻坚：并发竞态、性能退化、间歇性/不可复现故障 | 【直接相关】（对应 dispatcher 假死） |
| 09 | `hollow-knight-oc-design` | `hollow-knight-oc-design` | 《空洞骑士》二创角色与区域设定文档 | 【与本项目无关】 |
| 10 | `interviewer` | `interviewer` | 模拟面试、出题、评估候选人 | 【与本项目无关】 |
| 11 | `memory-management` | `memory-management` | 三层记忆（云端会话检索 / 用户级 MEMORY.md / 工作区日志）的分工与写入纪律 | 【直接相关】 |
| 12 | `parallel-execution` | `parallel-execution` | 并行/并发任务判断、冲突识别与结果合并纪律 | 【直接相关】 |
| 13 | `pixel-game-animation` | `pixel-game-animation` | 像素风逐帧 sprite animation 规范与提示词 | 【与本项目无关】 |
| 14 | `pixel-game-icon` | `pixel-game-icon` | 像素风游戏 Icon 设计规范与提示词 | 【与本项目无关】 |
| 15 | `rnd-workflow` | `rnd-workflow` | 从需求到交付验收的完整研发链路纪律 | 【直接相关】 |
| 16 | `scanned-pdf-text-replace` | `scanned-pdf-text-replace` | 扫描/图片型 PDF 文字替换（保排版、保印章） | 【与本项目无关】 |
| 17 | `shandong-chorography-research` | `shandong-chorography-research` | 抓取分析「山东地情档案」地方志论文 | 【与本项目无关】（属另一研究项目） |
| 18 | `skill-management` | `skill-management` | Skill 全生命周期：新建/命名/编写/自检/维护/合并拆分/废弃删除 | 【直接相关·仅维护用】 |
| 19 | `team-collaboration` | `team-collaboration` | 多智能体团队（Team / 多 worker / 并行分工）组织与协作纪律 | 【直接相关】（本次知识库即以此方式汇编） |

**按 description 语义分档**：【直接相关】11 个（02/03/04/05/07/08/11/12/15/18/19）、【可迁移·无调用记录】2 个（01/06）、【与本项目无关】6 个（09/10/13/14/16/17）。

> 【无法判断】是否存在 skill 的**实际调用记录** —— 本项目未保存这类日志，上表**不声称**任何 skill 在本项目里被真正执行过。
>
> 【实证·书证】但其中 **10 个是本项目期间按主题重写并验收的**：工作区 memory `2026-10-17.md:55` 记「用户要求把十个主题（团队协作/并行/技能管理/记忆管理/表达与理解/工程化/解决各种各样的问题/高难度问题/研发流程/面试官）提升到用户级，但工作区 `.workbuddy/skills` 目录**不存在**、全盘搜零命中 ⇒ 判断是当时在对话里生成了但没落盘；用户确认后改为**按十个主题重新写**，落到 `C:\Users\abc\.workbuddy\skills\<name>\SKILL.md`」；`:67` 记「十项技能验收结果（全过）」并逐项列出行数（team-collaboration 158、parallel-execution 144、skill-management 193、memory-management 152、expression-and-understanding 203、engineering-rigor 191、rnd-workflow 195、interviewer 243、general-problem-solving 133、hard-problem-solving 138），格式为 frontmatter + 五节正文（触发时机 / 核心原则 / 具体做法 / 反模式 / 常见陷阱）。
>
> ⇒ 精确说法：**这 10 个有「编写 + 验收」的书面证据，但没有「运行时被调用」的证据**；另外 9 个的来历【报告未载】。

### 2.1 项目级 / 工作区级 skills

【实测】`C:\Users\abc\WorkBuddy\2026-10-10-14-15-51\.workbuddy\skills\` **不存在**。所有 19 个 skill 都是**用户级**（跨项目），本项目没有专属 skill。

### 2.2 下载 / 安装方法

> 【待确认】这 19 个 skill 的**安装来源 URL / 版本 / 安装命令**本地文件里没有记录（只有已落盘的 `SKILL.md` 内容）。**不要凭记忆编造 marketplace 链接**。若 Codex 需要复现这套方法，直接读本表 + 各 `SKILL.md` 正文即可；安装渠道请向用户索取。
> 通用管理规则见 `skill-management` skill（用户级默认目录 `~/.workbuddy/skills/`，项目级 `{workspace}/.workbuddy/skills/`）。

---

## 3. 自研取证工具（**实测清单，含两处更正**）

### 3.1 工作区根目录（2 个 `.py`）

| 脚本 | 用途（读文件头 docstring【实证】） |
|---|---|
| `scan_logs.py` | **只读**校验 battle-tag 重组与 `|request|`/response 配对：扫 `E:\Tools\foul-play\logs\*.log`，按 tag 分组，检查 `|win|`/`|player|` 归属、按 `rqid` 配对、异常（重复 rqid / 孤儿 response / 未回答的非 wait request / 同时 active 与 forceSwitch）。`LOGS_DIR = sys.argv[1] if len(sys.argv)>1 else r"E:\Tools\foul-play\logs"`；`Prints a compact summary only. Does not write anything.` |
| `reconcile_counts.py` | **只读**核对冻结决策点定义：非 teamPreview、非 wait 的 `|request|` 与其后 `Choice:` 配对；四类统计 response 并校验配对。`Prints only. Writes nothing.` |

> ⚠️ `reconcile_counts.py` docstring 写「`rqid is the global primary key`」，但 `reports\metamon-audit-plan\04-data-model-dataset.md:319/:549` 与工作区 memory `2026-10-17.md:226` 已核定 **`rqid` 单独不唯一**（1181 个请求仅 159 个不同值），主键应为 **`(battle_tag, rqid)`**。以 04 章为准，别照 docstring 建表。
> ⚠️ `scan_logs.py` 默认读 live `logs\` ⇒ 按 `AGENTS.md` 红线 2，**必须传入冻结副本目录作为 `sys.argv[1]`**。

### 3.2 工作区 `tmp\` 探针脚本（**18 个 `.py`，实测**）

`fix_gamma.py`、`length_confound.py`、`main_head.py`、`power_gamma.py`、`probe3.py`、`probe_frames.py`、`probe_initlog.py`、`probe_initlog2.py`、`probe_sides.py`、`probe_ts.py`、`recalc.py`、`recalc2.py`、`s1_analysis.py`、`s1_analysis2.py`、`s1_length.py`、`t.py`、`verify_s1b.py`（+ 1 个，共 18 个）。

这些是**一次性的只读探针**（看 `init.log` 帧、验 γ、查长度混杂、算功效、重算 S1 统计），不是可复用产品线工具。其中 `t.py` 仅 **27 字节**（最小占位）。

> 【更正】早期取证材料称「`t.py` 为 0 字节空文件」，本次 `ls -l` 实测为 **27 字节**；又称「`tmp\` 有 13 个 `.py`」，实测为 **18 个**。以本节实测为准。
> 【更正】早期取证材料提及 `E:\Tools\foul-play\tools\verify_evidence.py`，本次 `find` 在 `E:\Tools\` 与 `E:\venvs\` 下**零命中** ⇒ 该文件**不存在**【待核】，不要引用它当复现入口。

### 3.3 主仓库工具与报告库

- `E:\Tools\foul-play\tools\` 下 A/B、端到端、日志、队伍工具全清单见 **`docs/01` §7**（`ab_ladder_pipeline.py`、`mde_two_prop.py`、`concurrent_realtime_test.py`、`e2e_hidenext_test.py`、`export_battle_log.py`、`replay_lead_advisor.py` 等）。
- **主报告库**在 `E:\Tools\foul-play\reports\`（`ls | grep` 可见 `NN方案计划.md`、`phase3_plan_20261020.md`、`phase3_opponent_model_online_report.md`、`metamon_opponent_modeling_longterm_value_analysis.md`、`metamon-audit-plan\` 等）。
- 工作区 `C:\Users\abc\WorkBuddy\2026-10-10-14-15-51\reports\` 只有 **2 个 `.md`**，`frozen_logs\` 只有 **1 个文件** ⇒ **不要把它当主报告库 / 主日志源**。权威数据源 A–E 见 `docs/12`。

---

## 4. 证据标签体系（**本库约定，无外部书面定义**）

【实测】工作区 `.workbuddy\memory\*.md` 与 `E:\Tools\foul-play\reports\*.md` 中**没有**这六个标签的定义 —— 它们是**整理本知识库时新建的约定**，不是项目历史里既有的规范。

**定义处（在本仓库内，Codex 可直接读）**：`AGENTS.md` §4「证据标记约定」表、`README.md`「证据标记约定」表、`docs/12` 证据等级段。三处一致，**以 `AGENTS.md` 为准**。

> ⚠️ 别混用另一套标签：取证材料曾提到「【确证】/【推断】/【未查到】」三态，并指向某份 `ps_bot_flag_root_cause_20261023.md`。本次实测：`find` 在 `E:\Tools\foul-play\reports\` 下**找不到该文件**，`grep -rn '【确证】|【未查到】'` 全库**零命中** ⇒【待核】，**不要凭转述补写进本库**。本仓库只用上面六类标签。

| 标记 | 含义 | 处理 |
|---|---|---|
| 【实证】 | 本地日志 / 实测 / 命令原始输出可复核 | 可用，改动前复核来源 |
| 【源码】 | PS 或 foul-play 源码行号可证 | 可用；线上版本可能与本地不同 |
| 【推测】 | 原资料明确标注未证实 | **不得当结论、不得据此改代码** |
| 【待核】 | 悬案 / 未闭环 | 先验证再动手 |
| 【更正】 | 跨文档推翻旧结论 | **以更正后为准**，旧文保留 |
| 【待确认】 | 无法从本地核实的下载地址 / 版本 | **不要凭记忆填 URL**，向用户索取 |

**维护纪律**：【推测】与【待核】**永远保留标签，不因时间推移自动升格为事实**（`README.md` 维护约定）。把已核实事实误标成【待确认】同样是错误 —— 本次整理就发生过一次（`docs/01` §5.1 曾把已存在的 Kakuna 权重直链标成待确认，已【更正】）。

---

## 5. 工作流纪律（**逐条给出可核验出处**）

> 出处全部在工作区 `C:\Users\abc\WorkBuddy\2026-10-10-14-15-51\.workbuddy\memory\`。这些文件**不在本仓库内**，所以本表把结论一并转写成 `AGENTS.md` 红线；Codex 以 `AGENTS.md` 为准即可。

| 纪律 | 原始出处（文件:行号） | 原文要点 |
|---|---|---|
| 默认 OFF 不擅自打开 | `2026-10-22.md:4`（「用户拍板：默认 OFF、只改源码 + pytest 验证；谨慎改、有依据」）、`:11`（「铁律：两开关默认 OFF，连天梯前绝不打开」）、`MEMORY.md:25` | 每一项默认 OFF 开关的开启动作都需用户明确授权 |
| 拟人节奏类旋钮绝不用于真天梯 | `2026-10-23.md:67` | 「旧号标记撤不掉且踩 PS 10-09 新政红线；只在未登记新号上有边际意义」 |
| 随跑随冻、只用冻结副本 | `2026-10-18.md:90`（「live logs/ 会边查边轮转…**长任务必须随跑随冻，live logs 不能当唯一数据源**」）、`2026-10-22.md:19`、`MEMORY.md:13` | 轮转上限 ≤200 文件 / 2GB（见 `docs/01` §2） |
| 判「对方」靠账号名，不靠 p1/p2 | `2026-10-21.md:145`、`:188`、`MEMORY.md:12` | 我方账号有时是 p2；胜负用 `|win|<name>` |
| 统计按 `battle_tag` 去重 | `2026-10-21.md:207` | 「三元组库按 (user,pokemon) 记观测，同一局被计入 6 次 → 不去重会让 1700+ 局数虚高 **7 倍**（492 vs 67）」；`:204` 去重后 67 局 / 35 唯一 / 16 重复 / 45.7% |
| 主键是 `(battle_tag, rqid)` | `2026-10-17.md:226` | 1181 个请求仅 159 个不同 rqid |
| `logs/` 是滚动流，按 tag 跨文件重组 | `2026-10-16.md:19` | 「每局开头落在上一局文件尾部」，因 `do_rollover` 在 `get_battle_tag_and_opponent` 返回后才切流 |
| 删除类操作先定安全准则 | `2026-10-21.md:18-19` | 「只删已有冻结副本的、保护 mtime 最新 5 个、不动 init.log」；结果删 15 个腾 1.06GB |
| 终止进程前确认进程树 | `docs/09` 事故、`rating_drop_diagnosis_20261018.md:88-93` | 父子双 `run.py` 是正常形态，误杀过活 bot |
| 任何「格式是 X」断言贴原始行 | **原始出处** `2026-10-21.md:215-222`；转写于 `AGENTS.md` §8 第 5 条 / `docs/12` §5 | 事故原文：凭记忆手写一条 `|move|` 样例去否定他人正确实现，被源 D **1215 条实测**反证后撤回 ⇒「任何『格式是 X』的断言，必须贴**从文件直接抓的原始行**（连同文件名与行号），不得手写构造；**尤其在否定他人实现时**」。同处写定：`|move|` = `<使用者>|<招式名>|<目标>|<附加>`，**parts[2] 是招式名** |

> ⚠️ **本表是转写，不是原文副本**。工作区 memory 是**日志式**记录（按日期追加），不是规范手册；同一条纪律在不同日期的文件里可能有更细或略异的表述。有冲突时以 `AGENTS.md` + 本知识库【更正】段为准。

---

## 6. 本知识库是怎么造出来的（可复用的取证流程）

**并行只读取证**（对应 `team-collaboration` / `parallel-execution` 两个 skill 的语义）：同时派出 5 个互不重叠的只读取证任务，各自带「只改不改、贴路径+行号、每条标证据等级」的约束：

| 子任务 | 范围 | 产出流向 |
|---|---|---|
| 代码地图 | `fp/config.py` 45 flag、调用链、模块职责、测试清单、git 状态 | `docs/02`、`docs/03` |
| 报告与实验 | `E:\Tools\foul-play\reports\` 全批 A/B、D-1/D-2、20 局、NN/Phase3 | `docs/06`、`docs/07` |
| 早期历史 | L2 落地、490 连败、并发六缺陷、隐身、队伍池时序 | `docs/05`、`docs/09` |
| 近期历史 | 对手建模、PS 服务端、匹配墙、dispatcher 修复、悬案 | `docs/04`、`docs/08`、`docs/11` |
| 环境与 skills | venv、下载方法、19 个 skill、工具链 | `docs/01`、`docs/10` |

**结果合并纪律**（关键，避免文档互相打架）：

1. 子任务报告是**二手转述**，凡要写进库的硬事实（URL、行数、行号、参数默认值、文件大小）**一律自己再跑一次只读命令复核**。本次就因此抓到并更正了 4 处：Kakuna 直链误标待确认、`t.py` 大小、`tmp/` 脚本计数、`verify_evidence.py` 不存在。
2. 跨日结论冲突**不删旧文**，写【更正】段保留演变链（例：「五个 bot 号全 userType=10」被逐号实测推翻；「1700+ 建模有效窗只在同日」被分母口径更正）。
3. 数字冲突（pytest 947 vs 887）标明**两套序列口径不同**，以实际运行输出为准，不硬选一个。
4. 「计划预期」与「当前实测」分开写（例：Phase3 规划 21+ 单测 vs 当前 11 个顶层用例）。
5. 全程**不写凭证、不写攻击研究、不写实时日志、不写 checkpoint 二进制**；推送脚本内置凭证扫描（见 §7）。

---

## 7. 交付与推送工具（本库自身的工程资产）

`C:\Users\abc\WorkBuddy\2026-10-10-14-15-51\tmp\push_kb_to_github.js`（**不在本仓库内**，是一次性交付脚本）：

- 用 GitHub **Git Tree API** 把 `tmp\repo_export\` 下所有 `.md` / `.txt` 合成**单个 commit** 推到 `n7140148-svg/pokemonshowdown-intel-robert` 的 `main`。
- **token 只从 `process.env.GITHUB_TOKEN` 读取**，绝不硬编码、绝不写入任何文件。
- 推送前**凭证扫描**：正则拒绝含 `github_pat_…` / `ghp_…` / PEM 私钥头的文件内容（命中即 `process.exit(2)`）。
- 支持 `--dry-run`（只列文件与字节数，不联网写入）。
- 流程：`GET /git/ref/heads/main` → `POST /git/trees`（带 `base_tree`）→ `POST /git/commits` → `PATCH /git/refs/heads/main`（`force: false`）。

> ⚠️ `base_tree` 方式**不会删除仓库里已有的其它文件**；如需移除旧文件，要显式在 tree 里传 `sha: null`，本脚本没这么做。

---

## 8. Codex 复现检查单

1. **不要装 skills**，也不要找用户级 `MEMORY.md`；读 `AGENTS.md` 就够了。
2. 环境与依赖照 `docs/01`：主 venv 用 `E:\Tools\foul-play\.venv\Scripts\python.exe`；**Python 一律用仓库自带 venv**（全局 python 没有 pytest）；`poke-engine` 的 `--config-settings` 整行照抄；**Rust 是构建前置**。
3. 要拿 Kakuna 权重：`docs/01` §5.1 有已核实的完整直链与落地路径（本地已有 571,581,627 字节的文件，**别重复下载**）；复用脚本时注意它关闭了 TLS 校验，应恢复并核对 sha256。
4. 要查对手库：用 `docs/01` §4 的 `query_elite.py`（`--user` / `--pokemon` / `--team-match` / `--list` / `--stats` / `--topn`）。
5. 跑任何日志分析前：先做**冻结副本**，传冻结目录给脚本；统计**按 `battle_tag` 去重**；断言帧格式时**贴原始行 + 文件名 + 行号**。
6. 改代码前：`docs/11` 查悬案 → `docs/03` 查开关 → `git status` 看工作树 → 定位根因到**文件路径 + 行号 + 触发条件 + 影响面**。
7. 改代码后：跑全量 pytest 并报**你自己的输出**（别引用文档里的历史数字）、补挡同类缺陷的回归测试、更新对应文档并写清日期 + 来源。
8. 任何默认 OFF 开关、真天梯运行、终止进程类操作，**先取得用户明确授权**。
9. 新结论与本文冲突 → 在 `docs/11` 记【更正】，不删旧文。
