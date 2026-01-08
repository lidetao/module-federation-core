# generateTypesAPI 参数详细说明

## 参数结构

`generateTypesAPI` 方法接收一个对象参数，包含 `dtsManagerOptions` 字段：

```typescript
generateTypesAPI({
  dtsManagerOptions: DTSManagerOptions
})
```

## DTSManagerOptions 接口定义

```typescript
interface DTSManagerOptions {
  remote?: RemoteOptions;           // 远程模块配置（必需）
  host?: HostOptions;               // 主机应用配置（可选）
  extraOptions?: Record<string, any>; // 额外选项
  displayErrorInTerminal?: boolean;  // 是否在终端显示错误
}
```

## RemoteOptions 参数详解

`RemoteOptions` 是生成类型定义的核心配置，继承自 `moduleFederationPlugin.DtsRemoteOptions`。

### 必需参数

#### 1. `moduleFederationConfig`
- **类型**: `moduleFederationPlugin.ModuleFederationPluginOptions`
- **说明**: Module Federation 插件配置对象
- **作用**: 包含 `name`、`exposes` 等关键配置，用于确定需要生成类型的模块
- **示例**:
  ```typescript
  moduleFederationConfig: {
    name: 'remote-app',
    exposes: {
      './Button': './src/Button.tsx',
      './Header': './src/Header.tsx',
    }
  }
  ```

### 可选参数

#### 2. `context`
- **类型**: `string`
- **默认值**: `process.cwd()`
- **说明**: 项目根目录路径
- **作用**: 作为所有相对路径的解析基础

#### 3. `outputDir`
- **类型**: `string`
- **默认值**: `''` (使用 tsconfig 中的 outDir)
- **说明**: 输出目录
- **作用**: 指定类型文件的输出位置

#### 4. `tsConfigPath`
- **类型**: `string`
- **默认值**: `'./tsconfig.json'`
- **说明**: TypeScript 配置文件路径
- **作用**: 指定用于编译的 tsconfig.json 文件位置

#### 5. `typesFolder`
- **类型**: `string`
- **默认值**: `'@mf-types'`
- **说明**: 类型文件夹名称
- **作用**: 指定生成的类型文件存放的文件夹名称

#### 6. `compiledTypesFolder`
- **类型**: `string`
- **默认值**: `'compiled-types'`
- **说明**: 编译后的类型文件夹名称
- **作用**: 指定编译后的类型文件存放的子文件夹名称

#### 7. `hostRemoteTypesFolder`
- **类型**: `string`
- **默认值**: `'@mf-types'`
- **说明**: Host 应用的远程类型文件夹
- **作用**: 当 `extractRemoteTypes` 为 true 时，从此文件夹复制类型

#### 8. `deleteTypesFolder`
- **类型**: `boolean`
- **默认值**: `true`
- **说明**: 是否在生成后删除类型文件夹
- **作用**: 控制是否保留中间生成的类型文件夹（归档文件会保留）

#### 9. `deleteTsConfig`
- **类型**: `boolean`
- **默认值**: `true`
- **说明**: 是否删除临时生成的 tsconfig 文件
- **作用**: 控制是否清理临时配置文件

#### 10. `additionalFilesToCompile`
- **类型**: `string[]`
- **默认值**: `[]`
- **说明**: 额外需要编译的文件列表
- **作用**: 指定除了 exposes 中的文件外，还需要编译的文件

#### 11. `compilerInstance`
- **类型**: `'tsc' | string`
- **默认值**: `'tsc'`
- **说明**: TypeScript 编译器实例
- **作用**: 指定使用的 TypeScript 编译器命令

#### 12. `compileInChildProcess`
- **类型**: `boolean`
- **默认值**: `false`
- **说明**: 是否在子进程中编译
- **作用**: 
  - `true`: 在独立子进程中执行，避免阻塞主进程
  - `false`: 在主进程中执行

#### 13. `implementation`
- **类型**: `string`
- **默认值**: `''`
- **说明**: 自定义 DTSManager 实现路径
- **作用**: 允许使用自定义的 DTSManager 实现类

#### 14. `generateAPITypes`
- **类型**: `boolean`
- **默认值**: `false`
- **说明**: 是否生成 API 类型文件
- **作用**: 
  - `true`: 生成 `@mf-types.d.ts` API 类型文件，提供类型提示
  - `false`: 不生成 API 类型文件

#### 15. `abortOnError`
- **类型**: `boolean`
- **默认值**: `true`
- **说明**: 遇到错误时是否中止
- **作用**: 
  - `true`: 遇到错误时抛出异常
  - `false`: 遇到错误时记录日志但不抛出异常

#### 16. `extractRemoteTypes`
- **类型**: `boolean`
- **默认值**: `false`
- **说明**: 是否提取远程类型
- **作用**: 
  - `true`: 从 Host 应用的 `typesFolder` 复制远程类型到当前项目的 `node_modules`
  - `false`: 不提取远程类型

#### 17. `extractThirdParty`
- **类型**: `boolean | { exclude?: string[] }`
- **默认值**: `false`
- **说明**: 是否提取第三方依赖的类型
- **作用**: 
  - `true`: 提取所有第三方依赖的类型定义
  - `{ exclude: ['package1', 'package2'] }`: 提取第三方类型，但排除指定包
  - `false`: 不提取第三方类型

## HostOptions 参数详解

`HostOptions` 用于配置 Host 应用相关的选项，主要用于类型消费场景。

### 必需参数

#### 1. `moduleFederationConfig`
- **类型**: `moduleFederationPlugin.ModuleFederationPluginOptions`
- **说明**: Module Federation 插件配置对象
- **作用**: 包含 `remotes` 配置，用于确定需要消费的远程模块

### 可选参数

#### 2. `context`
- **类型**: `string`
- **默认值**: `process.cwd()`
- **说明**: 项目根目录路径

#### 3. `typesFolder`
- **类型**: `string`
- **默认值**: `'@mf-types'`
- **说明**: 类型文件夹名称
- **作用**: 指定下载的远程类型文件存放位置

#### 4. `remoteTypeUrls`
- **类型**: `Record<string, string> | Promise<Record<string, string>> | undefined`
- **说明**: 远程类型 URL 映射
- **作用**: 手动指定远程模块的类型文件 URL，覆盖自动发现

#### 5. `consumeAPITypes`
- **类型**: `boolean`
- **默认值**: `false`
- **说明**: 是否消费 API 类型
- **作用**: 是否下载并合并远程模块的 API 类型文件

#### 6. `timeout`
- **类型**: `number`
- **默认值**: `60000` (60秒)
- **说明**: 下载超时时间（毫秒）

#### 7. `maxRetries`
- **类型**: `number`
- **默认值**: `3`
- **说明**: 最大重试次数
- **作用**: 下载失败时的重试次数

#### 8. `family`
- **类型**: `4 | 6`
- **默认值**: `4`
- **说明**: IP 地址族
- **作用**: 指定使用 IPv4 还是 IPv6

#### 9. `runtimePkgs`
- **类型**: `string[]`
- **默认值**: `['@module-federation/runtime', ...]`
- **说明**: 运行时包列表
- **作用**: 指定需要注入类型声明的运行时包

#### 10. `abortOnError`
- **类型**: `boolean`
- **默认值**: `true`
- **说明**: 遇到错误时是否中止

## extraOptions 参数

- **类型**: `Record<string, any>`
- **说明**: 额外选项对象
- **作用**: 用于传递自定义配置，供扩展功能使用

## displayErrorInTerminal 参数

- **类型**: `boolean`
- **说明**: 是否在终端显示错误信息
- **作用**: 控制错误信息的输出方式

## 参数使用示例

### 基础配置

```typescript
const options: DTSManagerOptions = {
  remote: {
    context: process.cwd(),
    outputDir: 'dist',
    moduleFederationConfig: {
      name: 'remote-app',
      exposes: {
        './Button': './src/Button.tsx',
      },
    },
    generateAPITypes: true,
    compileInChildProcess: false,
  },
};
```

### 完整配置

```typescript
const options: DTSManagerOptions = {
  remote: {
    context: process.cwd(),
    outputDir: 'dist',
    tsConfigPath: './tsconfig.json',
    typesFolder: '@mf-types',
    compiledTypesFolder: 'compiled-types',
    moduleFederationConfig: {
      name: 'remote-app',
      exposes: {
        './Button': './src/Button.tsx',
        './Header': './src/Header.tsx',
      },
    },
    generateAPITypes: true,
    compileInChildProcess: true,
    extractThirdParty: {
      exclude: ['@types/node'],
    },
    extractRemoteTypes: true,
    abortOnError: false,
    deleteTypesFolder: true,
    deleteTsConfig: true,
    additionalFilesToCompile: ['./src/types.ts'],
  },
  host: {
    context: process.cwd(),
    typesFolder: '@mf-types',
    moduleFederationConfig: {
      name: 'host-app',
      remotes: {
        'remote-app': 'remote-app@http://localhost:3001/remoteEntry.js',
      },
    },
    consumeAPITypes: true,
    timeout: 60000,
    maxRetries: 3,
  },
  extraOptions: {
    customOption: 'value',
  },
  displayErrorInTerminal: true,
};
```

