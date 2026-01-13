# consumeTypesAPI 执行流程详解

## 整体流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    consumeTypesAPI                           │
│                  (入口方法)                                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
         ┌─────────────────────────────┐
         │  处理 remoteTypeUrls         │
         │  (函数/Promise/对象)         │
         └──────────┬──────────────────┘
                    │
                    ▼
         ┌─────────────────────────────┐
         │  consumeTypes()             │
         │  (核心消费逻辑)               │
         └──────────┬──────────────────┘
                    │
                    ▼
         ┌────────────────────────────┐
         │  DTSManager.consumeTypes()  │
         │  (核心管理器)               │
         └──────────┬─────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
┌──────────────────┐   ┌──────────────────────┐
│ retrieveHost    │   │ consumeArchiveTypes  │
│ Config           │   │ (下载归档类型)        │
└────────┬─────────┘   └──────────┬───────────┘
         │                        │
         │                        ▼
         │         ┌────────────────────────────┐
         │         │  requestRemoteManifest    │
         │         │  (请求 Manifest)           │
         │         └──────────┬────────────────┘
         │                    │
         │                    ▼
         │         ┌────────────────────────────┐
         │         │  consumeTargetRemotes      │
         │         │  (下载并解压 ZIP)           │
         │         └──────────┬────────────────┘
         │                    │
         └──────────┬─────────┘
                    │
                    ▼
         ┌────────────────────────────┐
         │  downloadAPITypes (可选)    │
         │  (下载 API 类型)             │
         └──────────┬─────────────────┘
                    │
                    ▼
         ┌────────────────────────────┐
         │  consumeAPITypes (可选)     │
         │  (合并 API 类型)             │
         └────────────────────────────┘
```

## 详细执行步骤

### 阶段 1: 入口和参数处理

#### 步骤 1.1: 调用 consumeTypesAPI

```typescript
await consumeTypesAPI(dtsManagerOptions, callback);
```

**输入**:
- `dtsManagerOptions`: 完整的 DTS 管理器配置选项
- `callback`: 可选的回调函数

**处理**:
- 处理 `remoteTypeUrls` 参数（函数/Promise/对象）

#### 步骤 1.2: 处理 remoteTypeUrls

```typescript
const fetchRemoteTypeUrlsPromise =
  typeof dtsManagerOptions.host.remoteTypeUrls === 'function'
    ? dtsManagerOptions.host.remoteTypeUrls()
    : Promise.resolve(dtsManagerOptions.host.remoteTypeUrls);
```

**决策逻辑**:
- 如果是函数，调用函数获取 Promise
- 如果是 Promise，直接使用
- 如果是对象或 undefined，包装为 Promise

### 阶段 2: consumeTypes 函数调用

#### 步骤 2.1: 获取 DTSManager 构造函数

```typescript
const DTSManagerConstructor = getDTSManagerConstructor(
  options.host?.implementation,
);
```

**处理逻辑**:
- 如果提供了 `implementation`，加载自定义实现
- 否则使用默认的 `DTSManager` 类

#### 步骤 2.2: 创建 DTSManager 实例

```typescript
const dtsManager = new DTSManagerConstructor(options);
```

**初始化内容**:
- 保存配置选项
- 初始化运行时包列表
- 初始化远程别名映射
- 初始化已加载的 API 别名集合

### 阶段 3: 配置解析和验证

#### 步骤 3.1: 调用 retrieveHostConfig

```typescript
const { mapRemotesToDownload } = retrieveHostConfig(options.host);
```

**子步骤**:

1. **验证选项**:
   ```typescript
   validateOptions(options);
   // 检查 moduleFederationConfig 是否存在
   ```

2. **规范化配置**:
   ```typescript
   const hostOptions: Required<HostOptions> = { ...defaultOptions, ...options };
   ```

3. **解析 remotes 配置**:
   ```typescript
   const mapRemotesToDownload = resolveRemotes(hostOptions);
   ```

**resolveRemotes 处理逻辑**:
- 解析 `remotes` 配置，支持多种格式（对象、数组、函数）
- 从 `remoteTypeUrls` 构建初始远程信息映射
- 遍历 remotes 配置，调用 `retrieveRemoteInfo` 构建 `RemoteInfo`
- 合并 `remoteTypeUrls` 和 remotes 配置的结果

**retrieveRemoteInfo 处理逻辑**:
- 解析远程 URL，提取 entry 路径
- 如果 `remoteTypeUrls` 中有对应项，使用其 URL
- 否则根据 entry URL 构建 ZIP 和 API 类型 URL
- 返回 `RemoteInfo` 对象

**输出**:
- `hostOptions`: 完整的 Host 选项配置
- `mapRemotesToDownload`: 远程别名到 `RemoteInfo` 的映射

#### 步骤 3.2: 验证是否有需要下载的远程

```typescript
if (!Object.keys(mapRemotesToDownload).length) {
  return; // 没有需要下载的远程，直接返回
}
```

### 阶段 4: 消费归档类型

#### 步骤 4.1: 调用 consumeArchiveTypes

```typescript
const { downloadPromisesResult, hostOptions } =
  await this.consumeArchiveTypes(options.host);
```

#### 步骤 4.2: 请求远程 Manifest（如果需要）

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
    // ...
  },
);
```

**requestRemoteManifest 处理逻辑**:

1. **检查是否需要请求 Manifest**:
   ```typescript
   if (!remoteInfo.url.includes(MANIFEST_EXT)) {
     return remoteInfo as Required<RemoteInfo>;
   }
   ```

2. **请求 Manifest 文件**:
   ```typescript
   const res = await axiosGet(url, {
     timeout: hostOptions.timeout,
     family: hostOptions.family,
   });
   const manifestJson = res.data as unknown as Manifest;
   ```

3. **提取类型 URL**:
   ```typescript
   if (!manifestJson.metaData.types.zip) {
     throw new Error(`Can not get ${remoteInfo.name}'s types archive url!`);
   }
   ```

4. **处理 publicPath**:
   ```typescript
   let publicPath;
   if ('publicPath' in manifestJson.metaData) {
     publicPath = manifestJson.metaData.publicPath;
   } else {
     const getPublicPath = new Function(manifestJson.metaData.getPublicPath);
     publicPath = getPublicPath();
   }
   if (publicPath === 'auto') {
     publicPath = inferAutoPublicPath(remoteInfo.url);
   }
   ```

5. **构建完整 URL**:
   ```typescript
   remoteInfo.zipUrl = new URL(
     path.join(addProtocol(publicPath), manifestJson.metaData.types.zip),
   ).href;
   remoteInfo.apiTypeUrl = new URL(
     path.join(addProtocol(publicPath), manifestJson.metaData.types.api),
   ).href;
   ```

#### 步骤 4.3: 下载并解压类型归档

```typescript
return this.consumeTargetRemotes(
  hostOptions,
  this.remoteAliasMap[remoteInfo.alias],
);
```

**consumeTargetRemotes 处理逻辑**:

1. **验证 ZIP URL**:
   ```typescript
   if (!remoteInfo.zipUrl) {
     throw new Error(`Can not get ${remoteInfo.name}'s types archive url!`);
   }
   ```

2. **调用下载函数**:
   ```typescript
   const typesDownloader = downloadTypesArchive(hostOptions);
   return typesDownloader([remoteInfo.alias, remoteInfo.zipUrl]);
   ```

**downloadTypesArchive 处理逻辑**:

1. **构建目标路径**:
   ```typescript
   const destinationPath = retrieveTypesArchiveDestinationPath(
     hostOptions,
     destinationFolder,
   );
   // 返回: @mf-types/{alias}/
   ```

2. **重试循环下载**:
   ```typescript
   while (retries++ < hostOptions.maxRetries) {
     try {
       // 下载 ZIP 文件
     } catch (error) {
       // 重试或抛出错误
     }
   }
   ```

3. **下载 ZIP 文件**:
   ```typescript
   const response = await axiosGet(url, {
     responseType: 'arraybuffer',
     timeout: hostOptions.timeout,
     family: hostOptions.family,
   });
   ```

4. **验证内容类型**:
   ```typescript
   if (
     typeof response.headers?.['content-type'] === 'string' &&
     response.headers['content-type'].includes('text/html')
   ) {
     throw new Error(`Invalid content-type: ${response.headers['content-type']}`);
   }
   ```

5. **删除旧类型文件夹（可选）**:
   ```typescript
   if (hostOptions.deleteTypesFolder) {
     await rm(destinationPath, {
       recursive: true,
       force: true,
     });
   }
   ```

6. **解压 ZIP 文件**:
   ```typescript
   const zip = new AdmZip(Buffer.from(response.data));
   zip.extractAllTo(destinationPath, true);
   ```

7. **返回结果**:
   ```typescript
   return [destinationFolder, destinationPath];
   ```

#### 步骤 4.4: 等待所有下载完成

```typescript
const downloadPromisesResult = await Promise.allSettled(downloadPromises);
```

**说明**:
- 使用 `Promise.allSettled` 确保所有 Promise 都完成
- 即使某些下载失败，其他下载仍可继续

### 阶段 5: 下载 API 类型（可选）

#### 步骤 5.1: 检查是否需要下载 API 类型

```typescript
if (hostOptions.consumeAPITypes) {
  // 下载 API 类型
}
```

#### 步骤 5.2: 并行下载所有远程的 API 类型

```typescript
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
```

**downloadAPITypes 处理逻辑**:

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
   let apiTypeFile = res.data as string;
   ```

3. **替换别名标识符**:
   ```typescript
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

### 阶段 6: 合并 API 类型（可选）

#### 步骤 6.1: 调用 consumeAPITypes

```typescript
this.consumeAPITypes(hostOptions);
```

#### 步骤 6.2: 读取已存在的 API 类型文件

```typescript
const apiTypeFileName = path.join(
  hostOptions.context,
  hostOptions.typesFolder,
  HOST_API_TYPES_FILE_NAME,
);
try {
  const existedFile = fs.readFileSync(apiTypeFileName, 'utf-8');
  // 收集已导入的类型
} catch (err) {
  // 文件不存在，忽略
}
```

#### 步骤 6.3: 收集已导入的类型

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

#### 步骤 6.4: 生成导入语句和类型声明

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

#### 步骤 6.5: 生成运行时包声明

```typescript
const runtimePkgs: Set<string> = new Set();
[...this.runtimePkgs, ...hostOptions.runtimePkgs].forEach((pkg) => {
  runtimePkgs.add(pkg);
});

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

#### 步骤 6.6: 写入合并后的 API 类型文件

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

### 阶段 7: 完成和回调

#### 步骤 7.1: 记录成功日志

```typescript
logger.success('Federated types extraction completed');
```

#### 步骤 7.2: 执行回调函数

```typescript
// 在 consumeTypesAPI 中
.then(() => {
  typeof cb === 'function' && cb(remoteTypeUrls);
})
.catch(() => {
  typeof cb === 'function' && cb(remoteTypeUrls);
});
```

**说明**: 无论成功或失败，都会调用回调函数

## 输出文件结构

执行完成后，生成的文件结构如下：

```
@mf-types/
├── apis.d.ts                    # 合并后的 API 类型文件（如果启用）
└── {remote-alias}/              # 远程类型文件目录
    ├── apis.d.ts                # 远程 API 类型文件（如果启用）
    └── compiled-types/          # 编译后的类型文件
        ├── Button.d.ts
        ├── Header.d.ts
        └── ...
```

## 错误处理流程

```
┌─────────────────┐
│   执行步骤      │
└────────┬────────┘
         │
         ▼
    ┌─────────┐
    │ 发生错误 │
    └────┬────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
┌────────┐ ┌──────────────┐
│abortOn │ │abortOnError  │
│Error=  │ │=false        │
│true    │ │              │
└───┬────┘ └──────┬───────┘
    │             │
    ▼             ▼
┌────────┐  ┌─────────────┐
│抛出异常 │  │记录错误日志  │
│        │  │继续执行      │
└────────┘  └─────────────┘
```

**下载错误处理**:
- 支持重试机制，最多重试 `maxRetries` 次
- 达到最大重试次数后，根据 `abortOnError` 决定是否抛出异常
- 使用 `Promise.allSettled` 确保部分失败不影响其他下载

## 性能考虑

### 并行下载

- 使用 `Promise.allSettled` 并行下载所有远程类型
- 即使某个远程下载失败，其他远程仍可继续下载

### 缓存机制

- 使用 `remoteAliasMap` 缓存已请求的远程信息
- 避免重复请求 manifest 文件

### 重试机制

- 支持配置重试次数
- 自动重试失败的下载请求

### 错误容错

- 支持 `abortOnError` 配置，控制错误处理策略
- 部分失败不影响整体流程
