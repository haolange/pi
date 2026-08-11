---
kind: dependency_management
name: 基于 npm workspaces 与脚本化锁文件的依赖管理体系
category: dependency_management
scope:
    - '**'
source_files:
    - package.json
    - .npmrc
    - package-lock.json
    - scripts/check-pinned-deps.mjs
    - scripts/sync-versions.js
    - scripts/generate-coding-agent-shrinkwrap.mjs
    - scripts/generate-coding-agent-install-lock.mjs
    - packages/coding-agent/package.json
    - packages/ai/package.json
    - packages/server/package.json
---

## 1. 使用的系统与工具

- **包管理器**: npm（lockfileVersion 3），通过 `package-lock.json` 锁定整个 monorepo 的第三方依赖。
- **工作区管理**: 根 `package.json` 使用 `workspaces` 字段聚合 `packages/*`、`packages/session-backends/*` 以及 `packages/coding-agent/examples/extensions/*` 下的子包，实现跨包依赖解析与统一发布。
- **版本策略**: `.npmrc` 设置 `save-exact=true`，所有外部依赖以精确版本号声明；CI 通过 `scripts/check-pinned-deps.mjs` 强制校验——除内部 `@earendil-works/pi-*` 包与非 registry 来源（`workspace:`/`file:`/`git+`/`https:` 等）外，所有直接依赖必须匹配 `^\d+.\d+.\d+(-...)?(\+...)?` 精确版本模式。
- **Node 引擎约束**: 根 `engines.node >=22.19.0`，各子包也各自声明相同要求，保证运行环境一致。

## 2. 关键文件与脚本

- `package.json`（根）：定义 workspaces、顶层构建/检查/发布脚本、`overrides`（强制 `protobufjs@7.6.5`、`rimraf@6.1.2` 等）。
- `.npmrc`：全局启用 `save-exact=true` 与 `min-release-age=2`。
- `package-lock.json`（根）：全仓库唯一 lockfile，作为所有生成型锁文件的权威来源。
- `scripts/check-pinned-deps.mjs`：递归扫描所有 `package.json`，对非内部、非 registry 别名依赖执行精确版本校验。
- `scripts/sync-versions.js`：在发布流程中验证所有非私有包处于同一版本（lockstep），并将内部 `@earendil-works/pi-*` 依赖更新为 `^<version>`。
- `scripts/generate-coding-agent-shrinkwrap.mjs`：从根 `package-lock.json` 派生 `packages/coding-agent/npm-shrinkwrap.json`，仅包含运行时依赖、剥离 dev 元数据、将内部工作区替换为 npm 注册表 tarball URL，并校验不允许 `link/file/workspace` resolved、至少存在平台相关可选依赖、无未允许的 `hasInstallScript`。
- `scripts/generate-coding-agent-install-lock.mjs`：生成 `packages/coding-agent/install-lock/package.json` + `package-lock.json`，用于 Pi 安装器/更新器离线安装，同样禁止本地 resolved、校验内部包版本一致性、校验精确版本解析。
- `packages/coding-agent/package.json`：声明 `files` 包含 `npm-shrinkwrap.json`，并在 `prepublishOnly` 中调用 shrinkwrap 生成。

## 3. 架构与约定

- **单一 lockfile 源**：整个 monorepo 只维护一份根 `package-lock.json`，所有下游可复现产物均由该文件经脚本推导生成，避免多份 lockfile 漂移。
- **内部包版本 lockstep**：所有非私有发布的 `@earendil-works/pi-*` 包共享同一版本号；`sync-versions.js` 在 `version:*` 脚本后运行，把相互引用的内部依赖统一升级为 `^<新主版本>`。
- **外部依赖精确锁定**：`.npmrc` + `check-pinned-deps.mjs` 双重保障，不允许 caret/tilde 范围。
- **coding-agent 特殊隔离**：由于 `pi-coding-agent` 会被打包成独立二进制并通过安装器分发，项目额外生成两份派生锁文件：
  - `npm-shrinkwrap.json`：供 `npm pack` 时固定发布内容。
  - `install-lock/`：供安装器在目标机器上 `npm ci` 安装，剔除开发依赖、校验平台可选依赖存在性。
- **安全与可审计约束**：两个生成脚本均维护 `allowedInstallScriptPackages` 白名单（如 `@google/genai@1.52.0`、`protobufjs@7.6.5`），新增带 install script 的依赖需显式加入白名单并通过校验。
- **依赖覆盖集中管理**：根 `overrides` 与 `packages/coding-agent` 的 `overrides` 共同强制 `protobufjs`、`rimraf`、`gaxios.rimraf` 等脆弱依赖版本。

## 4. 约定与约束清单

| 规则 | 来源/执行方式 |
|---|---|
| 所有外部直接依赖必须使用精确语义化版本（含 prerelease/build metadata） | `.npmrc save-exact=true` + `scripts/check-pinned-deps.mjs`（CI `check:pinned-deps`） |
| 内部 `@earendil-works/pi-*` 包之间采用 lockstep 版本，引用时使用 `^<version>` | `scripts/sync-versions.js`（由 `version:patch/minor/major` 触发） |
| 根 lockfile 必须是 lockfileVersion 3 且包含 `packages` map | 两个生成脚本在启动时断言 |
| coding-agent 发布物不得包含 link/file/workspace 类型的 resolved | `generate-coding-agent-shrinkwrap.mjs` / `generate-coding-agent-install-lock.mjs` 校验失败即退出 |
| 派生锁文件中每个依赖的精确声明必须与实际解析版本一致 | `generate-coding-agent-install-lock.mjs` 逐条比对 |
| 派生锁文件必须至少包含一个平台相关可选依赖（os/cpu/libc） | 两个生成脚本的 `validateShrinkwrap` / `validateGeneratedFiles` |
| 任何 `hasInstallScript` 的包必须列入白名单 | 两脚本中的 `allowedInstallScriptPackages` 映射 |
| Node 版本不低于 22.19.0 | 根与各子包 `engines.node` |
| 构建前自动清理并重新构建所有 workspace | 根 `prepublishOnly` 与 `build` 脚本链 |

## 5. 适用性说明

本仓库是典型的 npm workspaces monorepo，依赖管理通过“精确版本 + 单锁文件 + 脚本化派生锁”的方式实现，具备完整的版本同步、校验与发布隔离机制。