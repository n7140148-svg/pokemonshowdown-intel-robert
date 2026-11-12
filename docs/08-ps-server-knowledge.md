# 08 · Pokémon Showdown 服务端工程机制

> **范围声明**：本文只记录**开发与调试必需的协议 / 服务端机制事实**（为什么匹配不到人、评分怎么算、日志帧怎么解析），不含任何针对线上服务的攻击研究或操作手法。
> 来源：工作区 memory `2026-10-21.md` ~ `2026-10-25.md`、`MEMORY.md`，以及本地 `E:\Tools\pokemon-showdown` 源码只读核对。

---

## 1. userType 映射【源码】

| 值 | 含义 | 实际区别 |
|---|---|---|
| 1 | 未注册 | 无天梯资格 |
| 2 | 注册 | 匹配墙的「正常人类」基准 |
| 3 | sysop | 全局管理员；token 里唯一能直接全局提权的值 |
| 4 | autoconfirmed | 跳过新号限制，非特权 |
| 5 | permalocked | 能登录但禁言 / 禁战 / 禁改名 |
| 6 | permabanned | 登录即踢硬封 |
| **10** | **bot** | `isUserBot=true` + autoconfirmed，触发匹配墙 + 同 IP 墙 |

登录服 `Session.getAssertion()` 逻辑【源码】：默认 `2`；isSysop → 3；`banstate >= 100` 注销；`>=40` → 2；`>=30` → 6；`>=20` → 5；`banstate <= -10` → 4；**`group === 10` → 10(bot)**；`banstate === 0` 且注册 > 7 天 + 有天梯胜 → 自动置 4；未登录分支 DB 无记录 → 1。

`isUserBot` 全仓仅两处写入（`users.ts:845` assertion 驱动、`users.ts:1022` 合并继承），**无自动检测链路** ⇒ 标记只能由人工 / staff 施加。

> 【口径差异·待核】关于 `userType==='10'` 是否同时置 `settings.allowLadderBots=true`：不同日期的记录不一致。10-25「之十一」以 commit `a17f847`（10-23 04:14 CST）「bot 登录置 allowLadderBots=true，对墙无实际影响」解释该差异。**以源码为准，别凭记忆。**

## 2. `/trn` assertion 格式与 userType 提取【源码 + 实证】

- 源码版：`data = challenge,userid,userType,date,hostname`，用 `signAsync('RSA-SHA1', data, Config.privatekey)` 签发。
- `users.ts:665` `tokenData.split(',')` → **第 3 个逗号字段（0-indexed `[2]`）即 userType**。
- **实测断言比 5 段版多 3 段**（`[5]`–`[7]` 为额外签名字段），**userType 位置不变（仍是 `[2]`）**；可用 date 字段反解时间戳与上下文自洽来自校验。
- Game server 侧校验（`users.ts:640-713`）：拆 `tokenData;tokenSig` → hostname ∈ `legalhosts` → 字段数 ≥5 → `signedUserid === userid` → `signedChallenge === connection.challenge` → 时间差 ≤ expiry（默认 25h）→ RSA-SHA1 验签 → **返回 userType 全链路信任**。

**两种读取 userType 的方法**【实证】：
1. 用户浏览器 console 里的 `/trn <user>,0,<assertion>` 行。
2. **bot 自己的 `init.log`** 也含 `/trn`（`DEBUG Sending message to websocket: |/trn ...`）。提取：
   ```
   grep -oE '/trn [^,]+,0,[^,]+,[^,]+,[0-9]+' init.log | awk -F, '{print $1,$5}'
   ```
   ⚠️ 该 grep 的 `[0-9]+` 会命中 PokemonSet 调试行里的科学计数浮点数（如 `1.2701874131101882e-07`）造成假阳性，查对局只能 grep 精确 battle id。

**探针红线**【实证】：连 sim3 取 `|challstr|` → POST `https://play.pokemonshowdown.com/api/login`({name,pass,challstr}) → 读 assertion。响应体**前面带一个 `]`**，直接 `.json()` 会炸。**探针绝不向服务器发 `/trn` 认领身份**，否则把正在跑的 bot 踢成 Guest。

## 3. `matchmakingOK` 四道关 + 评分窗口【源码】

位置 `ladders.ts:376-430`，是 `/search` 必经路径：`searchBattle → addSearch:455 → matchmakingOK`，另有 `periodicMatch:492` 每 60s 重评池内（把等待最久的 search 逐一试配，每 tick 每格式最多成一局）。

| # | 关卡 | 细节 |
|---|---|---|
| ① | **IP 去重** | `new Set(users.map(u=>u.latestIp)).size !== users.length → return false`；`noipchecks` 为真时全部跳过（本地服 :8000 因此绕墙） |
| ② | lastMatch 防重配 | 只挡**上一个**对手 |
| ③ | **评分窗口 searchRange** | gen9ou / gen9randombattle 起步 **±50**（其它格式 100）；`elapsed` 是毫秒，`+= elapsed/300` ⇒ 每 0.3s +1（每 3s +10）；>300 后每 3s +1；硬上限 **±600**；`elapsed = Date.now() - Math.min(...times)` 由等待更久那方驱动；约 **16 分钟封顶**（「等 25 分钟才宽」是错算） |
| ④ | **bot 墙** | `isUserBot && 对方未 opt-in → false` |

> 【更正】早期文档曾写「无评分窗口 / 评分段已排除」，属**错误前提**，已由 master 源码纠正。窗口确实存在，只是随时间放宽，只卡头几分钟。

**IP 限流**（monitor.ts）：`countPrepBattle` 12 次 / 3 分钟（弹窗）、`countBattle` 30 局 / 5 分钟（掐连接）。

**弹窗分类器坑**【实证】：不是所有 popup 都是拒绝。`Ladders.disabled` 不 return、`already searching` 只是不重复入队 ⇒ **见 popup 就 raise 会误杀并发路径重试**。

## 4. `allowLadderBots` 墙：政策与 commit 时间线【源码 + 实证】

| 时间 | 事件 |
|---|---|
| 10-03 | commit `2619487d63` 支持 userType '10' |
| 10-09 | 官方政策帖《Initial policy on bots》(smogon 3789528) |
| 10-13 | `/ht` 登记入口 + `/setbot`（`233620b398`） |
| **10-20** | **`/allowbots` + `allowLadderBots`（默认 false）**：`de1cf598c7`、`7777a79491` |
| 10-21 | 客户端开关 |
| 10-22 | 登记即刻生效（`03d56f41d3`） |

**UTC 校正**（关键，曾导致时间线自相矛盾）：
- `de1cf598c7`（墙 v1）= 10-20 21:59 UTC = **10-21 05:59 CST**
- `7777a79491`（墙 v2）= 10-20 23:14 UTC = **10-21 07:14 CST**（1.3 小时后）
- `a17f847`（bot 登录置 allowLadderBots=true）= **10-23 04:14 CST**
- `|updateuser|` flip 窗口（10-21 02:10 → 10-22 00:50 CST）含 commit 时刻 ⇒ 墙最晚 10-22 00:50 已上线；`ladders.ts` 此后零 commit（未回滚）。

**v1 有 bug**：
```
allOptedIn = users.every(x => x.settings.allowLadderBots && !x.isUserBot)
```
bot 自身不满足 `!isUserBot` → 恒 false → **v1 挡掉一切含 bot 的配对（含 bot-vs-bot）**。

**v2 修复**：改为 `users.some(isUserBot) && hasNonOptedInUsers` ⇒ **只挡 bot vs 未 opt-in 真人，放行 bot-vs-bot**。

**墙的语义**【实证】：被标号只能配「其他 bot」或「手动开 `allowLadderBots` 的人类」，普通人类**静默拒配、无 popup、无报错**（入队成功但永不配对）。自己开 `allowLadderBots` 没用 —— `ladders.ts:406` 只对 `!u.isUserBot` 求值（看的是**对方**）。

> 【更正】10-20 版根因报告把 `ladders.ts:409` 说成「弹窗拒绝」，10-23 纠正为「静默匹配池限制」。

## 5. 房间与隐身【源码 / 实证】

- **对战房 privacy 只能建房前设**；`/hidenext` 安全 —— `addSearch` / `matchmakingOK` 均不检查 hidden，`hiddenNextBattle` 只在 `getUpdateuserText` 同步用（`ps_users.ts:973`），只决定下场建房是否隐藏，**不影响匹配资格**（bot 只见最终长 id）。
- `/modjoin` **锁门即触发改名 → 运行时不可用**，别在跑局时用。
- `noipchecks=true` 使 `matchmakingOK` 提前 return true，本地服绕墙；本地 :8000 需 `noguestsecurity=true` + `noipchecks=true`。

## 6. 日志帧与解析规则（**最常踩坑**）【实证】

| 事实 | 说明 |
|---|---|
| **`|request|` 只含自己 side** | 40 万行实测 `side.name` 全是己方账号 ⇒ 对手 item / ability / teraType / 完整 moves **在日志里根本不存在**。`side.id` + `side.name` 是归属权威，`side.pokemon[]` 含 details / moves / baseAbility / ability / item / teraType |
| **判「对方」靠账号名，绝不靠 p1/p2** | 我方 `nutshell93` **有时是 p2**；bot 账号既可能 p1 也可能 p2 |
| 胜负判定 | `|win|<name>`；归属看 `|request|` 首帧 `side.pokemon[].ident` |
| `|player|` / `|win|` 在哪 | **主连接流 `init.log` 里有**，而 **per-battle 日志反而缺 `|player|` 头**（直接跳到 `|request|`）⇒ 评分轨迹 / 对手 census 都从 init.log 提 |
| `|player|` 重复 | 同一 tag 因重连重复：457 个 tag 恰 2 行、138 个 3 行、最多 ×19 ⇒ 直接数行会把 1 局算成多局 |
| `|move|` 格式 | `|move|<使用者>|<招式名>|<目标>|<附加>`，**parts[2] 是招式名**（源 D 1215 条实测）。曾误记为「parts[2] 是目标」，已更正 |
| `|poke|` | 行尾 `|` 是空字段分隔符；物种名须拆 species + form + gender |
| avatar | 可能是字符串（如 `cynthia`），rating 正则别写 `[0-9]*` |
| battle tag | 两段 / 三段两种：`battle-[a-z0-9]+-\d+(?:-[a-z0-9]+)?` |
| 去重 | 观测级统计**必须按 battle_tag 去重**，否则局数虚高约 7 倍 |

**live logs 轮转**【实证】：live logs 2.01GB 已逼近阈值（≤200 个 / 2GB），会在核查中**实时删文件** ⇒ 长任务必须**随跑随冻，只用冻结副本**，不可当唯一数据源。

> **方法论铁律**：曾凭记忆手写「`|move|` 样例」否定他人正确实现，被 1215 条实测反证撤回。⇒ 任何「格式是 X」断言必须贴**从文件直接抓的原始行（文件名 + 行号）**，不得手写构造。

## 7. 计分结论【实证】

被标账号只要在 rated 对局里（bot-vs-bot 或 opt-in 人类），**天梯分数照常涨落**。计分代码三层（`ladders-local.ts` / `ps_server_ladders-remote.ts` / 官方 `ntbb-ladder.lib.php`）**都无 bot 排除**。「分数不动」是因为**打不上 rated 局（无局可计）**，不是计分屏蔽。

旁证：10-24 有 3 处相邻对局 `nutshell93` 评分完全不变，与登录服 commit "Don't update human rankings in bot-involved matches" 吻合（**该场景下计分被跳过**）。

## 8. 匹配路径的工程约束【实证】

| 路径 | 约束 |
|---|---|
| 直接 `/challenge` | `rooms.ts` 无 `isUserBot` 检查、无同 IP 墙 ⇒ **可行**；不走 `matchmakingOK` |
| 本地服 :8000 | `noipchecks=true` 绕墙，用于端到端验证 |
| bot-vs-bot 天梯 | v2 墙放行，但**同 IP 下被 `ladders.ts:394` 挡死** |
| 换新号绕标记 | **不可行**：`t/lung`（手打、从没跑过 bot）也被标 → 操作者 / 网络 / 行为判定；且踩 PS 10-09 新政红线 |
| 摘标 | 只能 staff（`/helpticket` 或 DM chaos）；**未查到任何一例成功摘标的公开记录** |

## 9. 匹配墙盖棺时间线（含【更正】）【实证】

**bot 账号名单**：`thingsyoudo2me` / `iknow_why` / `nutshell93` / `FpConcA` / `FpConcB`（另手打号 `t/lung` → PS 去斜杠规范为 `tlung`）。三号曾共享恒定 clientid `e2bfee49c1c95d62`。

**逐号 userType 实测 —— 必须按更正后口径**：
| 账号 | userType | 证据 |
|---|---|---|
| thingsyoudo2me | **4** | 【更正】用户网页 console `/trn` 断言（约 10-24） |
| iknow_why | **4** | 【更正】用户 console 自查，10-25 之十确认 |
| nutshell93 | **10** | init.log 62 条 `/trn` 全为 10（第一个被实证的 bot 标记号） |
| FpConcA / FpConcB | **未读** | 【待核】 |
| t/lung → `tlung` | **10** | 手打也被标 |

> ⚠️ **【更正·最高优先级】**：10-23 曾盖棺「五个 bot 号 + t/lung 全部 userType=10，匹配墙把五个号墙死」，**10-25 被逐号实测推翻**。⇒ **「五个号全被墙死」这个归因从一开始就错**。详见 `docs/11`。

**对局时间线**：
- 10-21 00:50:47 → 03:32:08 `nutshell93` 连打 **35 场 rated 真人局**（评分 1590 → 峰值 → 1681；磁盘 battle 日志逐个 `|player|` 核实；init.log 另有 54 次 Initialized battle）。最后一次真人匹配 = 10-21 03:32:08（`battle-gen9ou-2701802467`，Jirachi Is Woke 1666 vs nutshell93 1681，`|rated|`）。
- **streak 最后一场比第一个墙 commit 早 2 小时 27 分** ⇒ 35 场能打完、之后骤然全停，时间线自洽。
- 服务器设置 schema 在 (10-21 02:10, 10-22 00:50) 间翻转（`|updateuser|` 出现 `"allowLadderBots":false`）。此后所有 `/search` 都进池（`searching:["gen9ou"]`）但永无人匹配，重搜被回 `Couldn't search: You are already searching`（原搜索挂池）。
- **10-21 03:32 → 10-24 18:18 约 3.6 天零对局**（`2701802467 < id < 2703799011` 窗口无任何 join）。
- 10-23 21:55 修队后实跑：`|updatesearch|{"searching":["gen9ou"]}` 铁证进池，300s 超时后每轮重搜被回 already searching，晚高峰 70+ 分钟池内零匹配。

**已排除的其它解释**【实证】：评分区间（热门段 + 晚高峰 70 分钟零匹配）、`/hidenext`（不看 hidden）、队伍非法（Pecharunt 泄露名 `Toxic Chain` → 已修 `Poison Puppeteer`，refused 弹窗彻底消失）。

**10-24 恢复局 = bot-vs-bot**【实证】：18 场 8 胜 8 负、只碰 **12 个不同对手**（XemOUr 4 次、luisowbro / vadmiral_slowbro / Kanha Greninja 各 2 次）；**对手 `FP.//06` 是另一个 foul-play bot，还赢 1 场**；3 处相邻对局评分完全不变 ⇒ 解释 A 成立：**墙已上线（v2 放行 bot-vs-bot），打的是 bot-vs-bot，不是人类**。

**空窗期构成**：① 墙挡人类 ② **10-22 02:47 → 10-23 16:08 整整 37 小时零登录（bot 离线）** ③ 10-22/23 bot 注册基建 commit（`d601443` / `03d56f4` / `f02a6ee` / `1636bf6`）早期无其他 bot 可配。52 次 `searching` 几乎全落在两个有对局时段。

**连接层与真人无协议差异**【实证】：同一 wss + 同一 `api/login` + 服务器签名 assertion 原样 `/trn`，**没有「我是 bot」握手标记**；差异只在行为（瞬时确定性决策、24h 无人值守、同开 5 局、队伍逐字相同、从不聊天）。

## 10. 待核 / 悬案（详见 `docs/11`）

- 【待核】`/version`：官方服输入可显示服务器 commit 哈希（≥ `de1cf598c7` ⇒ 墙已上线），**一锤定音未做，需用户本人在官方服输入**。
- 【悬案】`nutshell93` 1681（10-21 03:32）→ 1611（10-24 18:18）降 70 分，但中间**无任何有日志对局**（无日志对局 or 评分重置，未明）。
- 【待核】峰值评分 1773 / 1792 / 1812 三种记法；35 场 vs 54 局 vs「56W/58L」口径冲突。
- 【待核】FpConcA / FpConcB 的 userType 未读。
- 【待核】10-24 墙是否曾短暂失效（userType=10 的 nutshell93 18:08 登录、18:20 前被配走）——两种归因并存、未完全收口。
- 【待核】staff 判定 bot 的成文标准**不存在**（官方措辞 "imprecise science at best"）。
