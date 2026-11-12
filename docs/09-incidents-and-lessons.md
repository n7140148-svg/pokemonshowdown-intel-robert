# 09 · 事故台账与工程教训

> 来源：工作区 memory `2026-10-16.md` ~ `2026-10-18.md`、`2026-10-22.md`、`2026-10-24.md`。
> 格式统一为：**现象 → 根因（文件:行号）→ 修复 → 回归测试 → 教训**。Codex 做回归调试时先查这里，很多"新 bug"其实是老坑复发。

---

## 事故 1 ⭐ 490 连败 / rating 暴跌（2026-10-17，最严重）

**现象**：用户用 `E:\Tools\run_3_battle.bat` 启动 bot（账号 `thingsyoudo2me`），在并发 `y/n` 提示处选了 **y**，走了单账号多局并发路径。结果 **490 局全输**，rating 从 **1655 跌到 1003**。

**取证**（离线，490 个 `battle-gen9ou-*.log`，均 10-17 12:05 前后）：
- `Searching for a move using MCTS` 命中 **0** 次；`Choice:` **0**；`Winner:` **0**；`Metamon lead advisor` **0**。
- `Lost with team` 命中 120670 次（root logger 广播到所有 per-battle handler，次数被放大）。
- ⇒ **bot 从未做出任何一次决策，也从未有任何一局正常结束**。

**根因（两层）**：
1. **主因**：`fp/modes/base.py` 的 `start_battle_common` 里，并发 + 写日志分支执行 `battle._log_handler = handler`，而 `battle = Battle(battle_tag)` 在更早处已被构造 → `team_preview` 阶段的异常被吞。
2. 日志系统缺陷：并发路径下 per-battle 文件 handler 挂到 **root logger**，造成广播串写。

**修复**：重构 `start_battle_common` 的构造顺序 + 修日志 handler 注册；补 **16 个回归测试** → **854 passed**。

**教训**：
- 「一次决策都没做」比「打输了」严重得多 —— 排查时先 grep `Searching for a move`，别只看胜负。
- 默认 OFF 的开关铁律由此强化：**并发 flag 绝不连天梯开**，除非用户明确授权。
- 启动器交互式开关（`run_3_battle.bat` 的 `y/n`）是真实风险入口，改动必须做命令行构造验证（见事故 2）。

## 事故 2 启动器并发开关的构造错误（2026-10-16）

**现象**：`E:\Tools\run_3_battle.bat` 加「并发 y/n」后，需要证明各种输入下都拼出正确命令行，**绝不真的启动对战**。

**验证结论**：`run_3_battle.bat` 把 `PARALLEL` / `MAX_PARALLEL_BATTLES` 拼进 `run.py` 命令；`tools/h2h_battle.sh` 顶部变量 `PARALLEL=off`、`MAX_PARALLEL_BATTLES=2`，`on` 时才追加 `--concurrent-battles --max-concurrent-battles N`。

**教训**：批处理/脚本的开关必须做**纯构造验证**（echo 出命令行比对），不要靠真跑验证。

## 事故 3 并发功能审计的 6 个缺陷（2026-10-16 ~ 10-17）

来源：独立审阅 `general-purpose-3` 的怀疑式复核（**不是橡皮图章**）。逐条：

| # | 缺陷 | 修复要点 |
|---|---|---|
| 1 | flag-OFF 时行为不逐字一致 | 保证默认路径逐字不变 |
| 2 | `Battler` 判己方靠 p1/p2 | 改为按账号名（见下「判对方」铁律） |
| 3 | `asyncio.create_task` 无强引用 → GC 中途回收 | 保留 task 集合强引用 |
| 4 | `run_battle` 取消时未 `leave_battle` | try/finally 清理 |
| 5 | **per-room 队列泄漏**：`_route`（`fp/websocket_client.py:176-188`）为每个房间建 `asyncio.Queue`；`leave_battle`（:358-373）收到 deinit 帧后直接 return，从不删队列 → 每局泄漏一个 | 新增 `discard_room(roomid)` 实现 `self._rooms.pop(roomid, None)` |
| 6 | 脆弱顺序依赖 | 消除隐式时序假设 |

**修复后**：822 passed（#1/#2）→ 838 passed（#3–#6）；本地 PS 真机 **5/5 PASS**。

**并发链路（供 Codex 定位）**：`fp/main.py::_run_concurrent_battles` → `fp/run_battle.py::pokemon_battle`（:33-77 `while True`）→ `fp/modes/base.py::start_battle_common`（:278-317）→ `fp/websocket_client.py::_route`（:176-188）/ `leave_battle`（:358-373）；`fp/modes/standard_battle.py:8` 有类级 `_expected` 计数。

## 事故 4 per-battle 日志跨局互染（2026-10-17）

**现象**：本地端到端验证并发对战本身完全正常（2 局真并发、每局 46–55 次真实 MCTS 决策、rc=0），但**两个对局的日志写进同一个文件**。实测 `logs/battle-gen9randombattle-53_FpConcA.log` 里 102 行属于 53 局、124 行属于 54 局。

**根因**：root logger 广播 —— 每条记录发给所有 handler。

**修复**：`fp/log_routing.py` 新增 `BattleTagFilter`；`fp/modes/base.py` 增加 `_PER_BATTLE_LOG_HANDLERS` 注册表 + `pop_battle_log_handler()`；`fp/run_battle.py:69` 战斗结束后移除。→ **863 passed**（+9）。

## 事故 5 异常路径清理兜底缺失（2026-10-17）

**现象/风险**：`fp/run_battle.py` 的 `pokemon_battle`（:33-77）`while True` **没有 try/finally**。正常结束路径（:64-72）会 `leave_battle` → `pop_battle_log_handler`；但对局中途抛异常（网络断、帧解析错）时异常直接穿出：per-battle handler 泄漏、房间队列泄漏、`battle_ended` 未置位。

**修复**：补 try/finally → **867 passed**（+4）。

> **教训**：这条与事故 8（dispatcher 断线）是同一类问题 —— **异常路径上的清理永远要显式兜底**，不能只写正常路径。

## 事故 6 对手建模注入污染自生长库（2026-10-22）

**现象**：本地 PS 确定性验证发现，recorder 把**注入进去的假设招式**当成实战观测写入自生长库（Glimmora 记到 `sludgewave`，但对手根本不会）。

**根因**：Part A 注入把假设招式 `add_move` 进 `pkmn.moves`，Part B recorder 读同一字段。

**修复**：打标 `pkmn._injected_move_names` + `recorder._observed_moves` 排除 + 日志 `%s/%d` 预格式化；补 `test_record_excludes_injected_moves` → **905 passed**。

**【待核】遗留**：item / tera 注入同根因、低频，未修。

**教训**：**读同一份可变状态的两个模块必须显式区分「观测」与「假设」**。详见 `docs/04` 第 9 节。

## 事故 7 日志解析假阳性（2026-10-21 / 10-25）

- `awk` 取错 MCTS 字段，得出「MCTS 均值 220 次迭代」的荒谬结论；改用正确字段后为 **258,056**。
- `grep '2701[0-9]{6}'` 命中 PokemonSet 调试行里的科学计数浮点数（如 `count=1.2701874131101882e-07`），一度误以为空窗期有 6 场「隐形对局」；逐条看上下文后排除。
- 手写 `|move|` 样例否定他人正确实现，被 1215 条实测反证撤回。

**教训（写成铁律）**：
1. 任何「格式是 X」断言必须贴**从文件直接抓的原始行（文件名 + 行号）**，不得手写构造。
2. 统计数字必须说明**字段来源与分母口径**。
3. 正则可能命中浮点数/调试行，结论必须逐条看上下文。

## 事故 8 ⭐ dispatcher 断线导致进程假死（2026-10-24）

**现象**：进程全活、**CPU 0s**、日志 **33 分钟无更新**，看起来"还在跑"其实已死锁。进程树 PID 4420（父 `run.py` 3MB）→ 8612（子 421MB）→ 6 个 multiprocessing spawn；18:07:47 启动，参数 `--concurrent-battles --max-concurrent-battles 2`，账号 nutshell93 + stall_slow，run-count 10。

**根因链**【实证】：
```
18:40 最后一场 MCTS 完成、发出 /choose move earthquake
  → fp/websocket_client.py::_dispatch_loop 主 WS recv 抛异常 break（全程唯一一次断线，此前 33 分钟 7 场正常）
  → per-room queue 无人喂
  → fp/run_battle.py:46 `msg = await _src()` 永久阻塞（该路径无超时）
  → fp/main.py `asyncio.wait(active, FIRST_COMPLETED)` 死等
```
死前完成 6 场 rated（2 胜 4 负），1 场在局被打断。

**修复（3 文件，947 passed = 934 + 13）**：
- `fp/websocket_client.py`：新增 `DispatcherDeadError` + `DISPATCHER_DEAD` sentinel；`_dispatch_loop` 死亡改 **ERROR 日志**（原 debug 无痕）+ 置 `_dispatcher_dead` + `_wake_all_consumers()` 向 global/new-room/所有 room 队列推 sentinel；`_guarded_get()` 把 sentinel 转异常；`send_message` 死 socket **拒发（fail fast）**；`close()` 死线 no-op。
- `fp/main.py`：`_run_concurrent_battles` 捕获 `DispatcherDeadError`（发射循环 update_team / search / await_battle_room 三处 + battle task 结果两路）；**死亡不计败绩**（无 `|win|` 不是败绩，原 `except Exception` 会记成 loss 污染 W/L）；置 dispatcher_dead、取消剩余 task、退出 run。
- `fp/run_battle.py` 无需改：经既有 finally 自然清理。

**回归测试** `tests/test_concurrent_dispatcher_death.py` **13 个**：4 类消费者唤醒、ERROR 日志、send 拒发、close no-op、正常帧透传、sentinel 不泄漏、main 循环三语义。

**重启验证**【实证】：同参数 19:38 启动 21:15 结束（1h37m），10 场全打完 6 胜 4 负（60%），Elo 1590 / GXE 74 / 生涯 377W-338L，**本次未再触发断线**。

**【待核】断线根因仍未确诊**：33 分钟 7 场稳定后突发，最后动作是正常 `/choose`，无异号双登；疑 PS 服务端侧（踢号 / 限流）或本地网络。若再现先看 `Dispatcher socket recv ended` 前后是否有 `|error|` / 连接级帧。**dispatcher 修复目前只有单测验证，未经真机断线检验。**

## 事故 9 进程树误判（操作纪律）

父子双 `run.py` 是**正常形态**（父 CPU 0s / 内存 3MB），勿误杀。⇒ **终止类操作前先确认进程树并取得用户确认**。

---

## 横向教训汇总（Codex 请当 checklist）

1. **默认 OFF 是硬约束**，打开前必须回 `docs/03` 看原因并取得用户确认。
2. **异常路径必须 try/finally 兜底**；正常路径写了清理不等于异常时也清理。
3. **任何"永久 await"都要有超时或 sentinel 唤醒机制**。
4. **不把"没有 `|win|`"当成败绩**（会污染 W/L 统计）。
5. **logging 用预格式化字符串**，别在 foul-play 里用 `%s/%d`（老毛病，不替换）。
6. **判「对方」靠账号名，绝不靠 p1/p2**。
7. **观测 vs 假设必须显式打标区分**。
8. **数字结论要带字段来源 + 分母口径 + 去重规则**。
9. **日志轮转会实时删文件** → 长任务随跑随冻，只用冻结副本。
10. **验证要"怀疑式复核"而非"橡皮图章"**，尤其改并发 / flag-OFF 路径时。
