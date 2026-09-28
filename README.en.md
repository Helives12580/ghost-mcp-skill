# Ghost MCP Field Notes

> Installation & configuration guide plus hard-won gotchas for the desktop-automation MCP [Ghost](https://github.com/NORTHTEKDevs/ghost).
> Based on a full hands-on test on **Windows 11 + Ghost 0.23.4**.
> **Community notes, not official documentation.**
>
> [中文版 / Chinese](README.md)

---

## ⚠️ Read This Before You Start

Ghost can **read everything on your screen**, and it **may drive your real mouse and keyboard**. Know what you are handing over:

**What it can actually do:**

- **Read any window's contents** — including windows hidden behind others (screenshots see through occlusion), and control text read through the OS accessibility API
- **Drive the UI while you are not looking**: click, type, scroll, drag
- **Run command lines**: `ghost_shell` is on by default and executes with your account's full privileges

**Please note:**

1. **Use it only on devices you own, or are explicitly authorized to automate.**
2. **Do not unlock the focus policy and then forget about it.** While unlocked, any call may move your mouse and send keystrokes into whatever window happens to be focused. **The default locked state is safe** — it will never touch your mouse; it errors out and names the alternative route instead. See [Focus Policy](docs/Ghost-MCP-Install-and-Config-Guide.md#3-focus-policy-the-root-cause-of-the-mouse-grabbing).
3. **Screenshots and screen reading touch everything on screen**, including password fields, private chats and verification codes. Be extra careful when screen sharing, remoting in, or recording.
4. **Some security software will flag Ghost as suspicious** (it genuinely does window-message posting and injection-like work). It is usually a false positive, but allowing it is your call.
5. Every command here **was verified on one machine** — environments differ, so validate before you rely on it.
6. **Three things not to do with it**: bypass authorization on someone else's device, harvest other people's private content, or violate the terms of software you use.

> Any consequences of using this tool or this repository are the user's own responsibility.

---

## If You Only Want to Fix One Thing: Why Is Ghost Grabbing My Mouse?

**Because your configuration unlocked the focus lock.**

By default (enforced since 0.19, locked since 0.22) Ghost only takes **background routes** — posted window messages plus UI Automation — never touching your mouse or stealing focus. When it hits an action that **has no background route** (typically the content area of Chromium/Electron apps, which has no separate window handle), it **errors out and names the alternative** rather than silently grabbing your mouse.

But if the environment variable `GHOST_FOCUS_LOCK=off` is set, the agent can raise the policy to foreground and switch to **real input** — that is when your mouse jumps and your keystrokes land in the wrong window.

**Fix**: delete those two lines from your MCP config and restart the client.

```jsonc
"env": {
  "GHOST_FOCUS_LOCK": "off",          // ← delete
  "GHOST_FOCUS_POLICY": "background"  // ← and this
}
```

---

## What Is In This Repository

| Path | Description |
|---|---|
| [`docs/Ghost-MCP-Install-and-Config-Guide.md`](docs/Ghost-MCP-Install-and-Config-Guide.md) | **Main guide (EN)**: install, focus policy in depth, parameter reference, 20 field-tested gotchas, capability boundaries, common flows |
| [`docs/Ghost-MCP-安装与配置指南.md`](docs/Ghost-MCP-安装与配置指南.md) | 主文档中文版（Main guide, Chinese） |
| [`skill/SKILL.md`](skill/SKILL.md) | Entry point of the agent-facing manual |
| [`skill/references/focus-policy.md`](skill/references/focus-policy.md) | Full semantics of the three focus policies, four hard limits, alternative-route table |
| [`skill/references/tools.md`](skill/references/tools.md) | All 54 tools by category, with call costs |
| [`skill/references/no-a11y-apps.md`](skill/references/no-a11y-apps.md) | Apps exposing no accessibility tree (QQ / WeChat): screenshot reading + coordinate clicking + clipboard input |
| [`skill/references/nwjs-games.md`](skill/references/nwjs-games.md) | Taking over RPG Maker / NW.js games: version watershed and parameter details |
| [`skill/references/comfyui-canvas.md`](skill/references/comfyui-canvas.md) | ComfyUI canvas work via CDP-executed JS: load, arrange, rewire |

---

## Tested Capability Boundaries

| Target type | Read UI | Act | Route |
|---|---|---|---|
| Native Win32 (Notepad, dialogs) | ✅ UIA | ✅ background | `background` |
| Browsers | ✅ UIA + CDP | ✅ background | `background`; CDP is stronger |
| Electron apps (QQ / WeChat) | ⚠️ screenshot only | ⚠️ foreground only | screenshot + coordinates |
| NW.js games (recent) | ✅ CDP | ✅ execute JS | `--remote-debugging-port` |
| RGSS / Unity games | screenshot | coordinates + foreground | `foreground` |

**Seeing is not the same as clicking**: screenshots can read anything, but clicking relies on computed coordinates — scroll the UI and they are stale. Prefer UIA/CDP whenever available.

---

## Using the `skill/` Directory

`skill/` is written for **an AI agent**, not for humans. Drop the whole `skill/` directory into your client's skill folder — for example `~/.claude/skills/ghost-desktop/` for Claude Code, or the equivalent for your harness — and the agent will consult it automatically on desktop-automation tasks.

---

## Source & License

- Ghost project: <https://github.com/NORTHTEKDevs/ghost> (MIT License)
- This repository is **field notes**, not official documentation; every description of Ghost's behavior comes from actual testing
- License: **MIT** — see [LICENSE](LICENSE)
