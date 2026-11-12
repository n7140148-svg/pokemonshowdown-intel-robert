# 12 · 数据源、日志解析规则与复现方法

> Codex 做**任何数据 / 审计 / 统计类工作前必读**。这里记录了数据权威等级、哪些数据会消失、以及三条用代价换来的方法论纪律。

---

## 1. 数据源清单与权威等级【实证】

| 源 | 路径 | 规模 | 权威性 / 用途 |
|---|---|---|---|
| **A** | `init_log_archive_20261019\init.log` | **8.31 GB / 680 万行 / 76 会话 / 661 tag** | ⭐ **唯一完整 rating 源**；主连接流，含 `|player|` / `|win|` / `/trn` / `|updatesearch|` / 队伍 `|utm|` |
| **B** | `today_snapshot\` | 33 文件 | 无 `|player|`，对对手建模库**零增量** |
| **C** | `eraB_snapshot_20261018_2245\` | 32 文件 | 22 个完整 `|player|` 双行 |
| **D** | `live_snapshot_20261021\` | 37 文件 | **冻结副本**，`|player|` 完整带 rating；`|move|` 格式实测 1215 条 |
| **E** | `logs_incident_20261017_backup\` | 490 文件 | 可用增量 0，但 **rating 轨迹唯一来源**（242,187 条观测）；对应 490 连败事故 |
| 工作区 | `C:\Users\abc\WorkBuddy\2026-10-10-14-15-51\frozen_logs\` | 冻结日志 | 长任务用的副本 |
| 工作区 | `C:\Users\abc\WorkBuddy\2026-10-10-14-15-51\reports\` | 报告 | 研究结论文本 |
| 工作区 | `C:\Users\abc\WorkBuddy\2026-10-10-14-15-51\.workbuddy\memory\` | 16 个日志 + MEMORY.md | **项目过程记录**（本知识库的主要来源） |

**构建三元组库**：`build_user_config.py` 从源 A 流式读取，实测 **116 秒 @ 58 MB/s**；`selfcheck_user_config.py` 输出 `ALL CONSISTENT`；统计写 `build_stats.json`。

## 2. ⚠️ live logs 会消失【实证 · 铁律】

`E:\Tools\foul-play\logs\` 的 live 日志受轮转策略约束（**≤200 个文件 / 2GB**），实测 2.01GB 时已逼近阈值，**核查过程中会实时删文件**。

历史教训：
- attributor 的 4 份 490-loss 时代日志（tags 2699xxx）**已被轮转删除**，导致归因分析无法回做。
- 10-25 重建时间线时只能改用 `ls -lat` + `init.log` 反向重建。

⇒ **纪律**：长任务必须**随跑随冻**（边跑边把需要的文件复制到 `frozen_logs/` 一类目录），**只用冻结副本做分析**，绝不把 live 目录当唯一数据源。

同理，`init.log`（源 A 的归档版）本身是 1.7–8.3GB 级大文件，普通 `Read` 工具会报 "File is too large" ⇒ 用 `sed -n 'a,bp'` / `grep` / `awk` 流式处理，不要整文件读入。

## 3. PS 协议帧速查（照抄，别手写样例）【实证】

| 帧 | 格式 / 要点 |
|---|---|
| `|request|` | **只含自己 side**；`side.id` + `side.name` 是归属权威；`side.pokemon[]` 含 details / moves / baseAbility / ability / item / teraType；`request.stats` 是**合成后实际能力值，不能反推 EV** |
| `|player|` | 评分在 `split_msg[5]`；同一 tag 会因重连重复（457 个 tag 恰 2 行、138 个 3 行、最多 ×19）；**per-battle 日志反而缺 `|player|` 头** |
| `|win|` | `|win|<name>`；判胜负用名字匹配，**不靠 p1/p2** |
| `|move|` | `|move|<使用者>|<招式名>|<目标>|<附加>` → **parts[2] 是招式名**（源 D 1215 条实测） |
| `|poke|` | 行尾 `|` 是空字段分隔符；物种名拆 species + form + gender |
| `|enditem|` / `|detailschange|` | 对手侧少数可得的道具 / 形态信息 |
| `|updatesearch|` | `{"searching":["gen9ou"]}` 表示已进匹配池 |
| `|challstr|` | 登录挑战串，用于换 assertion |
| `/trn` | `|/trn <user>,0,<assertion>`；assertion 第 3 逗号段（0-indexed `[2]`）= userType |
| battle tag | 两段 / 三段两种：`battle-[a-z0-9]+-\d+(?:-[a-z0-9]+)?` |

**对手侧可得性**（决定建模上限）：对手 item / ability / teraType / 完整 moves **在日志里不存在**，只能拿 `|poke|` / `|move|` / `|enditem|` / `|detailschange|` 的已披露部分。覆盖率：species 100%、gender 64.8%、moves 21.5%、item 3.5%、ability/tera/完整配招/_stats **0%**。

## 4. 统计纪律【实证 · 三条代价换来的规则】

1. **按 battle_tag 去重**：观测级统计不去重会让局数虚高约 **7 倍**。
2. **统一分母口径**：曾把「1700+ 档复现率 45.7%（分母 35）」与「全档跨日率 20.4%（分母 530）」直接比较，得出不成立的 2.24× 叙事。任何跨源复现率对比先说明分母。
3. **贴原始行**：任何「格式是 X」断言必须贴**从文件直接抓的原始行（带文件名 + 行号）**，不得手写构造样例。曾凭记忆手写 `|move|` 样例去否定他人正确实现，被 1215 条实测反证后撤回。

**其它踩过的坑**：
- `grep '2701[0-9]{6}'` 命中 PokemonSet 调试行里的科学计数浮点数（`1.2701874131101882e-07`），一度误判「空窗期有 6 场隐形对局」⇒ 正则结论必须逐条看上下文。
- `awk` 取错字段得出「MCTS 均值 220 次迭代」，正确值 258,056 ⇒ 数字必须说明字段来源。
- avatar 字段可能是字符串（如 `cynthia`），rating 解析正则别写 `[0-9]*`。
- `|player|` 直接数行会把 1 局算成多局（重连重复）。

## 5. 常用复现命令（**只读 / 流式；按需替换路径**）

```bash
# 提取 bot 自己的 /trn 断言，读 userType
grep -oE '/trn [^,]+,0,[^,]+,[^,]+,[0-9]+' init.log | awk -F, '{print $1,$5}'

# 从 init.log 提评分轨迹与胜负（主连接流才有 |player|）
grep -E '^\|player\||^\|win\|' init.log > frozen_players.txt

# 查匹配池状态
grep -o '"searching":\[[^]]*\]' init.log | sort | uniq -c

# 按 battle_tag 去重后统计
# （先冻结副本，再对副本做，不要在 live 目录上跑长任务）
```

**冻结副本示意**：
```bash
# 只复制需要的文件到冻结目录，绝不移动/删除 live 文件
mkdir -p <冻结目录>
cp -n E:/Tools/foul-play/logs/battle-gen9ou-<tag>*.log <冻结目录>/
```

> ⚠️ 上面是**方法示意**，路径与 tag 需按当次任务替换。所有命令都是只读或复制到新目录；**绝不修改 / 删除 live logs**。

## 6. 工具脚本（工作区自带）【待 `docs/01` 补齐参数】

| 脚本 | 用途 |
|---|---|
| `scan_logs.py` | 扫描日志、提取结构化信息 |
| `reconcile_counts.py` | 对账 / 计数一致性复核 |
| `E:\venvs\metamon-audit\opponent_modeling\build_user_config.py` | 构建三元组库 |
| `query_user_config.py` / `query_elite.py` | 查询库 |
| `selfcheck_user_config.py` | 一致性自检 |
| `E:\Tools\foul-play\tools\replay_lead_advisor.py` | L2 首发模型历史日志回放 |

## 7. 证据等级速查

写结论时按此标注，禁止升格：
- 【实证】本地日志 / 实测 / 命令原始输出可复核
- 【源码】PS 或 foul-play 源码行号可证
- 【推测】原始资料明确标注未证实
- 【待核】悬案 / 未闭环
- 【更正】跨文档推翻旧结论，以最新为准
- 【待确认】无法从本地核实的下载地址 / 版本号（**不要凭记忆填 URL**）
