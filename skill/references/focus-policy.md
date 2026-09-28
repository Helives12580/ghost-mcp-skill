# 焦点策略与三条纪律

## 三个策略（语义以 README 634-648 / CHANGELOG 219、246、533 为依据）

| 策略 | 行为 |
|---|---|
| `background`（默认） | 只走后台路径。真 Win32 控件用投递消息（`BM_CLICK` / `WM_LBUTTONDOWN·UP` / `WM_SETTEXT` / `WM_MOUSEWHEEL`），无窗口控件（UWP/WinUI/Chromium/Electron）用 UIA pattern（`Invoke` / `ValuePattern`）。没有后台路径的操作**直接报错**，错误里给出不需要改策略的替代路线。 |
| `prefer_background` | 优先后台，**必要时自动回落到前台**（会真的 raise 窗口）。省心，但你无法预知哪一次调用会碰真实输入。 |
| `foreground` | 直接以前台方式驱动：先把目标窗口提到前台（AttachThreadInput + 确认）再操作。 |

`GHOST_FOCUS_LOCK=off` 是解锁开关。锁着时 `ghost_set_focus_policy` 只接受 `background`，另两个返回 `FocusLocked`。本机当前配置就是 `off` + 起始 `background`。

**本机约定（操作者 定）：起始策略保持 `background` 不动，只有他明确要求时才升级。**

## 三条纪律（硬性）

1. **升降成对** —— 升到 `foreground`/`prefer_background` 后，必须在**同一个流程内**设回 `background`。不许让它跨会话留着。
2. **升级前先告知** —— 他可能在打字。要用真实键鼠之前，先说清"这一步要用真实鼠标/键盘了"，让他有机会避开。
3. **优先走替代路线** —— 先找后台路线；找不到才升；升完立刻降回。

## background 的硬边界（没有后台路径的四类）

1. **修饰键组合** —— 后台只能处理 `Ctrl+C/X/V/Z/A` 这五个（翻译成语义消息 `WM_COPY`/`WM_CUT`/`WM_PASTE`/`WM_UNDO`/`EM_SETSEL`）。**之外的组合被明确拒绝**（`Ctrl+S`、`Ctrl+F`、`Ctrl+Shift+方向键`、`Alt+F4`、`Win+方向键`…），因为投递消息无法设置应用读取的修饰键状态。
2. **拖拽** —— `ghost_drag` 需要真实光标轨迹。
3. **只认真实硬件输入的程序** —— 游戏、模拟器、DirectInput 软件、部分全屏应用。注意：真实 `SendInput` 在隐藏桌面上**不可用**（Windows 拒绝在非输入桌面发），所以这类程序必须在用户自己的桌面上、且用 `foreground`。
4. **需要真实鼠标坐标的画布交互** —— 绘图/3D/地图软件；`right_click` / `double_click` / `hover` 打在无窗口控件（Chromium/UWP 元素）上也没有后台路径。

## 替代路线优先表（先查这张表再考虑升级）

| 想做的事 | background 下的正确路线 |
|---|---|
| 浏览器里按 Ctrl+S / Ctrl+F 等修饰键组合 | `ghost_tab_press(key, modifiers)` —— **CDP 支持完整修饰键**，不走 SendInput |
| 浏览器里点按钮 / 填表单 | `ghost_tab_click`(selector) / `ghost_tab_type`(selector, text) |
| 浏览器里做复杂操作 | `ghost_tab_eval`(expression) —— 页面 JS 上下文执行任意代码 |
| ComfyUI 画布操作（加节点/连线/布局） | `ghost_tab_eval` + JS，见 `comfyui-canvas.md`。**不需要升级** |
| 启动程序 | `ghost_window op=launch`（落在隐藏桌面，连激活事件都不产生） |
| 有 CLI / API 程序的自动化 | 直接调它的 CLI/API，别点 UI |
| 读任意窗口内容 | `ghost_see mode=text` |
| 读被遮挡窗口的像素 | `ghost_screenshot window=...`（`occlusion_proof: true`，PrintWindow 渲染） |

## 升级的代价（有实测数据，不是理论风险）

README 里的数字：在真人于另一个窗口持续打字的条件下，**0.23 之前约 490 次击键有 48 次送进错误的窗口**；0.23 之后六次运行分别是 0、0、0、1、0、0。开锁状态下**几乎不可能完全避免**偶发击键错位——Ghost 自己的结论是"撤销一次激活永远不可能是保证，要保证就让 Ghost 自己启动这个程序"。另外真实光标会跳，用户能直接看见。

## 查当前状态

`ghost_focus_policy` 返回 `policy` 与 `locked`；`ghost_session_state` 返回策略 + 真实光标位置 + 干扰审计（`synthetic_foreground_changes` 应始终为 0，否则说明有东西在合成前台切换）。
