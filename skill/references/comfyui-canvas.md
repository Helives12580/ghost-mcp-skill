# ComfyUI 画布协作（CDP + JS，全程后台）

## 场景（操作者 的实际用法）

他研究工作流时的常规路径：拿到一个 API 格式工作流示例 → 在画布中打开（节点会按序号自动排成树状图）→ 在画布上展开节点、格式化、提升辨识度 → 分辨清楚链路 → 再融进实际环境的 UI 画布。这一套手动做很花时间。

**Ghost 可以代做的是"从 API 工作流到可读画布，再到融合接线"这一段**，全部走 CDP + 页面 JS，**不需要升焦点策略、不动他的鼠标**。

## 前提：必须是他的浏览器

ComfyUI 的工作流是**前端内存状态**，服务端不知道。所以：

### 实测结论：默认 profile 上开不了调试端口

Edge 152 遵循 Chromium 136+ 的安全规则——`--remote-debugging-port` 与**默认 user-data-dir** 同用时被静默忽略。实测矩阵（同一分钟内对照）：

| 组合 | 结果 |
|---|---|
| `%TEMP%\...` + `--remote-debugging-port=0` | ✅ 成功，端口从 `DevToolsActivePort` 读 |
| `%TEMP%\...` + `--remote-debugging-port=9333` | ❌ 端口不开，URL 被转交给已有实例（用户浏览器里会多出空白标签） |
| `D:\根目录\...` + `port=0` | ❌ 进程当场退出（`D:\` 根对普通用户不可写，Access denied） |
| 默认 profile + `--remote-debugging-port=9333` | ❌ 参数被完全忽略 |

**唯一可行组合**：`--user-data-dir=<非默认且可写的目录> --remote-debugging-port=0`。端口必须写 `0`，由系统分配后从 `<dir>\DevToolsActivePort` 第一行读。

**本机已建好的副本**：`<edge-cdp目录>`（从默认 profile robocopy 而来，含 IndexedDB 422MB / Local Storage 24MB / 扩展 420MB / 登录数据；`Default\Network\Cookies` 因被运行中的 Edge 锁定而跳过——本地 ComfyUI 不依赖 cookie）。ComfyUI 的画布快照就在这份 IndexedDB 里，副本一开就在。

### 启动并接管的标准流程

```powershell
$dir = "<edge-cdp目录>"
$exe = "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
# 1. 清掉旧实例
Get-CimInstance Win32_Process -Filter "Name='msedge.exe'" |
  Where-Object { $_.CommandLine -match 'edge-cdp' } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue }
Start-Sleep 2
# 2. 删掉陈旧的端口文件（否则可能读到旧端口）
Remove-Item "$dir\DevToolsActivePort" -Force -ErrorAction SilentlyContinue
# 3. 启动
Start-Process $exe -ArgumentList "--user-data-dir=$dir","--remote-debugging-port=0",
  "--no-first-run","--no-default-browser-check","--disable-session-crashed-bubble","about:blank"
Start-Sleep 13
# 4. 读端口
$port = [int]((Get-Content "$dir\DevToolsActivePort")[0])
```

然后 `ghost_browser_attach(port=$port, id="user")` → `ghost_tab_open(url="http://127.0.0.1:8188")` → `ghost_tab_wait_for(selector="canvas")`（副本首次加载要 15~30 秒）→ `ghost_tab_eval`。

### 注意事项

- **端口每次启动都变**，必须重读 `DevToolsActivePort`；启动前先删旧文件。
- 这个窗口**用户看得见**，Chromium 创建窗口时**会抢一次前台**（Ghost 的 `ghost_browser_launch` 不会，是因为它开在隐藏桌面上）。
- Ghost **不会关闭** attach 的浏览器（"never closes a browser it did not start"）；用户可能自己关掉，那就重跑上面的流程。
- 副本与原浏览器**状态分叉**，别混着改同一份工作流。
- 画布标题的 `*` 前缀 + `activeFile: null` 只表示**当前画布没绑定服务端工作流文件**，**不等于用户有未保存的改动**——不要据此判断数据珍贵、也不要主动提议备份。
- **Ghost 自己起的浏览器**（`ghost_browser_launch`）是全新 profile，打开 8188 只有空默认画布；只适合做能力验证。

## 把工作流整理成「可读 + 分组 + 子图」（实测 2026-09-28）

参照形态是 操作者 的《多模型两次采样工作流》：**25 个中文组 + 8 个子图**。判据是**功能内聚、接口窄、参数平时不动、可复用**——连只有 2 个节点的 `独立质量词` 也做成子图，因为它是"一处定义、多处引用"的语义单元；组只是**语义标注**，`相机控制` 只含 1 个节点也建组。

### 1. 分区布局（一次 eval 完成）

按 `class_type` 正则把节点分到语义阶段，横向排列、每列最多 10 个折行：

```js
var S=[{n:'图像输入',r:/^LoadImage$/},
 {n:'模型与基础加载',r:/^(CheckpointLoaderSimple|CLIPLoader|VAELoader|UNETLoader|LoraLoaderModelOnly|TK Batch LoRA Loader|AnimaBoosterLoader|AnimaBoosterCheckpointLoader|PathchSageAttentionKJ|AnimaPresetEmptyLatent|ttN seed)$/},
 {n:'提示词与打标',r:/Prompt|Tagger|Embedding|Danbooru|JoinString|ShowText|String ?Router|CLIPTextEncode|CR Prompt Text|AnimaNormalizedAttentionGuidance/},
 {n:'ControlNet 串行',r:/AIO_Preprocessor|ACN_|GrowMaskWithBlur/},
 {n:'开关与总线',r:/Switch|Context|QuickGroupManager|GetNode|SetNode/},
 {n:'采样',r:/KSampler|SetLatentNoiseMask|VAEEncode|EmptySD3/},
 {n:'解码与后处理',r:/Upscale|ScaleToMax|ImageCompare|VAEDecode|SesquiLatent|RTXVideo/},
 {n:'输出',r:/BSK_SaveImage|PreviewImage/}];
// 每个阶段：arr.sort(by order) → pos=[X + col*470, 120 + row*170]，col = Math.floor(i/10)
// 然后 X += 子列数*470 + 170
```

**坑**：正则写成 `/StringRouter/` 会漏掉 `TK String Router`（名字里有空格），必须 `/String ?Router/`。**归类后一定要检查"未分类"清单**。

### 2. 建组

```js
g.groups.slice().forEach(gr => { try{ g.remove(gr); }catch(e){} });   // 重建前先清
var grp = new LGraphGroup('阶段名');
grp.pos  = [minX-40, minY-80];                                        // 标题条在上方
grp.size = [(maxX-minX)+COLW+80, (maxY-minY)+ROWH+140];
g.add(grp);
```

### 3. 转子图

```js
app.canvas.selectNodes(nodeArray);                    // 传数组即可
var sel = app.canvas.selected_nodes;
var s = (sel && typeof sel.size === 'number') ? sel : new Set(Object.values(sel || {}));
var inst = app.graph.convertToSubgraph(s);            // 返回子图实例节点
```

- `convertToSubgraph(e)` 内部检查 `e.size === 0` → **必须传 Set**；传数组会报 "nothing to convert"。
- 转换后 `graph.subgraphs`（Map）多一项，画布上留下一个 **UUID 类型**的实例节点；节点数 = 原数 − 选中数 + 1。
- **事后必须给实例补标题**：`inst.title = '名字'`。改 `sg.name` **不会**同步到实例（否则显示 `New Subgraph`）。
- **回退**：`graph.unpackSubgraph(instanceNode)` 把子图展开回节点。
- 转换前先 `serialize()` 存服务端（见上文备份写法），失败能恢复。

### 4. 接口宽度取决于有没有总线化

ACN 那份 27 个 ControlNet 节点转出的子图是 **in 17 / out 15**；原工作流的 `Controlnet` 大子图只有 **in 4 / out 11**。差别不在节点数，而在**数据传递方式**——原工作流用 rgthree 的 `Context` / `Context Big` 把一堆数据打包成一条总线传进去，所以子图只需暴露 `base_ctx / on_false / strength / end_percent`。**想让子图接口窄，得先做总线化改造，而不是挑节点的功夫**。

### 5. 实测收益（ACN 串行测试工作流，99 节点）

| 阶段 | 画布尺寸 |
|---|---|
| `loadApiJson` 自动展开 | 8616 × 6309 |
| 仅按拓扑分层（20 列） | 8930 × 6000（第一列挤 25 个源节点，没用） |
| **功能分区 + 子图** | **6830 × 1530** |

节点 99 → 73（27 个 ControlNet 节点收成 1 个子图），连线 158 → 120，中文组 8 个，零未分类。

## 改造工作流的三个坑（都踩过，代价很大）

### 坑一：`widgets_values` 不是生效值

用 `LiteGraph.createNode(type)` 建节点后只赋 `node.widgets_values = [...]`，**值不会生效**——真正被读取的是 `node.widgets[i].value`。只赋前者的后果是节点退回默认值：这次 13 条 LoRA 语法凭空消失，而节点**连接看起来完全正常**，一眼看不出问题。

```js
var n = LiteGraph.createNode(type);
app.graph.add(n);
if (n.widgets && n.widgets[0]) {
  n.widgets[0].value = myValue;                                  // 真正生效的值
  if (typeof n.widgets[0].callback === 'function') {
    try { n.widgets[0].callback(myValue, app.canvas, n, [0,0], null); } catch(e) {}
  }
}
n.widgets_values = [myValue, ...rest];                           // 序列化字段，也要同步
app.graph.change();
```

### 坑二：验证要读节点，不要读本地变量

生成语法后我统计的是**局部变量**里的 `<lora:` 个数，报"13 条 ✅"，而节点里其实是空的。必须回读 `node.widgets[0].value` 做 `===` 比对才算验证。

### 坑三：`SetNode`/`GetNode` 不能用脚本直接建

它们是**前端虚拟节点**（不在后端 `object_info` 里）。手工 `createNode` + 赋名只能让前端**显示**配对，`graphToPrompt()` 却会把它们**剔除而不重建连接**——下游整条链静默消失，图上完全看不出来。实测：替换 14 条长线后，目标节点的对应输入在 API 里直接空掉（`whoRefs2130: []`、目标节点不在 API 里）。

**唯一的等价性判据是 `graphToPrompt()`**，不是"图上看着连着"。每次改造前后都该跑一次比对。要无线连接就用 UI 手工建。

## 一次成功的合并实例（LoRA 链）

ACN 工作流里 7 个节点串成一条纯 MODEL 链，承担 13 个 LoRA：

```
AnimaBoosterLoader(1897) → 8(0.6) → 2124(0.6) → 2125(1.0) → 18(0.7)
  → 22(0.7) → 261(TK Batch，内含 7 条) → 1882(Turbo 1.0) → 2128
```

**合并前必须核查的三件事**（缺一不可）：
- 6 个 `LoraLoaderModelOnly` 只有 `model` 输入、只输出 MODEL（**不碰 CLIP**）
- 目标批量加载器的 `clip` 输入为空
- 它的 `CLIP` / `trigger_words` 输出都没接下游

三条都成立，才能收成一个**只接 model** 的批量加载器而不改行为。

**做法**：13 条按原顺序拼成 `<lora:name:weight>`（名字去 `.safetensors`、`\` 转 `/`，与已有批量节点风格一致）。节点 73 → 67。

**验证**：`graphToPrompt()` 里新节点 `lora_syntax` 长度 531、含 13 个 `<lora:`，且 `2128.model = ["2130",0]`、`1900.model = ["2128",0]`。

### 合并候选的接口对比（`/api/object_info/<name>` 可查）

| 节点 | 必需输入 | 输出 | 适用面 |
|---|---|---|---|
| **TK Batch LoRA Loader** | `model`, `lora_syntax` | MODEL, CLIP, STRING | 语法式，**条数不限**，还能输出触发词。最优 |
| AnimaMultiLoraLoader | `model`, `lora_list_json` | MODEL | JSON 列表，无 CLIP |
| easy loraStackApply | `lora_stack`, `model` | MODEL, CLIP | 需搭配 `easy loraStack`（两个节点） |
| bsk_MultiLoraLoaderWithPath | 7 组 switch/name/strength | MODEL, CLIP | 固定 7 条 |

### 判「死代码」要看连接，不能看 `graphToPrompt()`

`graphToPrompt()` 的输出**包含所有节点**（实测：55 个画布节点 → 81 个 API 节点，差额是子图展开出来的内部节点），断链节点一样在列。所以「某节点不在 API 里」**不能**当作删除依据——我一开始就是这么误判的。

真正的判据是**连接**：
- 输出槽的 `links` 为空 → 它的计算结果没人用
- 输入槽的 `link` 全为 `null` → 它是 no-op
- 沿上游追整条链，确认这条链**只服务这个死点**（链上每个节点的**其他**下游都要查）

ACN 实测可删的 12 个：`EmbeddingPrompt×4 → JoinStringMulti×2 → PromptCleaner×2 → CLIPTextEncode×2` 这条链末端两个 CLIPTextEncode 输出全空，加上两个输入全空的 `ImageCompare`。**链上的 `WeiLinPromptUIWithoutLora(96)` 有别的下游（`→175`）所以不能删**——删完要复查每个上游的其余下游，否则误伤。

删后自检三项：`orphans: 0`、`KSampler`/`SaveImage` 数量不变、`graphToPrompt()` 节点数按预期下降（93 → 81）。

### Get/Set 的正确用法（操作者 说明）

**SetNode** 的文本框写入自定义名字（接入的数据名）；**GetNode** 从**下拉里选中**那个名字，输出接口就**自动转换成该名字对应的数据类型**——不是通配符 `*`。

举例：LoRA 总链的模型输出 → SetNode 命名 `model`；另一处放 GetNode 选 `model`，输出口自动变成 MODEL 类型，接进 KSampler 即可。

**我上次失败的根因**：直接写 `widgets_values = ['name']` **既没建立下拉选项、也没触发类型转换**，前端只显示配对，转 API 时被剔除且不重建连接。要手工建，必须让 Get 的 combo widget **真实选中**这个名字（选项来自已存在的 Set 名字列表），验证点是 **`getNode.outputs[0].type` 是否从 `*` 变成了实际类型**——这比 `graphToPrompt()` 更早暴露问题。

**实测通过的正确流程（2026-09-28）**：

```js
// 1) Set：建节点 → 写名字 → 触发 callback → 接数据源
var s = LiteGraph.createNode('SetNode'); s.pos=[x,y]; app.graph.add(s);
var w = s.widgets[0];                       // widget 名是 'Constant'，type 是 'text'
w.value = 'clip_main';
if (typeof w.callback === 'function') w.callback('clip_main', app.canvas, s, [0,0], null);
srcNode.connect(0, s, 0);                   // 接上后 s.inputs[0].type 由 '*' 变成实际类型

// 2) Get：建节点 → combo 选名字
var gt = LiteGraph.createNode('GetNode'); gt.pos=[x,y]; app.graph.add(gt);
var gw = gt.widgets[0];                     // type 是 'combo'，options.values 自动含新 Set 名
gw.value = 'clip_main';
if (typeof gw.callback === 'function') gw.callback('clip_main', app.canvas, gt, [0,0], null);
gt.connect(0, dstNode, dstSlot);            // 输出已自动转成实际类型
```

**四个实测确认的点**：
1. **Get 的 combo 选项自动包含新 Set 的名字**，不需要手工维护选项列表。
2. **类型来自 Set 接入的数据**：Set 的输入为空时，Get 的选项里虽已出现名字，但输出类型**仍是 `*`**；把数据源接上后 Set 的输入类型变成实际类型，Get 的输出才跟着变。
3. **API 层透明**：图上 `2130 → Set("model") → Get("model") → 2128`，`graphToPrompt()` 里是 `2128.inputs.model = ["2130", 0]`，且 `getSetAsClass: 0`——Get/Set 自身不进入可执行图。
4. **子图 ↔ 父级是单向的**（操作者 实测补充，推翻了我原先"跨子图无效"的推断）：
   - **子图的 Get 能取到父级各 Set 的数据**——选项里以 `名字_parent` 的形式出现（**带 `parent` 后缀**）。所以父级的 Set 可以喂给子图内部，跨边界的长线是能消掉的。
   - **反向不行**：子图里的 Set **传不到父级**。
   - **多级嵌套未实测**：目前只知道能正常取到**最高父级**的 Set，中间层级是否逐级递归未经验证——要用就先自己试一次。
   - 且父级的 Set 名对子图可见性取决于**该子图实例**，多个实例各自独立取用。

**关于这套机制的用途**：操作者 日常跑《多模型两次采样工作流》时就大量使用 Get/Set，**大幅降低了飞线密度**——这是本工作流的标准做法，不是临时技巧。

**实测收益**：`CLIPLoader` 一个源喂 4 个主图目标（各 ~2050px 长线），替换后全部变成局部短线，节点 55 → 60，`refsOf2120` 五个引用完整、`orphans: 0`。

## 已实测确认的 API（本机 ComfyUI，3616 种节点类型已注册）

读：
- `app.graph._nodes` —— 节点数组；每个节点有 `.id .type .order .pos .title .inputs .outputs .widgets_values`
- `app.graph.serialize()` —— 整个画布（version / nodes / links），含每个节点的参数与输入输出定义
- `app.graphToPrompt()` —— **画布 → 可执行 API 格式**（async，内部会跑一遍图校验）

写：
- `LiteGraph.createNode(type)` → `app.graph.add(node)` —— 加节点
- `srcNode.connect(outSlotIndex, dstNode, inSlotIndex)` —— 连线，返回新 link（含 `id/origin_id/target_id`）；失败返回 `null`
- `app.graph.remove(node)` / `app.graph.removeLink(link)` —— 删
- `node.pos = [x, y]`、`node.size = [w, h]`、`node.flags.collapsed = false` —— 布局与展开
- `app.graph.arrange()` —— 自动排布
- `app.loadApiJson(json)` —— **API 格式工作流直接进画布**（就是他平时手动"在画布中打开"的那一步）
- `app.loadGraphData(json, clean, restoreView, workflow)` —— UI 格式载入
- `app.graph.change()` —— 改完触发更新

其它有用的入口：`app.queuePrompt`、`app.rootGraph`、`app.isGraphReady`、`LiteGraph.registered_node_types`。

## 标准流程

1. `ghost_tab_open(url="http://127.0.0.1:8188")`（或直接用他已有的标签）
2. **等前端就绪**：`app` 未定义时什么都不能做 —— 用 `ghost_tab_wait_for` 或先 eval 一次 `typeof window.app`，没就绪就等 5~10 秒再试。实测从打开到 `app.graph` 可用要 5~15 秒。
3. 备份：`window.__bak = app.graph.serialize()`（改坏能 `app.loadGraphData(window.__bak, true, true, false)` 恢复）
4. 操作：`loadApiJson` / `createNode` / `connect` / 布局 / 加 Note 写链路说明
5. 验证：`app.graph.serialize()` 看结构，`app.graphToPrompt()` 确认新节点**进了 API 输出**
6. 需要的话 `ghost_tab_screenshot` 出图给他看

## 坑（都踩过）

- **独立 profile = 空画布**。别用它冒充他的工作区。
- **`app` 就绪有延迟**，打开页面立刻 eval 只会拿到 `undefined`。
- **Note 节点不进 `graphToPrompt()` 的 output** —— 这是正确行为（注释节点不参与执行），不是丢失。UI 格式（`workflow.nodes`）里仍在。
- `pos` 在内存里是**数组** `[x,y]`，`serialize()` 后变成对象 `{0:x, 1:y}` —— 反着读会错。
- `graphToPrompt()` 是 **async**；`ghost_tab_eval` 会自动 await promise，直接返回结果即可。
- 连线方向是 `源节点.connect(输出槽, 目标节点, 输入槽)`，别写反。
- 改完画布**不等于保存**：他的画布状态在他浏览器内存里，要留档得让他存（或把 `serialize()` 的结果落盘）。

## 实测记录（2026-09-28）

- 起 CDP 浏览器 → 打开 8188 → 默认工作流 10 节点（SD3/Flux：CLIPLoader / UNETLoader / VAELoader / EmptySD3LatentImage / CLIPTextEncode×2 / ModelSamplingAuraFlow / KSampler / VAEDecode / SaveImage）
- 加 `PreviewImage`(id=72) → `vaedecode.connect(0, pv, 0)` → 返回 link id=83，`origin=65 → target=72`
- 加 `Note`(id=73) → 画布 12 节点；`graphToPrompt()` 输出 **11** 个 API 节点（Note 不计），新节点 72 在列，`class_type=PreviewImage`，`inputs=["images"]`
- 全程 `focus_preserved: true`、无前台切换，中途 操作者 自己切了标签页也不受影响

## 与 comfyui MCP 的分工

跑工作流永远走 `comfyui` MCP（comfy-cli 直连 8188）—— ComfyUI 前端点"运行"本身也只是把 JSON POST 给 `/prompt`。

Ghost 的价值只在这两处：
- **读**：把画布上的图工作流 dump 成 JSON（`serialize()` / `graphToPrompt()`）—— 这是 API 覆盖不到的洞（dsh-comfyui skill 里"图工作流要手动提取"那个）
- **写**：把生成的 API 工作流直接落进画布、自动排布、加注释、按需接线融合

闭环：`loadApiJson / graphToPrompt` ↔ `comfyui` MCP 执行。
