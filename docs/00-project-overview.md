# 00 · 项目本体与总览

> 证据标记约定见 [`../README.md`](../README.md)。本文标注【实证】【源码】【推测】【待核】【更正】。
> 资料截止：**2026-10-25**，整理成库：**2026-11-12**。所有"当前状态"以此日期为准，之后须重新核实。

---

## 1. 一句话定义

这是一个**围绕 Pokémon Showdown gen9 OU 对战机器人的长期研究 + 工程项目**：以第三方 bot 框架 `foul-play`（MCTS 决策）为本体，接入 `metamon` 研究的策略模型，并用真实天梯对战日志做离线审计、对手建模与决策质量评估。

**它不是**：一个要"冲天梯分数"的产品，也不是一个已发布的服务。核心产出是**可复现的工程资产 + 经过验证/否决的研究结论**。

## 2. 目标与成功标准

| 层次 | 目标 | 判据 |
|---|---|---|
| 工程底座 | bot 能稳定、可控、默认安全地在真机 / 本地服跑完对局 | 全量 pytest 全绿；本地 PS 端到端通过；默认 OFF 开关绝不误开 |
| 决策质量研究 | 量化 bot 决策与更强策略的分歧，定位「盲区」 | 离线审计产出带胜负回标的分歧清单；结论须分层、注明 MDE 与分母口径 |
| 对手建模 | 从历史日志重建高分段对手的固定队/已披露配置，安全注入 | curated 库稳定、runtime 库自生长且绝不污染 curated；注入只 plant 真子集 |
| 可交接性 | 新接手者（含 Codex）无需聊天记录即可继续开发 | **本仓库**即判据 |

## 3. 三层架构（Pokémon Showdown 侧）【源码】

```
[ Client：foul-play / 浏览器 ]
   │  ① 连 Game server，收 |challstr|
   │  ② 拿 challstr 去 Login server 换 assertion
   │  ③ 把 assertion 放进 /trn 发给 Game server
   ▼
[ Login server  (独立仓库 pokemon-showdown-loginserver, MIT) ]
   - 持有私钥；签发 RSA-SHA1 assertion
   - assertion 内容：challenge,userid,userType,date,hostname[,s1,s2,s3]
   - 决定 userType（含 isUserBot 标记）、rating、账号状态
   ▼
[ Game server  (pokemon-showdown, 即本项目的 E:\Tools\pokemon-showdown 同构镜像) ]
   - **只持公钥，无私钥**；用 verifier 校验 assertion 签名
   - 校验通过后完全信任 userType，据此决定 registered / isUserBot / 权限
   - 负责房间、匹配、对战协议帧、日志
```

本地实证锚点（我方只读核对）：
- `E:\Tools\pokemon-showdown\config\config.js:191-194` → `loginserver='http://play.pokemonshowdown.com/'`、`loginserverkeyalgo="RSA-SHA1"`、`loginserverpublickeyid=4` + RSA 公钥。
- `E:\Tools\pokemon-showdown\server\users.ts:640-713` → `validateToken()` 六步校验：拆 `tokenData;tokenSig` → hostname ∈ `legalhosts` → 字段数 ≥5 → `signedUserid === userid` → `signedChallenge === connection.challenge` → 时间差 ≤ `tokenexpiry`（默认 **25 小时**）→ 验签；通过则 **返回 userType 并被全链路信任**。
- `E:\Tools\pokemon-showdown\server\verifier.ts:21-25` → `crypto.createVerify('RSA-SHA1').verify(公钥, sig, 'hex')`，跑在子进程池。
- `config.js` 中 **`legalhosts` 与 `tokenexpiry` 均未配置**（grep 零命中）→ 本地服不检查 hostname（跨服重放防护缺失），expiry 走默认 25h。**这是自托管部署清单里必须补的一项**。

> ⚠️ 本节只描述**工程机制**（为了排查登录/匹配/评分问题）。本仓库不包含任何针对线上服务的攻击研究或操作手法，见 `README.md`「没有什么」。

## 4. 本地资产布局【实证，待 `docs/01` 逐项核实】

| 路径 | 角色 |
|---|---|
| `E:\Tools\foul-play` | **主工程**：Python 对战 bot，MCTS 决策，连接 PS；自带 `.venv` |
| `E:\Tools\src\metamon` | 研究框架：策略模型 / Kakuna / 离线数据集；L2 首发模型来源 |
| `E:\Tools\src\amago` | 若存在：metamon 依赖的 RL 框架 |
| `E:\Tools\pokemon-showdown` | 本地 PS 服务器（:8000），用于不连真天梯的端到端验证 |
| `E:\venvs\metamon-audit\venv` | 隔离审计环境（metamon editable + torch CPU），**与 foul-play 的 venv 分开** |
| `E:\venvs\metamon-audit\opponent_modeling\` | 对手建模库与查询脚本（见 `docs/04`） |
| `C:\Users\abc\WorkBuddy\2026-10-10-14-15-51\` | WorkBuddy 工作区：报告、冻结日志、脚本、`.workbuddy\memory\` 项目日志 |
| `C:\Users\abc\.workbuddy\skills\` | 用户级技能包（见 `docs/10`） |

**关键隔离原则**：foul-play 的 `.venv` 里**没有 metamon**；所有需要 metamon/torch 的推理必须走 `E:\venvs\metamon-audit\venv`。L2 首发模型之所以能进 bot，是把推理最小集 **vendored** 进 `fp/lead_model/`，而不是装依赖。

## 5. 能力栈（按"已落地 / 研究中 / 已否决"分层）

| 状态 | 能力 | 详见 |
|---|---|---|
| ✅ 已落地 | 对战隐身 `hide_next_battle`（真机 5/5 PASS） | `docs/03` |
| ✅ 已落地 | 单账号多局并发 `--concurrent-battles`（**默认 OFF**） | `docs/03`、`docs/09` |
| ✅ 已落地 | 多队伍随机选队 `--team-pool-source editor`（**默认 `file`，`editor` 需显式开启**） | `docs/03` |
| ✅ 已落地 | dispatcher 断线兜底（`DispatcherDeadError` + sentinel），947 passed | `docs/09` |
| ✅ 已落地 | L2：Metamon gen9ou 首发模型接入 `handle_team_preview` | `docs/05` |
| ✅ 已落地 | 对手建模 Part A 注入 + Part B 自生长（**两开关默认 OFF**） | `docs/04` |
| 🔬 研究中 | 辅助形态 A：用 Kakuna 离线审计历史决策，产出「MCTS 盲区清单」 | `docs/07` |
| 🔬 研究中 | Metamon 离线审计 / γ 预注册协议（canonical 993 决策点、18 胜 15 负） | `docs/06` |
| ❌ 已否决 | MCTS argmax(B1) / threads(B5) —— 均未过保留规则，默认 OFF | `docs/06` |

## 6. 术语表（Codex 请先对齐）

| 术语 | 含义 |
|---|---|
| `userType` | 登录服 assertion 里的账号类型码：1 未注册 / 2 注册 / 3 sysop / 4 autoconfirmed / 5 permalocked / 6 permabanned / **10 bot** |
| `isUserBot` | Game server 由 `userType==='10'` 推出的标志，触发匹配墙与同 IP 墙；**无自动检测链路，只能 staff 施加** |
| assertion | 登录服用 RSA-SHA1 签发的身份串，格式 `challenge,userid,userType,date,hostname[,s1,s2,s3];sig`；userType 是逗号第 3 段（0-indexed `[2]`） |
| `matchmakingOK` | `ladders.ts` 的配对准入函数，四道关（IP 去重 / lastMatch / 评分窗口 / bot 墙） |
| `searchRange` | 评分窗口，gen9ou 初始 ±50，每 0.3s +1，>300 后每 3s +1，硬上限 ±600（约 16 分钟封顶） |
| `allowLadderBots` | 2026-10-20 上线的开关（默认 false）；被标 bot 号只能配其他 bot 或手动 opt-in 的人类 |
| battle tag | 对战房 id，两段/三段两种，正则 `battle-[a-z0-9]+-\d+(?:-[a-z0-9]+)?` |
| curated 库 | 人工审定的精英对手库 `elite_profiles.jsonl`，**只读、绝不自动写** |
| runtime 库 | `runtime_opponent_log.jsonl`，bot 实战自生长写入，独立于 curated，可安全重跑/删除 |
| team_signature | 6 只宝可梦组成的队伍签名，用于对手复现匹配 |
| LOO | Leave-One-Out，留一验证；`team-exact` 指队伍成员完全命中 |
| 冻结副本 / 随跑随冻 | live logs 会轮转实时删文件，长任务须边跑边复制到 `frozen_logs/` 一类目录，**只用副本** |
| MCTS | Monte Carlo Tree Search，foul-play 的行动决策引擎 |
| 辅助形态 A | 用 Kakuna 对历史决策点重推理 → 统计分歧 → 按胜负回标 → 产出盲区清单 |
| vendored 最小集 | 把 metamon 推理所需文件复制进 `fp/lead_model/`，避免主 venv 装 torch |

## 7. 状态快照（截至 2026-10-25）【实证，注意口径冲突】

- 测试基线：工作区 memory 序列记为 **947 passed**（dispatcher 修复 934 + 13），节点 792 → 893 → 904 → 905 → 947；但 `E:\Tools\foul-play\reports\` 里另有一套 813 / 819 / 822 / 838 / 854 / 863 / 887 的数字。**两套序列口径不一致**（不同时点 / 分支 / 子套件），仓库内 `README.md`、CI、`pytest.ini` 均无基线数字 ⇒ **以你实际跑出的 pytest 输出为准**，详见 `docs/02` 第 6 节与 `docs/05` 第 6 节。
- 对手建模库：3309 三元组 / 530 用户 / 330 物种（v2 归一化后）；35 精英 = 29 固定队 + 6 对照。
- 天梯账号：`thingsyoudo2me` / `iknow_why` / `nutshell93` / `FpConcA` / `FpConcB`；逐号 userType 实测存在**结论演变**，见 `docs/11`。
- git：工作树曾积压多轮未提交改动，`main` 领先 `origin/main` 数个 commit；**提交策略待用户决定**（见 `docs/11`）。
- 【待核】「56 胜 58 负」记忆与日志冲突；峰值评分 1773/1792/1812 三种记法。

## 8. 铁律速查（完整版见 `docs/03` 与 `docs/11`）

1. **默认 OFF / opt-in 的开关，绝不擅自打开**，尤其上真天梯时。涉及：`--concurrent-battles`、`--team-pool-source editor`（默认 `file`）、对手建模两开关、MCTS A/B 变体、`--human-pacing` / `--team-repeat-guard`。完整表见 `docs/03`。
2. **runtime 日志绝不碰 curated 库**。
3. **判「对方」靠账号名匹配，绝不靠 p1/p2**。
4. **长任务随跑随冻，只用冻结副本**，绝不把 live logs 当唯一数据源。
5. **任何「格式是 X」的断言必须贴从文件直接抓的原始行（文件名 + 行号）**，不得手写构造样例。
6. **探针脚本绝不向服务器发 `/trn` 认领身份**（会把正在跑的 bot 踢成 Guest）。
7. **不把凭证、密码、token 写进任何文件 / 日志 / git 历史**。
8. 终止类操作前先确认进程树并取得用户确认（父子双 `run.py` 是正常形态，勿误杀）。
