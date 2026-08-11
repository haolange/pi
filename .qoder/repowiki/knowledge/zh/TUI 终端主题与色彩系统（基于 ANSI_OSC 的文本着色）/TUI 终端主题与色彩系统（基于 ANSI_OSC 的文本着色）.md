---
kind: frontend_style
name: TUI 终端主题与色彩系统（基于 ANSI/OSC 的文本着色）
category: frontend_style
scope:
    - '**'
source_files:
    - packages/tui/src/terminal-colors.ts
    - packages/tui/src/terminal.ts
    - packages/tui/src/components/text.ts
    - packages/tui/src/components/box.ts
    - packages/tui/src/components/v-stack.ts
    - packages/tui/src/components/h-stack.ts
    - packages/tui/src/components/select-list.ts
    - packages/tui/src/components/editor.ts
    - packages/tui/src/components/markdown.ts
    - packages/tui/src/components/image.ts
    - packages/tui/src/components/loader.ts
    - packages/tui/src/components/input.ts
    - packages/tui/src/terminal-image.ts
    - packages/coding-agent/examples/extensions/built-in-tool-renderer.ts
    - packages/coding-agent/docs/extensions.md
---

## 1. 整体方案
本仓库没有浏览器前端，因此不存在 CSS/Tailwind/styled-components 等 Web 样式体系。所有“前端风格”集中在 `packages/tui`——一个纯 Node.js 的终端用户界面库，通过 VT 转义序列、ANSI 颜色码和 OSC 控制序列在终端中渲染彩色文本、图像、布局与组件。

## 2. 关键文件与包
- `packages/tui/src/terminal-colors.ts`：定义 `RgbColor`、`TerminalColorScheme = "dark" | "light"`，解析终端返回的 OSC 11 背景色响应以及 `\x1b[?997;...` 颜色方案报告，从而自动检测终端是 dark/light 主题。
- `packages/tui/src/terminal.ts`：实现 `ProcessTerminal`，负责 raw 模式、Kitty 键盘协议协商、SIGWINCH resize、进度条 OSC 9;4 等底层 I/O；不直接管理颜色，但为上层组件提供写入能力。
- `packages/tui/src/components/*`：`text.ts`、`box.ts`、`v-stack.ts`、`h-stack.ts`、`scroll-view.ts`、`select-list.ts`、`editor.ts`、`markdown.ts`、`image.ts`、`loader.ts`、`cancellable-loader.ts`、`input.ts`、`settings-list.ts`、`spacer.ts`、`truncated-text.ts`、`alt-screen-flash.ts` 等构成 TUI 的可视化组件树。
- `packages/coding-agent/examples/extensions/built-in-tool-renderer.ts`：示例扩展，使用 `theme.fg(...)`、`theme.bold(...)` 对工具输出进行着色，体现主题 API 的使用方式。
- `packages/coding-agent/docs/extensions.md`：文档中大量展示 `ctx.ui.theme.fg("accent"|"dim"|"muted"|"toolTitle"|"success"|"warning"|"error")` 等语义化 token 用法，是主题 token 列表的事实来源。

## 3. 架构与约定
- **无 CSS**：整个 UI 由字符串拼接 + ANSI/VT/OSC 转义序列组成，最终通过 `process.stdout.write` 输出到终端。
- **主题模型**：主题以语义化 token 形式暴露给上层渲染器（如 `theme.fg("accent", text)`），token 包括 `accent`、`dim`、`muted`、`toolTitle`、`success`、`warning`、`error`、`bold` 等。这些 token 的具体 RGB 值由 TUI 根据终端能力与当前 dark/light 方案映射生成。
- **自动主题检测**：通过解析终端 OSC 11 背景色响应与 Kitty `\x1b[?997;n` 报告，将 `TerminalColorScheme` 推断为 `dark` 或 `light`，从而选择对应配色表。
- **组件分层**：`terminal.ts` 提供最低层 I/O；`components/*` 组合成布局树；`tui-main-screen.ts` / `tui-alt-screen.ts` 管理屏幕切换；`layout.ts` / `layout-node.ts` 描述布局节点；`terminal-image.ts` 处理终端内嵌图片（如 Sixel/Kitty graphics）。
- **跨包复用**：`@earendil-works/pi-tui` 作为独立 npm 包发布，被 `@earendil-works/pi-coding-agent` 消费，后者通过 `registerTool({ renderCall, renderResult })` 接收 `theme` 参数并调用其方法。

## 4. 约定与约束
- 所有可见文本必须通过 TUI 组件（如 `Text`）或主题 API 输出，避免硬编码 ANSI 颜色码散落各处。
- 颜色语义遵循固定 token 集合：`accent`（强调）、`dim`（弱化）、`muted`（更弱）、`toolTitle`（工具标题）、`success`/`warning`/`error`（状态）、`bold`（加粗）。新增视觉含义应优先复用已有 token。
- 主题感知依赖终端能力：当终端不支持 OSC 11 或 Kitty 997 报告时，回退到默认 dark/light 策略。
- 图像渲染通过 `terminal-image.ts` 统一处理，不同终端协议（Sixel、Kitty、iTerm2 等）由该模块封装，组件层无需关心具体协议。
- 构建产物不包含任何 `.css`/`.scss`/`tailwind.config.*` 文件；样式完全由运行时生成的转义序列决定。

## 5. 结论
本仓库的“frontend_style”实质上是**终端 TUI 的主题系统**：以 `packages/tui` 为核心，通过 ANSI/OSC 转义序列、语义化 color token（`accent`、`dim`、`success`、`warning`、`error`、`toolTitle`、`muted`、`bold` 等）以及暗/亮主题自动检测，为命令行应用提供一致的视觉风格。不存在 Web 前端样式技术栈。