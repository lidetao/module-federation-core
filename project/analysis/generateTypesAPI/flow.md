# generateTypesAPI 执行流程详解

## 整体流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    generateTypesAPI                         │
│                  (入口方法)                                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
         ┌─────────────────────────────┐
         │  getGenerateTypesFn          │
         │  选择执行模式                │
         └──────────┬──────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
┌───────────────┐      ┌──────────────────────┐
│ generateTypes │      │generateTypesInChild  │
│ (主进程模式)   │      │Process (子进程模式)   │
└───────┬───────┘      └──────────┬───────────┘
        │                         │
        │                         ▼
        │              ┌──────────────────────┐
        │              │   DtsWorker          │
        │              │   (RPC 通信)        │
        │              └──────────┬───────────┘
        │                         │
        └─────────────┬───────────┘
                      │
                      ▼
         ┌────────────────────────────┐
         │  DTSManager.generateTypes() │
         │  (核心生成逻辑)              │
         └──────────┬─────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
┌──────────────────┐   ┌──────────────────────┐
│ retrieveRemote   │   │ extractRemoteTypes   │
│ Config           │   │ (可选)               │
└────────┬─────────┘   └──────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  compileTs                           │
│  (TypeScript 编译)                    │
│  ┌─────────────────────────────────┐ │
│  │ 1. 创建临时 tsconfig            │ │
│  │ 2. 执行 tsc 编译                │ │
│  │ 3. 处理编译输出                 │ │
│  │ 4. 提取第三方类型(可选)         │ │
│  └─────────────────────────────────┘ │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│  createTypesArchive                 │
│  (创建 ZIP 归档)                     │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│  generateAPITypes (可选)             │
│  (生成 API 类型文件)                  │
└──────────┬──────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│  清理临时文件 (可选)                  │
└─────────────────────────────────────┘
```

## 详细执行步骤

### 阶段 1: 入口和模式选择

#### 步骤 1.1: 调用 generateTypesAPI

```typescript
await generateTypesAPI({ dtsManagerOptions });
```

**输入**:
- `dtsManagerOptions`: 完整的 DTS 管理器配置选项

**处理**:
- 调用 `getGenerateTypesFn` 根据 `compileInChildProcess` 选择执行函数

#### 步骤 1.2: 选择执行模式

```typescript
const fn = getGenerateTypesFn(dtsManagerOptions);
// 返回 generateTypes 或 generateTypesInChildProcess
```

**决策逻辑**:
- `compileInChildProcess === true` → 使用子进程模式
- `compileInChildProcess === false` → 使用主进程模式

### 阶段 2: DTSManager 初始化

#### 步骤 2.1: 获取 DTSManager 构造函数

```typescript
const DTSManagerConstructor = getDTSManagerConstructor(
  options.remote?.implementation,
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

#### 步骤 3.1: 调用 retrieveRemoteConfig

```typescript
const { remoteOptions, tsConfig, mapComponentsToExpose } =
  retrieveRemoteConfig(options.remote);
```

**子步骤**:

1. **验证选项**:
   ```typescript
   validateOptions(options);
   // 检查 moduleFederationConfig 是否存在
   ```

2. **解析 exposes 配置**:
   ```typescript
   const mapComponentsToExpose = resolveExposes(remoteOptions);
   // 将 exposes 配置转换为文件路径映射
   // 例如: { './Button': './src/Button.tsx' }
   ```

3. **读取 tsconfig.json**:
   ```typescript
   const tsConfig = readTsConfig(remoteOptions, mapComponentsToExpose);
   ```

4. **处理 tsconfig**:
   - 计算有效的 `rootDir`
   - 获取所有依赖文件
   - 配置编译器选项
   - 设置输出目录

**输出**:
- `remoteOptions`: 完整的远程选项配置
- `tsConfig`: 处理后的 TypeScript 配置
- `mapComponentsToExpose`: expose 键到文件路径的映射

#### 步骤 3.2: 验证是否有需要编译的文件

```typescript
if (!Object.keys(mapComponentsToExpose).length) {
  return; // 没有需要编译的文件，直接返回
}

if (!tsConfig.files?.length) {
  logger.info('No type files to compile, skip');
  return; // 没有类型文件，跳过编译
}
```

### 阶段 4: 提取远程类型（可选）

#### 步骤 4.1: 检查是否需要提取远程类型

```typescript
if (!remoteOptions.extractRemoteTypes) {
  return; // 不需要提取，跳过
}
```

#### 步骤 4.2: 检查是否有 remotes 配置

```typescript
let hasRemotes = false;
const remotes = remoteOptions.moduleFederationConfig.remotes;
// 检查 remotes 是否存在且不为空
```

#### 步骤 4.3: 从 Host 应用复制类型

```typescript
if (hasRemotes && this.options.host) {
  const { hostOptions } = retrieveHostConfig(this.options.host);
  const remoteTypesFolder = path.resolve(
    hostOptions.context,
    hostOptions.typesFolder,
  );
  const targetFolder = path.resolve(
    remoteOptions.context,
    path.join(mfTypesPath, 'node_modules'),
  );
  await fse.copy(remoteTypesFolder, targetFolder, { overwrite: true });
}
```

**作用**: 将 Host 应用下载的远程类型复制到当前 Remote 应用的 `node_modules`，用于类型解析

### 阶段 5: TypeScript 编译

#### 步骤 5.1: 创建临时 tsconfig

```typescript
const tempTsConfigJsonPath = writeTempTsConfig(
  tsConfig,
  remoteOptions.context,
  remoteOptions.moduleFederationConfig.name || 'mf',
);
```

**处理**:
- 生成基于配置内容的 MD5 哈希
- 创建临时文件: `node_modules/.cache/mf-types/tsconfig.{hash}.json`
- 写入处理后的 tsconfig 内容

#### 步骤 5.2: 初始化第三方类型提取器（可选）

```typescript
if (remoteOptions.extractThirdParty) {
  const thirdPartyExtractor = new ThirdPartyExtractor({
    destDir: resolve(mfTypePath, 'node_modules'),
    context: remoteOptions.context,
    exclude: remoteOptions.extractThirdParty.exclude,
  });
}
```

#### 步骤 5.3: 执行 TypeScript 编译

```typescript
const pmExecutable = resolvePackageManagerExecutable(); // 'npx', 'yarn', etc.
const cmd = `${pmExecutable} ${remoteOptions.compilerInstance} --project '${tempTsConfigJsonPath}'`;
await execPromise(cmd, { cwd: ... });
```

**执行**:
- 使用包管理器的可执行文件（npx/yarn）
- 运行 TypeScript 编译器（tsc/vue-tsc 等）
- 使用临时 tsconfig 文件

**错误处理**:
- 如果编译失败，清理 `.tsbuildinfo` 文件
- 抛出包含命令信息的错误

#### 步骤 5.4: 处理编译输出

```typescript
await processTypesFile({
  outDir: compilerOptions.outDir,
  filePath: compilerOptions.outDir,
  rootDir: compilerOptions.rootDir,
  mfTypePath,
  cb: thirdPartyExtractor.collectPkgs.bind(thirdPartyExtractor),
  mapExposeToEntry,
});
```

**处理逻辑**:
1. 遍历编译输出的所有 `.d.ts` 文件
2. 为每个 expose 的组件创建入口文件:
   ```typescript
   // 例如: @mf-types/compiled-types/Button.d.ts
   export * from './src/Button';
   export { default } from './src/Button';
   ```
3. 如果启用第三方类型提取，收集依赖包信息

#### 步骤 5.5: 复制第三方类型（可选）

```typescript
if (remoteOptions.extractThirdParty) {
  await thirdPartyExtractor.copyDts();
}
```

**作用**: 将第三方依赖的类型定义复制到 `node_modules` 目录

#### 步骤 5.6: 清理临时 tsconfig（可选）

```typescript
if (remoteOptions.deleteTsConfig) {
  await rm(tempTsConfigJsonPath);
}
```

### 阶段 6: 创建类型归档

#### 步骤 6.1: 获取类型文件路径

```typescript
const mfTypesPath = retrieveMfTypesPath(tsConfig, remoteOptions);
// 返回: @mf-types/compiled-types/
```

#### 步骤 6.2: 创建 ZIP 归档

```typescript
const zip = new AdmZip();
zip.addLocalFolder(mfTypesPath);
await zip.writeZipPromise(retrieveTypesZipPath(mfTypesPath, remoteOptions));
```

**输出**: `@mf-types.zip` 文件，包含所有类型定义文件

### 阶段 7: 生成 API 类型文件（可选）

#### 步骤 7.1: 检查是否需要生成 API 类型

```typescript
if (remoteOptions.generateAPITypes) {
  // 生成 API 类型
}
```

#### 步骤 7.2: 生成 API 类型内容

```typescript
const apiTypes = this.generateAPITypes(mapComponentsToExpose);
```

**生成内容示例**:
```typescript
export type RemoteKeys = 
  'REMOTE_ALIAS_IDENTIFIER/Button' | 
  'REMOTE_ALIAS_IDENTIFIER/Header';

type PackageType<T> = 
  T extends 'REMOTE_ALIAS_IDENTIFIER/Button' 
    ? typeof import('REMOTE_ALIAS_IDENTIFIER/Button') 
  : T extends 'REMOTE_ALIAS_IDENTIFIER/Header' 
    ? typeof import('REMOTE_ALIAS_IDENTIFIER/Header') 
  : any;
```

#### 步骤 7.3: 写入 API 类型文件

```typescript
apiTypesPath = retrieveMfAPITypesPath(tsConfig, remoteOptions);
// 路径: @mf-types.d.ts
fs.writeFileSync(apiTypesPath, apiTypes);
```

### 阶段 8: 清理和完成

#### 步骤 8.1: 清理类型文件夹（可选）

```typescript
if (remoteOptions.deleteTypesFolder) {
  await rm(retrieveMfTypesPath(tsConfig, remoteOptions), {
    recursive: true,
    force: true,
  });
}
```

**说明**: 删除中间生成的类型文件夹，但保留 ZIP 归档

#### 步骤 8.2: 记录成功日志

```typescript
logger.success('Federated types created correctly');
```

## 输出文件结构

执行完成后，生成的文件结构如下：

```
dist/
├── @mf-types.zip                    # 类型归档文件
├── @mf-types.d.ts                   # API 类型文件（如果启用）
└── @mf-types/                       # 类型文件夹（如果未删除）
    └── compiled-types/
        ├── Button.d.ts              # 编译后的类型文件
        ├── Header.d.ts
        └── node_modules/            # 第三方类型（如果提取）
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

## 性能考虑

### 增量编译

- 使用 TypeScript 增量编译，只编译变更的文件
- 缓存编译信息到 `.tsbuildinfo` 文件
- 仅在类型文件夹不存在时清理缓存

### 并行处理

- 子进程模式支持异步执行，不阻塞主进程
- 文件处理使用 Promise.all 并行处理

### 资源清理

- 自动清理临时文件
- 可选删除中间生成的类型文件夹
- 子进程自动退出和资源释放

