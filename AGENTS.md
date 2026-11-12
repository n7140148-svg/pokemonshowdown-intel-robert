# AGENTS.md — Codex 接手指令

> **这是本仓库的入口文件。任何接手本项目的智能体 / 工程师，开工前必须完整读完本文。**
> 本仓库是「FoulPlay × Metamon 对战机器人」的**工程知识库 / 交接文档**，不是源代码仓库。真正的代码在 `E:\Tools\foul-play` 等本地路径（见 `docs/00` 与 `docs/01`）。
> 资料截止 **2026-10-25**，整理成库 **2026-11-12**。

---

## 1. 项目是什么（一句话）

以第三方 bot 框架 **foul-play**（MCTS 决策、连接 Pokémon Showdown）为本体，接入 **metamon** 的策略模型，并用真实天梯对战日志做**离线审计、对手建模、决策质量评估**的长期研究工程。

**不是**冲分产品，也**不是**已发布服务。核心产出 = 可复现的工程资产 + 经过验证/否决的研究结论 + 可交接的知识。

**Codex 的主要工作对象**：`E:\Tools\foul-play`（Python bot）。`E:\Tools\src\metamon`（研究框架）、`E:\Tools\pokemon-showdown`（本地服 :8000）、`E:\venvs\metamon-audit\`（隔离审计环境）是配套。

---

## 2. ⛔ 红线（违反会造成真实损失，逐条记住）

1. **默认 OFF 的开关绝不擅自打开**，尤其连真天梯时。涉及：
   - `--concurrent-battles`（单账号多局并发）
   - `--team-pool-source editor`（多队伍随机选队；默认 `file`，`editor` 需显式开启）
   - `--opponent-model-enabled` / `--opponent-record-enabled`（对手建模注入 + 自生长）
   - MCTS 变体 `argmax(B1)` / `threads(B5)` / `threads=2`（均已否决）
   打开任何一项前，必须回 `docs/03` 看关闭原因并**取得用户明确授权**。历史事故：误开并发 → 490 连败、rating 1655→1003。
2. **绝不读 / 写 / 删 `E:\Tools\foul-play\logs\`**。那里是 1.9GB+ 实时日志，会轮转实时删文件，且 bot 可能正在跑。只读分析一律用**冻结副本**（工作区 `frozen_logs\` 或自建冻结目录）。
3. **绝不把凭证写进任何文件**：GitHub PAT、Pokémon Showdown 账号密码、登录凭证，**绝不**写进代码、文档、日志、git 历史、commit message。凭证由用户在运行时通过命令行 / 环境变量提供。
4. **探针脚本绝不向服务器发 `/trn` 认领身份** —— 会把正在跑的 bot 踢成 Guest。
5. **判「对方」靠账号名匹配，绝不靠 p1/p2**（我方账号有时是 p2）。胜负用 `|win|<name>`。
6. **不连真天梯做实验**，除非用户明确授权；端到端验证走本地 PS :8000。
7. **不跑未授权的推理 / 不装包 / 不改 venv**：需要 metamon/torch 的推理只走 `E:\venvs\metamon-audit\venv`；foul-play 主 venv **没有 metamon**。
8. **终止类操作前先确认进程树并取得用户确认**：父子双 `run.py` 是正常形态（父 CPU 0s / 内存 3MB），勿误杀。
9. 本仓库**不含**针对线上服务的攻击研究、威胁建模、渗透手法。别把那类内容写进来。

---

## 3. ⭐ 开工前必读：结论演变

> **本项目里有多条跨日被推翻的旧结论。引用旧结论去改代码 = 建立在错误前提上。**

最高优先级的一条：
- 2026-10-23 曾盖棺「**五个 bot 账号全部 userType=10、被匹配墙墙死**」。
- 2026-10-25 逐号实测**推翻**：`thingsyoudo2me=4`、`iknow_why=4`、`nutshell93=10`；`FpConcA`/`FpConcB` **未读**。
- ⇒ 「五个号全被墙死」的归因**从一开始就错**。涉及账号标记 / 匹配行为，必须**逐号读 `/trn` 断言**重新实测。

其它冲突（`userType=10` 是否置 `allowLadderBots`、峰值评分 1773/1792/1812、35 场 vs 54 局 vs「56W/58L」、评分窗口有无）列在 **`docs/11-open-items-and-roadmap.md` 第 1 节**。改任何相关逻辑前先读它。

> 【已更正·不再是悬案】`|move|` 字段位置：源 D 1215 条实测确认 `|move|<使用者>|<招式名>|<目标>|<附加>`，**parts[2] 是招式名**；旧记「parts[2] 是目标」已作废。见 `docs/04` §5、`docs/08` §4、`docs/12` §2。

---

## 4. 证据标记约定（**不得升格**）

| 标记 | 含义 | 处理 |
|---|---|---|
| 【实证】 | 本地日志 / 实测 / 命令原始输出可复核 | 可用，改动前复核来源 |
| 【源码】 | PS 或 foul-play 源码行号可证 | 可用；线上版本可能与本地不同 |
| 【推测】 | 原资料明确标注未证实 | **不得当结论、不得据此改代码** |
| 【待核】 | 悬案 / 未闭环 | 先验证再动手 |
| 【更正】 | 跨文档推翻旧结论 | **以更正后为准** |
| 【待确认】 | 无法从本地核实的下载地址 / 版本 | **不要凭记忆填 URL**，向用户索取 |

---

## 5. 文档地图（按任务选读）

| 文件 | 内容 | 何时读 |
|---|---|---|
| `README.md` | 仓库定位、范围、证据约定 | 第一次 |
| **`docs/11-open-items-and-roadmap.md`** | **悬案台账 + 结论演变 + 路线 + 接手检查单** | **每次开工** |
| `docs/00-project-overview.md` | 三层架构、路径布局、技术栈、术语表、铁律速查 | 第一次接手 |
| `docs/01-environment-and-toolchain.md` | venv、本地 PS 服务器、工具链、**下载 / 重建方法** | 搭环境、复现 |
| `docs/02-codebase-map.md` | 代码地图、调用链、模块职责、**精确行号** | 定位代码 |
| `docs/03-features-and-flags.md` | 已实现功能 + **全部命令行开关（含默认 OFF）** | 改功能 / 开开关前 |
| `docs/04-opponent-modeling.md` | 对手建模库、数据源、能力上限、Part A/B、污染修复 | 改对手建模 |
| `docs/05-metamon-integration.md` | L2 首发模型、lead_advisor、vendored 最小集、**测试基线演进表** | 改首发决策 |
| `docs/06-experiments-and-results.md` | MCTS A/B、Metamon 离线审计、20 局报告、实战成果 | 评估方案 |
| `docs/07-alternative-models.md` | **未采用 / 被否决 / 研究中**的模型与辅助形态 | 选新方案前（避免重复踩坑） |
| `docs/08-ps-server-knowledge.md` | userType、匹配墙、日志帧、评分、匹配路径约束 | 排查匹配 / 日志 / 评分 |
| `docs/09-incidents-and-lessons.md` | 事故台账（现象→根因→修复→回归）+ 横向教训 checklist | 回归调试 |
| `docs/10-skills-and-workflows.md` | WorkBuddy skills、工作流、取证纪律 | 复用工具与流程 |
| `docs/12-data-sources-and-repro.md` | 数据源 A–E 权威等级、帧格式、统计纪律、复现命令 | 做数据 / 审计 |

---

## 6. 环境与命令（速查；详见 `docs/01`）

| 项 | 值 |
|---|---|
| 主工程 | `E:\Tools\foul-play` |
| Python | **一律用仓库自带** `E:\Tools\foul-play\.venv\Scripts\python.exe`（全局 python 没有 pytest） |
| 测试 | `E:\Tools\foul-play\.venv\Scripts\python.exe -m pytest`；memory 序列最新 **947 passed**，但 `reports/` 另记 887 等数字，**口径冲突 ⇒ 以你实际跑出的输出为准**（`docs/02` 第 6 节） |
| 隔离审计环境 | `E:\venvs\metamon-audit\venv`（metamon editable + torch CPU）；**不要用它跑 foul-play 主程序** |
| 本地 PS 服务器 | `E:\Tools\pokemon-showdown`，:8000；需 `noguestsecurity=true` + `noipchecks=true`；`noipchecks` 使 `matchmakingOK` 提前 return true（绕墙） |
| 对手建模库 | `E:\venvs\metamon-audit\opponent_modeling\` |
| 主连接日志 | `E:\Tools\foul-play\logs\init.log`（**归档/冻结副本**，含 `|player|` / `|win|` / `/trn`）；per-battle 日志**缺** `|player|` 头 |

**跑测试前**：确认没有改任何默认 OFF 开关的值；确认不会读 `logs\`。若前台 pytest 被 SIGTERM，改用后台运行再读输出。

---

## 7. Codex 的工作约定

**改动前**
1. 读 `docs/11` 确认这件事不是已知悬案，且不以作废结论为前提。
2. 读 `docs/03` 确认涉及的开关状态与关闭原因。
3. 用 `git status` / `git log` 确认当前工作树 —— 本项目曾长期积压未提交改动，`main` 领先 `origin/main` 数个 commit，**提交策略待用户决定**（见 `docs/11` 第 2 节 A）。
4. 定位根因到**文件路径 + 行号 + 触发条件 + 影响面**，不凭现象猜着改。

**改动时**
- 保持 flag-OFF（默认）路径行为**逐字不变**（历史硬要求）。
- 异常路径显式 `try/finally` 兜底；任何"永久 await"要有超时或 sentinel 唤醒。
- 明确区分**观测**与**假设**（历史事故：注入的假设招式被当成实战观测写进自生长库）。
- logging 用预格式化字符串（foul-play 老毛病：`%s/%d` 不替换）。
- 不把"没有 `|win|`"记成败绩（会污染 W/L）。

**改动后**
- 跑全量 pytest 并报 passed 数；补能挡住同类缺陷的回归测试。
- 采用**怀疑式复核**而非橡皮图章，尤其改并发 / flag-OFF 路径时。
- 更新本仓库对应文档：写清日期 + 来源（文件路径:行号 或命令原始输出）；若与旧结论冲突，在 `docs/11` 记【更正】，**不删旧结论**。

**验证纪律**
- 验证 = 证明代码能跑通，不是确认文件存在。
- 端到端验证走**本地 PS :8000**，不连真天梯。
- 涉及默认 OFF 功能的验证，需用户明确授权才可在真机启用。

---

## 8. 铁律速查（完整版 `docs/03` / `docs/11`）

1. 默认 OFF 开关绝不擅自打开。
2. runtime 库（`runtime_opponent_log.jsonl`）**绝不碰 curated 库**（`elite_profiles.jsonl`）；两者物理独立。
3. 对手建模只 plant **已披露真子集**，绝不臆造 ability / tera / 完整配招 / EV（日志里根本不存在）。
4. 长任务**随跑随冻，只用冻结副本**。
5. 任何「格式是 X」断言必须贴**从文件直接抓的原始行（文件名 + 行号）**，不得手写构造。
6. 统计必须说明**字段来源 + 分母口径 + 去重规则**（按 `battle_tag` 去重，否则局数虚高 ~7 倍）。
7. 不把凭证写进任何文件 / 日志 / git 历史。
8. 终止类操作前确认进程树并取得用户确认。

---

## 9. 仓库定位

- GitHub：`n7140148-svg/pokemonshowdown-intel-robert`（**private**），默认分支 `main`。
- 本仓库只放**文档**，不放代码、凭证、二进制、checkpoint、实时日志。
- 新增结论时：写清日期 + 来源，保留证据标记，冲突走【更正】不删旧文。
