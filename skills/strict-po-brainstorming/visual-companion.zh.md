# 可视化协作指南

基于浏览器的可视化头脑风暴助手，用于展示原型图、架构图和选项。

## 何时使用

按问题决定，而非按会话决定。判断标准：**用户通过观看是否比阅读更容易理解？**

**使用浏览器**当内容本身是视觉化的：

- **UI 原型图** — 线框图、布局、导航结构、组件设计
- **架构图** — 系统组件、数据流、关系图
- **并排视觉对比** — 比较两种布局、两种配色方案、两种设计方向
- **设计润色** — 当问题涉及外观和感觉、间距、视觉层次时
- **空间关系** — 状态机、流程图、实体关系图

**使用终端**当内容是文本或表格形式：

- **需求和范围问题** — "X 是什么意思？"、"哪些功能在范围内？"
- **概念性的 A/B/C 选择** — 用文字描述的不同方案之间做选择
- **权衡列表** — 优缺点、对比表格
- **技术决策** — API 设计、数据建模、架构方法选择
- **澄清问题** — 任何答案是文字而非视觉偏好的问题

关于 UI 主题的问题不一定是视觉问题。"你想要什么样的向导？"是概念性的 —— 使用终端。"这些向导布局哪个感觉对？"是视觉性的 —— 使用浏览器。

## 工作原理

服务器监视目录中的 HTML 文件，并将最新的文件提供给浏览器。你将 HTML 内容写入 `screen_dir`，用户在其浏览器中看到它，可以点击选择选项。选择记录到 `state_dir/events`，你在下一轮读取。

**内容片段 vs 完整文档：** 如果你的 HTML 文件以 `<!DOCTYPE` 或 `<html` 开头，服务器会原样提供（只注入辅助脚本）。否则，服务器会自动将你的内容包装在框架模板中 —— 添加页头、CSS 主题、选择指示器和所有交互基础设施。**默认情况下编写内容片段。** 只有当你需要完全控制页面时才编写完整文档。

## 启动会话

```bash
# 使用持久化启动服务器（原型图保存到项目）
scripts/start-server.sh --project-dir /path/to/project

# 返回: {"type":"server-started","port":52341,"url":"http://localhost:52341",
#           "screen_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/content",
#           "state_dir":"/path/to/project/.superpowers/brainstorm/12345-1706000000/state"}
```

保存响应中的 `screen_dir` 和 `state_dir`。告诉用户打开 URL。

**查找连接信息：** 服务器将其启动 JSON 写入 `$STATE_DIR/server-info`。如果你在后台启动了服务器且没有捕获标准输出，读取该文件以获取 URL 和端口。使用 `--project-dir` 时，在 `<project>/.superpowers/brainstorm/` 中检查会话目录。

**注意：** 将项目根目录作为 `--project-dir` 传递，以便原型图持久保存在 `.superpowers/brainstorm/` 中，并在服务器重启后保留。没有它，文件会进入 `/tmp` 并被清理。提醒用户将 `.superpowers/` 添加到 `.gitignore`（如果尚未添加）。

**按平台启动服务器：**

**Claude Code (macOS / Linux):**
```bash
# 默认模式有效 —— 脚本本身会在后台运行服务器
scripts/start-server.sh --project-dir /path/to/project
```

**Claude Code (Windows):**
```bash
# Windows 自动检测并使用前台模式，这会阻塞工具调用。
# 在 Bash 工具调用上使用 run_in_background: true，以便服务器在
# 对话轮次之间保持运行。
scripts/start-server.sh --project-dir /path/to/project
```
通过 Bash 工具调用时，设置 `run_in_background: true`。然后在下一轮读取 `$STATE_DIR/server-info` 以获取 URL 和端口。

**Codex:**
```bash
# Codex 会回收后台进程。脚本自动检测 CODEX_CI 并
# 切换到前台模式。正常运行它 —— 不需要额外的标志。
scripts/start-server.sh --project-dir /path/to/project
```

**Gemini CLI:**
```bash
# 使用 --foreground 并在 shell 工具调用上设置 is_background: true
# 以便进程在轮次之间保持运行
scripts/start-server.sh --project-dir /path/to/project --foreground
```

**其他环境：** 服务器必须在后台持续运行，跨越对话轮次。如果你的环境回收分离的进程，使用 `--foreground` 并使用你平台的后台执行机制启动命令。

如果从浏览器无法访问 URL（在远程/容器化设置中常见），绑定非回环主机：

```bash
scripts/start-server.sh \
  --project-dir /path/to/project \
  --host 0.0.0.0 \
  --url-host localhost
```

使用 `--url-host` 控制返回的 URL JSON 中打印的主机名。

## 工作流程

1. **检查服务器是否存活**，然后 **将 HTML 写入** `screen_dir` 中的新文件：
   - 每次写入前，检查 `$STATE_DIR/server-info` 是否存在。如果不存在（或 `$STATE_DIR/server-stopped` 存在），服务器已关闭 —— 在继续前用 `start-server.sh` 重启它。服务器在无活动 30 分钟后自动退出。
   - 使用语义文件名：`platform.html`、`visual-style.html`、`layout.html`
   - **永远不要重用文件名** —— 每个屏幕获取一个新文件
   - 使用 Write 工具 —— **永远不要使用 cat/heredoc**（会将噪音转储到终端）
   - 服务器自动提供最新的文件

2. **告诉用户预期什么并结束你的回合：**
   - 每一步都提醒他们 URL（不仅仅是第一步）
   - 简要文字总结屏幕上显示的内容（例如，"显示首页的 3 种布局选项"）
   - 要求他们在终端中回复："看一看，告诉我你的想法。如果你想，可以点击选择一个选项。"

3. **在你的下一轮** —— 用户在终端回复后：
   - 如果存在，读取 `$STATE_DIR/events` —— 这包含用户的浏览器交互（点击、选择）作为 JSON 行
   - 与用户的终端文本合并以获得完整画面
   - 终端消息是主要反馈；`state_dir/events` 提供结构化交互数据

4. **迭代或推进** —— 如果反馈改变当前屏幕，写入一个新文件（例如 `layout-v2.html`）。只有当当前步骤验证通过后才进入下一个问题。

5. **返回终端时卸载** —— 当下一步不需要浏览器时（例如澄清问题、权衡讨论），推送一个等待屏幕以清除过时内容：

   ```html
   <!-- 文件名: waiting.html (或 waiting-2.html 等) -->
   <div style="display:flex;align-items:center;justify-content:center;min-height:60vh">
     <p class="subtitle">继续在终端中...</p>
   </div>
   ```

   这防止用户在对话已继续时盯着已解决的选择。当下一个视觉问题出现时，像往常一样推送新内容文件。

6. 重复直到完成。

## 编写内容片段

只编写进入页面的内容。服务器自动将其包装在框架模板中（页头、主题 CSS、选择指示器和所有交互基础设施）。

**最小示例：**

```html
<h2>哪种布局更好？</h2>
<p class="subtitle">考虑可读性和视觉层次</p>

<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>单列</h3>
      <p>简洁、专注的阅读体验</p>
    </div>
  </div>
  <div class="option" data-choice="b" onclick="toggleSelect(this)">
    <div class="letter">B</div>
    <div class="content">
      <h3>双列</h3>
      <p>侧边栏导航与主内容</p>
    </div>
  </div>
</div>
```

就是这样。不需要 `<html>`、CSS 或 `<script>` 标签。服务器提供所有这些。

## 可用的 CSS 类

框架模板为你的内容提供这些 CSS 类：

### 选项（A/B/C 选择）

```html
<div class="options">
  <div class="option" data-choice="a" onclick="toggleSelect(this)">
    <div class="letter">A</div>
    <div class="content">
      <h3>标题</h3>
      <p>描述</p>
    </div>
  </div>
</div>
```

**多选：** 在容器上添加 `data-multiselect` 以允许用户选择多个选项。每次点击切换项目。指示器栏显示计数。

```html
<div class="options" data-multiselect>
  <!-- 相同的选项标记 —— 用户可以选择/取消选择多个 -->
</div>
```

### 卡片（视觉设计）

```html
<div class="cards">
  <div class="card" data-choice="design1" onclick="toggleSelect(this)">
    <div class="card-image"><!-- 原型图内容 --></div>
    <div class="card-body">
      <h3>名称</h3>
      <p>描述</p>
    </div>
  </div>
</div>
```

### 原型图容器

```html
<div class="mockup">
  <div class="mockup-header">预览：仪表板布局</div>
  <div class="mockup-body"><!-- 你的原型图 HTML --></div>
</div>
```

### 分割视图（并排）

```html
<div class="split">
  <div class="mockup"><!-- 左侧 --></div>
  <div class="mockup"><!-- 右侧 --></div>
</div>
```

### 优缺点

```html
<div class="pros-cons">
  <div class="pros"><h4>优点</h4><ul><li>好处</li></ul></div>
  <div class="cons"><h4>缺点</h4><ul><li>缺点</li></ul></div>
</div>
```

### 模拟元素（线框图构建块）

```html
<div class="mock-nav">Logo | 首页 | 关于 | 联系</div>
<div style="display: flex;">
  <div class="mock-sidebar">导航</div>
  <div class="mock-content">主内容区域</div>
</div>
<button class="mock-button">操作按钮</button>
<input class="mock-input" placeholder="输入框">
<div class="placeholder">占位区域</div>
```

### 排版和章节

- `h2` — 页面标题
- `h3` — 章节标题
- `.subtitle` — 标题下方的辅助文字
- `.section` — 带有底部边距的内容块
- `.label` — 小写大写标签文字

## 浏览器事件格式

当用户点击浏览器中的选项时，他们的交互记录到 `$STATE_DIR/events`（每行一个 JSON 对象）。当你推送新屏幕时，文件会自动清除。

```jsonl
{"type":"click","choice":"a","text":"选项 A - 简单布局","timestamp":1706000101}
{"type":"click","choice":"c","text":"选项 C - 复杂网格","timestamp":1706000108}
{"type":"click","choice":"b","text":"选项 B - 混合","timestamp":1706000115}
```

完整的事件流显示用户的探索路径 —— 他们在确定之前可能点击多个选项。最后一个 `choice` 事件通常是最终选择，但点击模式可以揭示犹豫或值得询问的偏好。

如果 `$STATE_DIR/events` 不存在，用户没有与浏览器交互 —— 只使用他们的终端文本。

## 设计技巧

- **根据问题调整保真度** —— 布局用线框图，润色问题用精细设计
- **在每一页上解释问题** —— "哪种布局感觉更专业？" 而不是仅仅 "选一个"
- **在推进前迭代** —— 如果反馈改变当前屏幕，写入新版本
- **每屏最多 2-4 个选项**
- **在重要时使用真实内容** —— 对于摄影作品集，使用实际图片（Unsplash）。占位内容会掩盖设计问题。
- **保持原型图简单** —— 专注于布局和结构，而不是像素级完美设计

## 文件命名

- 使用语义名称：`platform.html`、`visual-style.html`、`layout.html`
- 永远不要重用文件名 —— 每个屏幕必须是一个新文件
- 迭代时：附加版本后缀如 `layout-v2.html`、`layout-v3.html`
- 服务器按修改时间提供最新文件

## 清理

```bash
scripts/stop-server.sh $SESSION_DIR
```

如果会话使用了 `--project-dir`，原型图文件会持久保存在 `.superpowers/brainstorm/` 中以供日后参考。只有 `/tmp` 会话在停止时被删除。

## 参考

- 框架模板（CSS 参考）：`scripts/frame-template.html`
- 辅助脚本（客户端）：`scripts/helper.js`
