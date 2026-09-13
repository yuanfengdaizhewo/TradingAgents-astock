# <功能名> — 规格

| 项 | 值 |
|----|-----|
| 状态 | 草案 / 已定稿 / 实现中 / 已发布 |
| 作者 | |
| 日期 | |
| 关联 issue | # |
| 落点 | 数据层 / Agent / Web / CLI（可多选） |

> 用法：复制本文件为 `docs/specs/NNN-<slug>.md`。一页纸为准——写不下通常是范围没收住。
> 技术选型理由写进 `DEV_LOG.md`（该仓库的 ADR 载体），不要塞在这里。

---

## 1. 目标

一句话，且**可验证**。反例：「提升分析质量」。正例：「新增北向资金分析师，其报告出现在 Web 12 阶段进度面板并在最终决策中被引用」。

## 2. 非目标

明确写"这次不做什么"。这一栏比目标更能防止范围蔓延。

- 不做 …
- 不改 …

## 3. 用户故事

- 作为 <角色>，我想要 <能力>，以便 <收益>。

## 4. 验收标准

**必须可执行**。本项目的 e2e 天然可测，用足：

- [ ] `python -m pytest tests/` 全绿（基线见 §9）
- [ ] 跑通一只真实标的（`600519`），完整报告含 `<字段>` 且非空
- [ ] Web：`tradingagents-web` 起来后，进度面板出现 `<阶段名>`，报告可展开
- [ ] 失败路径：数据源不可用时，报告顶部出现明确告警而非静默留空

## 5. 影响面

预估要碰的文件（写完对照实际改动回填，用于评估"fork 漂移"成本）：

| 层 | 文件 | 新增/修改 |
|----|------|----------|
| 数据 | `tradingagents/dataflows/…` | |
| Agent | `tradingagents/agents/…` | |
| 图 | `tradingagents/graph/…` | |
| Web | `web/…` | |
| CLI | `cli/…` | |

> ⚠️ **新文件优先**：凡能用新文件解决的，不要改上游文件；必须改时只动"注册点"，每个点一行。
> 这是将来 `git merge upstream/main` 能否保持干净的前提。

## 6. 接入点清单（新增 Agent 时逐项核对）

- [ ] `agents/analysts/<x>.py` 新建节点工厂（模板：`policy_analyst.py`，85 行）
- [ ] `agents/__init__.py` — `import` + `__all__`
- [ ] `agents/utils/agent_states.py` — `AgentState` 加 `<x>_report`
- [ ] `graph/propagation.py` — 初始状态初始化 `"<x>_report": ""`
- [ ] `graph/setup.py` — `ROLE_KEYS` + 默认 `selected_analysts` + `if "<x>" in …` 块
- [ ] `graph/trading_graph.py` — 默认列表 + `tool_nodes["<x>"]` + 落盘 dict
- [ ] `graph/conditional_logic.py` — `should_continue_<x>()`
- [ ] `agents/quality_gate.py` — 两个映射表
- [ ] **下游消费者**：bull / bear researcher、trader、3 个 risk debator 的 prompt 注入
      （⚠️ 本仓库最易漏的一步：不手工接，新报告不会流入辩论与决策）
- [ ] Web：`runner.py` 键列表、`progress.py` 阶段、`report_viewer.py` 展示、`pdf_export.py` 分区、`stock_display.py` 归一化
- [ ] `examples/run_cases.py` summary 字段
- [ ] 文档：README、README_en、CHANGES_FROM_UPSTREAM、DEV_LOG

> 注：CLI 侧目前只注册了 4 个分析师（`cli/models.py` 枚举 + `cli/utils.py` 的 `ANALYST_ORDER`）。
> 若要在 CLI 里可选，需额外补 `AnalystType` / `ANALYST_ORDER` / `cli/main.py` 的
> `ANALYST_MAPPING`、`REPORT_SECTIONS`、`save_report_to_disk`、`display_complete_report`。

## 7. 数据与防护（新增数据接口必答）

- [ ] 逐代码归一化走 `_normalize_ticker`（**不是** `safe_ticker_component`）——拒港股/美股
- [ ] 收 `curr_date` 就必须处理时点语义：能截断则截断，不能则用 `_snapshot_notice` 明示
      （新增接口拿了 `curr_date` 不用 = 静默未来函数）
- [ ] 东财端点走 `_em_get` 统一节流，不要裸 `requests.get`
- [ ] 兜底链：主源失败时降级到哪个 HTTP 源？降级后报告里如何体现数据来源？

## 8. 模型与角色

- [ ] 取模型只用 `self.llm_for("<角色名>")`，不直接引用 `quick_thinking_llm`
- [ ] 角色名加进 `graph/setup.py` 的 `ROLE_KEYS`
- [ ] 需要给该角色单独配模型时，`role_llms` 的用法已验证（写错角色名应报错，不静默忽略）

## 9. 测试计划

| 层 | 内容 | 文件 |
|----|------|------|
| 单测 | 节点函数 + mock llm | `tests/test_<x>.py` |
| 图级 | 拓扑/状态流转 | 参考 `tests/test_checkpoint_resume.py` |
| e2e | 真跑一只票 | 手动 |
| UI | `tradingagents-web` 走一遍 | 手动 |

基线（干净 clone）：**361 passed / 13 skipped / 0 failed**。出现 failed 即为真回归。

## 10. 上线与回滚

- 版本号三处同改：`pyproject.toml` / `CHANGELOG.md` / `CLAUDE.md`「当前版本」（有测试强制）
- CHANGELOG 条目（Keep a Changelog 风格，带 issue 号与署名）
- 回滚方式：`git revert <sha>` / 配置开关

## 11. 待定问题

- [ ] …
