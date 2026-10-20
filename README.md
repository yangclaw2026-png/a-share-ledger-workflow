# A股台账工作流（a-share-ledger-workflow）

> 8 步 A 股研究流程（方向 → 产业 → 链条 → 筛选 → 跟踪 → 门禁 → 持仓 → 复盘）
> ＋ 台账规范 ＋ 14 个脚本。零第三方依赖（纯 Python 标准库）。
>
> ⚠️ 本仓库是**流程与工具**，不含任何持仓/标的/token。投资有风险，
> 流程只保证"证据可追溯、判定可重算"，不保证收益。详见 `DISCLAIMER.md`。

## 1. 快速开始

```bash
# ① 准备数据目录（结构见第 2 节）
export STOCK_DATA_DIR="$HOME/stock_data"

# ② 门槛数据：复制模板并按你的标的填写
cp templates/bench.example.json $STOCK_DATA_DIR/bench.json
# 字段说明见模板顶部注释（每只标的 7 项：direction/bench_pt/pb_pctl/
# pe_pctl_3y/kf_ratio/a1/a1_note；constants 含判定阈值与 a1_base_date）

# ③ 自检（无飞书/无 Hermes 也能跑，缺的项会显式"跳过"）
python3 scripts/validate_ledgers.py

# ④ 跑通示例（隔离目录，不碰你的真实数据；含破坏测试）
# 步骤见 examples/sample_direction/README.md（预期 rc=0 + 2 项跳过提示）
```

## 2. 数据目录约定（`$STOCK_DATA_DIR`）

```
$STOCK_DATA_DIR/
├── bench.json                  # 门槛数据（必填，永不入仓；见 templates/bench.example.json）
├── 股票台账/
│   ├── 候选/                   # *_跟踪台账.md（建台账即自动纳入 candidate_gate 监控）
│   ├── 持仓中/                 # {名称}_{代码}_跟踪台账.md（建仓即自动纳入 profit_guard 监控）
│   ├── 流程状态表.md
│   ├── 待办与关键时点.md
│   └── 模板/                   # templates/ledger_template.md 拷过来做唯一权威模板
├── 方向记录/                   # 方向档案（含Q3披露监控用的代码名称映射，见第8项说明）
└── reports/                    # financial_analysis_reports 输出位
```

## 3. 环境变量

| 变量 | 必填 | 说明 |
|---|---|---|
| `STOCK_DATA_DIR` | ✅ | 你的股票数据根目录（见第 2 节）。未设 → 所有脚本明确报错退出 |
| `HERMES_HOME` | Hermes 用户 | Hermes profile 根（含 `scripts/`、`cron/jobs.json`）。未设 → cron 相关校验**跳过**（不计失败） |
| `FEISHU_APP_ID` / `FEISHU_APP_SECRET` | 飞书同步 | 飞书应用凭证（只走环境变量，永不写文件） |
| `FEISHU_FOLDER_TOKEN` | 飞书同步 | 目标根文件夹 token |
| `FEISHU_FOLDER_SUBDIR` | 飞书同步 | JSON，如 `{"ledgers":"...","status":"...","todo":"..."}` |
| `FEISHU_FOLDER_NOTICE` | 云文档留档 | `04_自动通知` 文件夹 token |
| `FEISHU_FOLDER_DIRECTION` | 方向同步 | `02_方向研究` 文件夹 token |
| `FEISHU_FOLDER_LEDGERS` | 去重 | `01_候选台账` 文件夹 token |
| `FEISHU_CHAT_ID` | 飞书告警 | 看门狗投递目标 |
| `Q3_WATCH_JSON` | 可选 | q3_watch 输出路径（默认 `/tmp/q3_watch.json`，两边保持一致） |

🔴 未配置飞书/邮箱 → 相关脚本**跳过**（stderr 一行提示，stdout 零污染，
cron 不会误报故障）。需要同步才配，不配也能用本地全套。

## 4. 常用命令

```bash
python3 scripts/validate_ledgers.py            # 台账 12 项校验（含 cron 脚本漂移第12项）
python3 scripts/rebuild_bench.py              # 从台账排名表重算 bench_pt，与 bench.json 对账（只读）
python3 scripts/selftest_dict_check.py        # 第7项双向自检（好状态不误报 + 坏状态必报）
python3 scripts/candidate_gate.py              # 候选门槛复查（A1+A2必过·B1-B5任一触发）
python3 scripts/candidate_gate.py --codes 600000 --json-out out.json
python3 scripts/profit_guard.py --with-json    # 持仓三模块（盈利/亏损/加仓前置）
python3 scripts/q3_watch.py --json-out $Q3_WATCH_JSON   # Q3 披露监控
python3 scripts/refresh_a1.py --set 600000:12.5:4.88:30.5  # 模型确认后回写 a1 基线
python3 scripts/trading_day.py YYYY-MM-DD      # 交易日判定（例：节后首个交易日应返回 ✅）
```

## 5. cron（按需自建，本仓库不带）

- 任务命名规范：引用一律写**完整任务名**，永不写 cron ID（ID 随重建变化）；
  任务名永不带数量（改名会让全部台账引用失效，校验器按名匹配）。
- 命名与分工示例见 `skills/ashare-notification-spec/SKILL.md` §3。
- Hermes cron 的 `--script` 脚本**只能放 `$HERMES_HOME/scripts/`**
  （cron 硬性要求），改完仓库版 `cp` 过去；两份不一致时校验器第 12 项会报"脚本漂移"。

## 6. 技能

`skills/` 下 15 个：主流程 `a-share-investment-workflow`（1–8 步）＋
台账规范 `ashare-ledger-standard` ＋ 各步子技能 ＋ 横向
（`dual-ai-stock-analysis` 第6步引用、`finance-stock-technical-analysis`
第5步引用、`ashare-notification-spec`、`ashare-data-source-fallbacks`、
`ashare-filing-verification`）。

## 7. 已知行为

🔴 **第7项（台账「相对中位」↔ `bench.json` 的 `bench_pt`）是 fail-closed 的**：
台账里提不到数值就**报错**，不静默跳过；每次运行还会打印「实际校验 X / 应有 Y」，
跳过/豁免的标的逐只列出。这么做是因为反面教训真实发生过 —— 旧实现遇到不认识的写法
直接 `continue`，32 只里只有 2 只被真正校验，输出却仍是「✅ 通过」。
⭐ 想确认它真的在工作：`python3 scripts/selftest_dict_check.py`（会注入坏状态、
并回滚旧正则验证覆盖率数字是实测值）。
⭐ `bench_pt` 不要手填：用 `scripts/rebuild_bench.py` 从台账排名表重算（口径固定为
「中位=平均」，方向内无对照的标的写 `null`，不要写 0.0 —— 0.0 会被 B4 读成「持平」）。
`bench_base`（可选）写明每个方向的基准起止，台账写了基准日时会被逐只核对，
口径不一致会报「基准日不符」，与「数值冲突」分开报。


- 无 `bench.json` → 门槛相关项 NEED_DATA/跳过（显式可见，不静默）。
- 无 cron（无 jobs.json）→ 第 1 项与时点校验跳过，退出码 0，跳过项会列出。
- `bench.json` 已加入 `.gitignore`（含全部标的明细，永不入仓）。

## 8. 安装（技能 / 脚本落位）

前置：Hermes（技能调用与 cron 用；纯脚本试跑不需要）、Python ≥ 3.10。

```bash
# ① 取仓库（只读本地，不 push）
git clone <本仓库地址> a-share-ledger-workflow && cd a-share-ledger-workflow

# ② 技能落位（默认位置，以你的 Hermes 文档为准）
cp -r skills/* <你的Hermes技能目录>/            # 如 ~/.hermes/profiles/<你>/skills/

# ③ 脚本落位（二选一）
#   A. 本地跑：直接用仓库内 scripts/（STOCK_DATA_DIR 指向你的数据目录即可）
#   B. Hermes cron 跑：cp 到 $HERMES_HOME/scripts/（cron 硬性要求，只能认该目录）
cp scripts/*.py $HERMES_HOME/scripts/

# ④ 零依赖（无需 pip install；requirements.txt 为空说明）
python3 -c "import sys; print(sys.version)"

# ⑤ 数据目录 + 示例验证（隔离目录）
export STOCK_DATA_DIR="$HOME/stock_data"
cp templates/bench.example.json $STOCK_DATA_DIR/bench.json   # 后按你的标的填写
rm -rf /tmp/demo && mkdir -p /tmp/demo
cp -r examples/sample_direction/* /tmp/demo/
STOCK_DATA_DIR=/tmp/demo python3 scripts/validate_ledgers.py  # 预期 rc=0 + 2 项跳过提示
```

外部脚本（可选依赖，不装则对应功能跳过，不报错）：

| 外部 | 提供方 | 缺失时行为 |
|---|---|---|
| `mailer.py`（`hermes-stock-mail` 技能） | Hermes 邮件技能 | `profit_guard.py` 邮件通道跳过（stderr 一行），不影响飞书 |
| `feishu_doc_write.py`（`feishu-doc-sync` 技能） | Hermes 飞书技能 | `sync_ledgers_to_feishu.py` 无法写云文档；先配好再跑同步 |
| `sources.py`（`grounded-citations` 技能） | Hermes 引用技能 | 技能文字里引用它做财报逐字校验；不用则手工贴来源 |

## 9. cron 自建思路（本仓库不带任务，只给分工参考）

命名与分工示例见 `skills/ashare-notification-spec/SKILL.md` §3。
建议起点（频率按你的标的数量级调整）：

- 候选门槛复查（`candidate_gate.py`）：每周 2 次（如周二/周五）。
- 财报披露跟踪（`q3_watch.py` → `refresh_a1.py --set`）：每天 1 次。
- 持仓三模块（`profit_guard.py --with-json`，no_agent）：每个交易日盘中 + 尾盘各 1 次。
- 台账同步（`sync_ledgers_to_feishu.py`，no_agent 零 token）：每周 1 次。
- 第6步批次盯盘：有标的通过门禁时**克隆模板任务**（平时 `enabled=false`），一股一任务。

## 10. 已知问题（第三方接口坑）

- 东财 K 线接口偶发拒连 → 脚本重试 3 次仍失败报 `DATA_FAIL`/`FETCH_FAIL`（不按"无数据"处理）。
- `MGJYXJJETZ` 字段名像"经营现金流同比"但不是 → 现金流同比必须用 `NETCASH_OPERATE_PK` 同期金额自算（详见 `ashare-data-source-fallbacks`）。
- 腾讯 K 线必须带 `sh/sz` 前缀（由代码规则推，`_exchange_pre`）；未知号段跳过并注明。
- cron 输出文件是完整对话记录（含 prompt 全文）→ `feishu_append_notice.py` 只取最后一次 `## Response` 之后正文；**绝不整份上传**。

## 11. 隐私说明

- 持仓数据永不上传：`sync_ledgers_to_feishu.py` 对 `持仓中/` 是**目录级硬排除**
  （不是读后过滤）；第 7 步只走飞书 IM + 邮件，不进云文档。
- `bench.json`（全部标的明细与估值分位）已进 `.gitignore`，永不入仓。
- 飞书 token / chat_id / 邮箱只走环境变量（见 §3），仓库内零凭据（已全仓扫描验证）。
