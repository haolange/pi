---
kind: build_system
name: 基于 npm workspaces + Bun 的 Monorepo 构建与发布流水线
category: build_system
scope:
    - '**'
source_files:
    - package.json
    - scripts/release.mjs
    - scripts/sync-versions.js
    - scripts/publish.mjs
    - scripts/build-binaries.sh
    - scripts/package-workspaces.mjs
    - test.sh
    - .github/workflows/ci.yml
    - .github/workflows/build-binaries.yml
---

## 1. 系统概览

该仓库使用 **npm workspaces** 作为 monorepo 编排核心，配合 **Bun** 进行二进制打包、**GitHub Actions** 实现 CI/CD、以及一系列 Node.js 脚本完成版本同步、包发布和产物归档。所有子包位于 `packages/*`（含 `session-backends/*`、`coding-agent/examples/extensions/*`），根 `package.json` 通过 `workspaces` 字段声明聚合关系。

## 2. 关键文件与职责

- `package.json`：定义 workspace、顶层脚本（`build`、`check`、`test`、`version:*`、`release:*`）、Node 引擎要求（`>=22.19.0`）及依赖覆盖。
- `scripts/release.mjs`：端到端发布编排器——校验工作区干净 → 验证公共包已在 npm 注册 → bump 版本 → 更新各包 `CHANGELOG.md` 的 `[Unreleased]` 段 → 生成模型目录/锁文件 → 运行 `npm run check`、`build:offline`、`./test.sh` → 提交并打 tag → 推送 main 与 tag 触发 CI。
- `scripts/sync-versions.js`：强制所有非私有包保持 lockstep 版本号，并将内部 workspace 依赖统一升级为 `^<version>`。
- `scripts/publish.mjs`：遍历公共工作区包，校验 `dist/` 存在、执行 `npm pack --dry-run`、去重后以 `--access public --provenance` 发布。
- `scripts/build-binaries.sh`：本地多平台打包脚本，镜像 `.github/workflows/build-binaries.yml`；安装跨平台原生绑定（`@mariozechner/clipboard-*`），用 `bun build --compile` 将 `packages/coding-agent/dist/bun/cli.js` 编译为 `pi` / `pi.exe`，并复制 WASM、主题、文档等静态资源，最终产出 `pi-darwin-arm64.tar.gz`、`pi-darwin-x64.tar.gz`、`pi-linux-x64.tar.gz`、`pi-linux-arm64.tar.gz`、`pi-windows-x64.zip`、`pi-windows-arm64.zip`。
- `test.sh`：在隔离临时 home 下运行测试，屏蔽 git/npm/AWS 配置，设置 `PI_NO_LOCAL_LLM=1` 避免调用本地 LLM。
- `.github/workflows/ci.yml`：PR/main 触发，Ubuntu runner 上安装系统依赖（`libcairo2-dev`、`libpango1.0-dev`、`libjpeg-dev`、`libgif-dev`、`librsvg2-dev`、`fd-find`、`ripgrep`），执行 `npm ci` → `npm run build` → `npm run check` → `npm test`。
- `.github/workflows/build-binaries.yml`：tag `v*` 或手动触发，分阶段执行：构建源码归档 → 用 `bun 1.3.14` + `node 22` 构建多平台二进制 → 生成 `SHA256SUMS` → 创建 draft GitHub Release → 通过 `npm-publish` 环境发布 npm 包 → 通过 `pi-model-upload` 环境向 R2 发布 pi.dev 公告 → 最后将 draft release 转为公开；失败时清理草稿。
- `scripts/package-workspaces.mjs`：递归扫描 `packages/**/package.json` 的工具函数，被 release 与 version 流程复用。
- `tsconfig.base.json`、`vitest.base.ts`、`biome.json`：共享 TypeScript/Vitest/Biome 配置，由子包继承。

## 3. 架构与约定

- **构建顺序**：根 `build` 脚本按固定顺序串行构建 `tui → telemetry → ai → agent → session-backends/sqlite-node → protocol → client → server → coding-agent`，体现包间依赖拓扑。
- **离线构建模式**：`build:offline` 对 `packages/ai` 使用 `build:offline` 目标，使发布产物内嵌模型数据，不依赖网络。
- **版本策略**：所有非私有包必须 lockstep 版本化（`sync-versions.js` 在版本 bump 后校验并写入 `^version` 依赖）；`release.mjs` 拒绝回退版本。
- **发布前置检查**：`publish.mjs` 要求每个包的 `dist/` 已存在，且所有包版本一致；`release.mjs` 在 bump 前通过 `npm view <pkg> version` 确认公共包已在 npm 注册。
- **二进制产物**：仅 `packages/coding-agent` 被编译为独立可执行文件；Windows 使用 `zip`，Unix 使用 `tar.gz` 并以 `pi` 为根目录名以兼容 mise。
- **CI 门禁**：`ci.yml` 是 PR/main 的单一入口；`build-binaries.yml` 仅在 tag 触发，保证二进制构建与 npm 发布解耦。

## 4. 约束与规则

- Node 版本锁定为 `>=22.19.0`（根 `engines`），CI 使用 `setup-node@v7` 的 `node-version: 22`。
- 发布流程禁止工作区有未提交变更（`git status --porcelain` 检测）。
- 公共工作区包必须在 npm 注册后才能执行版本 bump（`assertPackagesAreRegisteredWithNpm`）。
- 所有非私有包必须保持同一 semver 版本（`sync-versions.js` 输出错误并退出）。
- 二进制构建必须包含全部 6 个平台产物，CI 中通过 `test -f` 逐一断言。
- GitHub Release 必须先创建为 draft，校验资产集合与 `SHA256SUMS` 一致后才转为公开；若后续步骤失败，cleanup job 删除草稿。
- 测试运行于隔离环境：`HOME/TMPDIR/XDG_*` 指向临时目录，`GIT_CONFIG_NOSYSTEM=1`、`GIT_ASKPASS=false`、`AWS_EC2_METADATA_DISABLED=true`、`PI_NO_LOCAL_LLM=1`。
- 预发布钩子：`prepublishOnly` 自动执行 `clean → build → check`；`prepare` 安装 husky。
- 代码质量：`check` 同时运行 Biome (`biome check --write --error-on-warnings`)、依赖锁定检查、TS 相对导入检查、shrinkwrap 校验、浏览器冒烟测试、`tsgo --noEmit` 类型检查。