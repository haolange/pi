---
kind: logging_system
name: 日志系统：基于 @earendil-works/pi-telemetry 的遥测/追踪框架，无传统结构化日志输出
category: logging_system
scope:
    - '**'
source_files:
    - packages/telemetry/src/index.ts
    - packages/telemetry/src/memory.ts
    - packages/telemetry/src/noop.ts
    - packages/agent/src/harness/telemetry.ts
    - packages/agent/src/index.ts
    - packages/ai/src/types.ts
---

## 1. 使用的系统/方法

本仓库**没有传统意义上的日志系统**（如 winston、pino、bunyan、log4js 等），也没有统一的 `console.log` 封装或日志级别管理。相反，仓库通过一个独立的 `packages/telemetry` 包提供**遥测与分布式追踪**能力，作为可观测性的核心机制。

该包导出以下核心抽象：
- `TelemetryContext` / `TelemetrySpan`：定义 `startSpan`、`addEvent`、`setAttributes`、`setStatus` 的上下文与 span 接口。
- `NOOP_TELEMETRY_CONTEXT`：空实现，当应用未注入上下文时使用，保证调用方代码零开销。
- `InMemoryTelemetryContext`：内存中记录 span、事件、属性、状态，用于测试与调试。
- `createTypedSpanStarter` + `defineTelemetrySchema`：通过 TypeScript 类型推断约束 span 名称、起止属性与事件字段，使 span 使用具备强类型保障。

依赖此包的模块（如 `packages/agent`、`packages/ai`）通过 `@earendil-works/pi-telemetry` 导入并使用 span，而非直接写日志。

## 2. 关键文件

- `packages/telemetry/src/index.ts`：定义 `TelemetryContext`、`TelemetrySpan`、`SpanOptions`、`SpanStatus`、`TelemetrySchemaDefinition`、`createTypedSpanStarter`、`defineTelemetrySchema` 等全部公共 API。
- `packages/telemetry/src/memory.ts`：`InMemoryTelemetryContext` 的实现，记录 `RecordedTelemetrySpan` / `RecordedTelemetryEvent`，支持断言式测试。
- `packages/telemetry/src/noop.ts`：`NOOP_TELEMETRY_CONTEXT`，冻结的空实现。
- `packages/agent/src/harness/telemetry.ts`、`packages/agent/src/index.ts`：实际消费 telemetry 的示例，使用 `createTypedSpanStarter` 启动带类型约束的 span。
- `packages/ai/src/types.ts`：以 `TelemetryContext` 作为类型边界，表明 AI 层也通过该接口接入遥测。

## 3. 架构与约定

- **后端无关**：`TelemetryContext` 是纯接口，由上层应用注入具体实现；默认提供 noop 与内存实现，便于测试与生产替换。
- **span 即“日志”单位**：业务行为被建模为 span，附带 name、attributes、events、status；错误通过 `setStatus({ status: "error", error })` 显式标记，或在异常时自动推导。
- **类型驱动 schema**：通过 `defineTelemetrySchema` 声明 span 的起止属性与事件 schema，再由 `createTypedSpanStarter` 绑定到上下文，获得编译期校验——新增属性必须匹配 schema，否则 TS 报错。
- **被动记录**：所有 span 操作均包裹 try/catch，确保遥测失败不会污染主流程（见 memory 实现中的注释 “Recording is passive. Ignore malformed or unreadable telemetry payloads.”）。
- **测试友好**：`InMemoryTelemetryContext.getSpans()` 返回不可变快照，配合 `vitest` 断言 span 链路与属性。

## 4. 约定与约束

- **不使用传统日志库**：仓库内未发现对 winston/pino/bunyan/log4js/debug 等的引入；脚本类文件（如 `packages/ai/scripts/*.ts`）直接使用 `console.log` / `console.error` 进行一次性生成任务输出，不属于运行时日志体系。
- **遥测通过 `@earendil-works/pi-telemetry` 统一接入**：业务代码应通过注入的 `TelemetryContext` 创建 span，而不是自行维护 logger 实例。
- **span 命名与属性受 schema 约束**：新增 span 需先在 schema 中声明其 start/end attributes 与 events，否则无法通过类型检查。
- **错误处理模式固定**：span 可通过 `setStatus` 显式设置错误，或在 callback 抛错时自动设为 `{ status: "error" }`；未显式设置且成功结束时默认为 `{ status: "ok" }`。
- **无日志级别概念**：当前体系只有 span/event/status，没有 info/warn/error/debug 等分级；若需要传统日志输出，需在应用层另行集成（仓库未提供）。