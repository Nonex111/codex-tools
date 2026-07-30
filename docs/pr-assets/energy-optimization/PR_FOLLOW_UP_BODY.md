<!-- PR title / PR 标题: [EN/ZH] Harden Windows account switching and account recovery / 强化 Windows 账号切换与账号恢复 -->

> Follow-up to #160. This draft contains only changes that were not part of merged head `87ddf0c`.  
> #160 的后续 PR。本草稿仅描述未包含在已合并提交 `87ddf0c` 中的改动。

## Summary / 摘要

- Make Windows account switching restart the verified Store Codex desktop process tree without terminating independent Codex or ChatGPT processes. / Windows 账号切换仅重启经过验证的商店版 Codex 桌面进程树，不影响独立 Codex 或 ChatGPT 进程。
- Improve account recovery wording and diagnostics, display the primary HTTP status, and keep recovery actions out of the compact account-status badge. / 改进账号恢复文案与诊断，显示主要 HTTP 状态码，并避免在紧凑的账号状态提示中塞入恢复操作。
- Explain that membership expiry comes from the sign-in token; show “Not provided / 未提供” and a direct sign-in action when the date is empty. / 说明会员到期时间来自登录令牌；日期为空时显示“未提供”并提供直接重新登录入口。
- Let isolated development previews analyze production sessions through a separate read-only source and cache. / 允许隔离的开发预览通过独立只读来源与缓存分析正式会话。
- Hide macOS-only status-display controls on Windows while preserving the Windows tray behavior and stored settings. / 在 Windows 隐藏仅适用于 macOS 的状态展示控件，同时保留 Windows 托盘行为和已保存设置。

## Changes / 改动

### 1. Reliable Windows account switching / 可靠的 Windows 账号切换

**Root cause / 根因:** the previous implementation terminated every process named `Codex.exe`. Current Store builds use `ChatGPT.exe` inside the `OpenAI.Codex` package as the desktop root, while `resources\codex.exe` is an embedded app-server. Killing only the child left the UI alive and produced “ChatGPT stopped unexpectedly.” / 旧实现会结束所有名为 `Codex.exe` 的进程。当前商店版使用 `OpenAI.Codex` 包内的 `ChatGPT.exe` 作为桌面主进程，`resources\codex.exe` 则是内嵌 app-server；只结束子进程会让界面继续存活并触发“ChatGPT 已意外停止”。

The follow-up now identifies the verified Store Codex root, stops only that process tree, and then relaunches it. Independent Codex CLI processes, shims, unrelated ChatGPT applications, and legacy `Codex.exe` desktop installations remain supported and are not matched by name alone. / 本后续改动会识别经过验证的商店版 Codex 主进程，仅结束该进程树后再重新启动。独立 Codex CLI、shim、无关 ChatGPT 应用及旧版 `Codex.exe` 桌面安装不会再仅按进程名误匹配。

| Failure before the fix / 修改前的失败状态 |
| --- |
| ![Windows account switch failure before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/windows-account-switch-failure-before.png) |

### 2. Account recovery, diagnostics, and membership guidance / 账号恢复、诊断与会员提示

**Root cause / 根因:** the account action still used test-oriented wording even though it starts a real OAuth reauthorization. Composite refresh errors also searched for `401/403` before `5xx`, so a primary `503` followed by a fallback `403` could be misreported as an authorization failure. / 账号操作执行的是真实 OAuth 重新授权，但仍使用测试性质的文案；组合刷新错误也会先匹配 `401/403`、再检查 `5xx`，因此主要请求 `503`、回退请求 `403` 时可能被误报为授权异常。

- Rename **Test login / 测试登录** to **Sign in again / 重新登录**. / 将“测试登录”改为“重新登录”。
- Preserve and display the primary HTTP status, such as `503`. / 保留并显示主要 HTTP 状态码，例如 `503`。
- Keep the red refresh-status badge informational without embedding a sign-in action. Blocking authorization failures preserve the real HTTP status and use the compact form `400: Saved authorization snapshot is invalid; sign in again / 400：保存的授权快照已失效，请重新登录`; other failures retain their summarized status. / 红色刷新状态提示仅用于展示信息，不再内嵌重新登录操作；阻断授权错误保留真实 HTTP 状态码，并使用 `400：保存的授权快照已失效，请重新登录` 的紧凑格式，其他错误继续显示摘要状态。
- Keep both the account-list and detail-header status labels concise: **Needs attention / 异常账号** no longer appends a clipped copy of the error reason. The full coded failure remains in the refresh-status badge and in the status tooltip. / 左侧账号列表与右侧详情标题都只显示“异常账号”，不再追加被裁切的错误原因；带状态码的完整错误仍保留在刷新状态提示及状态悬浮说明中。
- Add one accessible custom membership-expiry tooltip without a duplicate native tooltip. Empty values show **Not provided / 未提供** with a direct **Sign in again / 重新登录** action that reauthorizes the selected account without switching to it first. / 增加一个可访问的自定义会员到期时间提示，不再重复显示原生 tooltip；空值显示“未提供”，并提供直接“重新登录”操作，只重新授权所选账号而不会先切换账号。
- Keep each account’s authorization snapshot isolated. Reauthorizing a non-current account preserves the current account and both membership dates through deduplication and the following all-account refresh. / 保持各账号授权快照隔离；重新登录非当前账号时，经过账号去重及随后的全账号刷新后，当前账号和双方会员日期均保持正确。

The existing #160 comparison below provides the account-detail context extended by this follow-up. / 下方沿用 #160 的账号详情对比图，作为本后续改动继续完善账号恢复与会员提示的界面上下文。

![Plus-to-Pro account resolution before and after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/account-resolution-before-after.png)

| Before / 修改前 |
| --- |
| ![Controlled membership expiry comparison before](https://raw.githubusercontent.com/Nonex111/codex-tools/44e52c7c387954e0fdb8aa3a8b9f304da2cfd3fa/docs/pr-assets/energy-optimization/membership-expiry-controlled-before.png) |

| After / 修改后 |
| --- |
| ![Controlled membership expiry comparison after](https://raw.githubusercontent.com/Nonex111/codex-tools/44e52c7c387954e0fdb8aa3a8b9f304da2cfd3fa/docs/pr-assets/energy-optimization/membership-expiry-controlled-after.png) |

### 3. Read-only production analytics in isolated previews / 隔离预览只读分析正式日志

**Root cause / 根因:** isolating all Codex data kept development previews safe but left Analytics empty. Sharing production session storage directly would mix writable cache state and expose destructive session deletion. / 隔离全部 Codex 数据虽然保证了开发预览安全，却会使分析页为空；直接共享正式会话目录又会混用可写缓存，并暴露破坏性会话删除风险。

- Add debug-only `CODEX_TOOLS_DEV_ANALYTICS_DIR` as a read-only analytics source. / 增加仅用于调试构建的只读分析来源 `CODEX_TOOLS_DEV_ANALYTICS_DIR`。
- Keep preview account data and writable Codex state isolated, with a separate analytics cache. / 继续隔离预览账号数据及可写 Codex 状态，并使用独立分析缓存。
- Reject session deletion while the read-only source is active. / 启用只读来源时拒绝删除会话。
- Keep release-build analytics behavior and the merged Token, fork-lineage, cost, and heatmap algorithms unchanged. / 保持发布构建分析行为以及已合并的 Token、fork 继承、成本和热力图算法不变。

### 4. Platform-specific status settings / 平台专属状态设置

The macOS menu-bar usage-title controls were visible on Windows even though the Windows tray does not support these display modes. The controls now render only on macOS; Windows keeps its normal tray icon and Open/Quit menu, and hidden settings remain preserved. / macOS 菜单栏用量标题控件此前会显示在 Windows，但 Windows 托盘并不支持这些展示模式。现在仅在 macOS 渲染该控件；Windows 继续保留常规托盘图标及打开/退出菜单，隐藏后也不会清除已保存设置。

| Before / 修改前 |
| --- |
| ![Windows status settings before](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/windows-status-settings-before.png) |

| After / 修改后 |
| --- |
| ![Windows status settings after](https://raw.githubusercontent.com/Nonex111/codex-tools/codex/energy-optimization/docs/pr-assets/energy-optimization/windows-status-settings-after.png) |

## Related work / 关联项

- Follow-up to #160 / #160 的后续改进
- Related to #159 / 关联 #159（账号重新授权与状态同步）
- #136 was already addressed by #160; this follow-up does not change the server-provided membership-expiry claim. / #136 已由 #160 处理；本后续 PR 不改变服务端提供会员到期字段的行为。

## Validation / 验证

- [x] **Frontend / 前端:** the touched frontend file passed ESLint, TypeScript checking passed, and the production Vite build completed. Full-repository ESLint still reports four pre-existing `react-hooks/set-state-in-effect` errors in untouched files. / 本次修改的前端文件通过 ESLint，TypeScript 检查及 Vite 生产构建通过；全仓 ESLint 仍报告 4 个位于未修改文件中的既有 `react-hooks/set-state-in-effect` 错误。
- [x] **Non-current reauthorization / 非当前账号重新授权:** the focused regression passed 1/1. Account A remains current while account B is reauthorized without switching; deduplication and the next refresh preserve both authorization snapshots and membership dates. / 专项回归 1/1 通过：账号 A 保持当前账号，账号 B 在不切换的情况下重新授权；账号去重及下一次刷新均保留双方授权快照与会员日期。
- [x] **Account service / 账号服务:** all focused account-service tests passed 23/23. / 账号服务专项测试 23/23 通过。
- [x] **Refresh and Windows process regressions / 刷新与 Windows 进程回归:** refresh-error tests passed 10/10 and focused Windows process tests passed 4/4. / 刷新错误测试 10/10、Windows 进程专项测试 4/4 通过。
- [x] **Windows real-machine validation / Windows 实机验证:** the local build ran beside Store Codex, preserved the two-account production store, restarted Store Codex successfully, and hid the macOS-only setting. / 本地构建可与商店版 Codex 同时运行，正式双账号库保持不变，可正确重启商店版 Codex，并隐藏仅适用于 macOS 的设置。

## Notes / 说明

- This PR should target upstream `main` after merge commit `689f1a5`. / 本 PR 应以合并提交 `689f1a5` 之后的上游 `main` 为目标分支。
- Membership expiry is a best-effort ID-token claim. Signing in again can request a fresh claim but cannot guarantee that the server returns one. / 会员到期时间是尽力读取的 ID token 字段；重新登录可以请求新字段，但无法保证服务端一定返回。
- The original merged PR body remains separate; this draft covers only post-merge follow-up changes. / 原已合并 PR 正文保持独立，本草稿仅覆盖合并后的后续改动。
