# generateTypesAPI 分析文档索引

本文档集详细分析了 `generateTypesAPI` 方法的实现机制、参数说明和执行流程。

## 文档列表

### 1. [概述文档](./01-analysis.md)

- 方法概述和核心功能
- 执行模式说明
- 快速开始示例
- 输出产物说明

### 2. [参数详细说明](./02-parameters.md)

- `DTSManagerOptions` 接口详解
- `RemoteOptions` 所有参数说明
- `HostOptions` 参数说明
- 参数使用示例

### 3. [实现细节分析](./03-implementation.md)

- 主进程模式实现
- 子进程模式实现
- `DTSManager.generateTypes()` 核心逻辑
- 关键工具函数说明
- 错误处理机制
- 性能优化策略

### 4. [执行流程详解](./04-flow.md)

- 完整的执行流程图
- 详细的步骤说明
- 输出文件结构
- 错误处理流程
- 性能考虑

### 5. [插件集成分析](./05-plugin-integration.md)

- `GenerateTypesPlugin` 如何调用 `generateTypesAPI`
- 参数构造流程 (`normalizeGenerateTypesOptions`)
- 前置处理（配置检查、资源信息获取、缓存检查）
- 后置处理（文件输出、错误处理）
- 生产环境与开发环境的差异
- Webpack 钩子集成机制

## 快速导航

### 按主题查找

**参数配置**:

- 查看 [参数详细说明](./02-parameters.md) 了解所有可配置参数
- 查看 [概述文档](./01-analysis.md) 了解快速开始

**实现原理**:

- 查看 [实现细节分析](./03-implementation.md) 了解内部实现
- 查看 [执行流程详解](./04-flow.md) 了解执行步骤
- 查看 [插件集成分析](./05-plugin-integration.md) 了解 Webpack 插件如何调用

**使用指南**:

- 查看 [概述文档](./01-analysis.md) 了解基本用法
- 查看 [参数详细说明](./02-parameters.md) 了解配置选项

## 关键概念

### 执行模式

- **主进程模式**: 在当前进程中直接执行，同步阻塞
- **子进程模式**: 在独立子进程中执行，通过 RPC 通信

### 核心步骤

1. **配置解析**: 解析 Module Federation 配置和 TypeScript 配置
2. **类型提取**: 可选地提取远程类型和第三方类型
3. **TypeScript 编译**: 编译暴露的组件为类型定义文件
4. **归档创建**: 将类型文件打包为 ZIP 归档
5. **API 类型生成**: 可选地生成 API 类型文件

### 输出产物

- `@mf-types.zip`: 类型归档文件
- `@mf-types.d.ts`: API 类型文件（可选）
- `@mf-types/compiled-types/`: 编译后的类型文件目录

## 相关文件位置

### 源代码文件

- **入口方法**: `packages/dts-plugin/src/plugins/GenerateTypesPlugin.ts`
- **主进程实现**: `packages/dts-plugin/src/core/lib/generateTypes.ts`
- **子进程实现**: `packages/dts-plugin/src/core/lib/generateTypesInChildProcess.ts`
- **核心管理器**: `packages/dts-plugin/src/core/lib/DTSManager.ts`
- **TypeScript 编译**: `packages/dts-plugin/src/core/lib/typeScriptCompiler.ts`
- **归档处理**: `packages/dts-plugin/src/core/lib/archiveHandler.ts`
- **配置解析**: `packages/dts-plugin/src/core/configurations/remotePlugin.ts`

### 接口定义

- **DTSManagerOptions**: `packages/dts-plugin/src/core/interfaces/DTSManagerOptions.ts`
- **RemoteOptions**: `packages/dts-plugin/src/core/interfaces/RemoteOptions.ts`
- **HostOptions**: `packages/dts-plugin/src/core/interfaces/HostOptions.ts`

## 更新日志

- 2024-XX-XX: 初始版本，完成 `generateTypesAPI` 方法的全面分析
- 2024-XX-XX: 新增插件集成分析文档，详细说明 `GenerateTypesPlugin` 的调用机制
