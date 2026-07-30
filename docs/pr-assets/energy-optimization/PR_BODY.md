<!-- PR title / PR 标题: [EN/ZH] Improve the macOS status bar and reduce background energy use / 优化 macOS 状态栏并降低后台耗电 -->

> **Draft status / 草稿状态:** This local draft integrates the current branch with the earlier remote draft in PR #160. Some current-branch changes have not yet been pushed to the PR. / 本地草稿已整合当前分支与 PR #160 的远端旧草稿；当前分支仍有部分改动尚未推送到该 PR。

## Summary / 摘要

- **Improve platform-specific desktop behavior / 改善不同平台的桌面体验:** enhance the macOS status item, show its display controls only where they apply, and fix Windows Store Codex restarts during account switching.
- **Make account information faster and more reliable / 让账号信息更快、更可靠:** render cached quota immediately, resolve Plus-to-PRO identity drift, show reference-only membership expiry, and distinguish recoverable authorization failures from service errors.
- **Unify local analytics / 统一本机分析口径:** calculate real Token increments consistently, handle fork lineage conservatively, align totals and heatmap costs, and protect production sessions during development previews.
- **Reduce recurring work / 降低周期性后台开销:** remove inference keepalive work, incrementally process changed logs every minute, and pause unnecessary polling while the window is hidden.
- **Improve release communication / 改善发布信息:** reuse bilingual changelog notes in GitHub Releases and the in-app updater without blocking releases when notes are missing.

## Changes / 改动

### 1. macOS status item and platform-specific settings / macOS 状态栏与平台专属设置

The macOS status item now uses the color application icon, defaults to one-week remaining usage, optionally shows `5h / 1w` labels, and disappears completely when “Hidden” is selected. Account meters retain the semantic `5h` and `1w` labels even when both periods currently report the same value.

macOS 状态栏现在使用彩色应用图标，默认显示一周剩余用量，可选择显示 `5h / 1w` 标签，并在选择“不显示”时完整隐藏。即使两个周期当前数值相同，账号用量栏仍分别保留 `5h` 与 `1w` 的语义标签。

These controls change the macOS menu-bar title rather than the basic Windows tray menu, so the entire setting row is now shown only on macOS. Windows keeps its tray icon and Open/Quit menu without showing controls that have no effect there.

这些选项控制的是 macOS 菜单栏标题，而不是 Windows 的基础托盘菜单，因此整行设置现在仅在 macOS 上显示。Windows 仍保留托盘图标及打开、退出菜单，但不会再显示无实际作用的选项。

**Appearance / 外观**

| Before / 修改前 | After / 修改后 |
| --- | --- |
| ![Status item before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/status-item-before.png) | ![Status item after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/status-item-after.png) |

**Settings / 设置**

| Before / 修改前 |
| --- |
| ![Status settings before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/status-settings-before.png) |

| After / 修改后 |
| --- |
| ![Status settings after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/status-settings-after.png) |

**Usage-window labels / 用量周期标签**

| Before / 修改前 | After / 修改后 |
| --- | --- |
| ![Usage labels before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/usage-labels-before.png) | ![Usage labels after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/usage-labels-after.png) |

### 2. Faster startup and clearer account freshness / 更快的启动与更清晰的账号状态

Returning installations render locally stored accounts and the latest quota snapshot immediately while remote quota refresh starts concurrently. During the first refresh, the UI identifies cached data as updating; success clears the badge, while failure or unavailable data remains visible with a concise reason and the complete error on hover.

已有本地数据时，应用会立即显示保存的账号和最新额度快照，同时并发刷新远端额度。首次刷新期间，界面会明确提示缓存数据正在更新；刷新成功后提示消失，失败或暂无数据时则保留状态，并直接显示简短原因，悬停后可查看完整错误。

A clean installation without a saved snapshot still waits for the first remote result. Cached values may also remain temporarily stale until the refresh finishes.

没有历史快照的全新安装仍需等待首次远端结果；已有缓存的数值在刷新完成前也可能暂时不是最新状态。

| Before / 修改前 |
| --- |
| ![Account usage loading before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/account-usage-loading-before.png) |

| After / 修改后 |
| --- |
| ![Account usage loading after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/account-usage-loading-after.png) |

### 3. Account identity, membership, and recoverable errors / 账号身份、会员信息与可恢复错误

**Plan identity and authorization recovery / 套餐识别与授权恢复**

Account matching now prioritizes stable identity over mutable plan metadata. If `auth.json` still reports Plus while the saved account and live quota are already PRO, startup reuses the PRO record and its cached quota instead of creating an empty Plus variant. A cooldown-protected token refresh may repair a known plan conflict; if ordinary refresh still cannot update stale identity claims, OAuth reauthorization synchronizes the new credentials back to the active `auth.json`.

账号匹配现在优先使用稳定身份，而不是容易变化的套餐字段。如果 `auth.json` 仍显示 Plus，而已保存账号和实时额度已经是 PRO，启动时会复用原有 PRO 记录及其缓存额度，不再创建空白 Plus 变体。遇到已知套餐冲突时，应用可在冷却限制下刷新一次令牌；若普通刷新仍无法更新陈旧的身份声明，OAuth 重新授权会把新凭据同步回当前 `auth.json`。

![Plus-to-Pro account resolution before and after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/account-resolution-before-after.png)

**Reference-only membership expiry / 仅供参考的会员到期时间**

The account detail card shows a future `chatgpt_subscription_active_until` ID-token claim when available. Missing or past claims are not presented as valid expiry dates, and the field is labeled **for reference only** because it may remain stale after a plan change. This is separate from the reset-card count and expiry requested in Issue #136.

账号详情会在存在有效未来数值时显示 ID token 中的 `chatgpt_subscription_active_until`。缺失或已过去的声明不会作为有效到期时间展示；由于套餐变化后该值仍可能陈旧，字段会明确标注为**仅供参考**。它与 Issue #136 中的重置卡数量及过期时间相互独立。

| Before / 修改前 |
| --- |
| ![Controlled membership expiry comparison before](https://raw.githubusercontent.com/Nonex111/codex-tools/44e52c7c387954e0fdb8aa3a8b9f304da2cfd3fa/docs/pr-assets/energy-optimization/membership-expiry-controlled-before.png) |

| After / 修改后 |
| --- |
| ![Controlled membership expiry comparison after](https://raw.githubusercontent.com/Nonex111/codex-tools/44e52c7c387954e0fdb8aa3a8b9f304da2cfd3fa/docs/pr-assets/energy-optimization/membership-expiry-controlled-after.png) |

**Actionable refresh failures / 可操作的刷新失败提示**

Refresh failures now preserve the primary HTTP status, such as `503`, instead of allowing a fallback `403` response to hide a service outage. Reauthorization is offered only for authentication states that signing in again can resolve, such as `401`, expired or revoked tokens, `invalid_token`, and `invalid_grant`. Outages, rate limits, timeouts, network failures, generic HTML `403` responses, and disabled accounts no longer incorrectly suggest reauthorization.

刷新失败现在会保留主要请求的 HTTP 状态码（例如 `503`），不会再让回退请求中的 `403` 掩盖服务中断。只有 `401`、令牌过期或吊销、`invalid_token`、`invalid_grant` 等可通过重新登录解决的状态才会显示重新授权；服务不可用、限流、超时、网络故障、通用 HTML `403` 及停用账号不会再被错误引导重新登录。

**Root causes / 根因:** plan metadata was previously treated as part of account identity, and the refresh classifier searched for `401/403` before `5xx` without separately deciding whether reauthorization was useful. / 此前套餐字段被视为账号身份的一部分，同时刷新分类器会先搜索 `401/403`、再检查 `5xx`，也没有独立判断重新授权是否真正有帮助。

### 4. Consistent local analytics and safe previews / 一致的本机分析与安全预览

Token summaries and detailed analytics now derive real increments from cumulative snapshots: unchanged counters contribute zero, monotonic increases contribute only component-wise deltas, and counter resets remain visible as anomalies. Total, seven-day, project, session, prompt, and heatmap values all use the same confirmed local-log deltas and model-pricing estimate, so complete project details add up to the displayed total.

Token 汇总与详细分析现在根据累计快照计算真实增量：计数不变时计为零，单调增长时只计算各 Token 分量的差值，计数器回退则保留为异常。总量、7 日、项目、会话、prompt 与热力图统一使用已确认的本机日志增量及同一套模型计价估算，因此完整项目明细可与显示总成本相加一致。

Forked files keep the immutable identity from their first physical `session_meta`. History inheritance (`forked_from_id`) is separated from agent ownership (`parent_thread_id`), and only a verified direct-parent record range is excluded. The matcher tolerates small replay insertions, omissions, regenerated IDs, and known default fields, but stops conservatively at the branch boundary. Missing, ambiguous, cyclic, or otherwise unverifiable lineage is excluded from confirmed totals and reported as unresolved.

fork 文件始终保留首条物理 `session_meta` 中的不可变身份。历史继承（`forked_from_id`）与 Agent 归属（`parent_thread_id`）分开处理，仅排除经验证的直接父会话记录区间。匹配器可以容忍回放时少量记录插入或缺失、重新生成的 ID 及已知默认字段，但会在真实分叉边界保守停止；父日志缺失、关系冲突、成环或无法验证时，不会计入已确认总量，并会显示为未解析。

These values are local-log Tokens and estimated API-equivalent costs—not official ChatGPT quota/Profile activity or API-proxy upstream usage. The seven-day card uses the previous seven completed local calendar days, while the budget alert uses the rolling latest 168 hours. The heatmap uses local time, localized tooltips, compact K/M values, and nine logarithmic levels relative to the largest one-hour bucket in the current view.

这些数值是本机日志 Token 与 API 等值成本估算，不是 ChatGPT 官方额度/Profile 活动量，也不是 API 反代上游用量。7 日卡片采用前 7 个完整本地自然日，预算预警采用滚动最近 168 小时。热力图按本地时间显示，提供本地化提示与 K/M 紧凑数值，并按照当前视图最大单小时值分配九档对数相对色阶。

| Before / 修改前 |
| --- |
| ![Controlled heatmap simulation before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/analytics-heatmap-controlled-before.png) |

| After / 修改后 |
| --- |
| ![Controlled heatmap simulation after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/analytics-heatmap-controlled-after.png) |

Development previews keep their account store and writable Codex state isolated, but may read production session logs through `CODEX_TOOLS_DEV_ANALYTICS_DIR`. They use a separate analytics cache and refuse session deletion while this read-only source is active, preventing Analytics-page testing from modifying production sessions.

开发预览继续隔离账号库及可写 Codex 状态，但可以通过 `CODEX_TOOLS_DEV_ANALYTICS_DIR` 读取正式会话日志。预览版使用独立分析缓存，并在只读来源启用时拒绝删除会话，避免测试分析页面时修改正式会话。

### 5. New log refresh strategy and lower resource use / 新的日志刷新策略，大幅降低能耗与资源占用

The app no longer sends an inference keepalive. After each process completes its initial scan, unchanged logs reuse parsed in-memory results and append-only files resume from their previous byte offset. Truncated, rewritten, or tail-mismatched files fall back to reparsing that file. The latest aggregate snapshot is persisted for fast display after restart, while the per-file index is rebuilt on the new process's first scan.

应用不再发送推理保活请求。每次进程完成首次扫描后，未变化日志会复用内存解析结果，仅追加文件则从上次字节位置继续读取；文件被截断、重写或尾部校验不匹配时，会回退为重新解析该文件。最新聚合快照会持久化，供重启后快速显示；逐文件索引则会在新进程首次扫描时重新建立。

Detailed local-log analytics follows this incremental path automatically every 60 seconds. Entering the Analytics page does not trigger another scan, while **Refresh analytics** remains available for an immediate update. This schedule affects only local session-log analysis; account quota refresh and API-proxy collection remain independent. Nonessential foreground polling also pauses while the main window is hidden.

本机会话日志分析固定每 60 秒执行一次上述增量刷新；进入分析页不会额外启动扫描，同时保留“刷新分析”用于立即更新。该周期只影响本机会话日志分析，账号额度刷新和 API 反代采集仍是独立链路。主窗口隐藏时，非必要的前台轮询也会暂停。

![Background work before and after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/background-work-before-after.png)

The modified build and clean v2.4.0 `main` build were measured on the same Mac in A-B-B-A order with isolated app data and the same credential-free 229-file corpus. Each valid round ran for 10 minutes with one Token event appended per minute; the modified build consumed all nine appended events in both rounds. Recurring Coalition CPU fell by about 97%, attributed reads by nearly 100%, and Apple Energy Impact by about 99%.

修改版与干净的 v2.4.0 `main` 构建在同一台 Mac 上按 A-B-B-A 顺序测试，使用隔离的应用数据和同一份不含凭据的 229 文件日志集。每个有效轮次持续 10 分钟，每分钟追加一条 Token 事件；修改版两轮均读取了全部 9 条新增事件。周期 Coalition CPU 约下降 97%，归因读取接近下降 100%，Apple Energy Impact 约下降 99%。

| Scenario<br>场景 | Coalition CPU<br>ms/min | Attributed reads<br>归因读取字节<br>B/s | Apple Energy Impact/s |
| --- | ---: | ---: | ---: |
| Default one-minute refresh with a growing log<br>默认每分钟刷新（日志增长） | 12,692 → 336 | 291,028,172 → 30 | 664.13 → 5.02 |

These are macOS process-group attribution metrics rather than whole-machine watt-hours. Coalition wakeups increased from 269 to 1,026 per minute in this short run, so this PR claims lower recurring CPU work, attributed reads, and Energy Impact—not fewer wakeups.

这些指标是 macOS 对应用进程组的归因统计，而不是整机瓦时。本次短时测试中的 Coalition 唤醒由每分钟 269 次增至 1,026 次，因此本 PR 只主张周期 CPU 工作、归因读取及 Energy Impact 降低，不主张唤醒次数减少。

### 6. Reliable Windows account switching / 修复 Windows 账号切换时 ChatGPT 应用报错

**Root cause / 根因:** the previous Windows implementation killed every process named `Codex.exe`. After the Store Codex moved to its current `ChatGPT.exe` host, `Codex.exe` became the embedded app-server while the UI process remained alive, so terminating only the backend produced “ChatGPT stopped unexpectedly.” / 旧版 Windows 实现会结束所有名为 `Codex.exe` 的进程。商店版 Codex 更新为当前由 `ChatGPT.exe` 承载的结构后，`Codex.exe` 变成内嵌 app-server，而界面进程仍继续运行，因此只终止后端会触发“ChatGPT 已意外停止”。

Windows account switching now identifies the actual Codex desktop root—`ChatGPT.exe` inside the `OpenAI.Codex` Store package—and stops only that verified process tree before relaunching. The embedded `resources\codex.exe` app-server is treated as a child, while independent Codex CLI processes, shims, and unrelated ChatGPT applications remain untouched. Legacy `Codex.exe` desktop installations remain supported.

Windows 账号切换现在会识别真正的 Codex 桌面主进程，即 `OpenAI.Codex` 商店包内的 `ChatGPT.exe`，并只在重新启动前结束这棵经过验证的进程树。内嵌的 `resources\codex.exe` app-server 仅作为子进程处理，独立 Codex CLI、shim 及无关 ChatGPT 应用不会受到影响；旧版 `Codex.exe` 桌面安装仍保持兼容。

### 7. Bilingual release notes and updater messages / 双语发布说明与应用内更新信息

When bilingual changelog notes exist, the release workflow reuses them for GitHub Release and the in-app updater. Missing notes produce a warning and generated fallback text without blocking a release. Debug-only redacted authorization diagnostics remain excluded from release builds.

存在双语更新日志时，发布流程会将其复用于 GitHub Release 与应用内更新弹窗。缺少说明时仅发出警告并生成兜底文字，不会阻断发布；脱敏授权诊断仍只存在于调试构建中。

## Related issue / 关联 Issue

Closes #136

## Validation / 验证

- [x] **Builds / 构建:** frontend lint and production build, TypeScript check, Rust check, debug build, and unsigned NSIS release build completed. / 前端检查与生产构建、TypeScript 检查、Rust 检查、调试构建及未签名 NSIS 发布构建均已完成。
- [x] **Focused regressions / 专项回归:** refresh-error tests passed 13/13 and Windows process tests passed 4/4; the full library run passed 189 tests, with only the pre-existing Windows SQLite cleanup lock failure remaining. / 刷新错误测试 13/13、Windows 进程测试 4/4 通过；完整库测试通过 189 项，仅保留既有的 Windows SQLite 清理文件锁失败。
- [x] **Windows real-machine test / Windows 实机测试:** the installed v2.4.0 build ran beside the Store Codex app, preserved the two-account production store, restarted Store Codex successfully during account switching, and hid the macOS-only status-display setting. / 安装后的 v2.4.0 可与商店版 Codex 同时运行，正式双账号库保持不变，切换账号时可正确重启商店版 Codex，并且不会显示仅适用于 macOS 的状态栏设置。
- [x] **Account recovery / 账号恢复:** real cold starts retained both accounts and cached PRO quota despite stale Plus metadata; OAuth reauthorization synchronized the active `auth.json`; success, timeout, network, authorization, rate-limit, service, and invalid-response states were exercised. / 使用陈旧 Plus 元数据进行真实冷启动时，两个账号及 PRO 缓存额度均被保留；OAuth 重新授权可同步当前 `auth.json`；额度成功、超时、网络、授权、限流、服务及响应异常状态均已覆盖。
- [x] **Analytics correctness / 分析正确性:** deterministic tests cover cumulative deltas, counter resets, fork identity and ownership, replay variation, unresolved lineage, local-time and cost windows, heatmap levels, and cache append/reparse/eviction. / 确定性测试覆盖累计差值、计数器回退、fork 身份与归属、回放差异、未解析继承关系、本地时间与成本窗口、热力图色阶及缓存追加、重解析与淘汰。
- [x] **Large-log and energy tests / 大日志与能耗测试:** a real 6.2 GB nested fork retained 156,799 inherited records up to the observed branch boundary and reduced the false maximum heatmap bucket from 4.90B to 109.7M Tokens; the controlled A-B-B-A run consumed every appended event while reducing recurring work. / 一个真实的 6.2 GB 嵌套 fork 在观察到的分叉边界前识别出 156,799 条继承记录，并将错误的热力图最大格从 4.90B 降至 109.7M Token；受控 A-B-B-A 测试读取了全部新增事件并降低了周期性开销。
- [x] **Release-note handling / 更新说明处理:** extraction covers versioned notes, Unreleased fallback, and missing-note fallback without blocking the build. / 提取逻辑覆盖指定版本、Unreleased 回退及缺失说明回退，均不阻断构建。
