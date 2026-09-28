---
name: ghost-desktop
description: >-
  用 mcp__ghost__* 工具直接操作本机 Windows 桌面：驱动没有 API 的 GUI 程序（安装器、WPF/Electron/UWP、
  厂商面板）、读任意窗口的控件树与文本、管窗口与进程、跑隐形终端、驱动已登录的浏览器标签页。
  当需求必须靠“点界面 / 读屏 / 操作系统级窗口”才能完成（pwsh 与文件工具够不到的东西）时使用。
---

# Ghost 桌面自动化（本机已装 v0.23.4）

## 本机事实

- 二进制目录 `<ghost目录>`：`ghost.exe`（CLI，已加入用户 PATH）、`ghost-mcp.exe`（stdio MCP 服务器，已挂进 dsh）、`ghost-http.exe`（REST，未常驻）。
- dsh 挂载点在 `~/.dsh/profiles/web/cordis.patch.yml` 的 `mcp-ghost` 条目，工具名为 `mcp__ghost__<tool>`，Windows 上共 54 个。
- 策略：focus policy = `background`，`GHOST_FOCUS_LOCK=off`（已解锁）。**起始策略按 操作者 的约定保持不变，只有他明确要求时才升级。**
- 自检命令：`ghost doctor`（本机 7 项全 PASS）。
- references：`tools.md`（54 个工具分类速查）、`focus-policy.md`（三个策略语义、background 的四类硬边界、替代路线优先表、三条纪律）、`comfyui-canvas.md`（画布协作：CDP+JS 载入/排布/接线/导出，含实测记录）、`no-a11y-apps.md`（QQ/微信/Discord 这类不暴露 a11y 的 Chromium 应用：截图读屏 + 坐标点击 + 剪贴板输入的四段式流程）、`nwjs-games.md`（RPG Maker MV/MZ 等 NW.js 游戏：版本分水岭、CDP 直达游戏对象、用浏览器拉起 HTML 的代价）。

## 铁律

1. **先锚定，再操作。** 第一次调用带 `window="标题子串"`，之后省略 `window=`：会话锁定那个窗口（anchor 跟着 hwnd 走，标题改了也能找回并返回 `title_drift`）。不要依赖“前台窗口”——那可能是用户正在打字的地方。
2. **看 `verified`，不是看 `ok`。** `verified: true` 才代表动作真的生效（读回控件值或像素比对）。`focus_preserved` / `cursor_preserved` 说明有没有惊动用户的鼠标键盘。
3. **读屏优先 `ghost_see mode=text`**（比截图省约 10 倍 token），结构不够时才 `ghost_screenshot`。
4. **能用 `pwsh` / `read` / `edit` 做的事，别走 ghost。** MCP 往返更贵，ghost 只用于“必须先有界面”的任务。
5. **急停**：用户按 `Ctrl+Alt+G` 冻结一切动作（只读查询仍可用）；冻结后用 `ghost_reset` 恢复。
6. **不抢焦点，守三条纪律。**（1）升降成对：升到前台后必须在同一流程内设回 `background`；（2）用真实键鼠前先告知 操作者——他可能在打字；（3）优先走替代路线：浏览器里的修饰键组合用 CDP 的 `ghost_tab_press`（支持完整修饰键），根本不用开锁。完整语义、硬边界与替代路线表见 `references/focus-policy.md`；ComfyUI 画布操作走 CDP+JS，同样不需要开锁，见 `references/comfyui-canvas.md`。

## 标准循环

```
ghost_window op=list                                  # 挑窗口
ghost_see window="标题" [mode=text|full]              # 拿控件（name / role / rect）
ghost_act window="标题" name="按钮名" role="button" action="click"
ghost_assert / ghost_wait for=element|value           # 断言与等待
```

`verified: false` 时先重看 `ghost_see`，不要盲目重试同一个动作。

## 启动程序

`ghost_window op=launch exe="app.exe"` 默认落在**隐藏桌面**（用户看不见、不抢焦点），返回 `pid` 与 `surface`。

- `surface: "hidden"` —— 在隐藏桌面，照常用 `window=标题` 驱动。
- 返回时 `window` 常为 `null`（慢启动程序 5 秒内不出窗）：用 `ghost_window op=list` 确认出现后再驱动。
- Win11 的记事本 / 画图 / 计算器 / 资源管理器是**单实例**：`launch` 会 handoff 到用户桌面上已运行的实例，返回 `surface: "user"`。要真隔离就换非单实例程序（如 `dxdiag.exe`），或先让用户关掉它。

## 常见坑

- 元素名常带快捷键后缀，用模糊匹配即可：`name="下一页"` 能命中 `下一页(N)`。
- 同名元素多个时加 `index=N`（0-based），响应的 `matches` 给出总数。
- UWP / WinUI / Chromium 控件没有窗口句柄，动作走 UIA pattern；Chromium 可能短暂激活自身窗口，Ghost 会在 30–50 ms 内把前台还回去（响应里的 `focus_guard` 记录此事）。
- 浏览器标签：对带 `--remote-debugging-port` 启动的 Chrome / Edge / Brave / Comet 用 `ghost_tab_*` 走 CDP，比 UIA 稳得多。
- 隐藏桌面上 `SendInput` 不可用（Windows 拒绝），只认真实硬件输入的目标才需要 `foreground` 策略。

## 不要做

- 不要对用户正在打字的窗口做非必要的点击。
- 不要用 `ghost_shell` 替代 `pwsh`（能力重叠，且它的输出被 tail 截断）。
- 不要留垃圾进程或窗口：测试用的东西自己关掉（`ghost_window op=state state=close name="..."`）。
