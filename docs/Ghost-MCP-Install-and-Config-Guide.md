# Ghost MCP Installation & Configuration Guide

> Based on version **0.23.4** (Windows), assembled from a complete hands-on test.
> Focused on **parameters** and **gotchas** — especially the question that scares
> newcomers off: **"Why is Ghost grabbing my mouse and keyboard?"**
>
> [中文版 / Chinese](Ghost-MCP-安装与配置指南.md)

---

## 0. The One Question Everyone Asks

**Q: After installing Ghost it starts grabbing my mouse and my keyboard jumps around — what do I do?**

**A: Because your configuration unlocked the focus lock, and when Ghost meets a target with no background route it escalates to the foreground policy and drives your real mouse and keyboard.**

This is by design, not a bug — but by default **it should never happen**. To rule it out completely:

- **Do not** set `GHOST_FOCUS_LOCK=off` in your Ghost MCP config (unless you actually want it driving real input)
- Keep the default (locked): it can only take background routes and **will never touch your mouse**

Details in [Section 3](#3-focus-policy-the-root-cause-of-the-mouse-grabbing).

---

## 1. What Ghost Is

A **desktop automation layer** for agents. Its selling points differ from other approaches:

| Feature | Description |
|---|---|
| **Background driving** | Posted window messages + UI Automation — **no focus stealing, no cursor movement** (enforced since 0.19, locked since 0.22) |
| **Every action verified** | Returns `verified` / `focus_preserved` / `cursor_preserved`, not a blind `ok:true` |
| **Drives API-less apps** | Win32 / WPF / Electron / UWP / browsers — the app does not need to cooperate |
| **No vision model key** | Your model reads `ghost_see`'s element tree or a screenshot itself |
| **Three entry points** | `ghost-mcp` (MCP server), `ghost` (CLI), `ghost-http` (REST) |

Repository: <https://github.com/NORTHTEKDevs/ghost> (MIT)

---

## 2. Installation

### 2.1 Download

Grab `ghost-windows-x64.zip` from the Releases page and unzip it anywhere (**avoid a drive root** — see gotcha #20).

You get three executables:

```
ghost-mcp.exe    ← the MCP server (the one you mostly use)
ghost.exe        ← CLI, for scripts
ghost-http.exe   ← REST API, optional
```

**Add that folder to your PATH** (user-level is fine) so the CLI is reachable.

### 2.2 Self-check first (important)

```powershell
ghost.exe doctor
```

It checks the Windows build, interactive desktop, UI Automation, DPI awareness, monitors and screen capture. **Continue only when everything PASSes**; a FAIL names the problem outright. Optional warnings (e.g. "no vision key") can be ignored.

### 2.3 Mount it into your MCP client

**Claude Desktop**: download `ghost-windows-x64.mcpb`, then *Settings → Extensions → Install from file*.

**Claude Code**:

```bash
claude mcp add ghost --scope user -- C:/path/to/ghost-mcp.exe
```

**Any other MCP client / custom harness** (stdio server):

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

> ⚠️ That `env` block **is the mouse-grabbing switch**. If you do not need real input, **delete the whole `env` block**.

**Note**: field names differ between clients (some use `mcpServers`, some live in a profile config file). After mounting, tools appear as `mcp__ghost__<tool>` — **54 tools** on Windows.

---

## 3. Focus Policy (the Root Cause of the Mouse Grabbing)

### 3.1 The three policies

| Policy | Behaviour |
|---|---|
| **`background`** (default) | Background routes only. Real Win32 controls get posted messages; windowless controls (UWP/Chromium/Electron) get UIA patterns. **An action with no background route errors out**, and the error names the alternative |
| **`prefer_background`** | Prefers background but **falls back to foreground automatically** when needed (it really does raise the window) |
| **`foreground`** | Drives via foreground from the start: raises the target window, then acts |

### 3.2 The lock (the key part)

`GHOST_FOCUS_LOCK` defaults to **on**:

- While locked, `ghost_set_focus_policy` **only accepts `background`**; the other two return a `FocusLocked` error
- **The agent cannot unlock itself** — that is the explicit intent of 0.22: if the lock could be lifted by the agent, it would lift it "the moment a background action was refused", which is exactly the moment a human sees the mouse jump

Set `GHOST_FOCUS_LOCK=off` to unlock.

### 3.3 Why unlocking leads to mouse grabbing

Once unlocked, when the agent (or Ghost on its behalf) hits something like:

```
Want the QQ input box focused → click it
→ Chromium content has no separate HWND; posted messages have no target
→ no background route exists
→ since it is unlocked, Ghost raises to foreground
→ SendInput: real mouse + real keyboard
→ your cursor jumps there and your keystrokes land in that window
```

**Measured evidence** — Ghost refuses explicitly and says why:

```
ghost: Core error: 'click' has no background path on the user's desktop
and the focus policy is 'background'. Real input on the user's desktop is
the operator's call: GHOST_FOCUS_LOCK=off in the server environment,
then ghost_set_focus_policy
```

**It refuses rather than silently grabbing your mouse. So as long as you do not unlock, it will never grab.**

### 3.4 Recommended configuration

| Your need | Configuration |
|---|---|
| **Only want the AI to click UI and read the screen** (most people) | **Defaults**, no env at all — it will never grab your mouse |
| Need to drive "real input only" programs (games, emulators, some Electron apps) | `GHOST_FOCUS_LOCK=off`, but **agree to set it back** to `background` when done |
| Do not want to switch manually | `GHOST_FOCUS_LOCK=off` + `prefer_background` (auto-fallback, but **you cannot predict which call will touch real input**) |

---

## 4. Parameter Reference

| Env var | Default | Purpose | Advice |
|---|---|---|---|
| `GHOST_FOCUS_LOCK` | on (locked) | `off` unlocks, allowing `prefer_background`/`foreground` | **Leave unset if unsure** |
| `GHOST_FOCUS_POLICY` | `background` | Starting policy | Keep default |
| `GHOST_SHELL` | on | `off` fully disables `ghost_shell` (commands run with your full privileges) | Set `off` to restrict |
| `GHOST_CDP_ROUTE` | on | Routes to CDP for browsers started with a debug port (far more reliable than UIA) | Keep default |
| `GHOST_SHELL_WARM` | on | Pre-warms a PowerShell instance (3–5× faster commands) | Keep default |
| `GHOST_VISION_API_KEY` etc. | none | Optional vision tier, for "describe the element in natural language" | **Not needed** (your model reads the tree itself) |

---

## 5. Field-Tested Gotchas

### Focus & input

1. **An action with no background route errors out — it does not silently steal focus.** The error names the alternative route (launch on a hidden desktop, or drive the browser over CDP). Read the error before deciding to escalate.

2. **Background supports only five modifier combos** — `Ctrl+C/X/V/Z/A` — translated into semantic messages. Everything else is refused (`Ctrl+S`, `Ctrl+F`, `Alt+F4`, `Win+Arrow`…), because posted messages cannot set the modifier state apps read.

3. **Real `SendInput` does not work on a hidden desktop** (Windows refuses it off the input desktop). Targets that only accept hardware input must run on **your own desktop** with `foreground`.

4. **`ghost_window op=launch` starts programs on a hidden desktop** — invisible to the user and unable to steal focus. This is the safest way to launch anything.

5. **Screenshots see through occlusion by default** (`occlusion_proof: true`) — a window covered by others still yields real content.

### Browsers / debug ports

6. **Chromium 136+ / Edge 152: `--remote-debugging-port` is silently ignored with the default user-data-dir.** It requires a **non-default, writable** `--user-data-dir`.

7. **Edge 152 additionally requires the port to be `0`** (a fixed port is ignored and the whole instance degrades into "hand the request to the existing instance" — which shows up as stray blank tabs in your browser). Read the real port from the first line of `<user-data-dir>\DevToolsActivePort`.

8. **The port changes on every launch** — never hard-code it, and delete the stale `DevToolsActivePort` file before starting.

9. **The debug port is bound only when the browser process is created.** If an instance is already running, a newly launched process hands off the request and exits — **its command-line flags are discarded**. On Windows, Edge's "Startup boost" keeps a background process alive specifically to cause this.

10. **Do not edit the taskbar shortcut's arguments** — clicking a pinned icon goes through AppUserModelID activation, which does not read the `.lnk` arguments.

### Apps with no accessibility tree (QQ / WeChat / Discord)

11. **Electron does not build an a11y tree by default**; `ghost_see` returns an empty array (without even a `filtered_offscreen` count). Screenshots are the only way in, and you **must use `PrintWindow(hwnd, hdc, 2)` (`PW_RENDERFULLCONTENT`)** — for D3D windows `flags=0` or `BitBlt` yields a black frame. Self-check: sample the non-black pixel ratio; normally >90%.

12. **There is no background click route for these apps on the user's desktop**, so acting requires `foreground`. For CJK/Unicode input do not simulate character by character (IME makes it fragile) — **use the clipboard plus a real `Ctrl+V`**. Prefer **clicking the send button** over pressing Enter (whether Enter sends or inserts a newline depends on user settings).

### NW.js games (RPG Maker MV/MZ and friends)

13. **The watershed is the NW.js version**: 0.6x and above honour `--remote-debugging-port`; 0.48 / 0.49 suppress it. Check `Game.exe`'s `VersionInfo.FileVersion` to know whether CDP is even possible.

14. **Pass the flag on the command line**:

    ```powershell
    Game.exe --remote-debugging-port=9333
    ```

    Putting it in `package.json`'s `chromium-args` **does not work** — that field only reaches the renderer process, while the port is bound by the **browser (main) process**.

15. Some games write `--disable-devtools` in `package.json`, killing the exe route. The fallback is **an HTTP server plus a browser** opening the game itself (CDP is native to browsers and cannot be disabled), at the cost of saves moving from files into localStorage.

### General

16. **Check `verified`, not `ok`.** `ok:true` only means the call did not throw; `verified:true` means the action actually took effect (control value read back, or pixel comparison).

17. **`ghost_see mode=text` costs about 10× fewer tokens than a screenshot.** Use text unless you genuinely need pixels.

18. **Chromium's a11y tree activates with a delay**: the first `ghost_see` may return only the browser chrome (address bar, tab strip). **Read it again** for the full page contents.

19. **A process exiting does not unload injected DLLs.** When you "exit a security tool to test", its modules injected into your browser are still resident — you must **restart the target program** for a clean test.

20. **Do not put working directories at a drive root.** On some machines a normal user cannot write `D:\`, and programs that fail to create a profile there (NW.js, for one) **exit silently** — expensive to debug.

---

## 6. Capability Boundaries

| Target type | Read UI | Act | Route |
|---|---|---|---|
| Native Win32 (Notepad, dialogs) | ✅ UIA | ✅ background | `background` |
| Browsers | ✅ UIA + CDP | ✅ background | `background`; CDP stronger |
| Electron apps (QQ / WeChat) | ⚠️ screenshot | ⚠️ foreground | screenshot + coordinates |
| NW.js games (recent) | ✅ CDP | ✅ execute JS | `--remote-debugging-port` |
| RGSS / Unity games | screenshot | coordinates + real input | `foreground` |

**Seeing is not the same as clicking**: screenshots read anything, but clicking relies on computed coordinates, which go stale the moment the UI scrolls — prefer UIA/CDP where available.

---

## 7. Common Flows

**CLI**

```bash
ghost doctor                       # self-check
ghost list-windows                 # list windows
ghost describe --window "Notepad"  # dump the control tree
ghost click --name "Save"          # click by accessible name
ghost type --role edit --text "hi"
ghost press Enter
ghost hotkey --mods Ctrl --key s
ghost screenshot --out shot.png
```

**The standard agent loop**

```
ghost_window op=list                       → find a window
ghost_see window="Title" [mode=text]       → read elements
ghost_act window="Title" name="Button" action="click"
ghost_assert / ghost_wait for=element      → verify and wait
```

**Anchor the window first**: pass `window="title substring"` on the first call, then omit it — the session remembers that window (it follows the window handle, so a renamed title still resolves). **Do not rely on "the foreground window"** — that may be exactly where the user is typing.

**Emergency stop**: `Ctrl+Alt+G` freezes every acting call immediately (read-only queries keep working). `ghost_reset` resumes.

---

## Appendix: The `skill/` Directory

`skill/` is the agent-facing manual — a detailed expansion of this guide:

| File | Content |
|---|---|
| `SKILL.md` | Entry point: environment facts, hard rules, index |
| `references/focus-policy.md` | Full semantics of the three policies, four hard limits, alternative-route table |
| `references/tools.md` | All 54 tools by category with call costs |
| `references/no-a11y-apps.md` | Screenshot reading + coordinate clicking + clipboard input, end to end |
| `references/nwjs-games.md` | NW.js game takeover: version watershed and parameter details |
| `references/comfyui-canvas.md` | ComfyUI canvas via CDP-executed JS: load, arrange, rewire |

To use them: copy these files into the skill directory your client reads
(for example `~/.claude/skills/` for Claude Code, or the equivalent for your harness),
and the agent will consult them automatically on desktop-automation tasks — no more
re-discovering everything from scratch every session.
