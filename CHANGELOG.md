# 更新记录

这个仓库是**零件仓库**（organs）：每个 `dsh-agent-*` 插件的源码与产物在这里各占一个目录。

★ 和 `dsh-agent-soul` 的关系：那边是**架构驱动插件**（教 agent 怎么搭五层），
这边是**零件本体**。改零件时两边都要动 —— 那边的 `plugins/` 是分发副本。

★ 完整的技术细节记在 `dsh-agent-soul/CHANGELOG.md`。这里只记**零件本身的变动**。

---

## 2026-10-19 · huanqi 吸收空回合判据；huanqi-panel 跟着改账本路径

### huanqi/

**改了什么**：判据从三条变四条，新增第四条「空回合」——挂在 `session/event` 的
`turn/end` 上，检测「整个回合结束了、一个字节都没吐」。命中后走换气自己的额度账本
（`canContinue` → `spendContinuation` → `onContinuation` → `save`），
和流内那三条**共用同一本账**。

**为什么**：第四种退化此前靠独立插件 `dsh-stalled-turn-continue` 兜底，而它自己记一套
`maxConsecutive: 3`（按会话），换气记的是共享的 `ledger` —— **两套账会让一轮被续成两轮**，
而且「额度」这个概念被劈成两半。合并之后：四条判据扣同一分、用同一个上限、触发同一个熔断。

**新增配置** `emptyTurnContinue`（默认 `true`），可单独关掉这一条。

**顺带修的三个坑**（都是发布包本来就有的）：

- `package.json` 的 `files` 写着 `README_EN.md` 和 `tools/analyze-session.mjs`，但文件不在包里 → 补上
- `tsconfig.json` 缺失 → 补上
- 判断条件写成 `config.emptyTurnContinue && ...`，而 zod 默认值只在经过 schema 解析时
  才生效 —— 绕过解析的路径拿到 `undefined`，整条监听不注册，**判据静默失效** → 改成 `!== false`

**验证**：`node --test test/*.spec.mjs` → **196/196**（新增 `test/empty-turn.spec.mjs` 11 例）。

### huanqi-panel/

**改了什么**：`LEDGER` 常量从 `state/dsh-huanqi.json` 改成 `state/dsh-agent-huanqi.json`。

**为什么**：换气的账本路径本来就随包名走（`dsh-agent-huanqi`），而面板里硬编码的是开发源的
包名（`dsh-huanqi`）。两个包同时装时，面板读的是一本没人写的账 —— 会一直显示「换气还没出过手」。

### echo/

**改了什么**：`echo-doctor.mjs` 里那句「换气只覆盖推理 ≥2048 的；`dsh-stalled-turn-continue`
覆盖全部无输出回合（**两者接力**）」改成「换气自己覆盖四种退化（含空回合）；
**不再需要 `dsh-stalled-turn-continue`** —— 两套账会让一轮被续成两轮」。

**为什么**：接力关系已经不成立了 —— 空回合并进换气之后，那个插件是多余的。

---

## 2026-10-16 · 零件仓库成形

把 13 个 `dsh-agent-*` 插件的源码从各个开发目录收拢到这里，各占一个目录。
`plugins/` 在 `dsh-agent-soul` 那边保持权威，这边的目录是它的镜像。

★ 当时定下的纪律：**改零件时先改这里，再同步到 soul 的 `plugins/`** ——
反过来会让分发副本变成事实上的源码，下次同步就分不清谁对。
