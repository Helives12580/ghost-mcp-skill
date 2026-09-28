# Ghost MCP 安装与配置指南

> 基于版本 **0.23.4**（Windows）的一次完整实测整理。
> 重点在**参数配置**与**踩坑**，尤其是最容易劝退新人的那个问题——
> **「Ghost 为什么抢我的鼠标和键盘？」**
>
> [English version](Ghost-MCP-Install-and-Config-Guide.md)

---

## 0. 先回答那个最常被问的问题

**问：装了 Ghost 之后，它开始抢我的鼠标、键盘还乱跳，怎么办？**

**答：因为你的配置「解锁」了焦点锁，而 Ghost 遇到没有后台操作路径的目标时，会升级到前台策略，用真实键鼠去操作。**

这是设计行为，不是 bug——但它默认**不该发生**。要彻底避免：

- **不要**给 Ghost MCP 配置 `GHOST_FOCUS_LOCK=off`（除非你确实要它操作真实键鼠）
- 保持默认（锁定状态），它就只能走后台路径，**永远不会碰你的鼠标**

详见 [第三节](#三焦点策略抢鼠标的根源)。

---

## 一、Ghost 是什么

给 Agent 用的**桌面自动化层**。核心卖点与别的方案不同：

| 特性 | 说明 |
|---|---|
| **后台驱动** | 通过投递窗口消息 + UI Automation，**不抢焦点、不动光标**（0.19 起强制，0.22 起锁定） |
| **每个动作都验证** | 返回 `verified` / `focus_preserved` / `cursor_preserved`，不是盲目的 `ok:true` |
| **无需 API 也能驱动** | Win32 / WPF / Electron / UWP / 浏览器，应用不必配合 |
| **不需要视觉模型 key** | 模型自己读 `ghost_see` 的元素树或截图 |
| **三件套** | `ghost-mcp`（MCP 服务器）、`ghost`（CLI）、`ghost-http`（REST） |

仓库：<https://github.com/NORTHTEKDevs/ghost>（MIT）

---

## 二、安装

### 2.1 下载

从 Releases 页面取 `ghost-windows-x64.zip`，解压到任意目录（**不建议放系统盘根目录**，见坑 #12）。

解压后得到三个可执行文件：

```
ghost-mcp.exe    ← MCP 服务器（主要用的）
ghost.exe        ← CLI，脚本用
ghost-http.exe   ← REST API，可选
```

**建议把该目录加入 PATH**（用户级即可），之后 CLI 可以直接调。

### 2.2 先自检（重要）

```powershell
ghost.exe doctor
```

会逐项检查：Windows 版本、交互桌面、UI Automation、DPI 感知、显示器、截图能力。
**全 PASS 才继续**，有 FAIL 会直接告诉你哪里不对。可选警告（如"没有视觉 key"）可以忽略。

### 2.3 挂载到 MCP 客户端

**Claude Desktop**：下载 `ghost-windows-x64.mcpb`，*Settings → Extensions → Install from file*。

**Claude Code**：

```bash
claude mcp add ghost --scope user -- C:/path/to/ghost-mcp.exe
```

**通用 MCP 客户端 / 自建 harness**（stdio 服务器）：

```json
{
  "mcpServers": {
    "ghost": {
      "command": "C:/path/to/ghost-mcp.exe",
      "env": {
        "GHOST_FOCUS_LOCK": "off",
        "GHOST_FOCUS_POLICY": "background"
      }
    }
  }
}
```

> ⚠️ 上面这段 `env` **就是抢鼠标的开关**。不需要真实键鼠的话，**整段 env 删掉**。

**注意**：不同客户端的配置字段名可能不同（有的叫 `mcpServers`，有的在 profile 配置文件里）。挂载后工具会以 `mcp__ghost__<工具名>` 出现在工具列表中，Windows 上共 **54 个**。

---

## 三、焦点策略（抢鼠标的根源）

### 3.1 三个策略

| 策略 | 行为 |
|---|---|
| **`background`**（默认） | 只走后台路径。真 Win32 控件用投递消息，无窗口控件（UWP/Chromium/Electron）用 UIA pattern。**没有后台路径的操作直接报错**，并在错误里给出替代路线 |
| **`prefer_background`** | 优先后台，**必要时自动回落到前台**（会真的把窗口提到前台） |
| **`foreground`** | 直接以前台方式驱动：先把目标窗口提到前台，再操作 |

### 3.2 锁（关键）

`GHOST_FOCUS_LOCK` 默认 **on**：

- 锁着时，`ghost_set_focus_policy` **只接受 `background`**，另两个返回 `FocusLocked` 错误
- **Agent 自己无法解锁**——这是 0.22 的设计意图：如果锁可以被动解锁，
  Agent 会在"后台动作被拒的那一瞬间"就去解锁，而那一瞬间正是人类看见鼠标乱跳的瞬间

设 `GHOST_FOCUS_LOCK=off` 才会解锁。

### 3.3 为什么解锁后会抢鼠标

解锁之后，Agent（或它调用的 Ghost）遇到这样的操作时：

```
想让 QQ 输入框获得焦点 → 点击
→ Chromium 内容没有独立 HWND，投递消息没有目标
→ background 路径不存在
→ 既然解锁了，Ghost 就升到 foreground
→ SendInput 真实鼠标 + 真实键盘
→ 你的鼠标跳过去、你的键盘输入被打进那个窗口
```

**实测证据**：Ghost 自己会明确拒绝并说明：

```
ghost: Core error: 'click' has no background path on the user's desktop
and the focus policy is 'background'. Real input on the user's desktop is
the operator's call: GHOST_FOCUS_LOCK=off in the server environment,
then ghost_set_focus_policy
```

**它是主动拒绝的，不会静默抢你的鼠标。所以只要不解锁，就永远不会被抢。**

### 3.4 推荐配置

| 你的需求 | 配置 |
|---|---|
| **只想让 AI 帮忙点点界面、读读屏幕**（大多数人的需求） | **默认配置**，不设任何 env。永远不会抢鼠标 |
| 需要它操作"只认真实键鼠"的程序（游戏、模拟器、某些 Electron 应用） | `GHOST_FOCUS_LOCK=off`，但**约定好用完必须降回** `background` |
| 懒得每次手动切 | `GHOST_FOCUS_LOCK=off` + `prefer_background`（会自动回落，但**你无法预知哪次调用会碰真实输入**） |

---

## 四、参数速查表

| 环境变量 | 默认 | 作用 | 建议 |
|---|---|---|---|
| `GHOST_FOCUS_LOCK` | on（锁定） | `off` 解锁，允许升到 `prefer_background`/`foreground` | **不确定就别设** |
| `GHOST_FOCUS_POLICY` | `background` | 启动时的策略 | 保持默认 |
| `GHOST_SHELL` | on | `off` 彻底禁用 `ghost_shell`（执行命令的工具有完整本机权限） | 需要限制时设 `off` |
| `GHOST_CDP_ROUTE` | on | 对带调试端口的浏览器走 CDP（比 UIA 稳得多） | 保持默认 |
| `GHOST_SHELL_WARM` | on | 预热 PowerShell 实例（命令快 3~5 倍） | 保持默认 |
| `GHOST_VISION_API_KEY` 等 | 无 | 可选的视觉层，用于"用自然语言描述元素" | **不需要**（模型自己读元素树） |

---

## 五、实测踩坑清单

### 焦点与输入

1. **没有后台路径的操作会报错，不是静默抢焦点。** 错误信息里会给出替代路线
   （隐藏桌面启动、浏览器走 CDP），先看错误再决定要不要升级。

2. **后台只支持 `Ctrl+C/X/V/Z/A` 这五个修饰键组合**，它们被翻译成语义消息。
   之外的全部被拒绝（`Ctrl+S`、`Ctrl+F`、`Alt+F4`、`Win+方向键`…），
   因为投递消息无法设置应用读取的修饰键状态。

3. **隐藏桌面上真实 `SendInput` 不可用**（Windows 拒绝在非输入桌面发送）。
   所以"只认硬件输入"的程序必须在**你自己的桌面**上、并用 `foreground`。

4. **`ghost_window op=launch` 启动的程序落在隐藏桌面**——用户看不见，
   也不会抢焦点。这是最安全的启动方式。

5. **截图默认能穿透遮挡**（`occlusion_proof: true`），窗口被别的程序盖住也能截到真实内容。

### 浏览器 / 调试端口

6. **Chromium 136+ / Edge 152：默认 user-data-dir 下 `--remote-debugging-port` 会被静默忽略。**
   必须配合**非默认且可写的** `--user-data-dir`。

7. **Edge 152 还要求端口写 `0`**（固定端口会被忽略，整个实例会降级成"向已有实例转交请求"——
   表现为你浏览器里莫名多出空白标签）。实际端口从 `<user-data-dir>\DevToolsActivePort` 第一行读。

8. **端口每次启动都变**，别写死。启动前先删掉旧的 `DevToolsActivePort` 文件。

9. **调试端口只有在"创建 browser 主进程"时才绑定**。若已有实例在跑，
   新启动的进程会把请求交给它然后自己退出，**命令行上的参数被丢弃**。
   Windows 上 Edge 的"启动增强"会在开机时拉起一个后台进程，专门制造这个问题。

10. **别改任务栏快捷方式的 arguments**——点击已固定的图标时 Windows 走的是
    AppUserModelID 激活逻辑，不读 `.lnk` 的参数。

### 无 a11y 的应用（QQ / 微信 / Discord 这类 Electron）

11. **Electron 默认不构建 a11y 树**，`ghost_see` 返回空数组（连 `filtered_offscreen` 都没有）。
    要读屏只能走截图，且**必须用 `PrintWindow(hwnd, hdc, 2)`（`PW_RENDERFULLCONTENT`）**——
    D3D 渲染的窗口用 flags=0 或 BitBlt 只会得到黑图。
    自检：抽样统计非黑像素比例，正常 >90%。

12. **这类应用在用户桌面上没有后台点击路径**，操作必须升级 `foreground`。
    输入中文别逐字模拟（要走 IME，极脆），**用剪贴板 + 真实 `Ctrl+V`**。
    发送优先**点按钮**而不是按回车（回车发送还是换行取决于用户设置）。

### NW.js 游戏（RPG Maker MV/MZ 等）

13. **分水岭是 NW.js 版本**：0.6x 以上支持 `--remote-debugging-port`；
    0.48 / 0.49 会抑制它。查游戏 `Game.exe` 的 `VersionInfo.FileVersion` 就知道能不能走 CDP。

14. **必须命令行直传参数**：

    ```powershell
    Game.exe --remote-debugging-port=9333
    ```

    写进 `package.json` 的 `chromium-args` **无效**——那个字段只透传到 renderer 进程，
    而端口由 **browser 主进程**绑定。

15. 有些游戏的 `package.json` 里明写 `--disable-devtools`，那 exe 路径就废了。
    后路是**用 HTTP 服务器 + 浏览器打开游戏本体**（CDP 是浏览器原生的，禁不掉），
    代价是存档从文件变成 localStorage。

### 通用

16. **看 `verified`，不要看 `ok`。** `ok:true` 只代表调用没抛异常，
    `verified:true` 才代表动作真的生效（读回控件值或像素比对）。

17. **`ghost_see mode=text` 比截图省约 10 倍 token**，能用文本就别用图。

18. **Chromium 的 a11y 有激活延迟**：第一次 `ghost_see` 可能只拿到浏览器外壳
    （地址栏、标签栏），**再读一次**才是完整网页内容。

19. **进程退出 ≠ 注入的 DLL 被卸载。** 比如安全软件退出托盘后，
    它注入浏览器的模块仍驻留，所以"退出某软件测试"不能作为排除依据，
    必须**重启目标程序**才算清干净。

20. **别把工作目录设在盘符根目录。** 有些机器上普通用户对 `D:\` 根不可写，
    而某些程序（如 NW.js）创建 profile 失败会**静默退出**，排查起来很费时间。

---

## 六、能力边界速查

| 目标类型 | 读界面 | 操作 | 走哪条路 |
|---|---|---|---|
| 原生 Win32（记事本、对话框） | ✅ UIA | ✅ 后台 | `background` |
| 浏览器 | ✅ UIA + CDP | ✅ 后台 | `background`，CDP 更强 |
| Electron 应用（含 QQ/微信） | ⚠️ UIA 可能为空 → **截图** | ❌ 后台 → **必须 `foreground`** | 截图 + 坐标 |
| NW.js 游戏（新版） | ✅ CDP | ✅ CDP 执行 JS | `--remote-debugging-port` |
| RGSS / Unity 游戏 | 截图 | 坐标 + 真实输入 | `foreground` |

**能"看"未必能"点"**：截图能读一切，但点击要靠坐标推算，
界面一滚动坐标就失效——所以能用 UIA/CDP 就别用坐标。

---

## 七、常用流程速查

**CLI 常用命令**

```bash
ghost doctor                     # 自检
ghost list-windows               # 列窗口
ghost describe --window "记事本"  # 看控件树
ghost click --name "保存"         # 按名字点
ghost type --role edit --text "hi"
ghost press Enter
ghost hotkey --mods Ctrl --key s
ghost screenshot --out shot.png
```

**给 Agent 的标准循环**

```
ghost_window op=list                     → 找窗口
ghost_see window="标题" [mode=text]       → 看元素
ghost_act window="标题" name="按钮" action="click"
ghost_assert / ghost_wait for=element     → 验证与等待
```

**先锚定窗口**：第一次调用带 `window="标题子串"`，之后可以省略——
会话会锁定那个窗口（跟着窗口句柄走，标题变了也能找回）。
**不要依赖"前台窗口"**，那可能是用户正在打字的地方。

**急停**：`Ctrl+Alt+G` 立刻冻结所有动作（只读查询仍可用），`ghost_reset` 恢复。

---

## 附：这份包里的另一个目录

`skill/` 是给 AI Agent 读的操作手册（本指南的细节展开版），包含：

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 主入口：本机事实、铁律、索引 |
| `references/focus-policy.md` | 三个策略的完整语义、四类硬边界、替代路线表 |
| `references/tools.md` | 54 个工具的分类速查与调用成本 |
| `references/no-a11y-apps.md` | 截图读屏 + 坐标点击 + 剪贴板输入的完整流程 |
| `references/nwjs-games.md` | NW.js 游戏接管：版本分水岭与参数细节 |
| `references/comfyui-canvas.md` | ComfyUI 画布协作：CDP 执行 JS 载入/排布/接线 |

用法：把这些文件放进你 Agent 客户端会读取的 skill 目录
（例如 Claude Code 的 `~/.claude/skills/`，或其他 harness 对应的技能目录），
Agent 就会在遇到桌面自动化任务时自动参考——省去每次重新摸索。
