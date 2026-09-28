# 操作「无 a11y 的 Chromium 应用」（QQ / 微信 / Discord 这类）

## 症状与根因

`ghost_see` 返回 `elements: []`，**连 `filtered_offscreen` 都没有**（对比 ComfyUI Desktop 那次至少有 3 个被过滤的元素）。诊断三步：

```
窗口类名        = Chrome_WidgetWin_1          ← 是 Chromium/Electron
子窗口          = Intermediate D3D Window     ← UI 全在 D3D 渲染内容里
渲染进程命令行  = --disable-features=...      ← 没有任何 accessibility 参数
```

**Electron 默认不构建 a11y 树**，要靠 `--force-renderer-accessibility` 或 app 层调 `setAccessibilitySupportEnabled(true)` 才暴露给 UIA。QQ(NT) 没开这个桥；Edge 能读是因为浏览器必须支持屏幕阅读器。**这不是配置问题，是应用没实现。**

## background 也点不了，Ghost 会明确拒绝

```
ghost: Core error: 'click' has no background path on the user's desktop
and the focus policy is 'background'. Real input on the user's desktop is
the operator's call: GHOST_FOCUS_LOCK=off in the server environment,
then ghost_set_focus_policy
```

原因：界面在 D3D 渲染的 Chromium 内容里，**没有独立 HWND** → `WM_LBUTTONDOWN` / `WM_SETTEXT` / `WM_PASTE` **全都没有投递目标**。所以这类应用必须走真实输入（`foreground`）。

（注意 CLI `ghost.exe` 是**独立进程**、策略默认 locked，它自己升不了级；只有 MCP 服务器那侧配了 `GHOST_FOCUS_LOCK=off` 才能 `ghost_set_focus_policy`。）

## 四段式流程（实测通过：在 QQ 群里发出一条中文消息）

### 1. 读 —— PrintWindow + PW_RENDERFULLCONTENT

```powershell
PrintWindow(hwnd, hdc, 2)     # 2 = PW_RENDERFULLCONTENT
```

- **必须用 flags=2**：D3D 窗口用 `flags=0` 或 `BitBlt` 通常拿到**黑图**
- **自检**：抽样统计非黑像素比例，正常 >90%（实测 99.7%）
- **坐标换算**：自绘标题栏也算在窗口内，所以 **屏幕坐标 = 窗口左上角 + 截图内坐标**（实测两轮一致）
- Ghost 自带的 `ghost_screenshot` 用的是同一原理（响应带 `occlusion_proof: true`），但**它把图以 `jpeg_base64` 字段返回**，落不到文件；要真正"看到"内容，就用上面这段自己落盘再读图

### 2. 点击 —— 坐标 + 真实输入

```
ghost_set_focus_policy(foreground)          # 先升策略
ghost_window op=focus name="QQ"             # 提窗口到前台（background 下只锚定不提升）
ghost_drag(from_x=X, from_y=Y, to_x=X, to_y=Y)   # from==to 即点击
```

**MCP 工具里没有 `ghost_click_at`**，坐标点击用 `ghost_drag` 代替（起止点相同）。

### 3. 输入中文 —— 剪贴板 + 真实 Ctrl+V

```
ghost_clipboard op=set text="..."
ghost_key keys="Ctrl+V"
```

**别逐字模拟**：中文要走 IME，逐字 SendInput 极脆；剪贴板是 Unicode，一次到位。剪贴板可以在**升策略之前**就设好（无风险）。

### 4. 发送 —— 优先点按钮，不要按 Enter

QQ 的"回车发送 / Ctrl+回车发送"**取决于用户设置**，按 Enter 可能只换行。点"发送"按钮更确定。

## 每一步之间都要截图验证

**发出前的"确认输入框内容"这步绝对不能省**——内容错了就是发出去收不回来的。发出后再截一次，确认输入框已清空且消息进入消息流。

## 代价（必须按 focus-policy.md 的纪律提前告知用户）

- 目标窗口会**抢到前台**，用户正在用的窗口失焦
- 期间**真实键鼠被占用**（约 10 秒），用户不能打字
- 做完**立刻降回 `background`**

## 局限

点按靠坐标推算，**界面滚动/布局一变坐标就失效**，每次操作前都要重新截图定位。对比 UIA 路径（能按名字点）稳定性差很多。能用 API 的应用优先用 API。

## 实测记录（2026-09-28）

QQ NT，在"某个群"发一条中文消息。

```
窗口:      (816,419)  960×705      → 截图坐标 + 该偏移 = 屏幕坐标
输入框:    窗口内 (485,637)  → 屏幕 (1301,1056)
发送按钮:  窗口内 (707,675)  → 屏幕 (1523,1094)
```

流程：PrintWindow 读界面 → `ghost_clipboard set` → 升 foreground → `ghost_window focus`
→ `ghost_drag` 点输入框 → `ghost_key Ctrl+V` → **截图确认文字已进输入框** →
`ghost_drag` 点发送 → 截图确认（输入框清空、消息入列、左侧预览更新）→ 降 background。

全程前台停在 QQ（降策略不会自动归还前台；要归还需再升一次策略后 `ghost_window op=focus` 到用户原来的窗口）。
