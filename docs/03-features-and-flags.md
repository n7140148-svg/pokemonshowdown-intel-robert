# 03 · 已实现功能与完整命令行开关表

> 来源：`E:\Tools\foul-play\fp\config.py` 直接读取（`cat -n` 行号），`configure()` 的 `add_argument` 共 **45 个**，位于 `:224-501`；类字段默认 `:119-220`；赋值 `:504-553`；校验 `:584-619`。
> ⚠️ **默认值以 `add_argument` 为准**：`configure()` 会把类字段默认值全部覆盖，所以 `:119-220` 的注释默认值**不一定等于运行时默认值**（第 4 节有实例）。

---

## 1. ⛔ 默认 OFF / 需显式开启的开关（**开工前必读**）

| 开关 | 默认 | 关闭原因 | 打开条件 |
|---|---|---|---|
| `--concurrent-battles` | `False` | 单账号多局并发曾导致 **490 连败事故**（rating 1655→1003）；help 原文 `must NEVER be enabled on the live ladder without explicit sign-off` | 用户明确授权；且 `--bot-mode search_ladder`（`validate_config` 硬性校验） |
| `--opponent-model-enabled` | `False` | Part A 注入是研究旋钮；help 原文 `research-only, never on the live ladder` | 用户明确授权 |
| `--opponent-record-enabled` | `False` | Part B 自生长是研究旋钮 | 用户明确授权 |
| `--human-pacing` | `False` | 拟人思考延迟，help 原文 `research only, never on the live ladder`；且 `(search_time_ms + human_pacing_max_ms)` 必须低于对局计时器，否则超时送掉 | 研究用途，**绝不用于真天梯**（旧号标记撤不掉 + 踩 PS 10-09 新政红线） |
| `--team-repeat-guard` | `False` | 禁止连续两局抽同一队（减少自动化痕迹），属行为研究旋钮 | 同上，默认 OFF |
| `--deep-hypotheses` | `False` | A/B 实验臂：少假设、双倍时间 | 仅 A/B 实验 |
| `--final-choice argmax` | 默认 `sample` | A/B 实验臂 B1，未过保留规则（见 `docs/06`） | 保持 `sample` |
| `--search-threads 2` | 默认 `1` | A/B 实验臂 B5，未过保留规则；`threads>1` 走 `mcts_threaded.rs` | 保持 `1` |
| `--team-pool-source editor` | 默认 **`file`** | 从队伍编辑器自动取池；默认走手动 `--team-list` 文件（向后兼容） | 需要显式指定 `editor` |
| `--log-to-file` | `False` | 写 DEBUG 日志到 `logs/`；与并发组合曾触发 490 事故的相关路径 | 按需开启，注意日志轮转 |

> **注意措辞**：早期记忆写「多队伍随机选队默认 OFF」，精确说法是 **`--team-pool-source` 默认 `file`，`editor` 池需显式开启**；`--team-pool-mode` 默认 `battle`（每局重选）。

---

## 2. 完整 45 个 `add_argument`（按功能分组，行号 = `fp/config.py`）

### 2.1 连接与身份（7 个）
| 行 | 参数 | 默认 / 类型 | 说明 |
|---|---|---|---|
| 224 | `--websocket-uri` | **required** | PS websocket URI；见第 3 节 `get_websocket` 别名与规范化 |
| 229 | `--ps-username` | **required** | 账号 |
| 230 | `--ps-password` | `None` | **绝不写进仓库**；`search_ladder` 模式缺它会直接 `ValueError`（`:596-601`） |
| 231 | `--ps-avatar` | `None` | |
| 233 | `--bot-mode` | **required** | `challenge_user` / `accept_challenge` / `search_ladder`（`BotModes` `:113-116`） |
| 236 | `--user-to-challenge` | `None` | `challenge_user` 时必填（`:585-588`） |
| 436 | `--room-name` | `None` | `accept_challenge` 时等待的房间；**与对战隐身无关** |

### 2.2 对战格式与队伍（9 个）
| 行 | 参数 | 默认 | 说明 |
|---|---|---|---|
| 241 | `--pokemon-format` | **required** | 如 `gen9ou` / `gen9randombattle` |
| 244 | `--smogon-stats-format` | `None` | 覆盖用于推断未知配置的 Smogon 统计格式；不设则用 `--pokemon-format` |
| 302 | `--team-name` | `None` | 文件名**或目录名**（相对 `./fp/teams/teams/`）；**目录 = 每局随机抽一队**；不设则用 format 值（`:524`） |
| 309 | `--team-list` | `None` | 队伍名清单文本，按顺序选；**优先于 `--team-name`** |
| 314 | `--team-select-mode` | `cycle` | `cycle`（顺序）/ `random`（随机），仅 `--team-list` 生效 |
| 321 | `--team-pool-mode` | `battle` | `battle` 每局重选（anti-meta）/ `session` 整场锁定一队（保持 A/B 两臂可比） |
| 329 | `--team-pool-source` | `file` | `file`（默认，`--team-list`）/`editor`（队伍编辑器全部队伍） |
| 340 | `--team-pool-scope` | `None` | `editor` 源下限定路径前缀，如 `gen9/ou_2026`；**Unset = 所有编辑器队伍** |
| 347 | `--team-repeat-guard` | `False` | 随机选队禁止连续两局同队 |

### 2.3 搜索 / MCTS（7 个）
| 行 | 参数 | 默认 | 说明 |
|---|---|---|---|
| 249 | `--search-time-ms` | `100` | **每个 state 的搜索毫秒数**（真实运行常配 3000） |
| 255 | `--search-parallelism` | `1` | 并行搜索的 state 数 |
| 261 | `--team-preview-search-parallelism` | `None` | 不设则回退 `--search-parallelism`（`:516-518`） |
| 267 | `--team-preview-search-time-ms` | `None` | 不设则回退 `--search-time-ms`（`:513-515`） |
| 290 | `--search-threads` | `1` | 每 state 线程数（poke_engine `monte_carlo_tree_search`） |
| 296 | `--run-count` | `1` | 打几局 |
| 494 | `--deep-hypotheses` | `False` | A/B 臂：`parallelism*2` 假设 @ `search_time_ms` → `parallelism` 假设 @ `2*search_time_ms`，墙钟不变 |

### 2.4 首发决策 / L2（2 个）
| 行 | 参数 | 默认 | 说明 |
|---|---|---|---|
| 273 | `--lead-policy` | **`auto`** | `auto` = Metamon 出建议就用、否则回退 MCTS，**两者都记录**；`metamon` = 总用模型并跳过 MCTS team-preview 搜索；`mcts` = 从不用模型（`LeadPolicy` `:101-110`） |
| 283 | `--lead-model-path` | `None` | `best_model.pt`；查找顺序：`fp/lead_model/assets/` → 上游 metamon checkout → 环境变量 `FOULPLAY_LEAD_MODEL` |

> ⚠️ `--lead-policy` 默认 `auto` ⇒ **首发实际由 Metamon advisor 主导**，MCTS 首发只旁路记录。见 `docs/05`。

### 2.5 终选策略 / A/B 旋钮（2 个）
| 行 | 参数 | 默认 | 说明 |
|---|---|---|---|
| 477 | `--final-choice` | `sample` | `sample`（按聚合 visit share 加权采样，**shipped**）/ `argmax`（确定性取最高权重，A/B 臂） |
| 486 | `--repeat-penalty` | `0.5` | 上回合自己用过的招的乘数；`0` 关闭；`:603-607` 校验必须 ≥0 |

**类字段注释里的实测背景**（`config.py:185-205`）：era-B 日志审计测得 **85.6%** 决策落在聚合 top-1（即 14.4% 没有）；`repeat_penalty` 触发 **243 次（22.8%）但对最终选择 0 次改变** ⇒ 实验可顺带检验它是不是死重。

### 2.6 拟人节奏（5 个，默认 OFF 组）
| 行 | 参数 | 默认 |
|---|---|---|
| 354 | `--human-pacing` | `False` |
| 362 | `--human-pacing-min-ms` | `800` |
| 367 | `--human-pacing-max-ms` | `4000` |
| 375 | `--human-pacing-long-pause-prob` | `0.05` |
| 380 | `--human-pacing-long-pause-ms` | `9000` |

### 2.7 ⭐ 对手建模 Part A / B（6 个）
| 行 | 参数 | 默认 | 说明 |
|---|---|---|---|
| 388 | `--opponent-model-enabled` | `False` | Part A 注入开关 |
| 396 | `--opponent-model-dir` | `E:\venvs\metamon-audit\opponent_modeling\elite_opponents` | curated 库目录（`elite_profiles.jsonl`） |
| 402 | `--opponent-record-enabled` | `False` | Part B 自生长开关 |
| 410 | `--opponent-record-threshold` | `1700` | 只记 ≥ 该 rating 的对手 |
| 416 | `--opponent-record-flush` | `5` | 每 N 局刷盘 |
| 422 | `--opponent-runtime-log` | `fp/runtime_opponent_log.jsonl`（相对 `fp/`） | **独立于 curated，绝不写 curated** |

### 2.8 并发（4 个）
| 行 | 参数 | 默认 | 说明 |
|---|---|---|---|
| 447 | `--concurrent-battles` | `False` | 见第 1 节 |
| 455 | `--max-concurrent-battles` | `2` | 每账号在飞局数上限 |
| 461 | `--no-concurrent-core-cap` | `store_false` → 默认 `True` | 关闭「MCTS 池按物理核数封顶」 |
| 469 | `--concurrent-core-budget` | `0` | 假定可用物理核数，`0` = 自动检测；多进程共享机器时显式设 |

**核心封顶的实测依据**（`config.py:170-178`）：本机 **6 物理核**，2 局并发 × 2 线程时每次搜索 MCTS 迭代数**下降 22.3%** ⇒ 封顶保证每次搜索仍能独占整核，并发收益来自「用一局的等待时间重叠另一局」，而不是把搜索叠在一起跑。

### 2.9 日志 / 回放（3 个）
| 行 | 参数 | 默认 | 说明 |
|---|---|---|---|
| 430 | `--save-replay` | `never` | `always` / `never` / `on_loss` / `on_win`（`SaveReplay` `:94-98`） |
| 440 | `--log-level` | `DEBUG` | |
| 442 | `--log-to-file` | `False`（`store_true` 隐式） | 写 `logs/`；轮转上限 `MAX_LOG_FILES=200` / `MAX_LOG_BYTES=2GB`（`config.py:18-22`） |

**无 CLI flag 的配置**：`challenge_attempts` 只由 `:523` `getattr(args, "challenge_attempts", 3600)` 提供默认 **3600**；`config.py` 全文 623 行**没有** `hide` / `privacy` / `spectator` / `hidenext` 相关 flag。

---

## 3. `get_websocket()` 别名与 URL 规范化（`config.py:557-578`）【实证·已修】

| 输入 | 输出 |
|---|---|
| `ps` / `pokemonshowdown` | `wss://sim3.psim.us/showdown/websocket` |
| `local` / `localhost` | `ws://localhost:8000/showdown/websocket` |
| `https://...` | `wss://...`（`:574-575`） |
| `http://...` | `ws://...`（`:576-577`） |
| 其它 | 原样返回 |

**【更正·重要】**：早期记忆记有「`sim3.psim.us` 返回 HTTP 307 跳 `play.pokemonshowdown.com`，`websockets` 库不认 https scheme → `InvalidURI: scheme isn't ws or wss` 崩溃（`fp/websocket_client.py:80`）」。当前 `get_websocket` **已实现 `http(s) → ws(s)` 规范化**（注释 `:564-578` 明确写了「直接粘贴浏览器 URL 会在连接时炸，导致根本进不了匹配」），该崩溃路径已被堵住。⇒ 若再遇 `InvalidURI`，先检查传入的 URI 是否被其它代码路径绕过 `get_websocket`。

同时 `validate_config` 增加了**游客号防呆**（`:590-601`）：`search_ladder` 缺 `--ps-password` 直接 `ValueError`，注释说明「游客能连能登录，但服务器静默拒绝匹配 ranked/search 格式，bot 会永远等一个永不开放的房间」——这正是历史上「连上真天梯但匹配不到人」的现象之一。

---

## 4. ⚠️ 类字段默认值 vs 运行时默认值不一致【实证·待核】

`config.py:142` 类字段写 `team_pool_scope: str = "gen9"`，注释说「避免抽进 `gl_05_26_gen9ou` 那批自动生成队」；但 `add_argument`（`:340`）的 default 是 **`None`**，且 `:529` 用 `args.team_pool_scope` 无条件覆盖类字段 ⇒ **运行时实际是 `None`（= 所有编辑器队伍）**。

⇒ 若启用 `--team-pool-source editor` 且**不显式传 `--team-pool-scope`**，可能会把 `gl_05_26_gen9ou`（6000+ 队，其中大量带泄露名 `Toxic Chain`）抽进池子，导致服务器拒队（见 `docs/11` 第 2 节 A）。

**Codex 处理建议**：启用 editor 池时**显式传** `--team-pool-scope gen9/ou_2026`；或修 `add_argument` 默认值使其与类字段注释一致（需补回归测试）。这是【待核】项，不要假设已修。

---

## 5. 已实现功能逐个（状态 + 证据位置）

| 功能 | 状态 | 证据 |
|---|---|---|
| **对战隐身 `/hidenext`** | ✅ 已落地，**无 CLI 开关、无条件发送** | `fp/websocket_client.py:506` `hide_next_battle()` → `:523` `send_message("", ["/hidenext"])`；调用点 `:525-527`(`challenge_user`)、`:554-556`(`search_for_match`)。真机 **5/5 PASS**。注释 `:509-521` 说明为何不能用 `/modjoin +`（中途锁门会重命名房间，战斗循环饿死） |
| **单账号多局并发** | ✅ 已落地，默认 OFF | `fp/main.py:194` `_run_concurrent_battles`；`fp/websocket_client.py:189` `_dispatch_loop` + `:303` `discard_room` + `:311` `_guarded_get`；本地 PS 端到端 5/5 → 6/6；见 `docs/09` 事故 3/4/5/8 |
| **dispatcher 断线兜底** | ✅ 已落地（947 passed） | `DispatcherDeadError`(`websocket_client.py:21`) + `DISPATCHER_DEAD`(`:37`) + `_wake_all_consumers`(`:219`)；`fp/main.py` 捕获后**不计败绩**；见 `docs/09` 事故 8 |
| **多队伍随机选队 / editor 池** | ✅ 已落地，默认 `file` | `--team-pool-*` 组；`fp/teams/load_team.py`（已修改 M）；`tests/test_team_pool.py` |
| **L2 Metamon 首发模型** | ✅ 已落地，`--lead-policy auto` 默认生效 | `fp/lead_advisor.py` + `fp/lead_model/`；见 `docs/05` |
| **对手建模 Part A 注入** | ✅ 已落地，默认 OFF | `fp/opponent_model.py:135` `inject_known_moves`；见 `docs/04` |
| **对手建模 Part B 自生长** | ✅ 已落地，默认 OFF | `fp/opponent_recorder.py:63` `record_battle`；污染已修（`_observed_moves`:47 剔除注入标记）；见 `docs/04` |
| **per-battle 日志路由** | ✅ 已落地 | `fp/log_routing.py:11` `BattleTagFilter`；`fp/modes/base.py:241/:358`；见 `docs/09` 事故 4 |
| **异常路径 try/finally 兜底** | ✅ 已落地 | `fp/run_battle.py:85-96`；见 `docs/09` 事故 5 |
| **弹窗守卫 / 搜索拒绝 fail loud** | ✅ 已落地（HEAD `5e9d8a9`） | `fp/websocket_client.py:17` `LadderSearchRejectedError`、`:40` `_FATAL_POPUP_MARKERS`、`:49` `fatal_popup_text`。注意 `already searching` **不是**拒绝（见 `docs/08`） |
| **拟人节奏 / 队伍去重 / A/B 旋钮** | ✅ 代码已落地，**默认 OFF / 默认 shipped 值** | 第 2.5–2.6 节；**绝不用于真天梯** |

---

## 6. 启动方式（**示意，不含任何凭证**）

```bash
# 本地服端到端验证（推荐，不连真天梯）
E:/Tools/foul-play/.venv/Scripts/python.exe run.py \
  --websocket-uri local \
  --ps-username <YOUR_BOT_ACCOUNT> \
  --ps-password <PROVIDED_AT_RUNTIME> \
  --bot-mode search_ladder \
  --pokemon-format gen9randombattle \
  --run-count 5 --log-level DEBUG

# 常用真机参数（历史上 run_3_battle.bat 的拼法）
#   --search-parallelism 5 --search-threads 1 --search-time-ms 3000
#   --run-count <N> --save-replay always --log-to-file
```

> ⚠️ 上面**只是命令行形状**，凭证一律运行时提供，**绝不写进文件**。真天梯运行属外部风险操作，执行前须与用户确认。

**启动器**：`E:\Tools\run_3_battle.bat`（交互式问用户名/密码/模式/局数/队伍 + 并发 `y/n`）；`tools\h2h_battle.sh` 顶部 `PARALLEL=off` / `MAX_PARALLEL_BATTLES=2`，`on` 时才追加 `--concurrent-battles --max-concurrent-battles N`。**改启动器开关必须做纯构造验证（echo 出命令行比对），不要真跑验证**（`docs/09` 事故 2）。
