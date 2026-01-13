# consumeTypesAPI 参数详细说明

## 参数结构

`consumeTypesAPI` 方法接收两个参数：

```typescript
consumeTypesAPI(
  dtsManagerOptions: DTSManagerOptions,
  cb?: (options: moduleFederationPlugin.RemoteTypeUrls) => void
)
```

## DTSManagerOptions 接口定义

```typescript
interface DTSManagerOptions {
  host?: HostOptions;               // 主机应用配置（必需）
  remote?: RemoteOptions;           // 远程模块配置（可选）
  extraOptions?: Record<string, any>; // 额外选项
  displayErrorInTerminal?: boolean;  // 是否在终端显示错误
}
```

## HostOptions 参数详解

`HostOptions` 是消费类型定义的核心配置，继承自 `moduleFederationPlugin.DtsHostOptions`。

### 必需参数

#### 1. `moduleFederationConfig`
- **类型**: `moduleFederationPlugin.ModuleFederationPluginOptions`
- **说明**: Module Federation 插件配置对象
- **作用**: 包含 `name`、`remotes` 等关键配置，用于确定需要消费的远程模块
- **示例**:
  ```typescript
  moduleFederationConfig: {
    name: 'host-app',
    remotes: {
      'remote-app': 'remote-app@http://localhost:3001/remoteEntry.js',
    }
  }
  ```

### 可选参数

#### 2. `context`
- **类型**: `string`
- **默认值**: `process.cwd()`
- **说明**: 项目根目录路径
- **作用**: 作为所有相对路径的解析基础

#### 3. `typesFolder`
- **类型**: `string`
- **默认值**: `'@mf-types'`
- **说明**: 类型文件夹名称
- **作用**: 指定下载的远程类型文件存放位置

#### 4. `remoteTypesFolder`
- **类型**: `string`
- **默认值**: `'@mf-types'`
- **说明**: 远程类型文件夹名称
- **作用**: 用于构建远程类型 ZIP 文件的 URL 路径

#### 5. `remoteTypeUrls`
- **类型**: `Record<string, { zip: string; api: string; alias: string }> | Promise<Record<string, { zip: string; api: string; alias: string }>> | (() => Promise<Record<string, { zip: string; api: string; alias: string }>>) | undefined`
- **默认值**: `undefined`
- **说明**: 远程类型 URL 映射
- **作用**: 
  - 手动指定远程模块的类型文件 URL，覆盖自动发现
  - 可以是对象、Promise 或函数
  - 格式: `{ 'remote-name': { zip: 'url', api: 'url', alias: 'alias' } }`

#### 6. `consumeAPITypes`
- **类型**: `boolean`
- **默认值**: `false`
- **说明**: 是否消费 API 类型
- **作用**: 
  - `true`: 下载并合并远程模块的 API 类型文件
  - `false`: 只下载类型归档文件，不处理 API 类型

#### 7. `timeout`
- **类型**: `number`
- **默认值**: `60000` (60秒)
- **说明**: 下载超时时间（毫秒）
- **作用**: 控制下载请求的超时时间

#### 8. `maxRetries`
- **类型**: `number`
- **默认值**: `3`
- **说明**: 最大重试次数
- **作用**: 下载失败时的重试次数

#### 9. `family`
- **类型**: `4 | 6`
- **默认值**: `4`
- **说明**: IP 地址族
- **作用**: 指定使用 IPv4 还是 IPv6

#### 10. `runtimePkgs`
- **类型**: `string[]`
- **默认值**: `[]`
- **说明**: 运行时包列表
- **作用**: 指定需要注入类型声明的运行时包，默认为 `['@module-federation/runtime', '@module-federation/enhanced/runtime', '@module-federation/runtime-tools']`

#### 11. `abortOnError`
- **类型**: `boolean`
- **默认值**: `true`
- **说明**: 遇到错误时是否中止
- **作用**: 
  - `true`: 遇到错误时抛出异常
  - `false`: 遇到错误时记录日志但不抛出异常

#### 12. `deleteTypesFolder`
- **类型**: `boolean`
- **默认值**: `true`
- **说明**: 是否在下载前删除已存在的类型文件夹
- **作用**: 控制是否清理旧的类型文件

#### 13. `typesOnBuild`
- **类型**: `boolean`
- **默认值**: `false`
- **说明**: 是否在构建时消费类型
- **作用**: 
  - `true`: 在生产环境构建时也消费类型
  - `false`: 仅在开发环境消费类型

#### 14. `implementation`
- **类型**: `string`
- **默认值**: `''`
- **说明**: 自定义 DTSManager 实现路径
- **作用**: 允许使用自定义的 DTSManager 实现类

## RemoteOptions 参数详解

`RemoteOptions` 在 `consumeTypesAPI` 中是可选的，主要用于类型生成场景。在消费类型时，通常不需要配置。

## extraOptions 参数

- **类型**: `Record<string, any>`
- **说明**: 额外选项对象
- **作用**: 用于传递自定义配置，供扩展功能使用

## displayErrorInTerminal 参数

- **类型**: `boolean`
- **说明**: 是否在终端显示错误信息
- **作用**: 控制错误信息的输出方式

## 回调函数参数

### cb 回调函数

- **类型**: `(options: moduleFederationPlugin.RemoteTypeUrls) => void`
- **说明**: 类型消费完成后的回调函数
- **参数**: `RemoteTypeUrls` - 远程类型 URL 映射对象
- **作用**: 在类型消费完成后执行，无论成功或失败都会调用

## 参数使用示例

### 基础配置

```typescript
const options: DTSManagerOptions = {
  host: {
    context: process.cwd(),
    moduleFederationConfig: {
      name: 'host-app',
      remotes: {
        'remote-app': 'remote-app@http://localhost:3001/remoteEntry.js',
      },
    },
    consumeAPITypes: true,
  },
};
```

### 完整配置

```typescript
const options: DTSManagerOptions = {
  host: {
    context: process.cwd(),
    typesFolder: '@mf-types',
    remoteTypesFolder: '@mf-types',
    moduleFederationConfig: {
      name: 'host-app',
      remotes: {
        'remote-app': 'remote-app@http://localhost:3001/remoteEntry.js',
      },
    },
    consumeAPITypes: true,
    timeout: 60000,
    maxRetries: 3,
    family: 4,
    runtimePkgs: ['@module-federation/runtime'],
    abortOnError: false,
    deleteTypesFolder: true,
    typesOnBuild: false,
  },
  extraOptions: {
    customOption: 'value',
  },
  displayErrorInTerminal: true,
};
```

### 使用 remoteTypeUrls

```typescript
const options: DTSManagerOptions = {
  host: {
    context: process.cwd(),
    moduleFederationConfig: {
      name: 'host-app',
      remotes: {
        'remote-app': 'remote-app@http://localhost:3001/remoteEntry.js',
      },
    },
    remoteTypeUrls: {
      'remote-app': {
        zip: 'http://localhost:3001/@mf-types.zip',
        api: 'http://localhost:3001/@mf-types.d.ts',
        alias: 'remote-app',
      },
    },
    consumeAPITypes: true,
  },
};
```

### 使用 Promise 形式的 remoteTypeUrls

```typescript
const options: DTSManagerOptions = {
  host: {
    context: process.cwd(),
    moduleFederationConfig: {
      name: 'host-app',
      remotes: {
        'remote-app': 'remote-app@http://localhost:3001/remoteEntry.js',
      },
    },
    remoteTypeUrls: async () => {
      // 异步获取远程类型 URL
      const response = await fetch('/api/remote-types');
      return response.json();
    },
    consumeAPITypes: true,
  },
};
```
