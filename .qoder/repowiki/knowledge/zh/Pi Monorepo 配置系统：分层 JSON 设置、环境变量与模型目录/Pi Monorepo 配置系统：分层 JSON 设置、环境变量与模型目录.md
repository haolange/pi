---
kind: configuration_system
name: Pi Monorepo 配置系统：分层 JSON 设置、环境变量与模型目录
category: configuration_system
scope:
    - '**'
source_files:
    - packages/coding-agent/src/config.ts
    - packages/coding-agent/src/core/settings-manager.ts
    - packages/coding-agent/src/core/model-config.ts
    - packages/ai/src/env-api-keys.ts
    - packages/ai/src/utils/provider-env.ts
    - packages/server/src/server.ts
    - package.json
    - .npmrc
---

## 1. 使用的系统与方案

本仓库没有引入统一的第三方配置框架（如 dotenv、configstore、zod-config 等），而是采用自实现的分层配置体系，核心由 packages/coding-agent 提供，辅以 packages/ai 的 API Key 发现逻辑。配置来源按优先级分为三层：

- 包级元数据：从 package.json 的 piConfig 字段读取应用名、配置目录名。
- 用户全局配置：JSON 文件 ~/.pi/agent/settings.json（可通过环境变量覆盖）。
- 项目级配置：JSON 文件 <project>/.pi/settings.json（受信任后才加载）。
- 模型配置：JSON 文件 ~/.pi/agent/models.json，使用 TypeBox schema 校验。
- 认证存储：~/.pi/agent/auth.json。
- 环境变量：用于覆盖路径、API Key、代理、分享查看器等运行时开关。

## 2. 关键文件与包

| 文件 | 职责 |
|---|---|
| packages/coding-agent/src/config.ts | 应用名、版本、包路径、用户目录 (~/.pi/agent)、CLI 安装方式检测、自更新命令生成 |
| packages/coding-agent/src/core/settings-manager.ts | 全局/项目双层 Settings 管理，JSON 持久化、深合并、迁移、锁文件并发保护 |
| packages/coding-agent/src/core/model-config.ts | models.json 的 TypeBox schema 定义与不可变加载 |
| packages/ai/src/env-api-keys.ts | 各 LLM 提供商的 API Key 环境变量映射与发现 |
| packages/ai/src/utils/provider-env.ts | 统一的环境变量读取入口（支持注入 ProviderEnv 便于测试） |
| packages/coding-agent/examples/extensions/tool-override.ts | 示例中声明 secrets/credentials 文件匹配模式（.json/.yaml/.yml/.toml） |
| packages/server/src/server.ts | 服务端通过构造函数选项注入配置（无文件/环境变量读取） |
| packages/client/src/*.ts | 客户端纯函数式 API，不直接读配置 |
| package.json (根) | npm workspaces、构建脚本、发布流程编排 |
| .npmrc | save-exact=true、min-release-age=2 等 npm 行为配置 |

## 3. 架构与设计约定

### 3.1 应用与用户目录解析 (config.ts)
- 通过 process.env.PI_PACKAGE_DIR 可覆盖包根目录（适配 Nix/Guix 等 store 路径）。
- 自动检测运行环境：Bun 编译二进制、Node dist、tsx 源码三种模式，分别定位主题、HTML 模板、交互资源目录。
- 通过扫描 __dirname 向上查找 package.json 确定包根。
- 根据 process.execPath 和 node_modules 路径推断安装方式（npm/pnpm/yarn/bun），并据此生成自更新命令。
- 应用名来自 package.json.piConfig.name，默认 @earendil-works/pi-coding-agent；配置目录名来自 piConfig.configDir，默认 .pi。
- 导出常量 ENV_AGENT_DIR = "${APP_NAME.toUpperCase()}_CODING_AGENT_DIR"，允许用 PI_CODING_AGENT_DIR / TAU_CODING_AGENT_DIR 覆盖用户目录。

### 3.2 设置管理器 (settings-manager.ts)
- 双层作用域：global（~/.pi/agent/settings.json）与 project（<cwd>/.pi/settings.json），最终通过 deepMergeSettings 合并。
- 项目可信度控制：当 projectTrusted=false 时跳过项目配置加载，且禁止写入项目配置（assertProjectTrainedForWrite）。
- 并发安全：使用 proper-lockfile 对 settings.json 加锁读写，避免多进程竞争。
- 增量持久化：追踪本次会话修改的字段与嵌套字段，仅写回变更部分，而非全量覆盖。
- 向后兼容迁移：内置 migrateSettings 处理历史字段重命名（如 queueMode→steeringMode、websockets boolean→transport enum、retry.maxDelayMs→retry.provider.maxRetryDelayMs、skills 对象→数组）。
- 错误隔离：全局/项目配置解析失败被记录到 errors 队列，不影响其他作用域。
- 可插拔存储：FileSettingsStorage 与 InMemorySettingsStorage 实现同一 SettingsStorage 接口，便于测试。

### 3.3 模型配置 (model-config.ts)
- 使用 typebox 定义 ModelsConfigSchema，对 providers 下的每个 provider 进行严格校验。
- 加载后调用 deepFreeze(structuredClone(provider)) 返回不可变快照。
- 支持注释的 JSON（通过 stripJsonComments）。
- 缺失文件视为空配置，解析/校验错误以字符串形式暴露给调用方。

### 3.4 API Key 与环境变量发现 (packages/ai/src/env-api-keys.ts)
- 维护一个 provider → env var 映射表（如 openai → OPENAI_API_KEY、anthropic → ANTHROPIC_AUTH_TOKEN/OAUTH_TOKEN/API_KEY、google → GEMINI_API_KEY 等）。
- 特殊处理：
  - google-vertex：支持显式 GOOGLE_APPLICATION_CREDENTIALS 或默认 ADC 路径（~/.config/gcloud/application_default_credentials.json）。
  - amazon-bedrock：支持 AWS_PROFILE、AWS_ACCESS_KEY_ID+SECRET、BEARER_TOKEN、ECS 容器凭证、IRSA 等多种来源。
  - github-copilot：使用 COPILOT_GITHUB_TOKEN。
- 所有环境变量通过 getProviderEnvValue(key, env) 读取，便于在测试中注入假 env。
- 浏览器环境通过动态 import() 延迟加载 node:fs/os/path，避免破坏 Vite 构建。

### 3.5 服务端/客户端配置传递
- server.ts 的 PiServer 通过构造函数 PiServerOptions 接收 listeners、maxFrameLength、handshakeTimeoutMs 等参数，并在 resolveOptions 中进行类型与范围校验，不直接读取文件或环境变量。
- client 包同样以构造选项形式接收传输配置，保持无状态。

## 4. 约定与约束

| 约定 | 说明 | 依据 |
|---|---|---|
| 用户配置目录固定为 ~/.pi/agent | 可通过 PI_CODING_AGENT_DIR 或 TAU_CODING_AGENT_DIR 覆盖 | config.ts 导出 ENV_AGENT_DIR 并使用 |
| 项目配置目录固定为 <cwd>/.pi | 可通过 package.json.piConfig.configDir 自定义 | config.ts 中 CONFIG_DIR_NAME |
| 设置文件必须为 JSON | 使用 JSON.parse 解析，支持注释（settings-manager 未启用注释，但 model-config 启用了 stripJsonComments） | settings-manager.ts 与 model-config.ts |
| 项目配置仅在“可信”时生效 | 不可信时不加载也不允许写入 | SettingsManager.setProjectTrusted 与 assertProjectTrainedForWrite |
| 并发写保护 | 所有 settings.json 写操作通过 proper-lockfile 加锁 | FileSettingsStorage.acquireLockSyncWithRetry |
| 新增设置需走 setter + 标记修改 | 直接修改内部对象不会持久化，必须通过 setter 调用 markModified | SettingsManager 的所有 setter 模式 |
| 旧设置字段会被迁移 | 迁移逻辑集中放在 migrateSettings | 代码中的 queueMode/websockets/skills/retry 迁移 |
| API Key 必须通过标准环境变量名 | 每个 provider 有固定的环境变量映射，不支持任意键 | getApiKeyEnvVars 映射表 |
| 浏览器环境不得静态导入 Node 模块 | 使用条件动态 import 避免破坏打包 | env-api-keys.ts 顶部注释与实现 |
| 服务端/客户端包不直接读配置 | 通过构造参数注入，保持可测试性 | server.ts、client.ts 的选项模式 |
| 根工作区使用 npm workspaces | 子包通过 packages/*、session-backends/*、coding-agent/examples/extensions/* 聚合 | 根 package.json.workspaces |
| npm 依赖锁定精确版本 | .npmrc 设置 save-exact=true | .npmrc |