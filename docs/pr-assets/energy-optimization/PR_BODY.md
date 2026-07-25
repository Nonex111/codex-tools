<!-- PR title / PR 标题: [EN/ZH] Improve the macOS status bar and reduce background energy use / 优化 macOS 状态栏并降低后台耗电 -->

> **Draft status / 草稿状态:** This local draft describes the current branch, including changes not yet pushed to the PR. / 本地草稿按当前分支编写，其中包含尚未推送到 PR 的改动。

## Summary / 摘要

- **Improve platform-aware desktop behavior / 改善平台相关的桌面体验:** enhance the macOS status item and usage labels, show its controls only on macOS, and make Windows Store account switching restart the correct Codex process tree.
- **Make account state faster and more trustworthy / 让账号状态更快、更可信:** restore cached accounts and quota immediately, resolve Plus-to-Pro identity drift, expose reference-only membership expiry, and turn refresh failures into accurate, actionable diagnostics.
- **Make local analytics correct, safe, and consistent / 让本机分析正确、安全且口径一致:** derive real Token deltas from cumulative logs, handle fork lineage conservatively, unify totals and heatmap calculations, and allow isolated previews to read production analytics without modifying production sessions.
- **Reduce recurring background work / 降低周期性后台开销:** remove inference keepalive work, incrementally refresh changed logs once per minute, pause unnecessary hidden-window polling, and reuse bilingual release notes in the updater.

## Changes / 改动

### 1. macOS status item and platform-aware settings / macOS 状态栏与平台相关设置

The macOS status item now uses the color application icon, defaults to one-week remaining usage, optionally shows `5h / 1w` labels, and disappears completely when “Hidden” is selected. Account meters preserve the semantic `5h` and `1w` labels even when both periods currently report the same value. Because these display modes control the macOS menu-bar title rather than the basic Windows tray menu, the entire setting row is now rendered only on macOS.

macOS 状态栏现在使用彩色应用图标，默认显示一周剩余用量，可选显示 `5h / 1w` 标签，并在选择“不显示”时完整隐藏。即使两个周期当前数值相同，账号用量栏仍分别保留 `5h` 与 `1w` 的语义标签。由于这些显示模式控制的是 macOS 菜单栏标题，而不是 Windows 的基础托盘菜单，因此整行设置现在仅在 macOS 上显示。

| Before / 修改前 | After / 修改后 |
| --- | --- |
| ![Status item before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/status-item-before.png) | ![Status item after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/status-item-after.png) |

| Before / 修改前 |
| --- |
| ![Status settings before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/status-settings-before.png) |

| After / 修改后 |
| --- |
| ![Status settings after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/status-settings-after.png) |

| Before / 修改前 | After / 修改后 |
| --- | --- |
| ![Usage labels before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/usage-labels-before.png) | ![Usage labels after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/usage-labels-after.png) |

### 2. Account startup, identity recovery, and diagnostics / 账号启动、身份恢复与诊断

Returning installations render locally stored accounts and the latest quota snapshot before the remote refresh finishes. The first refresh is visibly in progress; success clears the badge, while failure keeps a concise reason with the complete error available on hover. Clean installations without a snapshot still wait for the first remote result, and cached values may remain temporarily stale until refresh completes.

已有本地数据的安装会在远端刷新完成前显示已保存账号和最新额度快照。首次刷新会明确显示进行中状态；成功后提示消失，失败时则保留简短原因，并可悬停查看完整错误。没有历史快照的全新安装仍需等待首次远端结果，缓存值在刷新完成前也可能暂时陈旧。

| Before / 修改前 |
| --- |
| ![Account usage loading before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/account-usage-loading-before.png) |

| After / 修改后 |
| --- |
| ![Account usage loading after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/account-usage-loading-after.png) |

Account matching now prioritizes stable identity over mutable plan metadata. If `auth.json` still reports Plus while the saved account and live quota are already PRO, startup reuses the PRO record and its quota instead of creating an empty Plus variant. A cooldown-protected token refresh may repair a known plan conflict; when ordinary refresh cannot update stale identity claims, OAuth reauthorization synchronizes the new credentials back to the active `auth.json`.

账号匹配现在优先使用稳定身份，而不是可变化的套餐字段。如果 `auth.json` 仍显示 Plus，而已保存账号和实时额度已经是 PRO，启动时会复用该 PRO 记录及其额度，而不是创建空白 Plus 变体。遇到已知套餐冲突时，可以在冷却限制下刷新一次令牌；若普通刷新无法更新陈旧身份声明，OAuth 重新授权会把新凭据同步回当前 `auth.json`。

![Plus-to-Pro account resolution before and after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/account-resolution-before-after.png)

The account detail card shows a future `chatgpt_subscription_active_until` ID-token claim when available. Missing or past claims are not presented as valid expiry dates, and the field is labeled **for reference only** because it may remain stale after a plan change. This is separate from the reset-card count and expiry requested in Issue #136.

账号详情会在存在有效未来数值时显示 ID token 中的 `chatgpt_subscription_active_until`。缺失或已过去的声明不会作为有效到期时间展示；由于套餐变化后该值可能仍然陈旧，字段会明确标注为**仅供参考**。它与 Issue #136 中的重置卡数量及过期时间相互独立。

| Before / 修改前 |
| --- |
| ![Controlled membership expiry comparison before](https://raw.githubusercontent.com/Nonex111/codex-tools/44e52c7c387954e0fdb8aa3a8b9f304da2cfd3fa/docs/pr-assets/energy-optimization/membership-expiry-controlled-before.png) |

| After / 修改后 |
| --- |
| ![Controlled membership expiry comparison after](https://raw.githubusercontent.com/Nonex111/codex-tools/44e52c7c387954e0fdb8aa3a8b9f304da2cfd3fa/docs/pr-assets/energy-optimization/membership-expiry-controlled-after.png) |

Refresh failures now preserve the primary HTTP status, such as `503`, instead of allowing a fallback `403` response to hide a service outage. Reauthorization is offered only for actionable authentication states such as `401`, expired or revoked tokens, `invalid_token`, and `invalid_grant`; outages, rate limits, timeouts, network failures, generic HTML `403` responses, and disabled accounts no longer incorrectly suggest signing in again.

刷新失败现在会保留主要请求的 HTTP 状态码（例如 `503`），不会再让回退请求中的 `403` 掩盖服务中断。只有 `401`、令牌过期或吊销、`invalid_token`、`invalid_grant` 等可通过授权恢复的状态才会显示“重新登录”；服务不可用、限流、超时、网络故障、通用 HTML `403` 及停用账号不会再被错误引导重新登录。

**Root causes / 根因:** plan metadata was previously treated as part of account identity, and the refresh classifier searched for `401/403` before `5xx` without separately deciding whether reauthorization was useful. / 此前套餐字段被视为账号身份的一部分，同时刷新分类器会先搜索 `401/403`、再检查 `5xx`，也没有独立判断重新授权是否真正有帮助。

### 3. Consistent and safe local analytics / 一致且安全的本机分析

Token summaries and detailed analytics now derive actual increments from cumulative snapshots: unchanged counters contribute zero, monotonic increases contribute only component-wise deltas, and counter resets remain visible as anomalies. Total, seven-day, project, session, prompt, and heatmap values all use the same confirmed local-log deltas and model-pricing estimate, so complete project details add up to the displayed total.

Token 汇总与详细分析现在根据累计快照计算实际增量：计数不变时计为零，单调增长时仅计算各 Token 分量差值，计数器回退则保留为异常。总量、7 日、项目、会话、prompt 与热力图统一使用已确认的本机日志增量及同一套模型计价估算，因此完整项目明细可与显示总成本相加一致。

Forked files keep the immutable identity from their first physical `session_meta`. History inheritance (`forked_from_id`) is separated from agent ownership (`parent_thread_id`), and only a verified direct-parent range is excluded. The matcher tolerates small replay insertions, omissions, regenerated IDs, and known default fields, but stops conservatively at the branch boundary; missing, ambiguous, cyclic, or otherwise unverifiable lineage is excluded from confirmed totals and reported as unresolved.

fork 文件始终保留首条物理 `session_meta` 中的不可变身份。历史继承（`forked_from_id`）与 Agent 归属（`parent_thread_id`）分开处理，仅排除经验证的直接父会话区间。匹配器可以容忍回放时少量记录插入或缺失、重新生成的 ID 及已知默认字段，但会在真实分叉边界保守停止；父日志缺失、关系冲突、成环或无法验证时，不会计入已确认总量，并会显示为未解析。

These values are local-log Tokens and estimated API-equivalent costs—not official ChatGPT quota/Profile activity or API-proxy upstream usage. The seven-day card uses the previous seven completed local calendar days, while the budget alert uses the rolling latest 168 hours. The heatmap uses local time, localized tooltips, compact K/M values, and nine logarithmic levels relative to the largest one-hour bucket in the current view.

这些数值是本机日志 Token 与 API 等值成本估算，不是 ChatGPT 官方额度/Profile 活动量，也不是 API 反代上游用量。7 日卡片采用前 7 个完整本地自然日，预算预警则采用滚动最近 168 小时。热力图按本地时间显示，提供本地化提示与 K/M 紧凑数值，并按当前视图最大单小时值分配九档对数相对色阶。

| Before / 修改前 |
| --- |
| ![Controlled heatmap simulation before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/analytics-heatmap-controlled-before.png) |

| After / 修改后 |
| --- |
| ![Controlled heatmap simulation after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/analytics-heatmap-controlled-after.png) |

Development previews keep their account store and writable Codex state isolated, but may read production session logs through `CODEX_TOOLS_DEV_ANALYTICS_DIR`. They use a separate analytics cache and refuse session deletion while this read-only source is active, preventing Analytics-page testing from modifying production sessions.

开发预览继续隔离账号库及可写 Codex 状态，但可以通过 `CODEX_TOOLS_DEV_ANALYTICS_DIR` 读取正式会话日志。预览版使用独立分析缓存，并在只读来源启用时拒绝删除会话，避免测试分析页面时修改正式会话。

### 4. Incremental background processing and energy use / 增量后台处理与能耗

The app no longer sends an inference keepalive. After the initial scan in each process, unchanged logs reuse parsed in-memory results and append-only files resume from their previous byte offset. Truncated, rewritten, or tail-mismatched files fall back to reparsing that file. The latest aggregate snapshot is persisted for fast restart display, while the per-file index is rebuilt on the new process's first scan.

应用不再发送推理保活请求。每次进程完成首次扫描后，未变化日志会复用内存解析结果，仅追加文件则从上次字节位置继续读取；文件被截断、重写或尾部校验不匹配时，会回退为重新解析该文件。最新聚合快照会持久化以便重启后快速显示，而逐文件索引会在新进程首次扫描时重建。

Detailed local-log analytics refreshes every 60 seconds, with **Refresh analytics** available for immediate updates. Entering the Analytics page does not start another scan. This schedule affects only local session-log analysis; account quota refresh and API-proxy collection remain independent. Nonessential foreground polling also pauses while the main window is hidden.

本机会话日志分析固定每 60 秒刷新一次，同时保留“刷新分析”用于立即更新；进入分析页不会再启动额外扫描。该周期只影响本机会话日志分析，账号额度刷新和 API 反代采集仍是独立链路。主窗口隐藏时，非必要的前台轮询也会暂停。

![Background work before and after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/background-work-before-after.png)

In a same-Mac A-B-B-A comparison against clean v2.4.0 `main`, both builds used the same isolated 229-file corpus and received one appended Token event per minute. The modified build consumed every event while substantially reducing recurring CPU work, attributed reads, and Apple Energy Impact:

在同一台 Mac 上与干净 v2.4.0 `main` 进行 A-B-B-A 对比时，两版使用相同的隔离 229 文件日志集，并每分钟追加一条 Token 事件。修改版读取了全部新增事件，同时显著降低了周期 CPU 工作、归因读取和 Apple Energy Impact：

| Scenario<br>场景 | Coalition CPU<br>ms/min | Attributed reads<br>归因读取字节<br>B/s | Apple Energy Impact/s |
| --- | ---: | ---: | ---: |
| Default one-minute refresh with a growing log<br>默认每分钟刷新（日志增长） | 12,692 → 336 | 291,028,172 → 30 | 664.13 → 5.02 |

These are macOS process-group attribution metrics rather than whole-machine watt-hours. Wakeups did not improve in this short run, so this PR does not claim fewer wakeups.

这些指标是 macOS 对应用进程组的归因统计，而不是整机瓦时。短时测试中的唤醒次数没有改善，因此本 PR 不主张唤醒次数降低。

### 5. Reliable Windows Store account switching / 可靠的 Windows 商店版账号切换

Windows account switching now identifies the actual Codex desktop root—`ChatGPT.exe` inside the `OpenAI.Codex` Store package—and stops only that verified process tree before relaunching. The embedded `resources\codex.exe` app-server is treated as a child, while independent Codex CLI processes, shims, and unrelated ChatGPT applications remain untouched. Legacy `Codex.exe` desktop installations remain supported.

Windows 账号切换现在会识别真正的 Codex 桌面主进程，即 `OpenAI.Codex` 商店包内的 `ChatGPT.exe`，并只在重新启动前结束这棵经过验证的进程树。内嵌的 `resources\codex.exe` app-server 仅作为子进程处理，独立 Codex CLI、shim 及无关 ChatGPT 应用不会受到影响；旧版 `Codex.exe` 桌面安装仍保持兼容。

**Root cause / 根因:** the previous Windows implementation killed every process named `Codex.exe`. In current Store builds that name belongs to the embedded app-server while the surviving UI is `ChatGPT.exe`, so terminating only the backend produced “ChatGPT stopped unexpectedly.” / 旧版 Windows 实现会结束所有名为 `Codex.exe` 的进程；当前商店版中该名称属于内嵌 app-server，而继续存活的界面进程是 `ChatGPT.exe`，因此只终止后端会触发“ChatGPT 已意外停止”。

### 6. Bilingual release descriptions and updater UX / 双语更新说明与更新体验

When bilingual changelog notes exist, the release workflow reuses them for GitHub Release and the in-app updater. Missing notes produce a warning and generic fallback text without blocking a release. Debug-only redacted authorization diagnostics remain excluded from release builds.

存在双语更新日志时，发布流程会将其复用于 GitHub Release 与应用内更新弹窗。缺少说明时仅发出警告并使用通用兜底文字，不阻断发布；脱敏授权诊断仍只存在于调试构建中。

## Related issue / 关联 Issue

Closes #136

## Validation / 验证

- [x] **Build and automated tests / 构建与自动化测试:** frontend lint and production build, Rust check, debug build, and unsigned NSIS release build completed; focused refresh-error tests passed 13/13 and Windows process tests passed 4/4. The full library run passed 189 tests, with only the pre-existing Windows SQLite cleanup lock failure remaining. / 前端检查与生产构建、Rust 检查、调试构建及未签名 NSIS 发布构建均已完成；刷新错误专项测试 13/13、Windows 进程专项测试 4/4 通过。完整库测试通过 189 项，仅保留既有的 Windows SQLite 清理文件锁失败。
- [x] **Windows real-machine validation / Windows 实机验证:** the installed v2.4.0 build ran beside the Store Codex app, preserved the two-account production store, restarted Store Codex successfully during account switching, and hid the macOS-only status-display setting. / 安装后的 v2.4.0 可与商店版 Codex 同时运行，正式双账号库保持不变，切换账号时可正确重启商店版 Codex，并且不会显示仅适用于 macOS 的状态栏设置。
- [x] **Account recovery scenarios / 账号恢复场景:** real cold starts retained both accounts and cached PRO quota despite stale Plus metadata; OAuth reauthorization synchronized the active `auth.json`; success, timeout, network, authorization, rate-limit, service, and invalid-response refresh states were exercised. / 使用陈旧 Plus 元数据进行真实冷启动时，两个账号及 PRO 缓存额度均被保留；OAuth 重新授权可同步当前 `auth.json`；额度成功、超时、网络、授权、限流、服务及响应异常状态均已覆盖。
- [x] **Analytics correctness and performance / 分析正确性与性能:** deterministic tests cover cumulative deltas, counter resets, fork identity and ownership, replay variation, unresolved lineage, local-time and cost windows, heatmap levels, and cache append/reparse/eviction. A real 6.2 GB nested fork retained 156,799 inherited records up to the observed branch boundary and reduced the false maximum heatmap bucket from 4.90B to 109.7M Tokens; the controlled growing-log A-B-B-A run consumed every appended event while reducing recurring work. / 确定性测试覆盖累计差值、计数器回退、fork 身份与归属、回放差异、未解析继承关系、本地时间与成本窗口、热力图色阶及缓存追加、重解析与淘汰。一个真实的 6.2 GB 嵌套 fork 在观察到的分叉边界前识别出 156,799 条继承记录，并将错误的热力图最大格从 4.90B 降至 109.7M Token；受控增长日志 A-B-B-A 测试读取了全部新增事件并降低了周期性开销。
- [x] **Release-note handling / 更新说明处理:** extraction covers versioned notes, Unreleased fallback, and missing-note fallback without blocking the build. / 提取逻辑覆盖指定版本、Unreleased 回退及缺失说明回退，均不阻断构建。
