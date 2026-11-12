# 05 · Metamon 集成（L2 首发模型）

> 来源：工作区 memory `2026-10-13.md`、`2026-10-15.md`、`2026-10-16.md`、`2026-10-18.md`。
> 这是**目前唯一真正接进 bot 决策链的学习型模型**，其余（Kakuna 等）都在离线审计/研究阶段，见 `docs/07`。

---

## 1. 上游背景与选型判断【实证】

- `foul-play` 本身是 **NeurIPS 2025 PokéAgent Challenge Gen9 OU 冠军**（决赛 50–14，>80% GXE，峰值 rating 1879；随机对战 >90% GXE，峰值 2341）。
- `metamon` 最好的 gen9 模型是**第二**。
- 结论：**「在我们打的 Gen9 OU 环境里，搜索 > 学习」** ⇒ 不做整体替换，只把模型用在它比搜索强的位置。

**四层集成可行性结论**（2026-10-15 差距调研）：

| 层 | 方案 | 结论 |
|---|---|---|
| L1 | 数据层接入 | ❌ 不推荐 |
| **L2** | **首发层（team preview lead）** | ✅ **最推荐**：本地已有 `gen9ou_high_elo_v4/best_model.pt` 34.7MB，CPU 可跑；接口 `TeamPreviewModel.predict_lead(...)` → 6 类概率；落点 `fp/modes/base.py:308`；可用 39 场日志离线回放验证 |
| L3 | Kakuna 作 MCTS prior | 🔬 建议**先做离线反事实**，别直接接 |
| L4 | 整体替换 MCTS | ❌ 不推荐（gen9ou 上是倒退） |

**为什么首发层值得做**：FoulPlay 是 MCTS + poke_engine 精确模拟，gen9 team preview 已获对手全队，招式/道具按 Smogon 使用率做多假设加权投票（`fp/search/standard_battles.py:417 prepare_battles`）；**选首发 = MCTS 在 dummy active 上搜 3 秒**（`fp/modes/base.py:308 handle_team_preview`），**无跨回合记忆**。这是一个信息充分、但 MCTS 用得很浅的决策点，正好适合小模型。

调用链：`modes/standard_battle.py:108 handle_team_preview` → `modes/base.py:221` → `async_pick_move` → `format_decision`（`base.py:167`）。

## 2. 实现（2026-10-15 完成）【实证】

### `fp/lead_model/` — vendored 推理最小集
| 文件 | 说明 |
|---|---|
| `names.py` | 名字归一化 |
| `tokenizer.py` | 观测 → token |
| `model.py` | `TeamPreviewModel` |
| `DefaultObservationSpace-v1.json` | **2541 token**，md5 与上游一致（`cf75dac7...`） |
| `assets/gen9ou_high_elo_v4/best_model.pt` | **34,679,597 字节**（34.7MB） |

依赖只有 **torch(CPU) + einops**。⇒ 这是关键设计：**不往 foul-play 主 venv 装 metamon/torch**，而是把推理最小集复制进仓库，主 bot 保持轻量。

### `fp/lead_advisor.py` — `MetamonLeadAdvisor`
- **懒加载单例**，`torch.set_num_threads(1)`（实测推理 ~5ms/次）。
- **`_tokens_known` 词表预检**：任一 token 不在词表 → **整体回退**（防止 `nn.Embedding` 吃到 -1）。
- **显式路径**（CLI / env）**缺失即失败，不静默替换**。
- **全异常 containment**：任何异常返回 `None` → 回退 MCTS。
- argmax 确定性 OK。

### `fp/config.py`
- `--lead-policy {auto,mcts,metamon}`，**默认 `auto`**：advisor 可用就用，**MCTS 仍跑并双写日志**（保留对照）。
- `--lead-model-path`。
- ⚠️ **enum 坑**：`auto = auto()` 会重绑定类内的 `auto`，必须改用显式值。

### `fp/modes/base.py::handle_team_preview`
- advisor 有建议时 `/team` 首位用其 slot；
- `auto` 模式下 MCTS 照跑并记录双选（`MCTS lead choice for comparison`）；
- `last_selected_move` 始终按**最终选择**设置。

### 配套
- `tests/test_lead_advisor.py` **19 个测试**。
- `tools/replay_lead_advisor.py`：历史日志回放工具。

## 3. 验证证据【实证】

| 项 | 结果 |
|---|---|
| 全量 pytest | **792 passed**（原 773），零回归 |
| 真实日志回放 | **36 场**，advisor 全部作答、**0 回退**；与 MCTS **19 同意 / 17 不同意（53%）** |
| 分歧结构 | 高置信分歧（p≥0.87）集中在 **Samurott-Hisui 首发**；低置信分歧（p≤0.5）模型自己也没把握 |
| 推理耗时 | **~5ms/次**（`torch.set_num_threads(1)`） |
| `/team` 位数 | 矛盾解除，真实 `/team 412356` 吻合 |
| 对手敏感度 | Samurott 概率 0.484 → 0.571 → 0.648，Dragonite 0.095 → 0.178（随对手队伍变化） |
| MCTS 基线 | 8705 次采样：单次搜索中位 **201,000** 迭代、P90 **294,000**、最大 **1.92M**、均值 **258,056**（旧 awk 取错字段得「均值 220」的异常已修正） |

**上线后确认在工作**（10-16 12:20 运行健康检查）：
- 模型加载成功（`epoch 34, val_acc 0.5405, formats ['gen9ou'], argmax`）。
- 活动局：advisor 推 **darkrai（slot 4, p=0.495）** vs MCTS 想选 **greattusk（slot 6）** → auto 模式 advisor 胜出 → 实发 `/team 412356|2` 首位 4 ✓。
- 双选对照日志正常留痕。

## 4. ⚠️ 已知限制：上游词表缺陷【实证】

`DefaultObservationSpace-v1.json` **缺 Tapu 全系、Ferrothorn**（及 Poltchageist-Artisan 等新宠）。checkpoint 的 `token_emb` 正好 **2541 行**，证实词表未被裁剪。

⇒ 对阵含这些宝可梦时 advisor **回退 MCTS**（安全兜底，不会崩）。我方 2000 队池中仅 1 队含词表外宝可梦（`team_005645`）。

> Codex 若想修：扩词表需同步扩 `token_emb` 或做 unknown-token 映射，不能只改 json；且必须补回归测试验证不静默出错。

## 5. 与 MCTS 决策链的关系（10-18 全链代码审计）【实证】

`fp/search/main.py:319-369 find_best_move` 的真实流程：

```
对手假设采样
  → 每个假设独立 poke_engine.mcts(state, 3000ms, 0, 1)
  → select_move_from_mcts_results 聚合（Σ sample_chance × visit share）
  → POLICY_KEEP_FRACTION=0.75 过滤
  → REPEAT_PENALTY=0.5
  → random.choices 按权重采样（**不是 argmax**）
```

树搜索本体 = **poke_engine 0.0.48**（v0.0.48 tag 的 `mcts.rs` 与 main 分支逐字一致，已 WebFetch 双版本核对）：
- **UCB1 探索系数硬编码 2.0**，无先验；
- **rollout = 一步静态评估**（见 `docs/06` 方案 B 章节）；
- 伤害分支仅 `depth<2`；
- `threads>1` 走 `mcts_threaded.rs`（共享树 + DashMap + virtual loss）；
- `evaluate`（`genx/evaluate.rs`）纯手工线性权重，**无伤害预估 / 无速度线 / 无读招 / 无 PP**。

**关键现状**：`--lead-policy` 默认 `auto` ⇒ **team preview 首发实际由 metamon advisor 主导，MCTS 首发仅旁路记录**。改首发逻辑时别只看 MCTS 代码。

## 6. 测试基线演进对照表（**防止数字混用**）【实证】

所有数字都是**不同时点 / 不同子套件**的全量 pytest 结果，不是同一个基线：

| 数字 | 时点 / 事件 | 来源 |
|---|---|---|
| 773 | L2 之前 | 10-15 |
| **792** | **L2 实施完成**（+19 `test_lead_advisor`） | 10-15 |
| 813 | Phase 0 落地，+21 audit 测试，9.26 秒 | 10-16 |
| 822 | 并发 #1/#2 修复后 | 10-16/17 |
| 838 | punch-list #3–#6 修复后 | 10-17 |
| 854 | 490 事故修复后（+16 回归） | 10-17 |
| 863 | `BattleTagFilter` 日志路由后（+9） | 10-17 |
| 867 | try/finally 兜底后（+4） | 10-17 |
| 870 | `hide_next_battle` 后（+3） | 10-17 |
| 893 | 对手建模之前 | 10-22 前 |
| 894 | `--team-pool-source` 后（+7 `test_team_pool`） | 10-21 |
| 904 | Part A/B 落地（+10 `test_opponent_model`） | 10-22 |
| 905 | 注入污染修复（+1 `test_record_excludes_injected_moves`） | 10-22 |
| 934 | dispatcher 修复前基线 | 10-24 |
| **947** | **dispatcher 断线修复（+13）**，当前已知最新 | 10-24 |

> ⚠️ 894 出现在 10-21，而 893 记在 10-22 之前，两者顺序在不同文档里表述不完全一致；**以 git 历史和实际 `pytest` 输出为准**，别用本表倒推提交顺序。
>
> ⚠️ **口径冲突**：上表来自工作区 memory；`E:\Tools\foul-play\reports\` 里另有一套 813 / 819 / 822 / 838 / 854 / 863 / **887** 的数字（例如 `mcts_ab_experiment_plan.md:223` 记「887 passed, 1 warning，870 基线 + 17 新增」）。两套序列可能是不同分支 / 子套件 / 时点，**不可混用**。仓库内 `README.md`、`.github/workflows/ci.yml`、``pytest.ini`` 均无基线数字 ⇒ 以实际运行输出为准，详见 `docs/02` 第 6 节。

## 7. 复用建议

1. 想改首发：先读 `fp/lead_advisor.py` 的 containment 设计，保持「任何异常都回退 MCTS」不变。
2. 想换模型：注意 `predict_lead` 输出 6 类概率 + slot 映射，以及 `DefaultObservationSpace-v1.json` 词表约束。
3. 想做 L3（Kakuna 作 prior）：**先做离线反事实**，别直接接进线上决策，见 `docs/07`。
4. 别把 metamon/torch 装进 foul-play 主 venv —— 保持 vendored 最小集，这是刻意的隔离。
