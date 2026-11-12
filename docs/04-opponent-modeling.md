# 04 · 对手建模库（Part A 注入 + Part B 自生长）

> 来源：`C:\Users\abc\WorkBuddy\2026-10-10-14-15-51\.workbuddy\memory\` 的 `2026-10-21.md`、`2026-10-22.md`、`MEMORY.md`。
> 这一章是「已落地且默认关闭」的功能，Codex 改它之前必须读完第 6 节的能力上限和第 8 节铁律。

---

## 1. 它在解决什么问题

高分段对手**大量使用固定队伍**（29/35 = 82.9% 全程单队）。历史日志里能观察到对手已披露的宝可梦、招式、道具，若能在下次遇到同一对手时把这些**已知信息**注入到 bot 对对手的建模中，MCTS 搜索就能少依赖 Smogon 通用 fallback 的臆测。

**关键设计取舍**：只注入协议里**确实观测过**的信息（真子集），**绝不臆造**未披露的招式/道具。这决定了整个库的能力上限，也决定了它不会把 bot 带偏。

## 2. 库位置与文件清单【实证】

根目录 `E:\venvs\metamon-audit\opponent_modeling\`

| 文件 | 类型 | 用途 |
|---|---|---|
| `user_pokemon_config.jsonl` | **交付产物** | 用户 × 宝可梦 × 观测配置三元组库（v2：3309 条） |
| `config_index.json` | 索引 | 三元组库索引 |
| `build_user_config.py` | 脚本 | 从源 A 流式构建（8.31GiB，实测 116s @58MB/s） |
| `query_user_config.py` | 脚本 | 查询三元组库 |
| `selfcheck_user_config.py` | 脚本 | 一致性自检（结果 ALL CONSISTENT） |
| `build_stats.json` | 产物 | 构建统计 |
| `elite_opponents\elite_profiles.jsonl` | **curated 库** | 35 个精英对手画像（29 固定队 + 6 多队型对照） |
| `elite_opponents\query_elite.py` | 脚本 | 精英库查询 |
| `runtime_opponent_log.jsonl` | **runtime 库** | bot 实战自生长写入，独立于 curated |

`query_elite.py` 用法【实证】：`load` + `--user` / `--pokemon` / `--team-match` / `--list` / `--stats`。

> `user_pokemon_config.jsonl` / `config_index.json` / `build_*.py` / `query_user_config.py` 等文件位于 `opponent_modeling\` 根目录；`elite_profiles.jsonl` 与 `query_elite.py` 在 `elite_opponents\` 子目录（不同文档写作时层级表述略有差异，以实际磁盘为准）。

## 3. 权威数据源 A–E【实证】

| 源 | 路径 / 规模 | 权威性 |
|---|---|---|
| **A** | `init_log_archive_20261019\init.log`，8.31GB / 680 万行 / 76 会话 / 661 tag | **唯一完整 rating 源** |
| **B** | `today_snapshot\`，33 文件 | 无 `|player|`，对本库零增量 |
| **C** | `eraB_snapshot_20261018_2245\`，32 文件 | 22 个完整 `|player|` 双行 |
| **D** | `live_snapshot_20261021\`，37 文件 | **冻结副本**，`|player|` 完整带 rating |
| **E** | `logs_incident_20261017_backup\`，490 文件 | 可用增量 0，但 **rating 轨迹唯一来源**（242,187 条观测） |

**纪律**：源 D 是"随跑随冻"的冻结副本；live logs 会在核查过程中实时删文件（≤200 个 / 2GB），长任务绝不能拿 live 目录当唯一数据源。

## 4. 关键统计【实证，注意 v1/v2 两个版本】

**库规模**
- v1：3557 三元组 / 532 用户 / 454 物种（物种虚高，因未做名字归一化）。
- **v2 最终：3309 三元组 / 530 用户 / 330 物种**；归一化前后唯一物种 484 → 348 → 330。

**精英层与验证**
- 35 elite = **29 固定队型 + 6 多队型对照**；可做 LOO 者仅 10 人。
- **LOO 队伍成员命中率均值 1.000、最小 1.000（6 只全中）**，`team-exact = 1.000`。
- `chusei44` 用 `--team-match` 做 Jaccard 自匹配 **J=1.000**，第二三名仅 0.333 → 队伍签名区分度极高。
- 29 个队伍签名中 28 个互异，仅 1 组重复（套队率 3.4%）。

**高分复现（决定建模有效窗）**
- 1700+ 档 **35 人、复现率 45.7%**（按 battle_tag 去重后 67 局 / 35 唯一 / 16 重复）；全档 **20.4%（108/530）**，高分档是全档的 **2.24×**（也是 `<1400` 档 19.0% 的 2.41×）。
- 分档梯度（`crosscheck_1700_report.md` §3 原表）：`<1400` **19.0%**(395人/75复现) → `1400-1499` **3.5%** → `1500-1599` **10.3%** → `1600-1699` **30.9%** → `>=1700` **45.7%**(35人/16复现/67局)。**不是全程单调**：中间两档明显低于 `<1400`，正确表述是「**rating 越高、尤其 1600+ 复现率越高**」。
- **29/35 全程单队（82.9%）→ 固定队型是高分段主流**。
- 1700+ 只占 530 对手的 **6.6%**、贡献 **67 局（9.4%）**。

**⚠️【更正·最高优先级】建模有效窗：不是「只在同一天梯会话内」**
- 工作区 memory 曾写「跨日复现 1.9% vs 同日 45.7% ⇒ 建模有效窗 = 同一天梯会话内」——**这是分母混用造成的误读，已在原始产物里被明确警告**。
- 产物原文（`opponent_repeat_report.md` §1）：**不能拿「1700+ 档 45.7%」去比「全档跨天 1.9%」，两者分母不同**。
- 正确数据（`crosscheck_1700_report.md` §4）：1700+ 复现者 16 人中 **7 人（43.8%）跨天复现**、9 人仅同日；全档 108 个复现者中仅 10 人（9.3%）跨天。以**对手为分母**：1700+ 跨天率 **20.0%（7/35）** vs 全档 **1.9%（10/530）**，相差 **10.6×**。
- ⇒ **结论与旧记忆相反：高分对手跨天也反复出现，低分段对手才是「只在当天有效」**。⇒ 用户级建模的价值**不限于单会话内**，建档后可跨天复用（这正是 29/35 固定队型能长期有效的依据）。
- 三方法交叉验证（`USER_MODELING_REPORT_20261021.md` §3.1）另有 34.5% / 31.6% / 37.1% / 36.7% 四个数字，属**早期阶段**（方法①只扫源 A+D、方法③多含源 C），与最终五源合并主口径 45.7% 的覆盖不同，**不要当成互相矛盾**。口径 A/B/C 定义：`A`=该对手曾出现 rating>1700 且总≥2 局（主口径 35/16/45.71%）；`B`=该对手所有局都>1700（30/11/36.67%）；`C`=在>1700 的局里出现≥2 次（35/13/37.14%）。
- 独立第三方实现（`crosscheck_1700_independent_report.md`）：8 种「快照×阈值」组合全部逐位落在 35/16/45.71%；与正式报告对拍 **709 个 tag 共同、rating 不一致 0、opp_name 不一致 0**。

> ⚠️ **口径陷阱（历史踩过两次）**：① 全档跨日率分母 530，1700+ 复现率分母 35，不可直接比；② 曾把「1700+」误读成「1700 个不同对手」，据此推出「需 13 天历史」的错误结论，已在产物中删除更正。**任何跨源对比先统一分母与口径。**

**本组数字的原始出处**（比 memory 转述更权威）：
`E:\venvs\metamon-audit\opponent_modeling\` 下 `crosscheck_1700_report.md`（正式）、`opponent_repeat_report.md`、`USER_MODELING_REPORT_20261021.md`、`crosscheck_1700_independent_report.md`（独立验证）、`opponent_repeat_stats.json`、`crosscheck_1700_stats.json`。**这些不在 `E:\Tools\foul-play\reports\` 里**，两个来源不要混为一谈。

**决定性实证**：`chusei44` 5 局全同队，队伍 100% 拿得到、每只招式能抓到 1–4 个、关键道具能抓到。

## 5. ⭐ 能力上限（**Codex 改代码前必读**）【实证】

配置覆盖率：

| 字段 | 覆盖率 | 说明 |
|---|---|---|
| species | **100%** | `|poke|` 可得 |
| gender | 64.8% | |
| moves | **21.5%（仅曝光）** | 只有实际用过的招式 |
| item | 3.5% | 仅 `|enditem|` 等披露时 |
| **ability / teraType / 完整配招 / _stats** | **0%** | **日志里根本不存在** |

**根因（PS 协议硬约束）**：`|request|` 帧**只含 bot 自己 side，从不含对手**（源 A 抽 40 万行实测 `side.name` 全是己方账号）。对手侧只能拿 `|poke|` / `|move|` / `|enditem|` / `|detailschange|` 这几类已披露帧。另外 `request.stats` 是合成后的实际能力值，**不能反推 EV**。

⇒ **不要试图给对手补完整配招 / ability / tera / EV**，那不是"还没做"，是**数据源不存在**。任何声称能做到的方案都要先回答"信息从哪来"。

## 6. PS 协议事实（解析时照抄，别自己造样例）【实证】

- `|move|` = `|move|<使用者>|<招式名>|<目标>|<附加>` → **parts[2] 是招式名**（源 D 1215 条实测）。
  ⚠️ 曾有版本记成「parts[2] 是目标」，属错误，已更正。
- `|poke|` 行尾的 `|` 是空字段分隔符，物种名须拆成 species + form + gender。
- battle tag 两段 / 三段两种：`battle-[a-z0-9]+-\d+(?:-[a-z0-9]+)?`。
- **观测级统计必须按 battle_tag 去重**，否则局数虚高约 7 倍。
- `|player|` 行同一 tag 会因重连重复（457 个 tag 恰 2 行、138 个 3 行、最多 ×19）→ 直接数行会把 1 局算成多局。
- avatar 字段可能是字符串（如 `cynthia`），解析 rating 的正则别写 `[0-9]*`。

> **方法论事故（必须记住）**：曾凭记忆手写一条「`|move|` 样例」去否定他人正确实现，被 1215 条实测反证后撤回。⇒ 规则：任何「格式是 X」的断言必须贴**从文件直接抓的原始行（带文件名 + 行号）**，不得手写构造。

## 7. Part A：`fp/opponent_model.py` 注入【实证，2026-10-22 落地】

**调用点**：`handle_team_preview` 在 **deepcopy 之前**对 **live battle** 调 `inject_known_moves(battle)`。

**行为**：
1. 按 `opponent.account_name` 查 **curated(`elite_profiles.jsonl`) + runtime 合并库**（懒加载合并、招式取并集、名字归一化）。
2. 只把协议已披露的 `moves_seen`（**真子集，不臆造**）plant 到 reserve pokemon。
3. **item / tera 仅当库中只有唯一非空值时才设**。
4. 全程 `try/except` containment，异常不影响对局。

**为什么选 live battle 而不是副本**：注入的招式随 deepcopy 进入首发搜索副本，同时留在 live battle 被后续每回合 MCTS 继承 → **全对局生效**。依据 `standard_battles.py:117-123`：当 `pkmn.moves >= 4` 时尊重预置、不走 Smogon fallback。

**配套 4 处改动**：
| 文件 | 改动 |
|---|---|
| `config.py` | 新增 6 个 flag |
| `state.py` | `Battler` 新增 `rating` 字段（默认 None） |
| `base.py` | 解析 `|player|` 的 `split_msg[5]` 为 rating，经 `_player_line_rating` / `_assign_opponent_rating` 两个纯函数；加注入 hook |
| `run_battle.py` | battle 结束时调 `record_battle` hook |

`tests/test_opponent_model.py` **11 个用例**（`^def test_` 实测行号 65,71,77,97,108,134,146,160,194,211,234；含真库注入、runtime 写隔离、OFF/阈值守卫）；**904 passed**（893 + 11），无回归。
> 【更正】早期文档记「10 个用例」，以源码实测的 **11** 为准。

## 8. Part B：`fp/opponent_recorder.py` 自生长【实证】

- `record_battle(battle, winner)` 在每局结束时调用。
- **仅当 rating ≥ 1700（默认阈值）才缓冲** → 只记录高分段对手。
- **每 5 局 + `atexit`** 刷盘到独立的 `runtime_opponent_log.jsonl`。
- **绝不碰 curated 库**；runtime 日志可安全重跑 / 删除。
- 只记 `team_signature` + 实战观测招式 / 道具 / tera；live battle 的 `opponent pokemon.moves` 只由 `|move|` 帧增长，不含 Smogon 臆测。

## 9. ⚠️ 数据完整性 bug 与修复（**必读，防复发**）【实证】

**现象**：本地 PS 确定性验证中发现，recorder 记录 Glimmora `moves_seen=["powergem","sludgewave"]`，但 `sludgewave` 是**注入进去**的、对手根本不会（固定队只会 mortalspin / stealthrock / earthpower / powergem）。

**根因**：Part A 把假设招式 `add_move` 进 `pkmn.moves`，而 Part B recorder 读的是**同一个字段** → 假设被当成实战观测写进自生长库 → **污染**。

**修复（三处）**：
1. 注入时给假设招式打标 `pkmn._injected_move_names`。
2. `recorder._observed_moves` 排除被注入标记者。
3. 顺手修日志 `%s/%d` 不替换的老毛病（改为预格式化）。
4. 新增回归测试 `test_record_excludes_injected_moves`；**905 passed**。

**【待核】已知限制**：item / tera 注入**同根因、低频**，本次只修了招式，留待后续用同样打标法剔除。Codex 若改 item/tera 注入，必须同步处理 recorder 侧。

## 10. 铁律（逐条，改代码前复核）

1. 【铁律】两个开关 `--opponent-model-enabled` / `--opponent-record-enabled` **默认 OFF，连天梯前绝不打开**；需要用户明确授权才可开。
2. 【绝不】`runtime_opponent_log.jsonl` **绝不碰 curated 库**；两者物理独立，runtime 可安全重跑 / 删除。
3. 【能力边界】只 plant 已披露真子集，绝不臆造 ability / tera / 完整配招 / EV。
4. 【口径】任何跨源复现率对比先统一分母；观测统计按 battle_tag 去重。
5. 【纪律】断言格式必须贴原始行 + 文件名 + 行号。
