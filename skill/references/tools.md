# Ghost 工具速查（54 个，Windows）

工具前缀 `mcp__ghost__`。所有会“改变世界”的响应都带 `verified`、`focus_preserved`、`cursor_preserved`、`ms`、`target{hwnd,title,surface,source}`。

## 桌面核心（20 + 焦点策略）

| 工具 | 关键参数 | 用途 |
|---|---|---|
| `ghost_see` | `window`, `mode`(full/text/delta/marks/selection), `limit`, `since_seq`, `name`, `role` | 控件树或可见文本；`mode=text` 最省 token |
| `ghost_snapshot` | `window`, `actionable_only`, `limit` | 面向规划的视图：带 id / enabled / 可执行动作 |
| `ghost_find` | `name`, `role`, `description`, `text`, `index`, `mode`, `window` | 定位单个元素，返回中心点与 rect |
| `ghost_act` | `action`(click/type/double_click/right_click/hover), `name`, `role`, `text_input`, `index`, `window` | 找→做→验证，一次调用；`verified` 才可信 |
| `ghost_key` | `keys`(Enter/Tab/F5/Ctrl+C…), `window` | 后台把按键投给窗口的焦点控件 |
| `ghost_scroll` | `direction`, `amount`, `x/y`, `until_name/until_role`, `max_scrolls` | 滚动；“滚到某元素出现”用 until_* |
| `ghost_drag` | `from_x/from_y/to_x/to_y` 或 `from_name/to_name` | 拖拽（需窗口化控件） |
| `ghost_clipboard` | `op`(get/set), `text` | 剪贴板：给拒绝键入的程序当通道 |
| `ghost_screenshot` | `window`, `rect`, `name/role`, `full`, `max_dim`, `jpeg_quality` | 像素；`full=true` 才截全屏 |
| `ghost_window` | `op`(list/focus/anchor/state/launch), `name`, `exe`, `state`, `include_hidden`, `clear` | 窗口管理；`op=focus` 只锚定不提升 |
| `ghost_shell` | `op`(run/open/send/read/list/kill), `cmd`, `cwd`, `shell`, `id`, `timeout_ms` | 隐形控制台；持久会话靠 `open`+`send` |
| `ghost_wait` | `for`(element/value/text/navigate/idle/event/cond/ms), `timeout_ms`, … | 等条件，别 `for=ms` 死等 |
| `ghost_assert` | `predicate`(text-present/text-absent/element-exists/value-equals/value-contains), `name/role`, `text` | 把“成了没”变成机器判定 |
| `ghost_query` | `schema`(JSON Schema 或字段名数组), `region`, `window` | 批量抽字段：先 UIA 匹配，剩下的一次 VLM |
| `ghost_run` | `steps`/`script`/`json_flow`, `max_retries`, `stop_on_error` | 一个往返跑多步；支持 `${steps.N.center.x}` 取值链 |
| `ghost_stop` | — | 急停（同 Ctrl+Alt+G），并释放按住的修饰键 |
| `ghost_reset` | — | 急停之后恢复自动化 |
| `ghost_session_state` | — | 锚定窗口、真实光标、策略、干扰审计（前台抢占统计） |
| `ghost_stats` | — | 接地层遥测：cache/UIA/OCR/VLM 命中率、孤儿浏览器清扫 |
| `ghost_focus_policy` | — | 报告策略与是否被锁 |
| `ghost_set_focus_policy` | `policy`(background/prefer_background/foreground) | 改策略；本机已解锁，改完记得设回 background |

## 浏览器与标签（19）

- `ghost_browser_launch`(`browser`=chrome/comet/edge/brave, `mode`=headless/windowed, `id`) —— 自有隔离实例；windowed 也开在隐藏桌面。
- `ghost_browser_attach`(`port`, `id`) —— 接管用户**已登录**的浏览器（需它带 `--remote-debugging-port` 启动）；Ghost 只会断开，不会关掉它。
- `ghost_browser_tabs` / `ghost_browser_list_installed` / `ghost_browser_close`。
- 标签操作（全部要 `tab`，可选 `browser`）：`ghost_tab_open`, `ghost_tab_navigate`, `ghost_tab_text`, `ghost_tab_describe`, `ghost_tab_click`(selector), `ghost_tab_type`(selector, text, clear), `ghost_tab_press`(key, modifiers), `ghost_tab_eval`(expression), `ghost_tab_find`(query), `ghost_tab_scroll`(dx/dy, selector), `ghost_tab_select_option`(selector, value), `ghost_tab_screenshot`(full_page), `ghost_tab_wait_for`(selector), `ghost_tab_close`。

CDP 路径比 UIA 稳：选择器抗重渲染、支持全修饰键组合、能模拟 focus。窗口若是用 `--remote-debugging-port` 起的，同一个 `window=` 调用会自动走 CDP（响应带 `route:{browser,port,tab}`）。

## 隔离桌面（12，Windows 专属）

`ghost_desktop_create` / `close`(`id`)、`ghost_desktop_launch`(`command`)、`ghost_desktop_windows`、`ghost_desktop_describe`、`ghost_desktop_capture`、`ghost_desktop_click`(`hwnd,x,y,button`)、`ghost_desktop_type`、`ghost_desktop_press`、`ghost_desktop_scroll`、`ghost_desktop_shortcut`(`shortcut`=undo/cut/copy/paste/select_all 或 `Ctrl+Z`)、`ghost_desktop_wait_for_window`。

日常几乎用不到：`ghost_window op=launch` 已经自动把程序放在隐藏桌面 `auto`，普通动词带 `window=` 就能驱动。UIA、窗口消息、截图在隐藏桌面都正常；**真实 SendInput 不行**，所以只认硬件输入的目标必须回到用户桌面 + `foreground`。

## HTTP（2）

`ghost_http_get` / `ghost_http_post`（`url`, `headers`, `body`）—— 配合 `ghost-http.exe --addr 127.0.0.1:7878` 当 REST 用；平时用不到。

## 调用成本参考（README 实测）

文本读取 ~75 ms、验证过的点击 ~200 ms、列窗口 ~1 ms。真正的开销在往返次数：一次 `ghost_run` 打包多步、一次 `ghost_see` 拿到足够元素，比多次小调用便宜得多。
