# 01 · 环境、工具链与重建 / 下载方法

> 全部路径、版本、URL 均为**本地只读核实**（文件原文 / `git remote -v` / `pyvenv.cfg` / `package.json`）。
> **凡本地文件里查不到的下载地址或版本，一律标【待确认】，绝不凭记忆补 URL。**
> ⚠️ 本节所有命令是**重建说明**，不是「已执行」；整理本知识库时未安装任何包、未创建 venv、未启动服务器、未跑推理。

---

## 1. 环境总表（已核实存在）

| 组件 | 绝对路径 | 版本 / 状态 |
|---|---|---|
| 主工程 | `E:\Tools\foul-play` | git `origin` = `https://github.com/pmariglia/foul-play.git`（`git remote -v`） |
| 主 venv | `E:\Tools\foul-play\.venv` | `pyvenv.cfg`：**Python 3.13.14**，home = `C:\Users\abc\.workbuddy\binaries\python\versions\3.13.12`，`include-system-site-packages = false` |
| 审计 venv | `E:\venvs\metamon-audit\venv` | `pyvenv.cfg`：**Python 3.13.14**；`site-packages` 里有 **`torch-2.14.1.dist-info`**、**`metamon-1.6.0.dist-info`**、**`amago-3.4.0.dist-info`** |
| metamon 源码 | `E:\Tools\src\metamon` | `pyproject.toml`：`name=metamon`，**version 1.6.0**，`requires-python>=3.10`，MIT；**无 git remote**（`git remote -v` 空） |
| amago 源码 | `E:\Tools\src\amago` | `pyproject.toml`：`name=amago`，**version 3.4.0**，MIT；**无 git remote** |
| 本地 PS 服 | `E:\Tools\pokemon-showdown` | `package.json`：`name=pokemon-showdown`，**version 0.11.11**；`origin` = `https://github.com/smogon/pokemon-showdown.git` |
| 对手建模库 | `E:\venvs\metamon-audit\opponent_modeling\` | 见下 §4 |
| 托管运行时 | `C:\Users\abc\.workbuddy\binaries\python\versions\` | `3.13.12`、`current` |
| 托管 Node | `C:\Users\abc\.workbuddy\binaries\node\versions\` | `22.22.2-6`、`24.22.1`、`current`、`node24.zip` |
| WorkBuddy skills | `C:\Users\abc\.workbuddy\skills\` | 见 `docs/10` |
| 工作区 | `C:\Users\abc\WorkBuddy\2026-10-10-14-15-51\` | `reports/`、`frozen_logs/`、`tmp/`、`scan_logs.py`、`reconcile_counts.py` 等 |

> **隔离铁律**：foul-play 的 `.venv` **没有 metamon / torch**（`requirements.txt` 里没有）。所有需要 metamon 的推理走 `E:\venvs\metamon-audit\venv`。L2 能进 bot 是因为把推理最小集 **vendored** 进 `fp/lead_model/`，不是装依赖。

## 2. 依赖声明（原文，可直接照此重建）

### `E:\Tools\foul-play\requirements.txt`
```
requests==2.33.0
websockets==14.1
python-dateutil==2.8.0
poke-engine==0.0.48 --config-settings="build-args=--features poke-engine/terastallization --no-default-features"
```
> ⚠️ `poke-engine` 那行的 `--config-settings` **不是注释**，是安装参数（`--features poke-engine/terastallization --no-default-features`），用于启用太晶化特性。照抄整行，别只取版本号。

### `E:\Tools\foul-play\requirements-dev.txt`
```
-r requirements.txt
ruff==0.6.5
pytest==9.0.3
```

> ⚠️ **Rust 是构建前置条件【源码·`E:\Tools\foul-play\README.md`】**：`:30` `Note: Requires Rust to be installed on your machine to build the engine.`；`:79` `The engine must be built from source if installing locally so you must have rust installed on your machine.`；`:95` `make poke_engine GEN=<generation>`（`:100` 示例 `make poke_engine GEN=gen4`）。`poke-engine==0.0.48` 需从源码编译，**只装 pip 包不等于能跑**。Rust 工具链的下载地址 / 版本本地文件里没有 ⇒【待确认】，按官方 rustup 文档自行安装，别照抄任何记忆里的 URL。

### `E:\Tools\pokemon-showdown\package.json` 运行时依赖（核实）
`esbuild ^0.25.0`、`mysql2 ^3.14.2`、`preact ^10.5.15`、`preact-render-to-string`、`sockjs ^0.3.21`、`ts-chacha20 ^1.2.0`。

### `E:\Tools\src\metamon\pyproject.toml` 依赖（节选，原文）
`gymnasium>=0.26,<=0.29.1`、`torch>=2.6`、`numpy`、`gin-config`、`wandb`、`einops`、`tqdm`、`lz4`、`termcolor`、`rich`、`huggingface_hub`…（余下见原文件）。

**重建 / 安装命令（说明用，未执行）**：
```bash
# 主工程（用仓库自带 venv）
E:/Tools/foul-play/.venv/Scripts/python.exe -m pip install -r E:/Tools/foul-play/requirements.txt -r E:/Tools/foul-play/requirements-dev.txt

# 测试
E:/Tools/foul-play/.venv/Scripts/python.exe -m pytest

# 审计环境（若需重建；metamon / amago 为 editable 安装，源码在 E:\Tools\src\）
E:/venvs/metamon-audit/venv/Scripts/python.exe -m pip install -e E:/Tools/src/metamon -e E:/Tools/src/amago
```
> ⚠️ 上面最后一条的 `-e` 两个包同时安装是否可行、以及 `torch` CPU 版的 index，须以当时实际环境为准 —— 【待确认】，不要 Blindly 跑。已知 `torch 2.14.1` 的 CPU wheel 源出现在下载脚本里（`https://download.pytorch.org/whl/cpu`，见 §5）。

## 3. 本地 PS 服务器（:8000）

`E:\Tools\pokemon-showdown\config\config.js` 关键项（行号实证）：

| 行 | 配置 | 值 | 用途 |
|---|---|---|---|
| 8 | `exports.port` | `8000` | 本地服端口 |
| 16 | `exports.bindaddress` | `'0.0.0.0'` | 绑定全部网卡（**风险项**，见 `docs/11`） |
| 191 | `exports.loginserver` | `'http://play.pokemonshowdown.com/'` | 指向真实登录服 |
| 192 | `loginserverkeyalgo` | `"RSA-SHA1"` | assertion 验签算法 |
| 193 | `loginserverpublickeyid` | `4` | 公钥 id |
| 194 | `loginserverpublickey` | RSA 公钥（PEM） | 游戏服只持公钥 |
| **311** | `exports.noipchecks` | **`true`** | 使 `matchmakingOK` 提前 return true ⇒ **本地服绕过匹配墙 / 同 IP 墙** |
| **414** | `exports.consoleips` | `['127.0.0.1']` | console 访问白名单（**别放宽**） |
| 422 | `exports.watchconfig` | `true` | 配置热加载 |
| **463** | `exports.noguestsecurity` | **`true`** | 允许无 assertion 登录（游客级），本地测试刚需 |

> `legalhosts` / `tokenexpiry` 在 `config.js` **grep 零命中（未配置）** ⇒ 本地服不检查 hostname（跨服重放防护缺失），assertion 有效期走 `users.ts:692` 默认 **25 小时**。自托管部署清单里应补这两项。

**启动命令（说明用，未执行）**：
```bash
cd E:/Tools/pokemon-showdown
node build                       # 已编译产物 dist/server/index.js
node pokemon-showdown start 8000
```
`package.json:34-36` 的等价写法【实证】：`npm run build` = `node build`；`npm start` = `node pokemon-showdown start`（**不带端口参数**，端口靠 `config.js:8` 的 `exports.port = 8000`）。若需显式指定端口，用上面那条 `node pokemon-showdown start 8000`。

对应 bot 侧 `--websocket-uri local` → `ws://localhost:8000/showdown/websocket`（`fp/config.py:561-562`）。

**运营红线**（来自本地服安全审计结论）：**不授 `~` 权限、`consoleips` 不放宽、8000 不暴露公网**。

## 4. 对手建模库目录（已核实清单）

`E:\venvs\metamon-audit\opponent_modeling\`（`wc -l` 实测）：

| 文件 | 行数 / 说明 |
|---|---|
| `user_pokemon_config.jsonl` | **3309 行**（v2 三元组库，10.3 MB） |
| `elite_opponents\elite_profiles.jsonl` | **35 行**（curated 精英库） |
| `battles.jsonl` | 709 行（五源合并的对局） |
| `battles_A.jsonl` / `_B` / `_C` / `_D` / `_E` | 694 / 33 / 32 / 86 / 525 行 |
| `battles_stubs.jsonl` | 92 行（已剔除的 stub） |
| `crosscheck_1700_battles.jsonl` | 56 行 |

**构建 / 查询 / 自检脚本**（均在本目录，全部可执行 `.py`）：
`build_user_config.py`（31KB，主构建）、`build_user_config_v1_backup.py`、`extract_battles.py`、`add_team_signatures.py`、`analyze_repeats.py`、`crosscheck_1700.py`、`crosscheck_1700_stats.py`、`build_1700_answer.py`、`judge_move_columns.py`、`player_rating_agg.py`、`report_numbers.py`、`query_user_config.py`、`selfcheck_user_config.py`、`selfcheck_battles.py`、`verify_tag_count.py`、`diagnose_extraction.py`、`diagnose2.py`、`run_all.py`。

**产物报告（比 memory 转述更权威）**：
`USER_MODELING_REPORT_20261021.md`、`crosscheck_1700_report.md`（正式）、`opponent_repeat_report.md`、`crosscheck_1700_independent_report.md`（独立验证）、`crosscheck_1700_report.md`、`opponent_repeat_stats.json`、`crosscheck_1700_stats.json`、`build_stats.json`、`config_index.json`、`battles_summary.json`。

`elite_opponents\query_elite.py` 参数【实证·`grep -n add_argument` 原文】：`:86 --user`、`:87 --pokemon`、`:88 --team-match`、`:89 --list`(store_true)、`:90 --stats`(store_true)、`:91 --topn`(int, **default=5**)。

`elite_opponents\` 目录完整清单【实证·`ls -1`】：`build_elite_profiles.py`、`elite_index.json`、`elite_profiles.jsonl`、`elite_profiles_report.md`、`elite_stats.json`、`leave_one_out.json`、`leave_one_out.py`、`query_elite.py`（共 8 项）。

> 命令行开关的完整 45 项由 **`docs/03` 第 2 节**全量覆盖（来源 `fp/config.py:224-501` 的 `add_argument`）。本节末尾**没有**「CLI 参数表待确认」这类缺口，`fp/run.py` / `fp/main.py` 本身不解析 CLI，参数统一在 `fp/config.py::configure()`。

## 5. ⭐ 下载方法（**全部是从脚本里 grep 出的字面 URL，不是记忆**）

`E:\Tools\foul-play\tools\` 下与获取 / 安装相关的脚本：

| 脚本 | 作用 |
|---|---|
| `dl_kakuna.sh`、`dl_kakuna_v2.py`、`dl_kakuna_v3.sh`、`dl_kakuna_v4.sh` | 分四版迭代下载 **Kakuna 权重** |
| `download_gh_jsdelivr.py` | 走 jsDelivr CDN 抓 GitHub 仓库文件 |
| `robust_dl.sh` | 带重试的下载封装 |
| `install_env.sh` | 环境安装 |
| `inject_hf_cache.py` | 注入 HuggingFace 缓存 |

**脚本里出现的字面 URL**（`grep -hoE 'https?://...'` 去重结果）：
```
https://huggingface.co
https://hf.co
https://hf-mirror.com                                  ← 镜像
https://cdn.jsdelivr.net/gh/
https://data.jsdelivr.com/v1/packages/gh/
https://codeload.github.com/UT-Austin-RPL/metamon/tar.gz/refs/heads/main
https://codeload.github.com/UT-Austin-RPL/amago/tar.gz/refs/tags/v3.4.0
https://codeload.github.com/UT-Austin-RPL/poke-env/tar.gz/refs/heads/main
https://download.pytorch.org/whl/cpu                  ← torch CPU wheel
https://www.python.org/ftp/python/3.11.9/python-3.11.9-amd64.exe
http://127.0.0.1:7897                                 ← 本地代理端口
```
### 5.1 ⭐ Kakuna 权重：**完整直链已核实**（【更正】上一版曾误标「待确认」）

> 【更正·最高优先级】本节早期版本写「脚本里没有可直接复制的完整直链，只有域名与镜像」——**这是错的**。当时只 grep 了 `E:\Tools\foul-play\tools\`，漏了 `E:\venvs\metamon-audit\download_kakuna.py`。该脚本里就有 repo id + 完整文件名 + 落地路径。此类「把已核实事实误标成待确认」同样会误导接手者，已按原始文件复核改正。

**`E:\venvs\metamon-audit\download_kakuna.py` 原文关键行【实证·`cat` 全文】**：
```py
URL = "https://hf-mirror.com/jakegrigsby/metamon/resolve/main/kakuna/ckpts/policy_weights/policy_epoch_34.pt"
OUT = r"E:\venvs\metamon-audit\kakuna-ckpts\kakuna\ckpts\policy_weights\policy_epoch_34.pt"
```
- 脚本用 `urllib.request` 直连，**分块 1MiB 下载并算 sha256**，收尾打印 `DONE bytes=%d sha256=%s`。
- ⚠️ **安全提示【源码】**：脚本里写了 `ctx.check_hostname = False` / `ctx.verify_mode = ssl.CERT_NONE`（关闭 TLS 校验），是当时为绕过镜像证书问题加的。复用时**应恢复证书校验**；权威 sha256 见下方，下载后先核对再加载。
- 域名 `hf-mirror.com` 是镜像；脚本里另有 `https://huggingface.co` / `https://hf.co` 字面量，但**没有**拼好的上游完整 URL ⇒ 上游直链【待确认】，不要凭记忆改写。

**落地文件已存在【实证·`ls -l` 原始输出】**：
```
-rw-r--r-- 1 abc 197121 571581627 Oct 18 13:00 policy_epoch_34.pt
```
即 **571,581,627 字节（≈545 MiB）**，目录 `E:\venvs\metamon-audit\kakuna-ckpts\kakuna\ckpts\policy_weights\`。
**权威 sha256【原文·`E:\Tools\foul-play\reports\metamon-audit-plan\06-离线审计引擎实现规格.md:17`】**：
`7e7474d5b2ec6624afdeb29a7cf1dc5c84817d5aee8ec39c1d35a153b18df26c`

**上游模型标识【源码·`E:\Tools\src\metamon\metamon\rl\pretrained.py`】**：
| 行 | 原文 | 含义 |
|---|---|---|
| `:153` | `HF_REPO_ID = "jakegrigsby/metamon"` | HuggingFace repo id（与直链中的路径一致） |
| `:214-215` | `huggingface_hub.hf_hub_download(repo_id=self.HF_REPO_ID, ...)` | 上游标准下载入口 |
| `:893` | `class Kakuna(PretrainedModel):` | Kakuna 模型类 |
| `:915` | `default_checkpoint=34` | **默认 checkpoint = epoch 34**，与 `policy_epoch_34.pt` 对应 |

**精确参数与强度口径**：`142,832,563 params`（`06-离线审计引擎实现规格.md:29` 记 `initialize_agent(checkpoint=34)` 打印 `Checkpoint validated: 642 keys, 142,832,563 params == ckpt`；报告里的「142M」是约数）。`pretrained.py:893-905` docstring 明确 `gen9ou: ~71%` 是**对人类、`competitive` TeamSet 的估计 GXE**（不是胜率），并写明 **Kakuna = Superkazam 的微调版**（在「提高温度采集的自对弈 +7.8M 局」上继续训练）。

> ✅ **【口径已解释·原标待核】「1.6GB」从哪来**：`E:\Tools\foul-play\reports\metamon-audit-plan\03-kakuna-env-alignment.md:269` 原文「权重 **3 个 policy 文件**各 **571,573,427 / 571,581,627 / 571,581,627 字节**（`ls -la` 实测，**合计约 1.6GB**）」。⇒ 「1.6GB」是**整套目录里 3 个 policy 权重文件的合计**；本地 `policy_epoch_34.pt` 只是其中**一个**（571,581,627 字节）。`d1_kakuna_audit_report_20261019.md` 自身未写口径，但数量级由此得到解释。**两者不是同一个数字，不要合并引用。**

> **加载 Kakuna 的已知坑【原文·`06-离线审计引擎实现规格.md:14-30`】**：① 必须在 `import metamon` **之前**设 `METAMON_CACHE_DIR`（否则模块级检查直接抛 `ValueError`）；② 用 `LocalPretrainedModel` 而**不是**公开 `Kakuna()` 类（后者走 HF Hub 下载，本机 HF 被墙）；③ 需 3 条 gin override（`SDPAAttention`、`window_size=(96,0)`、`traj_save_len` 钉成 int，amago 3.4.0 默认 `1e10` 是 float 会让 `random.randint` 抛 TypeError），并镜像 `MetamonPerceiverTstepEncoder.tokenizer`；④ 加载后必须 `exp.policy.eval()`（`policy.training` 默认是 True）；⑤ `superkazam.gin` 本就选 SDPAAttention，**不需要 flash-attn**。

> ⚠️ **两套落地路径别混**：`tools\dl_kakuna*.sh` 里的字面目标是 `E:\venvs\mm_models\kakuna\`（脚本写作 `/e/venvs/mm_models/kakuna`），而**实际成功下载**用的是 `download_kakuna.py` 的 `E:\venvs\metamon-audit\kakuna-ckpts\...`。前者是早期迭代脚本，后者是当前生效路径。

> ⚠️ **版本口径差异**：`install_env.sh` 里字面是 **Python 3.11.9** 安装包（`https://www.python.org/ftp/python/3.11.9/python-3.11.9-amd64.exe`），而两个现有 venv 都是 **3.13.14**。这是历史脚本遗留，不要照它降级主环境。

### 5.2 源码获取（**全部已核实，无待确认缺口**）

```bash
git clone https://github.com/pmariglia/foul-play.git
git clone https://github.com/smogon/pokemon-showdown.git
```
metamon / amago 本地 checkout **无 git remote**，但仓库主页与源码包地址有实证：

| 项目 | 出处 | 值 |
|---|---|---|
| metamon 主页 | `E:\Tools\src\metamon\pyproject.toml:42-44` | `Homepage = "https://metamon.tech"`；`Repository = "https://github.com/UT-Austin-RPL/metamon"` |
| amago 主页 | `E:\Tools\src\amago\pyproject.toml:54-55` | `Repository = "https://github.com/UT-Austin-RPL/amago"` |
| 源码 tar 包 | `tools\` 脚本字面 URL | `codeload.github.com/UT-Austin-RPL/metamon/tar.gz/refs/heads/main`、`.../amago/tar.gz/refs/tags/v3.4.0`、`.../poke-env/tar.gz/refs/heads/main` |

> **PyPI 上的包名 / 版本**仍【待确认】（本地只有 editable 源码与 `pyproject.toml` 的 `name`/`version`，没有 PyPI 页面证据）；但**仓库主页已核实**，不需要再向用户索取。

## 6. 工作区工具脚本（只读校验）

| 脚本 | 用途（读文件头 docstring 实证） |
|---|---|
| `scan_logs.py` | **只读**校验 battle-tag 重组与 `|request|`/response 配对：扫 `E:\Tools\foul-play\logs\*.log`，按 tag 分组，检查 `|win|` / `|player|` 归属、按 `rqid` 配对、异常（重复 rqid / 孤儿 response / 未回答的非 wait request / 同时 active 与 forceSwitch）。**`LOGS_DIR = sys.argv[1] if len(sys.argv)>1 else r"E:\Tools\foul-play\logs"`**；`Prints a compact summary only. Does not write anything.` |
| `reconcile_counts.py` | **只读**核对冻结决策点定义：非 teamPreview、非 wait 的 `|request|` 与其后 `Choice:` 配对；分四类统计 response 并校验配对。`Prints only. Writes nothing.` |

> ⚠️ `reconcile_counts.py` docstring 写「`rqid is the global primary key`」，但 `reports\metamon-audit-plan\04-data-model-dataset.md:319/:549` 已核定 **`rqid` 单独不唯一，主键应为 `(battle_tag, rqid)`**。以 04 章为准，别照 docstring 建表。
>
> ⚠️ `scan_logs.py` 默认读 `logs\`（实时日志目录）—— 按 `AGENTS.md` 红线 2，实际使用时**传入冻结副本目录作为 `sys.argv[1]`**，不要对 live 目录跑长任务。

## 7. 其它工具（`E:\Tools\foul-play\tools\`）

A/B 与实验：`ab_ladder_pipeline.py`、`ab_mcts_stats.py`、`ab_mcts_test.py`、`ab_mcts_test_round2.py`、`mde_two_prop.py`、`h2h_stats.py`、`hp_schedule_bench.py`。
端到端验证：`concurrent_realtime_test.py`、`e2e_hidenext_test.py`、`e2e_log_concurrent_test.py`、`e2e_modjoin_test.py`、`ladder_battle.sh`、`h2h_battle.sh`、`cwatch.sh`。
日志与队伍：`export_battle_log.py`、`extract_opponent_evidence.py`、`replay_lead_advisor.py`、`team_editor\`、`gen9_team_stats\`、`disk_scan.py`、`audit\`。
其它：`fp_acceptance.py`、`opponent_evidence_recent.json`、`hp_schedule_results.json`(+`.prebugfix`)。

启动器 `E:\Tools\run_3_battle.bat`（工作区外）与 `tools\h2h_battle.sh` 见 `docs/03` 第 6 节。

## 8. 重建顺序建议（Codex 拿到一台新机器时）

1. 拿源码：`foul-play` + `pokemon-showdown`（remote 已核实）；metamon / amago 按 §5 的 codeload 地址或向用户索取。
2. 建主 venv（Python 3.13.x），按 `requirements.txt` + `requirements-dev.txt` 安装，**注意 `poke-engine` 的 `--config-settings` 整行**。
3. 跑 `pytest`，把**你自己的输出**作为基线（别引用报告里的历史数字）。
4. 需要离线审计才建 `E:\venvs\metamon-audit\venv`（torch CPU + metamon/amago editable）。
5. 本地服：`node build` → 起 :8000，确认 `noguestsecurity` / `noipchecks` 两项。
6. 对手建模库与 Kakuna 权重：库按 §4 脚本重建；Kakuna 单权重直链**已在 §5.1 核实**（`download_kakuna.py` 的 `hf-mirror.com/jakegrigsby/metamon/resolve/main/.../policy_epoch_34.pt`），可直接复用或走上游 `huggingface_hub.hf_hub_download`。**本地已存在的文件不要重复下载**（571,581,627 字节）。
7. 全程**默认 OFF 开关一律不动**；先读 `AGENTS.md` 与 `docs/11`。
