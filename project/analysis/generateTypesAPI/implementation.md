# generateTypesAPI 实现细节分析

## 方法实现

### 核心实现代码

```typescript
// packages/dts-plugin/src/plugins/GenerateTypesPlugin.ts

const getGenerateTypesFn = (dtsManagerOptions: DTSManagerOptions) => {
  let fn: typeof generateTypes | typeof generateTypesInChildProcess =
    generateTypes;
  if (dtsManagerOptions.remote.compileInChildProcess) {
    fn = generateTypesInChildProcess;
  }
  return fn;
};

export const generateTypesAPI = ({
  dtsManagerOptions,
}: {
  dtsManagerOptions: DTSManagerOptions;
}) => {
  const fn = getGenerateTypesFn(dtsManagerOptions);
  return fn(dtsManagerOptions);
};
```

### 执行模式选择

`generateTypesAPI` 根据 `compileInChildProcess` 参数选择执行模式：

1. **主进程模式** (`generateTypes`):
   - 直接在当前进程中执行
   - 同步阻塞，适合简单场景
   - 实现文件: `packages/dts-plugin/src/core/lib/generateTypes.ts`

2. **子进程模式** (`generateTypesInChildProcess`):
   - 在独立子进程中执行
   - 通过 RPC 通信，避免阻塞主进程
   - 实现文件: `packages/dts-plugin/src/core/lib/generateTypesInChildProcess.ts`

## 主进程模式实现

### generateTypes 函数

```typescript
// packages/dts-plugin/src/core/lib/generateTypes.ts

async function generateTypes(options: DTSManagerOptions) {
  const DTSManagerConstructor = getDTSManagerConstructor(
    options.remote?.implementation,
  );
  const dtsManager = new DTSManagerConstructor(options);
  return dtsManager.generateTypes();
}
```

**执行步骤**:
1. 获取 DTSManager 构造函数（支持自定义实现）
2. 创建 DTSManager 实例
3. 调用 `generateTypes()` 方法

## 子进程模式实现

### generateTypesInChildProcess 函数

```typescript
// packages/dts-plugin/src/core/lib/generateTypesInChildProcess.ts

async function generateTypesInChildProcess(options: DTSManagerOptions) {
  const dtsWorker = new DtsWorker(options);
  return dtsWorker.controlledPromise;
}
```

**执行步骤**:
1. 创建 `DtsWorker` 实例
2. 返回受控的 Promise，等待子进程完成

### DtsWorker 类

```typescript
// packages/dts-plugin/src/core/lib/DtsWorker.ts

export class DtsWorker {
  rpcWorker: RpcWorker<RpcMethod>;
  private _options: DtsWorkerOptions;
  private _res: Promise<any>;

  constructor(options: DtsWorkerOptions) {
    this._options = cloneDeepOptions(options);
    this.removeUnSerializationOptions();
    this.rpcWorker = createRpcWorker(
      path.resolve(__dirname, './fork-generate-dts.js'),
      {},
      undefined,
      true,
    );
    this._res = this.rpcWorker.connect(this._options);
  }

  get controlledPromise(): ReturnType<DTSManager['generateTypes']> {
    // 确保子进程正确退出
    return Promise.resolve(this._res)
      .then(() => {
        this.exit();
        ensureChildProcessExit();
      })
      .catch((err) => {
        ensureChildProcessExit();
      });
  }
}
```

**关键特性**:
- 使用 RPC 机制与子进程通信
- 自动清理不可序列化的选项（如 manifest）
- 确保子进程正确退出

## DTSManager.generateTypes() 核心实现

### 方法签名

```typescript
// packages/dts-plugin/src/core/lib/DTSManager.ts

async generateTypes() {
  // 实现细节见下文
}
```

### 完整执行流程

#### 1. 配置验证和获取

```typescript
const { remoteOptions, tsConfig, mapComponentsToExpose } =
  retrieveRemoteConfig(options.remote);
```

**retrieveRemoteConfig 函数** (`packages/dts-plugin/src/core/configurations/remotePlugin.ts`):
- 验证配置选项
- 解析 `exposes` 配置，构建 `mapComponentsToExpose` 映射
- 读取并处理 `tsconfig.json`
- 生成用于编译的临时 tsconfig

**关键处理**:
- 解析 exposes 路径，支持多种文件扩展名（.ts, .tsx, .vue, .svelte）
- 计算有效的 `rootDir`
- 获取所有依赖文件
- 配置编译器选项（`emitDeclarationOnly: true`, `declaration: true`）

#### 2. 提取远程类型（可选）

```typescript
await this.extractRemoteTypes({
  remoteOptions,
  tsConfig,
  mapComponentsToExpose,
});
```

**extractRemoteTypes 方法**:
- 如果 `extractRemoteTypes` 为 `true` 且配置了 `remotes`
- 从 Host 应用的 `typesFolder` 复制远程类型到当前项目的 `node_modules`
- 用于支持 Remote 应用依赖其他 Remote 应用的类型

#### 3. 编译 TypeScript

```typescript
await compileTs(mapComponentsToExpose, tsConfig, remoteOptions);
```

**compileTs 函数** (`packages/dts-plugin/src/core/lib/typeScriptCompiler.ts`):

**步骤**:
1. **创建临时 tsconfig**:
   - 使用 `writeTempTsConfig` 创建临时配置文件
   - 文件名基于配置内容的 MD5 哈希

2. **执行 TypeScript 编译器**:
   ```typescript
   const cmd = `${pmExecutable} ${remoteOptions.compilerInstance} --project '${tempTsConfigJsonPath}'`;
   await execPromise(cmd, { cwd: ... });
   ```
   - 使用 `tsc` 或自定义编译器实例
   - 通过 `--project` 指定临时 tsconfig

3. **处理编译输出**:
   - 遍历编译后的 `.d.ts` 文件
   - 为每个 expose 的组件创建入口文件
   - 入口文件格式:
     ```typescript
     export * from './relative/path';
     export { default } from './relative/path';
     ```

4. **提取第三方类型（可选）**:
   - 如果 `extractThirdParty` 为 `true`
   - 使用 `ThirdPartyExtractor` 提取第三方依赖的类型
   - 复制到 `node_modules` 目录

5. **清理临时文件**:
   - 如果 `deleteTsConfig` 为 `true`，删除临时 tsconfig

#### 4. 创建类型归档

```typescript
await createTypesArchive(tsConfig, remoteOptions);
```

**createTypesArchive 函数** (`packages/dts-plugin/src/core/lib/archiveHandler.ts`):

**步骤**:
1. 获取类型文件目录路径: `@mf-types/compiled-types/`
2. 使用 `AdmZip` 将整个目录打包为 ZIP 文件
3. 输出文件: `@mf-types.zip`

#### 5. 生成 API 类型文件（可选）

```typescript
if (remoteOptions.generateAPITypes) {
  const apiTypes = this.generateAPITypes(mapComponentsToExpose);
  apiTypesPath = retrieveMfAPITypesPath(tsConfig, remoteOptions);
  fs.writeFileSync(apiTypesPath, apiTypes);
}
```

**generateAPITypes 方法**:

**生成的类型定义**:
```typescript
export type RemoteKeys = 'REMOTE_ALIAS_IDENTIFIER/Button' | 'REMOTE_ALIAS_IDENTIFIER/Header';

type PackageType<T> = 
  T extends 'REMOTE_ALIAS_IDENTIFIER/Button' ? typeof import('REMOTE_ALIAS_IDENTIFIER/Button') :
  T extends 'REMOTE_ALIAS_IDENTIFIER/Header' ? typeof import('REMOTE_ALIAS_IDENTIFIER/Header') :
  any;
```

**作用**:
- 提供类型安全的远程模块导入
- 支持 IDE 自动补全和类型检查
- 使用 `REMOTE_ALIAS_IDENTIFIER` 作为占位符，在 Host 应用消费时会被替换

#### 6. 清理类型文件夹（可选）

```typescript
if (remoteOptions.deleteTypesFolder) {
  await rm(retrieveMfTypesPath(tsConfig, remoteOptions), {
    recursive: true,
    force: true,
  });
}
```

**说明**:
- 如果 `deleteTypesFolder` 为 `true`，删除中间生成的类型文件夹
- ZIP 归档文件会保留
- 减少输出目录的大小

## 关键工具函数

### retrieveMfTypesPath

```typescript
export const retrieveMfTypesPath = (
  tsConfig: TsConfigJson,
  remoteOptions: Required<RemoteOptions>,
) =>
  normalize(
    tsConfig.compilerOptions.outDir!.replace(
      remoteOptions.compiledTypesFolder,
      '',
    ),
  );
```

**作用**: 计算类型文件的根目录路径（`@mf-types/`）

### retrieveMfAPITypesPath

```typescript
export const retrieveMfAPITypesPath = (
  tsConfig: TsConfigJson,
  remoteOptions: Required<RemoteOptions>,
) =>
  join(
    retrieveOriginalOutDir(tsConfig, remoteOptions),
    `${remoteOptions.typesFolder}.d.ts`,
  );
```

**作用**: 计算 API 类型文件的路径（`@mf-types.d.ts`）

### retrieveTypesZipPath

```typescript
export const retrieveTypesZipPath = (
  mfTypesPath: string,
  remoteOptions: Required<RemoteOptions>,
) =>
  join(
    mfTypesPath.replace(remoteOptions.typesFolder, ''),
    `${remoteOptions.typesFolder}.zip`,
  );
```

**作用**: 计算类型归档文件的路径（`@mf-types.zip`）

## 错误处理

### 错误处理策略

```typescript
try {
  // 生成类型
} catch (error) {
  if (this.options.remote?.abortOnError === false) {
    if (this.options.displayErrorInTerminal) {
      logger.error(error);
    }
  } else {
    throw error;
  }
}
```

**行为**:
- 如果 `abortOnError` 为 `true`（默认），抛出异常
- 如果 `abortOnError` 为 `false`，记录错误但不抛出
- 根据 `displayErrorInTerminal` 决定是否在终端显示错误

## 性能优化

### 增量编译

- 使用 TypeScript 的增量编译功能（`incremental: true`）
- 缓存编译信息到 `.tsbuildinfo` 文件
- 仅在类型文件夹不存在时清理缓存

### 子进程隔离

- 使用子进程模式避免阻塞主进程
- 通过 RPC 通信，支持异步执行
- 自动清理子进程资源

### 临时文件管理

- 临时 tsconfig 文件使用哈希命名，避免冲突
- 支持自动清理临时文件
- 使用系统临时目录存放临时文件

## 扩展性

### 自定义 DTSManager

通过 `implementation` 参数可以指定自定义的 DTSManager 实现：

```typescript
remote: {
  implementation: './custom-dts-manager.js',
  // ...
}
```

### 自定义编译器

通过 `compilerInstance` 参数可以指定自定义编译器：

```typescript
remote: {
  compilerInstance: 'vue-tsc',
  // ...
}
```

