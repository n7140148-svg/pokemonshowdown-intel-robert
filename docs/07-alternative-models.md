# 07 · 未采用 / 被否决 / 研究中的机器人模型与方案

> **本文定位**：只写**没有成为 bot 默认决策链**的模型、训练范式与研究方案，回答「有哪些别的机器人模型、研究到哪一步、为什么没用」。
> 已落地并实际生效的见 `docs/05`（L2 Metamon 首发 advisor）、`docs/04`（对手建模 Part A/B）、`docs/03`（全部 CLI 开关）；已跑完并有数字的成果见 `docs/06`。
> **一手来源**：`E:\Tools\foul-play\reports\*.md`、`E:\Tools\src\metamon\metamon\rl\pretrained.py`、`E:\Tools\foul-play\fp\config.py`、`E:\Tools\foul-play\reports\phase3_opponent_model_online_report.md`。本文所有行号均指这些本地文件。

---

## 0. 状态分类（本文每个条目归入且只归入一类）

| 状态 | 含义 | Codex 能不能据此改代码 |
|---|---|---|
| 【已采用·生效】 | 已进入 bot 默认决策链 | 能，但先看 `docs/03` 默认 OFF 红线 |
| 【离线评估·未采用】 | 跑过实验、有数字，结论是不接入 | **不能**据此改默认配置；结论是「不接入」 |
| 【规划·未落地】 | 只有方案 / 规格 / 待编码，无实测 | **不能**，先补实测 |
| 【明确否决】 | 预登记 A/B 或端到端验证判负 | **不能**，别重复跑 |
| 【上游存在·未评估】 | metamon 源码里有，但本项目报告里**一次都没评估过** | 不能；这是候选池，不是结论 |
| 【报告未载】 | 本地所有报告都没有该数字 / 结论 | **禁止推测补写** |

> ⚠️ 【更正·贯穿全文】本项目里 **「不替换 MCTS」不是架构禁令**。报告原文给的是**实测梯度 + 可行性判断**（上游强度低于本 bot、CPU 推理延迟未测、替换会丢掉精确模拟与对手采样），并在 D-2 被实证反证过一次；**没有任何一份报告写「原理上不可能替换」**。`metamon_opponent_modeling_longterm_value_analysis.md:15` 那句「learned prior 只能辅助、不能替换」是**分析性判断**，不是实验结论，引用时必须区分。

---

## 1. 总表：所有备选方案一眼看完

| 模型 / 方案 | 状态 | 有实测数字？ | 没进 bot 的原因 | 一手来源 |
|---|---|---|---|---|
| **L2 TeamPreviewModel**（`gen9ou_high_elo_v4/best_model.pt`） | 【已采用·生效】 | ✅ 792 passed / 36 场回放 / val_acc 0.5405 | —— 唯一进决策链的学习型模型 | `docs/05` |
| **Kakuna**（142M, epoch34） | 【离线评估·未采用】 | ✅ D-1 166 局 / D-2 182 局 | 只作离线参照策略；「更像 Kakuna ≠ 更能赢」 | `docs/06` §2、`d2_kakuna_audit_report_20261020.md` |
| **Kakuna 作 MCTS prior（L3）** | 【规划·未落地】 | ❌ | 建议「先做离线反事实，别直接接」 | `docs/05` §1 四层表 |
| **整体替换 MCTS（L4）** | 【离线评估·未采用】 | ⚠️ 无对拍胜率 | gen9ou 上是倒退 + 延迟 + 丢精确模拟 | `metamon_opponent_modeling_longterm_value_analysis.md:13`、`:95-99` |
| **对手建模 辅助形态 A**（Metamon 使用率交叉校验） | 【规划·未落地】 | ❌ | 「两者同源，价值有限」，性价比低 | 同文件 `:48-52` |
| **对手建模 辅助形态 B**（Kakuna 标注对手身份判断） | 【规划·未落地】 | ❌ | Kakuna 输出的是动作不是队伍假设；环境未就绪 | 同文件 `:54-58` |
| **对手建模 辅助形态 C**（foul-play 原生重复对手建模） | 【已采用·但默认 OFF】 | ✅ 35 精英 / 3309 配置行 | 推荐路线，落地为 Part A/B 两模块；**默认 OFF 是研究旋钮** | 同文件 `:60-67`、`docs/04` |
| **长程价值 辅助形态 A**（Kakuna 离线审计历史决策） | 【规划·未落地】 | ❌ DoD 未完成 | 「先决条件是环境」，产出「MCTS 盲区清单」**未交付** | 同文件 `:103-107`、`:145` |
| **长程价值 辅助形态 B**（运行时候选剪枝） | 【规划·未落地】 | ❌ | 剪掉的动作永无翻身机会；Rust 侧 MCTS 不接受候选列表 | 同文件 `:109-113` |
| **长程价值 辅助形态 C**（蒸馏轻量局面评估） | 【规划·未落地】 | ❌ | 需改 Rust 侧叶子评估，工程量大 | 同文件 `:115-119` |
| **NN 方案 A**（现成模型做「第二意见」/ 对拍） | 【规划·未落地】 | ❌ | 目的是「学习 NN 怎么想」，不是变强；收益不确定 | `NN方案计划.md:60-73` |
| **NN 方案 B**（自建轻量 NN） | 【明确否决】 | ❌ | 200M CPU 训不了、60 局日志监督信号不足 | `NN方案计划.md:75-79` |
| **NN 方案 C**（腾磁盘 + 建 E 盘 venv） | 【已执行·基建】 | ✅ 环境已建 | 是前置不是模型；成果见 `docs/01` | `NN方案计划.md:81-88` |
| **Kadabra1-4 / Alakazam / Superkazam / Minikazam / Abra / V2A× / Tauros×** | 【上游存在·未评估】 | ❌ | 源码有 50 个类，本项目报告**只评估过 Kakuna** | `metamon/rl/pretrained.py`（见 §3） |
| **SyntheticRLV2 / SmallIL / SmallRL / Medium* / Large*** | 【上游存在·未评估】 | ❌ | 计划文档里列过参数表，但没有本地推理 / 对拍结果 | `NN方案计划.md:26-34` |

---

## 2. 决策背景：四次「不替换」判断的原文

### 2.1 四层集成可行性（`docs/05` §1，2026-10-15）

| 层 | 方案 | 结论 |
|---|---|---|
| L1 | 数据层接入 | ❌ 不推荐 |
| **L2** | **首发层（team preview lead）** | ✅ 最推荐 → **已落地** |
| L3 | Kakuna 作 MCTS prior | 🔬 建议先做离线反事实，别直接接 |
| L4 | 整体替换 MCTS | ❌ 不推荐（gen9ou 上是倒退） |

### 2.2 「不替换」的四条具体理由（**原文，不是推断**）

`metamon_opponent_modeling_longterm_value_analysis.md:95-99` 的替换方案缺点栏：

1. 公网 Gen9OU 强度排在 foul-play 之后（上游 README:282），替换大概率掉分；
2. Kakuna 权重 1.6GB，CPU 推理延迟未测，若 >50ms 反而比 MCTS 慢；
3. 丢掉 foul-play 的精确模拟与对手采样 —— 这两项是 foul-play 排名的根基；
4. 排位赛版本变动时 IL 策略会静默退化，而 MCTS 靠 poke_engine 规则自动正确。

`NN方案计划.md:26-45` 的硬约束补充：本机**无 NVIDIA GPU**（`nvidia-smi` 不存在）、内存 16.9GB（当时可用仅 4.0GB）、C 盘剩 2.1GB；metamon README 原文「Pretrained models can run without research GPUs」⇒ **推理可行、训练不现实**；安装要求 Linux + Python 3.10+，而本机 Windows 11 是「未验证的风险点」。成绩对照：FoulPlay >80% GXE 冠军，Metamon-Transformer-200M「接近 Top 500 人类」，论文自评 "still below human level"。

### 2.3 ⭐ 实证反证：「更像 Kakuna」≠「更能赢」

`d2_kakuna_audit_report_20261020.md`（L-1 数据，182 局 / 5341 决策点）：

| 臂 | 与 Kakuna 一致率 | 同批胜率 |
|---|---|---|
| B0 | 41.1% | 50.0% |
| B1 | 37.8% | 48.3% |
| **B5**（threads=2, 4.53× visits） | **47.0%（+5.9pp，最高）** | **53.7%（+3.7pp, p=0.7239，未过保留规则）** |

⇒ 4.5× 算力**确实**把决策推向 Kakuna（可检测），但胜率**没有可检测变化**。证伪了「逼近参照策略就能变强」这条推理链。Kakuna 是合理参照策略，但不是这个天梯池里的致胜策略；MCTS 与 Kakuna 的分歧大部分是**风格差异而非错误**（详见 `docs/06` §2.3）。

### 2.4 L2 成立条件表（为什么只有首发层能做）

`metamon_opponent_modeling_longterm_value_analysis.md:127-135`，三个条件：子问题边界清晰无搜索结构 / 现成实现明显更弱或没有 / 失败可无害回退。首发层三条全 ✅（12 输入→6 选 1、MCTS 在 team preview 搜得很浅、异常回退 MCTS 零损失、~5ms）；对手建模与长程价值各缺 2–3 条，尤其「剪枝 / 蒸馏的错误不可无害回退」。

> 共性风险（`:137`）：**上游词表 / 表示口径缺陷会静默缩小适用范围**。L2 的 `DefaultObservationSpace-v1.json` 缺 Tapu 全系与 Ferrothorn，checkpoint `token_emb` 正好 2541 行证明不是复制损坏 ⇒ 任何接入 Metamon 组件的方案都必须带「未知即回退」预检（`fp/lead_advisor.py::_tokens_known` 就是为此写的）。

---

## 3. ⭐ 上游模型谱系：源码里有 50 个类，本项目只评估过 1 个

【源码·实证】`E:\Tools\src\metamon\metamon\rl\pretrained.py` 共 **50 个 `class`**，其中 49 个是 `PretrainedModel` 子类。按族分组（`grep -nE '^class '` 原文行号）：

| 族 | 类（行号） | 本项目评估情况 |
|---|---|---|
| 早期小/中/大 IL-RL | `SmallIL:372`、`SmallILFA:385`、`SmallRL:398`、`SmallRL_ExtremeFilter:411`、`SmallRL_BinaryFilter:427`、`SmallRL_Aug:440`、`SmallRL_MaxQ:453`、`MediumIL:466`、`MediumRL:479`、`MediumRL_Aug:492`、`MediumRL_MaxQ:505`、`LargeRL:518`、`LargeIL:531` | ❌ 未评估；仅 `NN方案计划.md:26-34` 列过参数表 |
| SyntheticRL | `SyntheticRLV0:544`、`V1:557`、`V1_SelfPlay:570`、`V1_PlusPlus:583`、`V2:596` | ❌ 未评估；计划文档称「Gen9 OU 可用的只有 SyntheticRLV2」 |
| Gen9 beta / 本地模型 | `SmallRLGen9Beta:620`、`LocalPretrainedModel:293`、`LocalFinetunedModel:319` | ❌ 未评估 |
| **Abra 系（进化链）** | `Minikazam:653`、`Abra:682`、`Kadabra:713`、`Kadabra2:738`、`Kadabra3:770`、`Kadabra4:798`、`Alakazam:831`、`Superkazam:863` | ❌ 未评估；仅 `:13` 提过「Kadabra3 公网 Gen9OU 排名仅次于 foul-play」 |
| **Kakuna 系** | `Kakuna:893`、`KakunaEnsemble:1314`、`TaurosV0:1293`、`TaurosEnsemble:1469` | ✅ **仅 `Kakuna` 被离线评估**（D-1/D-2） |
| V2A 系列（消融/变体） | `V2A:946`、`Seed2:967`、`Beta01:988`、`Beta1:1009`、`Beta3:1030`、`Beta10:1051`、`BNBeta3HLGauss:1072`、`…Vanilla:1093`、`BNBeta3:1114`、`MixedGens:1135`、`NoMG:1157`、`IL:1180`、`GroupedV2ISFilter:1203`、`GroupedV2Patched:1224`、`GroupedV2ArchAblation:1245`、`GroupedV2DataAblation:1272` | ❌ 未评估 |

**关键类参数【源码·`grep -E 'model_name=|default_checkpoint='`】**：

| 类 | 行 | `model_name` | `default_checkpoint` | 备注 |
|---|---|---|---|---|
| `Kadabra3` | `:770` | `"kadabra3"` | `20` | 报告里被点名「Gen9OU 排名仅次于 foul-play」，**但从未本地评估** |
| `Alakazam` | `:831` | `"alakazam"` | `8` | 未评估 |
| `Kakuna` | `:893` | `"kakuna"` | **34** | docstring `:895` 明确是 **Superkazam 的微调版**（提高温度采集的自对弈 **+7.8M 局**）；`:897-905` 的 `gen9ou: ~71%` 是**对人类、`competitive` TeamSet 的估计 GXE**，不是胜率。精确参数 **142,832,563**；对应 `policy_epoch_34.pt` |
| `KakunaEnsemble` | `:1314` | `"Kakuna"`（大写）×3 | — | 集成类，未评估 |
| `TaurosEnsemble` | `:1469` | `"TaurosV0"` ×3 | — | 继承 KakunaEnsemble，未评估 |
| `PretrainedModel` 基类 | `:153` | `HF_REPO_ID = "jakegrigsby/metamon"` | `:167` 基类默认 `40` | `:214-215` 用 `huggingface_hub.hf_hub_download` |

> ⚠️ **口径演变【更正】**：`NN方案计划.md:26-34`（2026-10-13）只列了 7 行模型清单，并写「Gen 9 OU 可用的只有 SyntheticRLV2（其余是 gen1-4）」。这是**当时的简版认知**，与 metamon 1.6.0 源码里 50 个类（含 Kadabra3 / Alakazam / Kakuna 等 gen9 模型）**不一致**。以源码为准；不要把计划文档那句「只有 SyntheticRLV2」当成事实。
>
> ✅ **【口径已解释·原标待核】「1.6GB」**：`reports\metamon-audit-plan\03-kakuna-env-alignment.md:269` 原文「权重 **3 个 policy 文件**各 **571,573,427 / 571,581,627 / 571,581,627 字节**（`ls -la` 实测，**合计约 1.6GB**）」。⇒ 1.6GB 是**三个 policy 权重文件的合计**，本地 `policy_epoch_34.pt` 只是其中一个（571,581,627 字节）。`d1` 报告自身未写口径，数量级由此得到解释；**两者不是同一个数字，不要合并引用**。详见 `docs/01` §5.1。

**下载入口**：`docs/01` §5.1 已给出核实过的完整直链（`download_kakuna.py`）、权威 sha256（`7e7474d5…26c`）、上游 repo id / checkpoint 对应方法，以及加载 Kakuna 的 5 个已知坑（`METAMON_CACHE_DIR` / `LocalPretrainedModel` / gin override / `policy.eval()` / 无需 flash-attn）。**其它 48 个模型的权重本项目没有下载过**，也没有核对过文件名 ⇒【报告未载】，不要凭命名规律拼 URL。

---

## 4. 训练范式：IL / RL / 回放数据 / amago（**offline RL 报告未载**）

【源码】`pretrained.py` 的类名体现范式分野：`*IL` vs `*RL` vs `SyntheticRL*`（含自对弈合成数据）vs `V2A*`（消融变体）。【报告原文】`NN方案计划.md:36-39` 的参数规模：SmallIL/SmallRL 15M、MediumIL/RL 50M、LargeIL/RL 200M、SyntheticRLV2 200M（100 万人类 + 400 万自对弈）。报告里只用 `IL/RL` **缩写 + 「人类回放训练」**（`d1_kakuna_audit_report_20261019.md:113`），**没有**给出 IL 全称定义；把 `IL` 展开成「imitation learning / 模仿学习」属【推断】，不要写成报告结论。

> ⚠️ **【报告未载】不要把本项目写成「评估过 offline RL」**：对 `E:\Tools\foul-play\reports\*.md` 全量只读检索 `offline RL / 离线强化 / 离线 RL` **零命中**。报告里的「离线」全部是**工程口径**（离线审计 / 离线标注 / 离线裁判 / 纯离线），不是训练范式。`metamon_opponent_modeling_longterm_value_analysis.md:88` 提到的 shaped reward 是 RL 训练细节（`AggressiveShapedReward`），也不等于 offline RL 项目。

> ⚠️ **四组数据集规模数字不可合并 / 相加**（来源与口径不同）：① `00-辅助形态A实施方案.md:29`「160 万局人类回放 + 780 万局自对弈」；② `03-kakuna-env-alignment.md:65`「+7.8M 局自对弈」（Kakuna 微调增量）；③ `NN方案计划.md:40`「100 万人类 + 400 万自对弈」（SyntheticRLV2）；④ `05-statistical-validation.md:36`「5M+ 人类回放轨迹 + 20M+ 自对局」（上游 README 总口径）。报告未声明它们可互算。

【实证·环境】`E:\Tools\src\amago`（`amago 3.4.0`，`pyproject.toml:54-55` 仓库 `UT-Austin-RPL/amago`）是 metamon 的 **RL 库**（`NN方案计划.md:46`「metamon → amago → PyTorch」；`03-kakuna-env-alignment.md:14`「硬必需，ckpts 配置头 5 行直接 `import amago.*`」，`:76` 权重是 amago `Experiment` 的序列化产物、**不是裸 `state_dict`**）。`E:\venvs\metamon-audit\venv` 里 `amago-3.4.0.dist-info` 存在。**但本项目从未训练任何模型**，只做预训练权重的离线推理 / 审计。

> 【报告未载】本项目没有 IL vs RL 的对照实验、没有训练 loss 曲线、没有 amago 的配置调优记录。`NN方案计划.md:75-79` 明确把「自建轻量 NN」判为现阶段排除（200M CPU 训不了、60 局日志监督信号不足）。

---

## 5. 辅助形态 A / B / C：**两套分类，别混**

> 最容易出错的一点：`metamon_opponent_modeling_longterm_value_analysis.md` 里**有两个研究方向，各自都有形态 A/B/C**，名字相同、内容完全不同。引用时必须写「对手建模形态 X」或「长程价值形态 X」。

### 5.1 方向一：对手建模（`:19-67`）

| 形态 | 做法（原文摘要） | 报告给的可行性 | 状态 |
|---|---|---|---|
| **A** `:48-52` | Metamon 使用率 JSON 与 Smogon 统计逐宝可梦对比，差异超阈值告警 | 高，但**性价比低**（两者同源，预期差异很小） | 【规划·未落地】 |
| **B** `:54-58` | Kakuna 标注 foul-play 的「对手身份判断」，统计推断系统性偏差 | 中，依赖环境先行；Kakuna 输出的是动作不是队伍假设 | 【规划·未落地】 |
| **C（推荐）** `:60-67` | **foul-play 原生重复对手建模，根本不需要 Metamon**：按对手名聚合历史日志，统计该对手的队伍 / 招式 / 首发习惯，作为私先验覆盖或加权 Smogon 群体先验 | **高**，「唯一高价值 × 低难度」 | 【已采用·默认 OFF】→ 落地为 `docs/04` Part A/B |

形态 C 的证据（`:62`）：`logs/` 文件名即对手名（`fp/modes/base.py:172-174` 的 `do_rollover("{}_{}.log".format(battle_tag, opponent_name))`）；当时日志里 `thingsyoudo2me` 出现 9 个文件、`lupiiit` 2 个、`Captain_slowbro2` 2 个 ⇒ 同一对手重复出现是常态。已知缺点：冷启动（首局回退群体先验）、9 局样本需平滑防过拟合。

> **注意**：形态 C 的规划是「私先验覆盖 / 加权 Smogon 群体先验」，**实际落地被大幅简化**为「把已观测招式的真子集 plant 到 reserve pokemon + 独立 runtime 日志」，并没有实现 duck-type 替换 `SmogonSets`。见 §6 漂移链。

### 5.2 方向二：长程价值（`:71-121`）

定义（`:75`）：「一个决策的收益不在当期体现，而在若干回合后兑现」。

| 形态 | 做法（原文摘要） | 可行性判据 | 状态 |
|---|---|---|---|
| **A** `:103-107` | **离线标注**：用 Kakuna 审计 foul-play 历史决策，对每个决策点记录双方选择、按最终胜负回标分歧，产出「**MCTS 盲区清单**」 | 中高；「先决条件是环境」（任务 #11：装 amago + 修 venv），零运行时风险 | 【规划·未落地】**DoD 未完成，盲区清单未交付**（`docs/06` §8 第 2 项） |
| **B** `:109-113` | **运行时候选剪枝**：Kakuna 前向取 top-k，MCTS 只搜这 k 个 | 低；剪掉的动作永无翻身机会、Rust 侧 MCTS 不接受候选列表、CPU 前向延迟未测。重估条件：CPU 前向 <10ms 且 top-k 包含率 >99% | 【规划·未落地】 |
| **C** `:115-119` | **蒸馏**：Kakuna 标注大量局面 → 训练轻量局面评估 → 嵌入 MCTS 叶子评估 | 低（工程口径）；需改 Rust 侧或改 Python 侧模拟终止条件，属 L3+ | 【规划·未落地】 |

结论原文（`:121`）：**「长程价值方向，先做形态 A（离线审计）；B/C 等 A 的结论再谈。」**
下一步排序（`:143-147`）：① 形态 C 重复对手私先验 → ② 环境基建把 Kakuna 跑起来（测 CPU 单局面推理延迟，这个数字决定形态 B 是否值得重估）→ ③ 形态 A 离线审计产出盲区清单 → ④ 观察项：L2 auto 模式积累 100+ 局双选记录后统计 advisor 高置信时 MCTS 是否常错。

> ⚠️ 【报告未载】Kakuna **CPU 单局面推理延迟实测值**始终没测（多份报告明确「未测」），只给判据：≤50ms 整批可跑 / >500ms 形态 B 判死。这是形态 A/B 长期停在规划态的直接原因之一。

---

## 6. ⭐ Phase 3 规划 → 落地的**四处同源漂移**（统一口径，勿写成两个矛盾）

> `reports\phase3_opponent_model_online_report.md`（2026-10-16 规划）描述的是**一套未实现的方案**；当前仓库里的是**另一套更轻的实现**。以下四处差异**同源**：都是「规划稿的 ambitious 设计被简化落地」，不是四条独立 bug，也不构成「文档互相矛盾」。

| # | 规划稿原文 | 当前实测 | 证据 |
|---|---|---|---|
| 1 | `fp/opponent_model/` **四文件包**：`store.py` / `prior.py` / `observe.py` / `cluster.py`（`:43-46`） | 实际是**两个单文件模块**：`fp/opponent_model.py` + `fp/opponent_recorder.py` | `docs/02`；`grep` 无 `fp/opponent_model/` 目录 |
| 2 | **1 个** feature flag：`FoulPlayConfig.enable_opponent_model` 默认关（`:37`） | **2 个** flag：`--opponent-model-enabled` / `--opponent-record-enabled`，分别管注入与自生长 | `fp/config.py:213`、`:215`（类字段）；`docs/03` §2.7 |
| 3 | 字段名 `enable_opponent_model`（`:37`） | 字段名 `opponent_model_enabled` / `opponent_record_enabled` | `fp/config.py:548`、`:550` 赋值 |
| 4 | `tests/test_opponent_model.py`「**21+ 单测**（合成夹具，零 I/O）」（`:48`） | **11 个**顶层 `def test_` | `grep -n '^def test_'` 行号：65 / 71 / 77 / 97 / 108 / 134 / 146 / 160 / 194 / 211 / 234 |

**规划稿里没落地的部件**（`:37`、`:43-50`）：指数衰减 / 新鲜度、有界 store + LRU 淘汰、`OpponentPrior` duck-type 替换 `SmogonSets` 的两个方法（`get_all_remaining_trait_combinations` / `all_pkmn_counts`）、archetype 轻量聚类、`prepare_battles` / `sample_pokemon` 透传。接入点行号也从规划稿的 `base.py:166` 漂到当前 `fp/modes/base.py:227`（`start_battle_common`，这是代码演进导致的行号变化，**不是规划写错**）。

> 【口径差异·勿写成矛盾】「21+」是**计划文档的设计预期值**，11 是**当前实测顶层用例数**，且 `#84–#87` 后续改动过测试。当前文件无类、无 parametrize、无 conftest 钩子，结构上不存在收集数远大于 11 的机制。引用时务必注明是「计划预期」还是「当前实测」。

---

## 7. 被否决 / 跳过的实验臂（**别重复跑**，数字见 `docs/06` §1）

| 臂 | 裁定 | 一句话理由 |
|---|---|---|
| **B1** `--final-choice argmax` | 【明确否决】默认保持 `sample` | phase-1 +14.6pp 看着有效，L-1 复现失败（48.3% vs B0 50.0%）；预登记规则未过 |
| **B5** `--search-threads 2` | 【明确否决】默认保持 `1` | 4.53× visits 确认生效，但胜率 +3.7pp / p=0.7239 无 detectable 效果 |
| **B2** `--repeat-penalty 0` | 【明确否决】不跑天梯 | D-1 Phase C：216 个 penalty 覆盖 top1 中 Kakuna 认同覆盖后选择 88(40.7%)、认同被覆盖 top1 仅 9(4.2%) ⇒ penalty 在做正确的事 |
| **B3** `--deep-hypotheses` | 【明确否决】跳过 | 早期输局 37.9% vs 胜局 37.9% 完全相等，gate 失败；D-2 二次 null |
| **B4** `--lead-policy mcts` | 【明确否决】维持 B0 | −6.7pp / p=0.5832 ⇒ `DIRECTIONAL_ONLY`；且反向也不显著 ⇒ **也不声称「advisor 有价值」** |
| **长局终盘专项** | 【明确否决】不单设臂 | 长局 29-40 / 41+ 回合 Kakuna 一致率输胜完全持平、无终盘塌陷 ⇒ 「长局败局」正名为「**磨败的中盘局**」 |
| **`/modjoin +` 观战锁** | 【明确否决】端到端 FAIL | 锁门触发房间 rename → 并发战斗循环饿死 + hang，5/5 轮次复现；现用无条件 `/hidenext`（`docs/03` §5） |

### 7.1 两条容易压扁的口径（写新方案时保持这个粒度）

**2-A：A/B 实验不是「纯 MCTS 编排层」实验。** `reports\mcts_ab_experiment_plan.md:27` 明确 `--lead-policy mcts` 是**零代码对比项**（消融 advisor），era B 的 team-preview 首发实际由 **Metamon advisor 主导**，MCTS 首发只旁路记录。`fp/config.py:273` 的字面默认值是 **`auto`（不是 `metamon`）**：advisor 可用就用、否则回退 MCTS，两者都记录。所以 B0/B1/B5 对比的是**编排层参数**，但基线本身已含学习模型，不要把「B0」描述成「纯 MCTS bot」。

**2-F：并发不是一个布尔开关，是四个非独立子旋钮 + 一个外部交互。** `docs/03` §2.8（行号 `fp/config.py:447/455/461/469`）：`--concurrent-battles`（主开关，默认 False）、`--max-concurrent-battles`（默认 2）、`--no-concurrent-core-cap`（默认 True，即封顶生效）、`--concurrent-core-budget`（默认 0=自动）。外加与 `--search-threads` 的交互：`config.py:170-178` 记本机 **6 物理核**，2 局 × 2 线程时每次搜索 MCTS 迭代数**下降 22.3%**；`multibattle_concurrency_verification.md:223` 手工分核把损失收窄到 **−5.1%** ⇒ 同机多进程必须显式设 `--concurrent-core-budget`。**不要写成「并发全部默认 OFF」或「全部默认 ON」**。

---

## 8. 明确「报告未载」的项（**禁止推测补写**）

1. Kakuna **CPU 单局面推理延迟实测值**（只给判据，未测）。
2. 长程价值形态 A 的最终「**MCTS 盲区清单**」产出 —— 00/02/06 均为方案 / 规格 / 待编码，DoD 未完成。
3. Phase 3 / 形态 C 的**实测收益数字** —— 「+15~30%」「+几个 Elo」都是「边界预期，非承诺值，必须回放实测」。
4. NN 方案的 **baseline 实测胜率 / 阶段 3 对拍结果** —— `NN方案计划.md` 是计划文档，阶段 2 的 `python -m metamon.rl.evaluate ... --agent Kakuna --total_battles 100` 没有落地输出记录。
5. 除 Kakuna 外 **48 个上游模型**的权重文件名 / 参数 / 本地推理结果。
6. IL vs RL 的对照实验、训练配置、amago 调优记录。
7. 任何「整体替换 MCTS」的**对拍胜率**（只有可行性分析与上游排名引用，没有本地 head-to-head 数字）。
8. 「56 胜 58 负」、摘标成功率、staff 判定 bot 的成文标准（见 `docs/06` §8 / `docs/11`）。
9. **各模型「未采用」的逐字理由** —— 报告**没有**逐条写「不用 Kadabra3 / Alakazam / SyntheticRLV2 / SmallIL 是因为 X」。可核的只有 `NN方案计划.md:42`「Gen 9 OU 可用的只有 SyntheticRLV2（其余是 gen1-4）」、`:105`「先试 Kakuna 142M」、`:156`「模型选择待确认」、`:79`「方案 B 自建轻量 NN：现阶段排除」。本文 §1 总表里其余「没进 bot 的原因」是**基于缺席的推断**，不是报告结论。
10. **「形态 C（对手建模）/ 形态 A（长程价值）已被实际落地」** —— 报告只写「结论：做形态 C / 先做形态 A」与配套计划；判定为「已落地」依据的是现存文件（`fp/opponent_model.py`、`fp/opponent_recorder.py`、`metamon-audit-plan/00-辅助形态A实施方案.md`、D-1/D-2 报告）与任务状态，属【推断】。
11. **offline RL 作为训练范式 / 数据集规模的统一口径** —— 见 §4，报告未载，不可合并四组数字。

---

## 9. Codex 若要选新模型 / 新方案：决策检查单

1. 先看 §1 总表：这个方案是不是已经落在【明确否决】或【规划·未落地】里？否决过的别重跑。
2. 确认你引用的是**哪一套形态 A/B/C**（对手建模 or 长程价值）。
3. 若是上游模型：先确认权重能否拿到（`docs/01` §5.1 只核实过 Kakuna），再补 CPU 延迟实测 —— 这是形态 A/B 的共同前置。
4. 任何「改进」上真天梯前，走 `docs/06` §0 的**预登记保留规则**（点估计高于 B0 且 Wilson CI 明显不重叠 / Fisher p<0.05 且差 ≥ MDE）；过规则前开关保持默认 OFF。n≈60/臂时 MDE≈23–24pp，resolve 10pp 需 ~600 局/臂，**别用小规模单期结果下结论**。
5. 保持「**未知即回退**」：词表 / 表示口径缺陷会静默缩小适用范围（L2 的教训）。
6. 接入前重读 `AGENTS.md` 红线：默认 OFF 不擅自打开、不连真天梯做实验、端到端走本地 PS :8000。
7. 若结论与本文冲突：在 `docs/11` 记【更正】，**不删旧文**，写清日期 + 来源（文件路径:行号 或命令原始输出）。
