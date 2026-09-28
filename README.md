# Ghost MCP 实战笔记与总结下来的skill

> 桌面自动化 MCP [Ghost](https://github.com/NORTHTEKDevs/ghost) 的安装配置指南与踩坑记录。
> 基于 **Windows 11 + Ghost 0.23.4** 的完整实测整理。
> **这是社区笔记，不是官方文档。**

---

## ⚠️ 使用前必读

Ghost 能**读写你屏幕上的内容**，并**可能操作真实的鼠标和键盘**。动手前先心里有数：

**它实际能做到：**

- **读取任何窗口的内容**——包括被其他窗口遮住的（截图能穿透遮挡），以及通过系统辅助功能接口读到的控件文本
- **在你没注意的时候驱动界面**：点击、输入、滚动、拖拽
- **执行命令行**：`ghost_shell` 默认开启，拥有你当前账号的完整权限

**请务必注意：**

1. **只在你本人拥有、或明确获得授权的设备上使用。**
2. **不要把焦点锁解开后就不管了。** 解锁状态下任何一次调用都可能移动鼠标、把按键送进当时的前台窗口。**默认的锁定状态是安全的**——它永远不会碰你的鼠标，遇到没有后台路径的操作只会报错并给出替代方案。详见[焦点策略](docs/Ghost-MCP-安装与配置指南.md#三焦点策略抢鼠标的根源)。
3. **截图与读屏会接触到屏幕上的全部内容**，包括密码框、私人对话、验证码通知。共享屏幕、远程协助、录屏环境下尤其要留意。
4. **部分安全软件会把 Ghost 判为可疑程序**（它确实会做窗口消息投递与注入类操作）。这通常是误报，但放行与否需要你自己判断。
5. 本仓库涉及的命令**均在本机实测通过**，但环境差异客观存在，请自行验证后再使用。
6. **不要用它做这三件事**：绕过他人设备的授权、抓取他人的隐私内容、违反你所使用软件的服务条款。

> 因使用本工具及本仓库内容造成的任何后果，由使用者自行承担。

---

## 如果你只想解决一个问题：Ghost 为什么抢我的鼠标？

**因为你的配置解锁了焦点锁。**

Ghost 默认（0.19 起强制、0.22 起锁定）只会走**后台路径**——投递窗口消息 + UI Automation，不动鼠标、不抢焦点。当它遇到**没有后台路径的操作**时（典型如 Chromium/Electron 应用的内容区，它们没有独立窗口句柄），它会**直接报错并给出替代方案**，而不是偷偷抢你的鼠标。

但如果环境变量里设了 `GHOST_FOCUS_LOCK=off`，Agent 就能把策略升到前台、改用**真实键鼠**——这时候你的鼠标会跳、键盘会打进别的窗口。

**解决**：把 MCP 配置里的那两行删掉，重启客户端。

```jsonc
"env": {
  "GHOST_FOCUS_LOCK": "off",          // ← 删掉这行
  "GHOST_FOCUS_POLICY": "background"  // ← 和这行
}
```

---

## 仓库内容

| 路径 | 说明 |
|---|---|
| [`docs/Ghost-MCP-安装与配置指南.md`](docs/Ghost-MCP-安装与配置指南.md) | **主文档**：安装、焦点策略详解、参数速查表、20 条实测踩坑、能力边界、常用流程 |
| [`skill/SKILL.md`](skill/SKILL.md) | 给 AI Agent 读的操作手册入口（主文档的细节展开版） |
| [`skill/references/focus-policy.md`](skill/references/focus-policy.md) | 三个焦点策略的完整语义、四类硬边界、替代路线优先表 |
| [`skill/references/tools.md`](skill/references/tools.md) | 54 个工具的分类速查与调用成本 |
| [`skill/references/no-a11y-apps.md`](skill/references/no-a11y-apps.md) | QQ / 微信这类不暴露辅助功能的应用：截图读屏 + 坐标点击 + 剪贴板输入的完整流程 |
| [`skill/references/nwjs-games.md`](skill/references/nwjs-games.md) | RPG Maker 等 NW.js 游戏的接管：版本分水岭与参数细节 |
| [`skill/references/comfyui-canvas.md`](skill/references/comfyui-canvas.md) | ComfyUI 画布协作：用 CDP 执行 JS 载入 / 排布 / 接线 |

---

## 实测覆盖的能力边界

| 目标类型 | 读界面 | 操作 | 走哪条路 |
|---|---|---|---|
| 原生 Win32（记事本、对话框） | ✅ UIA | ✅ 后台 | `background` |
| 浏览器 | ✅ UIA + CDP | ✅ 后台 | `background`，CDP 更强 |
| Electron 应用（QQ / 微信） | ⚠️ 需截图 | ⚠️ 需前台 | 截图 + 坐标 |
| NW.js 游戏（新版） | ✅ CDP | ✅ 执行 JS | `--remote-debugging-port` |
| RGSS / Unity 游戏 | 截图 | 坐标 + 前台 | `foreground` |

**能"看"未必能"点"**：截图能读一切，但点击要靠坐标推算，界面一滚动坐标就失效——能用 UIA/CDP 就别用坐标。

---

## skill 目录怎么用

`skill/` 是给 **AI Agent** 读的，不是给人读的。把整个 `skill/` 放进你客户端的技能目录
（例如 Claude Code 的 `~/.claude/skills/ghost-desktop/`，或你所用 harness 对应的位置），
Agent 在遇到桌面自动化任务时会自动参考，省去每次重新摸索。

---

## 来源与许可

- Ghost 项目：<https://github.com/NORTHTEKDevs/ghost>（MIT License）
- 本仓库为**实测整理**，非官方文档；文中对 Ghost 行为的描述基于实际测试
- 许可：**MIT**，见 [LICENSE](LICENSE)
