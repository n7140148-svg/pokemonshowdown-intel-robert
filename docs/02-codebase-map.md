# 02 · 代码地图（精确行号）

> 来源：对 `E:\Tools\foul-play` 的只读取证（Read `cat -n` + 单文件 Grep 二次核对），**行号以当前工作树为准**。
> ⚠️ 取证时仓库工作树有大量未提交改动（见第 7 节），**行号可能随后续提交漂移**；改代码前请用 `Read` / `Grep` 复核。

---

## 1. 入口与调用链

```
run.py:12  asyncio.run(run_foul_play())
   └─ fp/main.py:47  run_foul_play()
        ├─ :48-50   FoulPlayConfig.configure() → init_logging() → apply_mods()
        ├─ :55-59   PSWebsocketClient.create(...) + login()
        ├─ :98      mode = battle_mode(format_spec.battle_type)
        ├─ :100-106 ── 并发分支 ──> fp/main.py:194  _run_concurrent_battles()
        └─ :114-181 ── 串行分支（默认）──> fp/main.py:160  pokemon_battle(...)

fp/run_battle.py:34  pokemon_battle(ps_websocket_client, battle_type, team_dict, msg_getter=None)
   ├─ :40-44   _src = msg_getter or receive_message
   ├─ :45-46   while True: msg = await _src()          ← 断线死锁点（见 docs/09 事故 8）
   ├─ :47      battle_is_finished(...)
   ├─ :57      record_battle(battle, winner)            ← Part B 自生长 hook
   ├─ :59-70   save_replay 分支
   ├─ :71-79   leave_battle / 清理 / return winner
   ├─ :81-84   async_update_battle → async_pick_move → send_message
   └─ :85-96   finally: pop_battle_log_handler(...) + discard_room(battle_tag)
```

`run.py` 全文 21 行；`finally` 里 `run.py:20` 调 `shutdown_executor()`。

### 并发调度细节
- `fp/main.py:194` `_run_concurrent_battles(...)`（docstring `:195-203`）。
- `:204` `ps_websocket_client.start_dispatcher()`。
- `:249` `while len(active) < max_concurrent and launched < run_count`。
- `:277-283` `get_message_for(battle_tag)` → `asyncio.create_task(pokemon_battle(..., msg_getter=msg_getter))`。
- `:263-275` / `:297-310` 捕获 `DispatcherDeadError`（**死亡不计败绩**）。
- `:334` `stop_dispatcher()`；`:183-190` `print_battle_report(...)` + `close()`。

## 2. 模式注册表（`fp/modes/__init__.py:8-17`）

| BattleType | 实现 | 文件 | requires_team |
|---|---|---|---|
| RANDOM_BATTLE | `RandomBattleMode` | `fp/modes/random_battle.py:25` | False (`:27`) |
| STANDARD_BATTLE | `StandardBattleMode` | `fp/modes/standard_battle.py:26` | True (`:28`) |
| BATTLE_FACTORY | `BattleFactoryMode` | `fp/modes/battle_factory.py:24` | False (`:26`) |
| BSS | `BSSMode` | `fp/modes/bss.py:17` | 继承 Standard |

`battle_mode(battle_type)` 在 `fp/modes/__init__.py:16-17`。

**基类 `BattleMode`**（`fp/modes/base.py:126`）：`start_battle`(135)、`search_params`(140)、`prepare_battles`(143)、`dataset_possibilities`(210)、`assume_spread_for_speed_check`(213)、`sample_mega_evolution`(161)、`start_battle_common`(227)、`handle_team_preview`(364 方法)。
模块级函数：`format_decision`(368)、`async_pick_move`(411)、`apply_human_pacing`(438)、`handle_team_preview`(464)、`get_battle_tag_and_opponent`(550)、`get_first_request_json`(588)。

各模式要点：
- `standard_battle.py`：`_smogon_and_team_targets`(34)、`start_battle`(58)、`initialize_team_preview_datasets`(144)、`_opponent_snapshot_write_lock`(159)、`search_params`(182)、`prepare_battles`(210) → `standard_battles.prepare_battles`；**team preview 唯一调用点 `fp/modes/standard_battle.py:140`**。
- `random_battle.py`：`prepare_battles`(95) → `prepare_random_battles`；Zoroark 推断 `check_zoroark_from_move`(101) / `check_zoroark_from_immune`(154)；**无 team preview**。
- `battle_factory.py`：`initialize_team_preview_datasets`(33)、`prepare_battles`(47)、`dataset_possibilities`(73)。
- `bss.py`：`opponent_possible_mega_evolutions`(20)、`handle_team_preview`(45，**覆写**，bring6pick3)、`prepare_battles`(93)、`search_params`(98)。

## 3. 决策链

### 3.1 MCTS 入口
- `fp/modes/base.py:18` `from fp.search.main import find_best_move`。
- `fp/modes/base.py:411` `async_pick_move(battle)` → `:425-428` 线程池 `loop.run_in_executor(pool, search_context.run, find_best_move, battle_copy)`。
- **`fp/search/main.py:334` `find_best_move(battle)` = MCTS 决策入口**：
  - `:355-356` `mode.search_params(battle)` / `mode.prepare_battles(battle, num_battles)`
  - `:362-374` 提交共享进程池
  - `:382` `select_move_from_mcts_results(mcts_results, last_selected)`
- `fp/search/main.py:14` `from poke_engine import State as PokeEngineState, monte_carlo_tree_search, MctsResult`；真正调 MCTS 在 `get_result_from_mcts`：
```python
323→def get_result_from_mcts(state, search_time_ms, index, threads):
327→    poke_engine_state = PokeEngineState.from_string(state)
329→    res = monte_carlo_tree_search(poke_engine_state, search_time_ms, threads=threads)
```
- 终选 `fp/search/main.py:262` `select_move_from_mcts_results`：`POLICY_KEEP_FRACTION=0.75`(`:27`)、`repeat_penalty`(`:295`)、`final_choice=="argmax"` 分支(`:308-315`)。
- 进程池：`_get_executor`(177)、`concurrent_pool_size`(245)、`_cap_pool_by_cores`(209)、`shutdown_executor`(196)。

> 树搜索本体是 **poke_engine 0.0.48**（Rust/pyO3）：UCB1 探索系数硬编码 2.0、rollout 一步静态评估、`threads>1` 走 `mcts_threaded.rs`。详见 `docs/06`。

### 3.2 team preview（首发）三处定义 + 一处调用
| 位置 | 角色 |
|---|---|
| `fp/modes/base.py:364-365` | 基类方法，委托给模块函数 |
| **`fp/modes/base.py:464`** | **标准实现** `async def handle_team_preview(battle, ps_websocket_client)` |
| `fp/modes/bss.py:45` | BSS 覆写（bring6pick3） |
| `fp/modes/standard_battle.py:140` | **唯一调用点** |

标准实现内部：`inject_known_moves`（`:470`，Part A）、`get_advisor().recommend(battle)`（`:489-490`，L2 Metamon）、MCTS 对照（`:507-509`）。

> ⚠️ `--lead-policy` 默认 `auto` ⇒ **首发实际由 Metamon advisor 主导**，MCTS 首发只旁路记录。改首发逻辑别只看 MCTS 代码，见 `docs/05`。

### 3.3 招式 fallback（`fp/search/standard_battles.py`）
`sample_pokemon_moveset_with_known_pkmn_set`（`:117`）：
```python
120→    pkmn_known_moves = [m.name for m in pkmn.moves]
121→    num_known_moves = len(pkmn_known_moves)
122→    if num_known_moves >= 4:
123→        return pkmn_known_moves
```
补全后二次短路 `:154-156`；Smogon 补全循环以 4 为退出界 `:168-170`。

⇒ **这是 Part A 注入能全对局生效的依据**：注入后 `pkmn.moves >= 4` 就不再走 Smogon 臆测。被 `fp/opponent_model.py:22-26` 显式引用。

## 4. 已实现模块逐个（职责 + 关键函数）

| 文件 | 行数 | 关键内容 |
|---|---|---|
| `fp/opponent_model.py` | 210 | docstring `:3-27`（默认 OFF / 只注入已披露 / live battle deepcopy 前 / 全 try/except）；`_read_jsonl`(50)、`_merge_profiles`(69)、`_load_index`(111 合并 curated+runtime)、`get_opponent_profile`(123)、`inject_known_moves`(**135**)、`clear_cache`(207)；常量 `DEFAULT_ELITE_DIR`(40)、`DEFAULT_RUNTIME_LOG`(44)。开关守卫 `:140-141` |
| `fp/opponent_recorder.py` | 176 | docstring `:3-18`；`_observed_moves`(**47**，剔除 `_injected_move_names`)、`record_battle`(**63**)、`flush`(161)、`_ensure_atexit`(40)。守卫 `:65-71`（flag + rating≥阈值）；刷盘 `:154-156` |
| `fp/lead_advisor.py` | 274 | docstring `:1-13`（fail-safe 返回 None 回退 MCTS、`_tokens_known` 拒绝 OOV 避免 `nn.Embedding(-1)`）；`LeadRecommendation`(62)、`MetamonLeadAdvisor`(97)：`available`(116)、`_ensure_loaded`(121)、`recommend`(**157**)、`_recommend`(176)、`_tokens_known`(229)；`get_advisor`(256)、`reset_advisor`(270)；`_resolve_model_path`(45) 候选 `_IN_REPO_MODEL`(32-34) / `_METAMON_REPO_MODEL`(37-41) / env `FOULPLAY_LEAD_MODEL`(42) |
| `fp/lead_model/names.py` | 75 | `FORMAT_ALIASES`(13)、`format_for_agent`(19)、`clean_no_numbers`(24)、`clean_name`(29)、`pokemon_name`(34)、`move_name`(53)、`consistent_pokemon_order`(57)、`consistent_move_order`(68) |
| `fp/lead_model/tokenizer.py` | 96 | `UNKNOWN_TOKEN=-1`(12)、`PokemonTokenizer`(15，`__getitem__`(39) 未知返回 -1)、`PREMADE_TOKEN_LISTS`(73)、`:83` `"DefaultObservationSpace-v1": "DefaultObservationSpace-v1.json"`、`get_tokenizer`(87) |
| `fp/lead_model/model.py` | 327 | `CrossAttentionBlock`(25)、`SelfAttentionBlock`(52)、`TeamPreviewModel`(77)（`classifier = nn.Linear(latent_tokens*d_model, 6)` :120）、`forward`(122)、`predict_lead`(**209**)、`load_from_checkpoint`(295)；`:9-10` 「12 inputs (6 ours + 6 opponent) → 1 output (which of our 6 to lead)」 |
| `fp/lead_model/DefaultObservationSpace-v1.json` | 单行 | `:1` token→int 词表，起始 `{"<anychoice>":0,"<blank>":1,"<conditions>":2,"<forcedswitch>":3,...}`；**2541 token**，缺 Tapu 全系 / Ferrothorn 等（见 `docs/05` 第 4 节） |
| `fp/log_routing.py` | 38 | `CURRENT_BATTLE_TAG`(8)、`BattleTagFilter`(11，`filter`(23) 比对 `CURRENT_BATTLE_TAG.get(None) == self.battle_tag`)、`battle_log_scope`(27)。使用点 `fp/modes/base.py:17/:241/:358`、`fp/websocket_client.py:12/:289` |
| `fp/websocket_client.py` | 580 | 见下节 |
| `fp/battle/state.py` | — | `class Battler`(`:215`)、`:229` `self.rating = None  # opponent's ladder rating, parsed from |player| (split_msg[5])` |

### `fp/websocket_client.py`（dispatcher / per-room 队列 / 隐身）
- 异常与哨兵：`LadderSearchRejectedError`(17)、`DispatcherDeadError`(**21**)、`DISPATCHER_DEAD = object()`(37)、`_FATAL_POPUP_MARKERS`(40)、`fatal_popup_text`(49)。
- `PSWebsocketClient`(102)；`__init__`(`:111-124`)：`_rooms={}`(120)、`_global_queue`(121)、`_new_room_queue`(122)、`_dispatch_task`(123)、`_dispatcher_dead`(124)。
- dispatcher：`start_dispatcher`(166)、`stop_dispatcher`(179)、`_dispatch_loop`(**189**)、`_wake_all_consumers`(219)、`_route`(234)、`discard_room`(**303**)。
```python
189→    async def _dispatch_loop(self):
190→        socket_died = False
191→        while True:
192→            try:
193→                message = await self.websocket.recv()
194→            except Exception as e:
```
```python
215→        if socket_died:
216→            self._dispatcher_dead = True
217→            await self._wake_all_consumers()
```
- `_route` 建 per-room 队列（`:294-301`）；`discard_room` 核心 `:309` `self._rooms.pop(roomid, None)`，被 `fp/run_battle.py:96` 与 `:575`(`leave_battle`) 调用。
- `_guarded_get`(`:311-328`) 把哨兵转 `DispatcherDeadError`：
```python
320→        async def _get():
321→            item = await queue.get()
322→            if item is DISPATCHER_DEAD:
323→                raise DispatcherDeadError(
324→                    "websocket dispatcher stopped; no more frames will arrive"
325→                )
```
- 其余 getter：`get_message_for`(330)、`get_new_room_message`(343)、`await_battle_room`(352)、`get_global_message`(373)、`receive_message`(147)。
- **对战隐身**：`hide_next_battle`(**506**) → `:523` `await self.send_message("", ["/hidenext"])`；调用点 `challenge_user`(525→527)、`search_for_match`(554→556)。注释 `:509-521` 说明「建房前 `/hidenext` 隐藏；中途 `/modjoin +` 会重命名房间导致战斗循环饿死」，见 `reports/modjoin_spectator_lock_verification.md`。
  > ⚠️ **无 CLI 开关**：`fp/config.py` 全文 623 行无 `hide`/`privacy`/`spectator`/`hidenext` flag；`--room-name`(:436-439) 与之无关。隐身是**无条件发送**的。

### `fp/modes/base.py` 的 `|player|` 评分解析
- `_player_line_rating`(`:22-30`)：`:28-29` 取 `split_msg[5].strip()`。
- `_assign_opponent_rating`(`:33-43`)：`:38-41` `int()` 转换、异常返回 None。
- 解析循环 `:281-303`（`:301` 调 `_player_line_rating`、`:302-303` 存 `player_ratings[side_id]`）；协议注释 `:266-270` 与 `:298-300`：`|player|<side>|<name>|<avatar>|<rating>`。
- 赋值 `:331-335` `battle.opponent.rating = _assign_opponent_rating(...)`。

## 5. 配置 `fp/config.py`
- `_FoulPlayConfig` 类(`:119`)，类字段默认 `:120-220`，单例 `FoulPlayConfig = _FoulPlayConfig()`(`:622`)，`configure()`(`:222`)，`:503` `parse_args()`。
- 枚举：`SaveReplay`(94-98)、`LeadPolicy`(101-110)、`BotModes`(113-116)。
- **`add_argument` 共 45 个**，位于 `fp/config.py:224-501` → 完整表见 **`docs/03`**。
- `validate_config()`(`:584-619`)：`search_ladder` 必须有 password(`:596-601`)、`repeat_penalty>=0`(`:603-607`)、**`concurrent_battles` 仅支持 `search_ladder`**(`:613-619`)。
- `get_websocket()`(`:557-578`)。
- `challenge_attempts` **无 CLI flag**：`:523` `self.challenge_attempts = getattr(args, "challenge_attempts", 3600)`。
- 日志轮转常量 `:18-22`：`MAX_LOG_FILES=200`、`MAX_LOG_BYTES=2GB`；`init_logging` `:86-91` 仅在 `log_to_file` 为真时建 `CustomRotatingFileHandler("init.log")`。

## 6. 测试

`tests/` 共 **30 个 `test_*.py`** + `__init__.py` + `conftest.py`：
```
test_apply_mods  test_audit  test_battle  test_battle_modifiers  test_battle_privacy
test_champions  test_concurrent_battle_cleanup  test_concurrent_battles
test_concurrent_dispatcher_death  test_concurrent_log_handler_regression
test_concurrent_log_routing  test_concurrent_pool_and_guards  test_format_spec
test_generations  test_helpers  test_human_pacing  test_initialize_battler
test_lead_advisor  test_modes  test_observed_evidence  test_opponent_model
test_pkmn_sets  test_poke_engine_serialization  test_search_bss  test_search_guard
test_search_main  test_search_random_battles  test_search_standard_battles
test_team_converter  test_team_pool
```

**用例数（单文件 Grep `^def test_` 实测）**：
- `tests/test_concurrent_dispatcher_death.py`：**13**（行 105,131,151,169,187,205,223,241,262,350,358,366,374）
- `tests/test_opponent_model.py`：**11**（行 65,71,77,97,108,134,146,160,194,211,234）；`:227` `_file_lines` 是辅助函数
  > 【更正】早期文档记「10 个用例」，实测为 **11**。

**测试基线**：仓库内 `README.md`、`.github/workflows/ci.yml:28`(`pytest tests`)、`Makefile:13`、`requirements-dev.txt:3`(`pytest==9.0.3`)、`conftest.py` 及 30 个测试文件里**都没有** passed 基线数字；仓库根**无** `pyproject.toml` / `setup.cfg` / `pytest.ini` / `tox.ini`。历史数字散落在 `reports/`（文字记录，非实测）：
| 文档 | 记载 |
|---|---|
| `reports/mcts_ab_experiment_plan.md:10` | 测试基线 887 passed（870 旧 + 17 新） |
| `reports/mcts_ab_experiment_plan.md:223` | 回归 **887 passed, 1 warning**（870 基线 + 17 新增；2026-10-18 夜跑） |
| `reports/concurrent_e2e_verification_round2.md:6` | 863 passed |
| `reports/concurrent_loss_fix_verification.md:28/:33` | 854 passed, 1 warning in 8.48s；修复前 838 passed |
| `reports/multibattle_concurrency_verification.md:8/:113/:265` | 819 → 822 → 838 passed |
| `reports/multibattle_concurrency_plan.md:55` | 813 passed |

> ⚠️ **数字口径警告**：工作区 memory 里另有一套 792→…→947 的演进数字，与 `reports/` 里的 813/819/822/838/887 序列**不完全一致**（可能是不同分支 / 不同时点 / 不同子套件）。**以你实际跑出来的 pytest 输出为准**，不要引用任何文档里的数字当作当前基线。详见 `docs/05` 第 6 节。

## 7. ⚠️ 仓库状态（只读取证，2026-10-25 时点）

`git log -5 --oneline`（需 `GIT_PAGER=cat git --no-pager ...`，直连 pager 会 SIGTERM 挂起）：
```
5e9d8a9 Fail loudly when the ladder search is refused by the server (popup guard)
ff3df09 fix(team-editor): 补全道具/特性/招式词表，补上 Light Clay 等 110 个合法道具
8145feb fix(team-pool): 让战斗池与队伍编辑器侧栏完全一致，scope 默认 gen9
8f541f8 Data update from smogon/pokemon-showdown (#352)
6c467c0 logging battle state happens outside of subprocess
```

**`git status --short` 关键事实**：`logs/` 被 `.gitignore:1` 忽略（故 `git status` 不扫描其内容）。
- **已修改(M)**：`fp/battle/protocol.py`、`fp/battle/state.py`、`fp/config.py`、`fp/data/sets/{base,smogon,team_datasets}.py`、`fp/main.py`、`fp/modes/{base,random_battle,standard_battle}.py`、`fp/run_battle.py`、`fp/search/{main,standard_battles}.py`、`fp/teams/load_team.py`、`fp/websocket_client.py`、`run.py`、`tests/{test_modes,test_pkmn_sets,test_team_pool}.py`。
- **已删除(D)**：`fp/teams/teams/` 下 6 个示例队（gen3/ou/sample、gen7/ou_sample、gen8/ou/{balance,band_toad,clef_sand}、gen9/ou/example）。
- **未跟踪(??) 且是核心资产**（⚠️ **这些代码尚未提交进 git**）：
  `fp/lead_advisor.py`、`fp/lead_model/`、`fp/log_routing.py`、`fp/opponent_model.py`、`fp/opponent_recorder.py`、`fp/battle_report.py`、
  `tests/test_audit.py`、`test_battle_privacy.py`、`test_concurrent_*`(5 个)、`test_human_pacing.py`、`test_lead_advisor.py`、`test_observed_evidence.py`、`test_opponent_model.py`、`test_search_main.py`、
  `reports/`、`tools/`（含 `h2h_battle.sh`、`replay_lead_advisor.py`、`team_editor/`、各类 `dl_kakuna*` 下载脚本）、`fp/teams/gl_05_26_gen9ou/`、`logs_incident_20261017_backup/`。
- 还有若干疑似临时产物：`1`、`83`、`269`、`.pytest_tmp/`、`ab_out/`、`*.bak2`、`test_local_opp_runtime.jsonl` 等（**清理前须取得用户确认**）。

> **Codex 必读**：L2 首发模型、对手建模 Part A/B、并发修复、日志路由这些**全部还在工作树里未提交**。任何 `git clean` / `git reset --hard` / 切换分支都可能永久丢失它们。动 git 之前先 `git status` 并询问用户提交策略（见 `docs/11` 第 2 节 A）。

## 8. 取证时的两处路径更正

1. 任务里写的 `fp/state.py` **不存在**，实际是 `fp/battle/state.py`。
2. `handle_team_preview` 有 **3 处定义**（`base.py:364` 方法 / `base.py:464` 模块函数 / `bss.py:45` 覆写）+ **1 处调用**（`standard_battle.py:140`）。
