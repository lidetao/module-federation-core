# consumeTypesAPI 实现细节分析

## 方法实现

### 核心实现代码

```typescript
// packages/dts-plugin/src/plugins/ConsumeTypesPlugin.ts

export const consumeTypesAPI = async (
  dtsManagerOptions: DTSManagerOptions,
  cb?: (options: moduleFederationPlugin.RemoteTypeUrls) => void,
) => {
  const fetchRemoteTypeUrlsPromise =
    typeof dtsManagerOptions.host.remoteTypeUrls === 'function'
      ? dtsManagerOptions.host.remoteTypeUrls()
      : Promise.resolve(dtsManagerOptions.host.remoteTypeUrls);
  return fetchRemoteTypeUrlsPromise.then((remoteTypeUrls) => {
    consumeTypes({
      ...dtsManagerOptions,
      host: {
        ...dtsManagerOptions.host,
        remoteTypeUrls,
      },
    })
      .then(() => {
        typeof cb === 'function' && cb(remoteTypeUrls);
      })
      .catch(() => {
        typeof cb === 'function' && cb(remoteTypeUrls);
      });
  });
};
```

### 执行流程

1. **处理 remoteTypeUrls**: 
   - 如果是函数，调用函数获取 Promise
   - 如果是 Promise，直接使用
   - 如果是对象，包装为 Promise
   - 如果是 undefined，返回 undefined

2. **调用 consumeTypes**: 
   - 将处理后的 `remoteTypeUrls` 合并到 `host` 配置中
   - 调用 `consumeTypes` 函数执行类型消费

3. **执行回调**: 
   - 无论成功或失败，都会调用回调函数
   - 回调函数接收 `remoteTypeUrls` 作为参数

## consumeTypes 函数实现

### 函数签名

```typescript
// packages/dts-plugin/src/core/lib/consumeTypes.ts

async function consumeTypes(options: DTSManagerOptions): Promise<void> {
  const DTSManagerConstructor = getDTSManagerConstructor(
    options.host?.implementation,
  );
  const dtsManager = new DTSManagerConstructor(options);
  await dtsManager.consumeTypes();
}
```

**执行步骤**:
1. 获取 DTSManager 构造函数（支持自定义实现）
2. 创建 DTSManager 实例
3. 调用 `consumeTypes()` 方法

## DTSManager.consumeTypes() 核心实现

### 方法签名

```typescript
// packages/dts-plugin/src/core/lib/DTSManager.ts

async consumeTypes() {
  // 实现细节见下文
}
```

### 完整执行流程

#### 1. 配置验证和获取

```typescript
const { options } = this;
if (!options.host) {
  throw new Error('options.host is required if you want to consumeTypes');
}
const { mapRemotesToDownload } = retrieveHostConfig(options.host);
if (!Object.keys(mapRemotesToDownload).length) {
  return;
}
```

**retrieveHostConfig 函数** (`packages/dts-plugin/src/core/configurations/hostPlugin.ts`):
- 验证配置选项
- 解析 `remotes` 配置，构建 `mapRemotesToDownload` 映射
- 处理 `remoteTypeUrls`，构建远程信息映射

**关键处理**:
- 解析 remotes 配置，支持多种格式（对象、数组、函数）
- 从 `remoteTypeUrls` 或 remotes URL 构建 `RemoteInfo`
- 计算类型文件的 URL 路径

#### 2. 消费归档类型

```typescript
const { downloadPromisesResult, hostOptions } =
  await this.consumeArchiveTypes(options.host);
```

**consumeArchiveTypes 方法**:

**步骤**:
1. **获取配置**:
   ```typescript
   const { hostOptions, mapRemotesToDownload } = retrieveHostConfig(options);
   ```

2. **请求远程 Manifest（如果需要）**:
   ```typescript
   const downloadPromises = Object.entries(mapRemotesToDownload).map(
     async (item) => {
       const remoteInfo = item[1] as RemoteInfo;
       if (!this.remoteAliasMap[remoteInfo.alias]) {
         const requiredRemoteInfo = await this.requestRemoteManifest(
           remoteInfo,
           hostOptions,
         );
         this.remoteAliasMap[remoteInfo.alias] = requiredRemoteInfo;
       }
       return this.consumeTargetRemotes(
         hostOptions,
         this.remoteAliasMap[remoteInfo.alias],
       );
     },
   );
   ```

3. **并行下载**:
   ```typescript
   const downloadPromisesResult = await Promise.allSettled(downloadPromises);
   ```

**requestRemoteManifest 方法**:
- 如果 `remoteInfo.url` 包含 `MANIFEST_EXT`，请求 manifest 文件
- 从 manifest 中提取 `types.zip` 和 `types.api` URL
- 处理 `publicPath`，支持 `auto` 模式
- 构建完整的类型文件 URL

**consumeTargetRemotes 方法**:
- 调用 `downloadTypesArchive` 下载类型归档
- 返回 `[alias, destinationPath]` 元组

#### 3. 下载 API 类型（可选）

```typescript
if (hostOptions.consumeAPITypes) {
  await Promise.all(
    downloadPromisesResult.map(async (item) => {
      if (item.status === 'rejected' || !item.value) {
        return;
      }
      const [alias, destinationPath] = item.value;
      const remoteInfo = this.remoteAliasMap[alias];
      if (!remoteInfo) {
        return;
      }

      await this.downloadAPITypes(
        remoteInfo,
        destinationPath,
        hostOptions,
      );
    }),
  );
  this.consumeAPITypes(hostOptions);
}
```

**downloadAPITypes 方法**:

**步骤**:
1. **检查 API 类型 URL**:
   ```typescript
   const { apiTypeUrl } = remoteInfo;
   if (!apiTypeUrl) {
     return;
   }
   ```

2. **下载 API 类型文件**:
   ```typescript
   const res = await axiosGet(url, {
     timeout: hostOptions.timeout,
     family: hostOptions.family,
   });
   ```

3. **替换别名标识符**:
   ```typescript
   let apiTypeFile = res.data as string;
   apiTypeFile = apiTypeFile.replaceAll(
     REMOTE_ALIAS_IDENTIFIER,
     remoteInfo.alias,
   );
   ```

4. **写入文件**:
   ```typescript
   const filePath = path.join(destinationPath, REMOTE_API_TYPES_FILE_NAME);
   fs.writeFileSync(filePath, apiTypeFile);
   ```

**consumeAPITypes 方法**:

**步骤**:
1. **读取已存在的 API 类型文件**:
   ```typescript
   const apiTypeFileName = path.join(
     hostOptions.context,
     hostOptions.typesFolder,
     HOST_API_TYPES_FILE_NAME,
   );
   const existedFile = fs.readFileSync(apiTypeFileName, 'utf-8');
   ```

2. **收集已导入的类型**:
   ```typescript
   const existedImports = new ThirdPartyExtractor({
     destDir: '',
   }).collectTypeImports(existedFile);
   existedImports.forEach((existedImport) => {
     const alias = existedImport
       .split('./')
       .slice(1)
       .join('./')
       .replace('/apis.d.ts', '');
     this.loadedRemoteAPIAlias.add(alias);
   });
   ```

3. **生成导入语句**:
   ```typescript
   const importTypeStr = [...this.loadedRemoteAPIAlias]
     .sort()
     .map((alias, index) => {
       const remoteKey = `RemoteKeys_${index}`;
       const packageType = `PackageType_${index}`;
       packageTypes.push(`T extends ${remoteKey} ? ${packageType}<T>`);
       remoteKeys.push(remoteKey);
       return `import type { PackageType as ${packageType},RemoteKeys as ${remoteKey} } from './${alias}/apis.d.ts';`;
     })
     .join('\n');
   ```

4. **生成类型声明**:
   ```typescript
   const remoteKeysStr = `type RemoteKeys = ${remoteKeys.join(' | ')};`;
   const packageTypesStr = `type PackageType<T, Y=any> = ${[
     ...packageTypes,
     'Y',
   ].join(' :\n')} ;`;
   ```

5. **生成运行时包声明**:
   ```typescript
   const pkgsDeclareStr = [...runtimePkgs]
     .map((pkg) => {
       return `declare module "${pkg}" {
         ${remoteKeysStr}
         ${packageTypesStr}
         export function loadRemote<T extends RemoteKeys,Y>(packageName: T): Promise<PackageType<T, Y>>;
         export function loadRemote<T extends string,Y>(packageName: T): Promise<PackageType<T, Y>>;
       }`;
     })
     .join('\n');
   ```

6. **写入文件**:
   ```typescript
   const fileStr = `${importTypeStr}
   ${pkgsDeclareStr}
   `;
   fs.writeFileSync(
     path.join(
       hostOptions.context,
       hostOptions.typesFolder,
       HOST_API_TYPES_FILE_NAME,
     ),
     fileStr,
   );
   ```

## 关键工具函数

### retrieveHostConfig

```typescript
export const retrieveHostConfig = (options: HostOptions) => {
  validateOptions(options);
  const hostOptions: Required<HostOptions> = { ...defaultOptions, ...options };
  const mapRemotesToDownload = resolveRemotes(hostOptions);
  return {
    hostOptions,
    mapRemotesToDownload,
  };
};
```

**作用**: 
- 验证并规范化 Host 配置
- 解析 remotes 配置，构建远程信息映射

### retrieveRemoteInfo

```typescript
export const retrieveRemoteInfo = (options: {
  hostOptions: Required<HostOptions>;
  remoteAlias: string;
  remote: string;
}): RemoteInfo => {
  // 解析远程 URL
  // 构建 zipUrl 和 apiTypeUrl
  // 处理 remoteTypeUrls 覆盖
}
```

**作用**: 
- 从 remotes 配置或 `remoteTypeUrls` 构建 `RemoteInfo`
- 计算类型文件的 URL 路径

### downloadTypesArchive

```typescript
export const downloadTypesArchive = (hostOptions: Required<HostOptions>) => {
  let retries = 0;
  return async ([destinationFolder, fileToDownload]: string[]): Promise<
    [string, string] | undefined
  > => {
    // 重试逻辑
    // 下载 ZIP 文件
    // 解压到目标目录
  };
};
```

**作用**: 
- 下载类型归档文件
- 支持重试机制
- 解压 ZIP 文件到目标目录

### axiosGet

```typescript
export async function axiosGet(url: string, config?: AxiosRequestConfig) {
  const httpAgent = new http.Agent({ family: config?.family ?? 4 });
  const httpsAgent = new https.Agent({ family: config?.family ?? 4 });
  return axios.get(url, {
    httpAgent,
    httpsAgent,
    ...{
      headers: getEnvHeaders(),
    },
    ...config,
    timeout: config?.timeout || 60000,
  });
}
```

**作用**: 
- 统一的 HTTP 请求方法
- 支持 IPv4/IPv6 配置
- 支持环境变量中的自定义 headers
- 默认超时时间 60 秒

## 错误处理

### 错误处理策略

```typescript
try {
  // 消费类型
} catch (err) {
  if (this.options.host?.abortOnError === false) {
    fileLog(
      `Unable to consume federated types, ${err}`,
      'consumeTypes',
      'error',
    );
  } else {
    throw err;
  }
}
```

**行为**:
- 如果 `abortOnError` 为 `true`（默认），抛出异常
- 如果 `abortOnError` 为 `false`，记录错误但不抛出
- 使用 `fileLog` 记录错误日志

### 下载错误处理

```typescript
catch (error: any) {
  fileLog(
    `Error during types archive download: ${
      error?.message || 'unknown error'
    }`,
    'downloadTypesArchive',
    'error',
  );
  if (retries >= hostOptions.maxRetries) {
    logger.error(
      `Failed to download types archive from "${fileToDownload}". Set FEDERATION_DEBUG=true for details.`,
    );
    if (hostOptions.abortOnError !== false) {
      throw error;
    }
    return undefined;
  }
}
```

**行为**:
- 支持重试机制，最多重试 `maxRetries` 次
- 达到最大重试次数后，根据 `abortOnError` 决定是否抛出异常
- 记录详细的错误日志

## 性能优化

### 并行下载

- 使用 `Promise.allSettled` 并行下载所有远程类型
- 即使某个远程下载失败，其他远程仍可继续下载

### 缓存机制

- 使用 `remoteAliasMap` 缓存已请求的远程信息
- 避免重复请求 manifest 文件

### 重试机制

- 支持配置重试次数
- 自动重试失败的下载请求

## 扩展性

### 自定义 DTSManager

通过 `implementation` 参数可以指定自定义的 DTSManager 实现：

```typescript
host: {
  implementation: './custom-dts-manager.js',
  // ...
}
```

### 自定义 remoteTypeUrls

支持多种形式的 `remoteTypeUrls`：
- 对象：直接指定 URL 映射
- Promise：异步获取 URL 映射
- 函数：动态获取 URL 映射
