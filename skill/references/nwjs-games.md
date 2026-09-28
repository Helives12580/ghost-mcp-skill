# RPG Maker MV/MZ、TyranoBuilder 这类 NW.js 游戏

## 分水岭：NW.js 版本（实测 2026-09-28）

| 游戏 | NW.js | `--remote-debugging-port` | 备注 |
|---|---|---|---|
| <某MV游戏>（MV） | 0.49.2 | ❌ 抑制 | 参数只到 renderer，browser 进程不绑端口 |
| **<某MZ游戏>（MZ）** | **0.69.1** | ✅ **可用** | Chrome/106，`ghost_browser_attach` 直接连 |
| 星辰婚礼！（MZ） | 0.48.4 | ❌ | `chromium-args` 里**显式** `--disable-devtools` |
| 逆梦的危险幻域（MZ） | 0.48.4 | ❌ | 同上 |

**结论：0.6x 以上的 NW.js 能用；0.48/0.49 抑制该开关。** 试之前先读游戏的 `package.json` 看 `chromium-args`，再读 exe 的 `VersionInfo.FileVersion` 拿 NW.js 版本，能省一轮试错。

## 启动方法（关键细节）

**必须命令行直传，只写 `package.json` 的 `chromium-args` 无效**：

```powershell
Start-Process "<游戏>\Game.exe" -ArgumentList "--remote-debugging-port=9333" -WorkingDirectory "<游戏>"
```

实测原因：`chromium-args` 里的参数**只透传到 renderer 进程**，而**调试端口由 browser（主）进程绑定**。写了 `chromium-args` 后，renderer 命令行里能看到该参数，主进程命令行却仍是干净的 → 端口不开。加 `--user-data-dir=<独占目录>` 也**不能**解决（在 0.49.2 上验证过）。

启动后 `Get-NetTCPConnection -LocalPort 9333 -State Listen` 确认，再 `ghost_browser_attach(port=9333)`。

## 接管后能做什么

MZ/MV 的游戏状态全在全局对象上，`Runtime.evaluate` 直接读写：

```js
Utils.isNwjs()                                  // true
$gamePlayer.x / $gamePlayer.y                   // 坐标，改了即瞬移
$gameMap.mapId()                                // 当前地图
$gameVariables.value(n) / setValue(n, v)        // 游戏变量
$gameSwitches.value(n) / setValue(n, true)      // 开关（跳剧情/解锁）
$gameParty.gold() / gainItem() / members()      // 金钱、道具、队伍
SceneManager._scene.constructor.name            // 当前场景
SceneManager.push(Scene_Map)                    // 场景跳转
$gameMessage.add("文本")                        // 直接推对话
```

**实测（<某MZ游戏>）**：`isNwjs: true`，`$gamePlayer` / `$gameMap` / `$gameVariables` / `$gameSwitches` / `$gameParty` 全部可访问，当前 `Scene_Title`。

## 替代路径：用浏览器拉起 HTML

exe 走不通时（禁用 devtools / NW.js 老旧），可以直接加载游戏本体：

```powershell
cd "<游戏目录>"
python -m http.server 8888            # 或 npx serve
msedge.exe --remote-debugging-port=9333 --user-data-dir=<edge-cdp目录> http://127.0.0.1:8888
```

**三个必须知道的代价**：

1. **必须用 HTTP 服务器，不能 `file://`** —— MV/MZ 通过 XHR 读 `data/*.json`（地图、数据库），`file://` 下被 CORS 拦，游戏起不来。
2. **存档隔离（最关键）** —— exe 模式下存档是**文件**（`save/fileN.rmmzsave`、`config.rmmzsave`）；浏览器模式进 **localStorage**，两边不通，浏览器打开看不到原存档。但 MZ 的 `StorageManager` 接口统一（`loadObject` / `saveObject`），**可以用 CDP 执行 JS 迁移存档**（文件 ↔ localStorage 双向）。
3. **NW.js 特有插件失效** —— 用 `require('fs')` / `nw.*` 的插件会报错；MZ 核心对两种环境都有分支，核心功能不受影响。

**优势**：不依赖 NW.js 的调试支持，CDP 是浏览器原生的。**对 `--disable-devtools` 的游戏这是唯一出路。**

## 不是 NW.js 的游戏别走这条路

- `Demon's GunPlay 2` 的产品标识是 **Ruby Game Scripting System**（RPG Maker VX/Ace 的 RGSS）
- Unity 游戏（`Isekai Succubus`、`ふたなり彼女２`）有自己的调试协议但不是 CDP

这两类**没有 CDP**，只能走截图 + 坐标（见 `no-a11y-apps.md`）。

## 与 MTool 的关系

MTool（`inject.exe` + `mzHook.dll` + `winmm.dll`/`version.dll` 劫持）走的是 **DLL 注入 + 内存 hook**，目标是截文本做翻译，需要改游戏目录、启动链是 `与工具一同启动.bat`。

Ghost + CDP 是**零侵入的运行时连接**，能做的是**读写游戏状态**而非替换文本。两者互补：**用 CDP 调试游戏时不要走那个 bat**（它会转手启动链），直接启原始 exe。
