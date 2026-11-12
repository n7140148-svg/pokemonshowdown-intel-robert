# 06 · 实战成果与实验结论

> 来源：`E:\Tools\foul-play\reports\*.md`、`reports\metamon-audit-plan\00-06`、工作区 `foulplay_gen9ou_3000ms_20game_report.html`。所有数字后附 `文件:行号`（HTML 用 `L行号`）。
> **本报告只写「已经跑完并有数字」的成果**。方案 / 规格 / 待编码 / 预期值一律放到 `docs/07` 并显式标注状态。

---

## 0. 贯穿全部实验的预登记纪律（**Codex 评估任何"改进"前必读**）

来源 `reports\mcts_ab_experiment_plan.md:144-150`（§5.2），phase-2/3 沿用：

- **保留规则**：臂胜率点估计高于 B0 **且**（Wilson CI 与 B0 明显不重叠 **或** Fisher p<0.05 且差 ≥ MDE）。
- **回退规则**：点估计低于 B0 **且** Fisher p<0.10。
- 否则判「证据不足 / DIRECTIONAL_ONLY」。
- **铁律**（`mcts_ab_phase2_plan_20261019.md:70`）：**任何臂在过预登记保留规则之前，开关保持默认 OFF。**

**MDE 工具**：`tools/mde_two_prop.py`，复算命令 `.venv\Scripts\python.exe tools/mde_two_prop.py --p1 0.479 --n 80 87 94 130`。n=22→37.4pp / 50→26.4 / 100→19.2 / 200→13.7 / 400→9.8（α=0.05 双侧、power=0.80、p₁=0.52，`mcts_ab_experiment_plan.md:113-120`）。

> ⚠️ **最重要的方法论教训**（`L1_final_report_20261020.md:88`）：**预登记 + 复现纪律抓到一个假阳性**。只看 phase-1，argmax 看起来值 +14.6pp；做了预登记的复现检验后效应消失。⇒ 任何「方向有利」的**单期**结论，在追加数据前不得作为开默认开关的依据。

---

## 1. MCTS 编排层 A/B 实验链（B0 / B1 / B5 / B4）

### 1.1 臂定义与固定参数
| 臂 | 参数 | 定位 |
|---|---|---|
| **B0** | `parallelism 5 / threads 1 / 3000ms / sampling` | 对照（era-B 逐字行为） |
| **B1** | B0 + `--final-choice argmax` | 终选取最高权重 |
| **B2** | B0 + `--repeat-penalty 0` | **未跑天梯**（见 §1.5） |
| **B3** | B0 + `--deep-hypotheses`（10 假设×3s → 5 假设×6s） | **跳过** |
| **B4** | B0 + `--lead-policy mcts`（advisor 消融） | 见 §1.4 |
| **B5** | B0 + `--search-threads 2`（phase-2 新增，4.5× 算力） | 见 §1.3 |

固定搜索参数 `parallelism 5 / threads 1 / 3000ms`（`mcts_ab_experiment_plan.md:57`）；全程同一队伍 Team B = (ceruledge, darkrai, great tusk, hatterene, iron crown, zamazenta)（`mcts_ab_final_report_20261019.md:21`），排除换队污染。账号 nutshell93（用户 2026-10-19 授权），真实天梯 `sim3.psim.us` gen9ou。

### 1.2 phase-1：argmax 看起来有效（`mcts_ab_final_report_20261019.md`，2026-10-19）

| 臂 | 局数 | 胜-负 | 胜率 | Wilson 95% CI |
|---|---|---|---|---|
| B0 | 94 | 45-49 | **47.9%** | [38.1%, 57.9%] |
| B1 | 80 | 50-30 | **62.5%** | [51.5%, 72.3%] |

- 点估计差 **+14.63pp**；OR=1.815；Fisher 双侧 **p=0.0669**（`:26-33`）。
- MDE（p₁=0.479）：n=94→20.0pp / n=87→20.8pp / n=80→21.6pp。**实测 14.6pp 低于 MDE**。
- 机制自检：B0 358 决策 294 选 top1 = **82.1%**；B1 327 决策 317 选 top1 = **96.9%**（argmax 确实生效，`:39-40`）。
- 深挖（176 局 / 5439 决策点，`:42-53`）：contested 偏离 top1 B0 53.5% vs B1 13.9%；决策空间 72.7% 仅 1 候选；**中局（22-28 回合）分层 B0 40.7% vs B1 77.3%，Fisher p=0.0193**；KO 差 B0 −0.68 vs B1 +0.14。
- **裁定**：CI 重叠 [51.5%,57.9%] 存在、p=0.067>0.05、14.6pp<20.0pp ⇒ **不满足，不保留**，开关维持 OFF（`:57-59`）。

### 1.3 L-1 主天梯实验：复现失败 + threads=2 被否（`L1_final_report_20261020.md`）

窗口 2026-10-19 19:15:50 → 10-20 10:54:00，**15h38m**；18 段（各臂 6 段，s19-s36）；188 日志 → 剔 1 无结果局 → **计分 187 局（B0 60 / B1 60 / B5 67）**；队伍一致 187/187；段均 49.7 分钟（`:3-17`）。

| 臂 | 局数 | 战绩 | 胜率 | Wilson 95% CI |
|---|---|---|---|---|
| B0 | 60 | 30W-30L | **50.0%** | [37.7%, 62.3%] |
| B1 | 60 | 29W-31L | **48.3%** | [36.2%, 60.7%] |
| B5 | 67 | 36W-31L | **53.7%** | [41.9%, 65.1%] |

**问题 1（主）B5 vs B0 → 未过保留规则，threads=2 默认 OFF**：+3.7pp、Fisher **p=0.7239**、MDE 23.3/24.5pp、Wilson 大幅重叠（`:29-33`）。
**机制自检通过，阴性不是开关没生效**：Iterations visits 中位 B0 218,000（17,190 样本）/ B1 221,000（16,245）/ **B5 988,000（16,255）= 4.53×**（争抢基准 4.4×，`:57-65`）。
**问题 2（次）B1 中局分层复现 → 未复现**：B0 n=16 37.5% vs B1 n=21 47.6%，Fisher p=0.7388；phase-1 同层是 40.7% vs 77.3%（p=0.0193）→ 效应缩到 +10.1pp 不显著（`:35-43`）。
**KO 差辅证**：B0 −0.50 / B1 −0.77 / B5 +0.10；B5−B0 差 +0.60，bootstrap 95%CI [−0.43,+1.65] 跨 0（`:47-53`）。
**长局结构**：>28 回合胜率 B0 38.5%(n=26) / B1 22.2%(n=18) / B5 25.0%(n=12)；短局 ≤10 回合全部 100% 胜；回合中位 25/26/22（`:69-71`）。
**跨期合并**：B0 两期 154 局 48.7% / B1 两期 140 局 56.4%，Fisher p=0.1997；**B1 跨期摆动 14.2pp**（62.5%→48.3%），B0 两期稳定 47.9%/50.0%（`:73-81`）。
**结论**（`:85-92`）：① threads=2 无 detectable 效果 → OFF；② argmax 信号未复现 → OFF；③ 方法论教训见 §0。**功效局限**：n≈60-67/臂，MDE≈23-24pp，只能排除巨大效应；resolve 10pp 需 **~600 局/臂**。

### 1.4 B4：`--lead-policy mcts` 消融（`B4_final_report_20261021.md`）

每臂 **60 局**（6 段×10 局），s37–s48 交替，冷却 120s，窗口 10-20 12:41 → 10-21 03:32。

| 臂 | W-L | n | 胜率 | Wilson 95% | MDE |
|---|---|---|---|---|---|
| B0（advisor 选首发） | 34-26 | 60 | **56.7%** | [44.1, 68.4] | 23.5 |
| B4（MCTS 选首发） | 30-30 | 60 | **50.0%** | [37.7, 62.3] | 24.5 |

- 差 **−6.7pp**，Fisher **p=0.5832**，Wilson 大幅重叠 ⇒ **`DIRECTIONAL_ONLY`**；反向判定需差<0 且 p<0.10，不满足 ⇒ **也不声称「advisor 有价值」**（`:15-27`）。
- 机制自检：B0 六段每段 21 条 `Metamon lead advisor` 全存在；B4 六段 **0 条**；B0 首发 darkrai 37(62.7%)/ceruledge 15(25.4%)/ironcrown 7(11.9%)；B4 hatterene 29(49.2%)/zamazenta 14(23.7%)/darkrai 7/greattusk 6/ceruledge 3（`:31-38`）。
- 辅证：KO 差 B0 +0.03 vs B4 −0.47；**长局 >28 回合 B0 9/18=50.0% vs B4 4/18=22.2%（−27.8pp，p=0.1642）**，短局两臂完全相同各 25/41=61.0% ⇒ **全部 −6.7pp 落在长局**（`:44-55`）。
- 段间波动：B4 相邻两段 88.9%→20.0%，**单段摆动 69pp**（`:59-65`）。
- 事故链：C 盘写满（`modify_backup` 14GB）、双重 `run.py` 同账号互踢、`logs_frozen` 污染；最终主终点只用 12 条 `exit=0` 干净段（`:69-77`）。
- **结论**（`:81-83`）：**B4 不启用，维持 B0（advisor 选首发）为默认**；n=60/臂时 ±23pp 以下点估计不可作决策依据。

**B4 预检**（`phase3_plan_20261020.md:17-27`，364 局日志）：MCTS 与 advisor 的 lead 分歧率 **337/364 = 92.6%**（B0 96.8% / B1 92.9% / B5 82.4%）；advisor lead 分布 darkrai 62.6% / ceruledge 30.5% / ironcrown 6.9%；按 lead 胜率 darkrai 50.4%(n=226) / ceruledge 56.4%(n=110) / ironcrown 56.0%(n=25)；模型 `gen9ou_high_elo_v4` val_acc 0.5405。

### 1.5 被否决 / 跳过的臂（**别重复跑**）

| 臂 | 裁定 | 依据 |
|---|---|---|
| **B2**（penalty=0） | 不跑天梯 | D-1 Phase C：216 个 penalty 覆盖 top1 中，Kakuna 认同覆盖后选择 88(40.7%)、认同被覆盖 top1 仅 9(4.2%)、都不认同 119(55.1%)，Fisher p<0.0001 ⇒ **repeat-penalty 在做正确的事**（`d1_kakuna_audit_report_20261019.md:95-101`）；期望效应 1-3pp，n=45/臂 MDE≈27pp |
| **B3**（deep-hypotheses） | 跳过 | D-1 早期 gate：早期输局 37.9% vs 胜局 37.9%（**完全相等**）→ 早期 gate 失败（`d1_kakuna_audit_report_20261019.md:70-72`）；D-2 二次确认 null（37.9% vs 40.0%）。另 B3 只改对手未充分暴露分支，中后期两臂完全一致 |
| **长局终盘专项** | 不单设实验臂 | `phase3_plan_20261020.md:6-13`：长局内部分层（29-40 / 41+ 回合）Kakuna 一致率输局 33.6%/34.3% **完全持平、无终盘塌陷**；112 个长局「不可回归点」中位输局第 25 回合 / 胜局第 27 回合；75% 长局输局中 22/75 在第 28 回合后仍 KO 均势或领先；换人率各阶段持平。⇒ **「长局败局 bucket」应正名为「磨败的中盘局」** |

**共同结论**：三个已测旋钮（argmax / threads=2 / penalty 保留）全部有明确答案，**默认配置不需要改**（`d2_kakuna_audit_report_20261020.md:39`）。

---

## 2. Kakuna 离线参照审计（D-1 / D-2）

### 2.1 方法（两份报告一致）
- 参照策略 **Kakuna**（metamon 基线，**142M 参数，epoch 34 checkpoint**），argmax 推理，top-5 动作概率。
- 对齐：位置式 1:1（team preview 两侧均不计）；引擎标准口径剔除 `chosen_missing`；分桶 early≤10 / mid 11-28 / long>28；battle-level 中位数抗聚簇。
- 一致率 = Kakuna argmax 动作索引 == 我方实际选择索引。

### 2.2 D-1（`d1_kakuna_audit_report_20261019.md`，phase-1 176 局冻结日志）
- 覆盖：176 冻结日志（B0 96 / B1 80），剔无结果 2、截断 1、metamon 解析失败 7 → **实际审计 166 局 / 5050 决策点**（B0 90/94=95.7%、B1 76/80=95.0%）；计分 4697 行。7 个失败全是形态变体（Sinistcha-Masterpiece / Keldeo-Resolute 无 tera 类型）与回放截断，非数据损坏。
- 结果：总体一致率 **B0 38.6%（2529 点）vs B1 42.1%（2168 点）**，gap +3.5pp。
- **长局诊断**：长局输局 35.9% vs 胜局 56.0%（battle-median 31.6% vs 56.3%）→ 曾据此认为「threads 深度方向正确」。
- **早期 gate**：早期输局 37.9% vs 胜局 37.9% → B3 不值得跑。
- 高频分歧 top5：116 drainingkiss→psychicnoise、74 bodypress→irondefense、56 icebeam→darkpulse、50 switch zamazenta→switch greattusk、47 shadowsneak→bitterblade。

### 2.3 D-2 ⭐（`d2_kakuna_audit_report_20261020.md`，L-1 数据）
- 目的：回答预登记问题「**B5 的 4.5× 搜索算力有没有把决策推向参照策略？**」
- 覆盖：L-1 188 场 → **182 局 / 5341 决策点**（B0 57 / B1 58 / B5 67），计分 4898 行。
- 一致率：**B0 41.1%(1663) / B1 37.8%(1649, −3.3pp) / B5 47.0%(1586, +5.9pp)**。
- **核心发现**：B5 的 4.5× 算力**确实**把决策推向参照策略（+5.9pp，三臂最高；visits 218k→988k），但同批胜率 B5 53.7% vs B0 50.0%（+3.7pp，p=0.7239，未过保留规则）。
  ⇒ **「搜索算力 → Kakuna 一致率↑（可检测）→ 胜率不动（不可检测）」**。
  ⇒ 证伪 D-1 隐含推理链的后一半：**「更像 Kakuna」≠「更能赢」**；Kakuna 是合理参照策略但不是这个天梯池里的致胜策略；MCTS 与 Kakuna 的分歧大部分是**风格差异而非错误**。
- 复现项：长局输局 39.3% vs 胜局 59.5%（同 D-1）；早期 gate 依旧 null；分歧模式稳定（drainingkiss→psychicnoise 78× 等）。

---

## 3. 20 局 3000ms Elo 报告（工作区 HTML，2026-10-16）

来源 `C:\Users\abc\WorkBuddy\2026-10-10-14-15-51\foulplay_gen9ou_3000ms_20game_report.html`。

- 账号 **thingsyoudo2me**；配置 **`--search-time-ms 3000 --search-threads 4`**（L43-44）。
  > ⚠️ **这是独立批次**：`threads 4` 与 MCTS A/B 的 `threads 1/2` 不同，不要合并比较。
- 核心数字（L47-52）：基线 Elo **1663** → 终局 **1691**（净 **+28**）；**11 胜 9 负（55.0%）**；峰值 **1740 / +77**（G14）。
- 关键统计（L103-108）：最长连胜 **5 局**（G8–G12）；最长连负 2 局；**爆冷失利 4 局**（G4/G15/G16/G19，对手分别低 102/10/120/107 分）；下克上 3 局（G9/G14/G20）；对手平均分 ≈1666；单局最大 −27（G16）/ +26（G14）。
- 走势结论（L112）：前 14 局主升段，5 连胜冲到 1740；G15–G19 陷入 1 胜 4 负回撤，**4 次爆冷是主要失分来源**；末局 G20 收复部分失地。整体小幅正收益，末段稳定性偏弱。
- **局限（报告未载）**：纯描述性 Elo 报告，**无显著性检验、无置信区间、无对照组**。数据来源 `E:\Tools\foul-play\logs`（只读 Grep，未运行程序），链校验「每局旧分=上局新分」全程自洽（L114）。

---

## 4. rating 掉分诊断（`rating_drop_diagnosis_20261018.md`）

- **结论**：程序正常工作；未检出 bot 在败局中下可识别的坏棋；掉分最符合「新队伍在当前天梯变弱/被克」——**但后被修正**（见下）。
- 证据：今日约 31 局，Python Traceback=0；Kakuna 首发顾问触发 30 次；每局 MCTS 决策 11–67 次；与 10-17 的 **490 连败事故**（MCTS 决策 0 次、`Winner:` 0 次）本质不同。
- Kakuna 离线审计：31 局 / **862 可计分决策点**；胜局 16 局 378 决策分歧率 50.79%，败局 15 局 484 决策分歧率 55.37%；对局级 bootstrap 95%CI（败−胜）**[−12.7%, +4.0%] 跨零不显著**；267 个「Kakuna 高置信分歧」无鉴别力。固有约 **53% 分歧率**（S1 批 55%）。
- 队伍核验：**Team A（新）** = Great Tusk/Zamazenta/Darkrai/Hatterene/Iron Crown/**Ogerpon-Wellspring**；**Team B（原）** = 同 5 只 + **Ceruledge**；**两队唯一差异一个格子**。
- 全量战绩：S1 批（Team B）35 局 24-11 **68.6%** CI[52.0,81.4]；时代 A（Team A）32+1 无记录 17-15 **53.1%**；时代 B（Team B）21+1 被误杀 11-10 **52.4%**。Fisher A vs B **p=1.000**；S1 vs A p=0.219；S1 vs B p=0.264。
- **修正后结论**（`:98-104`）：「换回原队伍后胜率回暖」前半句属实（七胜一负 +142），但「大概率是队伍原因」**未被数据坐实**；当前证据更支持「**正常波动 + rating 爬升后对手变强的回归效应**」；队伍差异既未证实也未排除；建议维持 Team B 再打 30–50 局。
- **⚠️ 操作事故**（`:88-93`）：初查把两个同命令行 python 进程（10036/20784）误判为「挂死重复启动」并 `taskkill //T`，实为**正常父子结构**，杀掉活 bot 及 5 个搜索 worker，一局进行中被打断。⇒ 见 `AGENTS.md` 红线 8。

---

## 5. 工程验证类成果（并发 / 隐身 / 启动器）

| 报告 | 结论 |
|---|---|
| `multibattle_concurrency_plan.md` | 服务端允许单账号最多 **5 场/账号**（`users.ts`/`monitor.ts:244-249`/`rooms.ts:2201`）；**关键修正：并发只提升对局/小时吞吐，不提升（甚至降低）单局强度**（全局 MCTS 池≈6 进程，N 场并发每场≈6/N）；flag 默认 OFF |
| `multibattle_concurrency_verification.md` | flag-OFF 零回归 **819 passed / 0 failed**(`:8`)；缺陷清单表**编号至 #7**（含 `RandomBattleMode.start_battle` 缺 `msg_getter` 致命项，已修复、**822 passed** `:113`）——⚠️ 报告**没有「共 7 项缺陷」的明确计数句**，勿写成确定结论；pytest 819→822→**838 passed/0 failed**(`:265,:276`)；**手工分核把损失从 −22.3% 收窄到 −5.1%**(`:223`) ⇒ 同机多进程必须显式设 `--concurrent-core-budget` |
| `concurrent_e2e_verification.md` | 第二关 **5/6**：事故签名全消、并发跑通，但 per-battle 日志仍跨局污染 |
| `concurrent_e2e_verification_round2.md` | **6/6 全过**，互染修复有效 |
| `concurrent_loss_root_cause.md` | 确切异常 **`UnboundLocalError: cannot access local variable 'battle'...`，抛在 `fp/modes/base.py:200`**；离线 mock socket 复现（未连 PS） |
| `concurrent_loss_fix_verification.md` | 全量 **854 passed / 0 failed**；新增 16 个离线回归测试；用「临时把缺陷改回去」证明测试非空转 |
| `hidenext_verification.md` | **5/5 PASS**；双 bot 同账号并发（FpConcA/FpConcB）本地 PS gen9randombattle 正常结束 `rcs={'FpConcA':0,'FpConcB':0}` 无 hang；`hidenext_sends=4` |
| `modjoin_spectator_lock_verification.md` | **总结论 FAIL，不可放行**：① 与决策消息同波发送被限流丢弃，锁门间歇失效；② 锁门一旦生效（房间 rename）→ 并发战斗循环饿死 + hang、串行 gg/leave 失败；5/5 并发轮次复现。**明确「试过但被否决」** |
| `run3_parallel_switch_verification.md` | 只做启动器输入分支探针，**全过程没有任何一次 `run.py` 被执行**，未连 PS |
| `bat_launcher_recheck.md` | 九场景运行时探针 **9/9 通过**；修复 `:164` 一处预存缺陷（未转义 `>`） |
| `foulplay_review_2026-10-21.md` | 上次 P1 已修（模块级 `_EXECUTOR` 单例跨决策复用，消除跨进程整局 pickle 开销）；**S1/S2/S3/S4/S8 与 P3/P4 仍未改**；P3 `|immune|` 热路径同步 deepcopy+全量伤害计算仍在（`fp/battle/protocol.py:1618-1620`） |

---

## 6. 测试基线演进（**口径冲突，务必看清**）

> ⚠️ **首要限定**：下表数字**全部是 `reports/` 里已记录的历史运行结果，不是本次知识库取证实测**。整理本库时全程只读、未运行任何 pytest、未联网。**不得写成「当前测试结果」或「本次验证通过」。**

`E:\Tools\foul-play\reports\` 里记载的一套：

| 数字 | 精确来源 | 备注 |
|---|---|---|
| **813** | `multibattle_concurrency_plan.md:55` | 基线；另 `phase3_opponent_model_online_report.md:61`（S1 快照 27 局回放「已验证」） |
| **819 / 822 / 838** | `multibattle_concurrency_verification.md:8`(819 passed/0 failed)、`:113`(819→822)、`:265`(838 passed/0 failed) | |
| **854** | `concurrent_loss_fix_verification.md:28`（**854 passed, 1 warning in 8.48s**，原始命令输出）、`:33`（修复前基线 **838 passed**）、`:13`（结论摘要「854 passed / 0 failed = 838 + 16」） | `warning ≠ failed`，`:28` 与 `:13` 不矛盾 |
| **863** | `concurrent_e2e_verification_round2.md:6`（**亲手复跑全量 pytest**） | 同报告另记 6/6 端到端全过 |
| **870 → 887** | `mcts_ab_experiment_plan.md:10`（测试基线 **887 passed = 870 旧 + 17 新**）、`:223`（回归 **887 passed, 1 warning**，2026-10-18 夜跑 `.venv/Scripts/python.exe -m pytest`） | **「870 基线」的出处就是这两行** |

工作区 memory 另有 **792 → 893 → 904 → 905 → 934 → 947** 一套。**两套序列不一致**（不同分支 / 时点 / 子套件）。

**无配置文件可交叉佐证**：仓库根**无** `pyproject.toml` / `setup.cfg` / `pytest.ini` / `tox.ini`；`README.md`、`.github/workflows/ci.yml`（只是 `pytest tests`）、`Makefile`、`requirements-dev.txt`(`pytest==9.0.3`)、`conftest.py` 及 30 个测试文件里**都没有** passed 数字 ⇒ **这些基线数字的唯一文字来源就是 `reports/` 里的报告本身**。

**测试文件口径**：`tests/` 共 30 个 `test_*.py`；`test_concurrent_dispatcher_death.py` **13 个**用例、`test_opponent_model.py` **11 个**用例（口径 = 单文件 `^def test_` grep，不含类内方法 / 参数化展开）。
> 【口径差异·勿写成矛盾】`phase3_opponent_model_online_report.md:48` 设计表写 `tests/test_opponent_model.py`「**21+ 单测**」，那是**计划文档的设计预期值**；当前仓库顶层函数计数是 11，且 #84–#87 后续改动过测试。引用时须注明「计划预期」还是「当前实测」。

> ⇒ **以你实际跑出的 pytest 输出为准**。别引用任何文档数字当当前基线，更别用它们倒推提交顺序。

---

## 7. 复现工具与台账路径

| 工具 / 台账 | 用途 |
|---|---|
| `tools/ab_ladder_pipeline.py` | A/B 天梯流水线；复现命令 `python tools/ab_ladder_pipeline.py --arms B0 B1 B5 --sessions-per-arm 6 --start-index 19`（`mcts_ab_phase2_plan_20261019.md:44`） |
| `tools/mde_two_prop.py` | 两比例 MDE 计算 |
| `tools/ab_winrate_stats.py` | B4 胜率统计 |
| `tools/replay_lead_advisor.py` | L2 首发模型 36 场真实日志回放 |
| `E:\venvs\metamon-audit\ab_pipeline_20261019\LEDGER.md` | **A/B 权威累计记录**；`sessions\*\logs_frozen\`、`B4_tally_clean.log`、`B4_run.log` |
| `E:\venvs\metamon-audit\ab_audit\D1_analysis_output.txt` / `C_penalty_details.jsonl` | D-1 原始输出 |
| `E:\venvs\metamon-audit\ab_audit_d2\D2_analysis_output.txt` | D-2 原始输出 |
| `analysis/L1_final_stats_output.txt`、`analysis/long_game_study.py` | L-1 与长局专项 |
| `scan_logs.py` / `reconcile_counts.py` | 工作区只读校验脚本（见 `docs/12`） |

> ⚠️ `reconcile_counts.py` 的 docstring 写「rqid is the global primary key」，但 `04-data-model-dataset.md:319/:549` 已核定 **rqid 单独不唯一、主键应为 `(battle_tag, rqid)`**。以 04 章为准。

---

## 8. 已明确「报告未载」的项（**禁止推测补写**）

1. Kakuna **CPU 单局面推理延迟实测值** —— 多份报告明确「未测」；只给判据（≤50ms 整批可跑 / >500ms 形态 B 判死）。
2. 辅助形态 A 的最终「MCTS 盲区清单」产出 —— 00/02/06 均为方案/规格/待编码，DoD 未完成。
3. Phase 3 / 形态 C 的**实测收益数字** —— +15~30%、+几个 Elo 均为「边界预期，非承诺值，必须回放实测」。
4. NN 方案的 baseline 实测胜率 / 阶段 3 对拍结果 —— `NN方案计划.md` 是计划文档，未载实测。
5. 20 局报告的统计显著性 / CI / 对照组。
6. 摘标成功率 / staff 判定 bot 的成文标准。
7. 「56 胜 58 负」 —— 本地所有日志均无此记录，与日志冲突，暂不可信。
8. 对手建模的「1700+ 档人数 / 复现率 / 固定队比例」 —— **本批 `reports/` 里没有**；这些数字来自 `E:\venvs\metamon-audit\opponent_modeling\` 的产物与工作区 memory（见 `docs/04`），**两个来源不要混为一谈**。
