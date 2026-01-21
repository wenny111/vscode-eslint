# VS Code ESLint 扩展 - 架构与调试指南

## 2. 项目架构

### 2.1 整体架构模式

这是一个基于 **Language Server Protocol (LSP)** 的 VS Code 扩展，采用 **Client-Server 分离架构**：

```
┌─────────────────────────────────────────────────────────┐
│              VS Code Extension Host                     │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Client (client/src/)                            │   │
│  │  - extension.ts: 扩展入口，延迟激活               │   │
│  │  - client.ts: LSP 客户端实现                     │   │
│  │  - settings.ts: 配置读取                         │   │
│  │  - tasks.ts: 任务提供者                          │   │
│  └──────────────────────────────────────────────────┘   │
│                    ↕ LSP Protocol                       │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Server (独立 Node.js 进程)                       │   │
│  │  - eslintServer.ts: LSP 服务器实现                │   │
│  │  - eslint.ts: ESLint 封装与抽象                   │   │
│  │  - 处理诊断、代码操作、格式化                     │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### 2.2 核心目录结构

| 目录 | 职责 |
|------|------|
| `client/` | VS Code 扩展前端代码，负责激活、LSP 客户端、任务提供、状态集成 |
| `server/` | LSP 服务器实现，负责 lint 编排、诊断、代码操作 |
| `$shared/` | 共享协议消息和设置类型（通过符号链接到 `client/src/shared` 和 `server/src/shared`） |
| `build/` | CI 脚本、辅助 Node 脚本（`bin/all.js`、安装/链接步骤） |
| `playgrounds/` | 用于手动测试的示例项目（跨 ESLint 版本和配置风格） |

### 2.3 关键设计决策

#### 2.3.1 延迟激活（Lazy Activation）
- 扩展在 `onStartupFinished` 时注册，但不立即激活
- 只有当打开需要验证的文档或配置改变时才真正激活
- 减少资源消耗，提升启动性能

```typescript
// extension.ts 中的激活逻辑
function didOpenTextDocument(textDocument: TextDocument) {
    if (activated) return;
    if (validator.check(textDocument) !== Validate.off) {
        activated = true;
        realActivate(context);
    }
}
```

#### 2.3.2 ESLint 版本抽象
- 透明处理 ESLint 版本差异：
  - CLIEngine API（旧版本）
  - ESLint Class API（v8+）
  - Flat Config（v8.57+ / v9+）
- 通过 `eslint.ts` 统一接口，自动检测并使用合适的 API

#### 2.3.3 设置管理
- 设置按资源（resource）作用域管理，支持全局或按文件夹/文件
- 设置必须通过 `TextDocumentSettings` 缓存，不能存储在全局变量中
- 新增设置的步骤：
  1. 在根 `package.json` 的 `contributes.configuration.properties` 中添加 schema
  2. 在 `$shared/settings` 中扩展 `ConfigurationSettings`
  3. 在 `client/src/client.ts` 的 `readConfiguration()` 中映射检索
  4. 在 `eslint.ts` 逻辑中使用

#### 2.3.4 诊断拉取模式（Diagnostic Pull）
- 使用 LSP 的 `DocumentDiagnosticRequest` 而非推送模式
- 支持按需验证：`onChange`、`onSave`、`onFocus`
- 根据 `eslint.run` 设置过滤触发时机

### 2.4 核心工作流程

#### 2.4.1 文档验证流程
```
1. 用户打开/编辑文档
   ↓
2. Client 检测是否需要验证（validator.check()）
   ↓
3. Client 通过 LSP 发送 DocumentDiagnosticRequest
   ↓
4. Server 接收请求，解析设置（ESLint.resolveSettings()）
   ↓
5. Server 调用 ESLint.validate() 执行 lint
   ↓
6. Server 返回诊断结果（Diagnostic[]）
   ↓
7. Client 显示诊断信息（错误、警告、信息）
```

#### 2.4.2 代码操作流程
```
1. 用户触发代码操作（Quick Fix / Source Fix All）
   ↓
2. Client 发送 codeAction 请求
   ↓
3. Server 分析问题，生成代码操作（修复、禁用规则、建议等）
   ↓
4. Client 显示操作菜单
   ↓
5. 用户选择操作
   ↓
6. Client 发送 executeCommand 请求
   ↓
7. Server 执行操作，返回 WorkspaceEdit
   ↓
8. Client 应用编辑
```

### 2.5 多工作区支持

- 支持多个工作区文件夹
- 每个文件夹可以有不同的 ESLint 配置
- 通过 `eslint.workingDirectories` 设置工作目录
- 支持 glob 模式匹配（如 `{ "pattern": "code-*" }`）

---

## 3. 核心调试观察

### 3.1 调试配置

项目使用 `.vscode/launch.json` 进行调试：

```json
{
    "type": "extensionHost",
    "request": "launch",
    "name": "Launch Client",
    "runtimeExecutable": "${execPath}",
    "args": ["--extensionDevelopmentPath=${workspaceFolder}"],
    "sourceMaps": true,
    "autoAttachChildProcesses": true,
    "outFiles": [
        "${workspaceFolder}/client/out/**/*.js",
        "${workspaceFolder}/server/out/**/*.js"
    ],
    "preLaunchTask": "npm: watch"
}
```

**关键点：**
- `extensionHost` 类型：在扩展主机中运行客户端
- `autoAttachChildProcesses`：自动附加到子进程（服务器进程）
- `sourceMaps`：启用源映射，支持 TypeScript 调试
- `preLaunchTask`：启动前运行 `watch` 任务（编译 TypeScript）

### 3.2 构建系统

#### 3.2.1 开发模式
```bash
npm run watch          # TypeScript 增量编译（watch 模式）
npm run webpack:dev    # Webpack 打包（开发模式，无压缩）
```

#### 3.2.2 生产模式
```bash
npm run webpack        # Webpack 生产打包（压缩优化）
npm run esbuild       # 使用 esbuild 打包（更快）
```

#### 3.2.3 编译流程
- TypeScript 编译：`tsc -b`（项目引用模式）
- Webpack 打包：将 TypeScript 输出打包为单个文件
- 符号链接：`$shared` 通过 `symlink:lsp` 链接到 client/server

### 3.3 调试技巧

#### 3.3.1 客户端调试
- 在 `client/src/extension.ts` 或 `client/src/client.ts` 设置断点
- 观察扩展激活流程、LSP 客户端创建、配置读取

#### 3.3.2 服务器调试
- 在 `server/src/eslintServer.ts` 或 `server/src/eslint.ts` 设置断点
- 观察诊断生成、代码操作计算、ESLint 调用

#### 3.3.3 输出通道
- ESLint 输出通道：`View > Output > ESLint`
- 包含服务器日志、错误信息、调试输出
- 可通过 `eslint.debug` 设置启用 ESLint 调试模式

#### 3.3.4 LSP 跟踪
- 设置 `eslint.trace.server` 为 `"messages"` 或 `"verbose"`
- 在输出通道中查看完整的 LSP 通信

### 3.4 常见调试场景

#### 3.4.1 扩展未激活
- 检查是否有需要验证的文档打开
- 检查 `eslint.enable` 设置
- 检查 `eslint.probe` 设置是否包含文档语言

#### 3.4.2 服务器启动失败
- 检查 Node.js 路径（`eslint.runtime`）
- 检查 PATH 环境变量
- 查看输出通道的错误信息

#### 3.4.3 诊断不显示
- 检查 ESLint 配置是否存在
- 检查 `eslint.validate` 设置
- 检查文件是否被 `.eslintignore` 忽略
- 启用 `eslint.debug` 查看详细日志

#### 3.4.4 代码操作不工作
- 检查问题是否可修复（`fixable` 属性）
- 检查 `eslint.codeAction.disableRuleComment.enable` 设置
- 查看服务器日志中的错误

### 3.5 性能监控

#### 3.5.1 时间预算
- `eslint.timeBudget.onValidation`：验证时间预算（默认 4000ms 警告，8000ms 错误）
- `eslint.timeBudget.onFixes`：修复计算时间预算（默认 3000ms 警告，6000ms 错误）
- 超出预算会在语言状态指示器中显示

#### 3.5.2 状态指示器
- VS Code 语言状态栏显示 ESLint 状态
- 显示验证时间、错误状态、性能警告

### 3.6 测试

#### 3.6.1 单元测试
```bash
npm test              # 运行客户端测试
```

#### 3.6.2 手动测试

`playgrounds/` 目录包含多个示例项目，用于测试不同 ESLint 版本和配置风格。

**重要：ESLint 版本解析机制**

扩展会按以下顺序查找 ESLint 库：

1. **从文件所在目录向上查找** `node_modules/eslint`
   - 从文件所在目录开始，向上遍历父目录
   - 找到第一个包含 `node_modules/eslint` 的目录

2. **从工作目录查找**（如果设置了 `eslint.workingDirectories`）

3. **从全局路径查找**（如果本地找不到）

**这意味着：**
- 如果 playground 目录**没有自己的 `node_modules`**，会向上查找到**项目根目录**的 ESLint
- 项目根目录的 `package.json` 中 ESLint 是 `^9.28.0`
- 但 playground 的配置文件可能是为 ESLint 8.x 写的

**手动测试步骤：**

1. **打开 playground 目录作为工作区**
   ```bash
   # 方式 1：在 VS Code 中打开 playground 目录
   code playgrounds/flat-config

   # 方式 2：使用多根工作区
   code playgrounds/testing.code-workspace
   ```

2. **安装 playground 的依赖**（如果需要特定版本）
   ```bash
   cd playgrounds/flat-config
   npm install
   ```
   这会在 playground 目录创建 `node_modules`，使用其 `package.json` 中指定的 ESLint 版本。

3. **检查 ESLint 版本**
   - 查看输出通道：`View > Output > ESLint`
   - 应该看到：`ESLint library loaded from: <path>`
   - 路径会显示实际使用的 ESLint 位置

4. **测试不同场景：**
   - `playgrounds/7.0/` - ESLint 7.x + eslintrc 配置
   - `playgrounds/8.0/` - ESLint 8.x + eslintrc 配置
   - `playgrounds/9.0/flat/` - ESLint 9.x + flat config
   - `playgrounds/flat-config/` - ESLint 8.x/9.x + flat config（需要修复配置）

**常见问题：**

**问题：** 配置文件报错 `Unexpected non-object config at user-defined index 0`

**原因：** ESLint 9.x 不支持字符串形式的配置（如 `"eslint:recommended"`）

**解决：**
- 在 playground 目录安装对应版本的 ESLint：`npm install`
- 或更新配置文件以兼容 ESLint 9.x：
  ```javascript
  // ❌ ESLint 9.x 不支持
  module.exports = ["eslint:recommended", {...}]

  // ✅ ESLint 9.x 正确格式
  const js = require('@eslint/js');
  module.exports = [js.configs.recommended, {...}]
  ```

**问题：** 使用了项目根目录的 ESLint 而不是 playground 的

**解决：**
- 在 playground 目录运行 `npm install` 创建自己的 `node_modules`
- 或使用 `eslint.nodePath` 设置指定 ESLint 路径

**测试清单：**
- [ ] 打开 playground 目录
- [ ] 安装依赖（`npm install`）
- [ ] 检查输出通道确认 ESLint 版本
- [ ] 打开测试文件（如 `app.js`）
- [ ] 验证是否显示 ESLint 错误/警告
- [ ] 测试代码操作（Quick Fix）
- [ ] 测试保存时自动修复

---

## 4. 关键链路总结（@CORE 标记）

项目中所有标记了 `@CORE:` 注释的位置构成了扩展的核心执行链路。以下是按执行顺序整理的关键链路：

### 4.1 扩展激活链路

```
1. extension.ts:activate()
   扩展激活入口，实现延迟激活机制
   ↓
2. extension.ts:didOpenTextDocument()
   文档打开时检查是否需要激活
   验证文档是否需要 ESLint 处理
   ↓`
3. extension.ts:realActivate()
   真正激活 ESLint 扩展，创建 LSP 客户端并启动服务器
   ↓
4. client.ts:ESLintClient.create()
   创建 ESLint LSP 客户端，配置服务器选项和客户端选项
   创建 LanguageClient 实例（管理 LSP 通信和服务器进程）
   ↓
5. client.ts:createServerOptions()
   创建服务器选项，指定服务器入口文件和 IPC 传输方式
   ↓
6. client.ts:createClientOptions()
   创建客户端选项，配置文档选择器、中间件、诊断拉取等
   ↓
7. extension.ts:client.start()
   启动 LSP 客户端（会启动服务器进程并通过 IPC 连接）
```

### 4.2 服务器初始化链路

```
1. eslintServer.ts:createConnection()
   创建 LSP 服务器连接（监听 stdin，写入 stdout，处理 JSON-RPC 消息）
   ↓
2. eslintServer.ts:ESLint.initialize()
   初始化 ESLint 封装层，注入依赖（连接、文档管理器、路径推断、动态加载函数）
   ↓
3. eslintServer.ts:connection.onInitialize()
   处理初始化请求
   ↓
4. eslintServer.ts:connection.listen()
   开始监听 LSP 消息（stdin）
```

### 4.3 诊断生成链路

```
1. client.ts:middleware.didOpen()
   拦截文档打开，只同步需要验证的文档到服务器
   ↓
2. eslintServer.ts:connection.languages.diagnostics.on()
   处理诊断拉取请求（LSP DocumentDiagnosticRequest）
   ↓
3. eslint.ts:ESLint.resolveSettings()
   解析文档设置（包含动态加载 ESLint 库、解析工作目录、配置等）
   ↓
4. eslint.ts:resolveSettings() 内部
   从工作区 node_modules 向上查找 ESLint 库路径
   动态加载 ESLint 库（从解析的路径）
   使用动态 require 加载 ESLint 模块
   动态加载标准 ESLint API
   缓存已加载的库（避免重复加载）
   ↓
5. eslint.ts:ESLint.validate()
   执行 ESLint 验证，返回诊断结果并记录修复信息
   ↓
6. eslint.ts:withClass()
   统一接口执行 ESLint 操作（处理工作目录切换、版本抽象）
   切换工作目录（如果需要）
   创建 ESLint 实例并执行操作
   恢复原始工作目录
   ↓
7. eslint.ts:newClass()
   创建 ESLint 实例，统一不同版本的 API（CLIEngine vs ESLint Class）
   ESLint 8.57+ 使用 loadESLint（支持 Flat Config）
   ESLint 7.x 根据设置选择 API
   ESLint < 7.0 使用 CLIEngine（通过模拟器统一接口）
   默认使用 ESLint Class
   ↓
8. eslint.ts:validate() 内部
   调用 ESLint API 执行 lint
   将 ESLint 问题转换为 LSP 诊断
   记录可修复的问题（用于 Code Action）
   记录所有问题的修复信息（fix、suggestions）
```

### 4.4 代码操作（Code Action）链路

```
1. client.ts:middleware.provideCodeActions()
   拦截代码操作请求，过滤只处理 ESLint 诊断，并监控性能
   过滤出 ESLint 诊断
   转发到服务器处理代码操作
   ↓
2. eslintServer.ts:connection.onCodeAction()
   处理代码操作请求（LSP textDocument/codeAction），生成修复、禁用规则、建议等操作
   获取之前验证时记录的修复信息
   处理 "Fix All" 类型的代码操作
   计算所有可修复的问题
   遍历问题，为每个问题生成代码操作（修复、建议、禁用规则等）
   生成单个修复操作
   ↓
3. eslintServer.ts:connection.onExecuteCommand()
   执行代码操作命令（LSP workspace/executeCommand），应用修复编辑
   计算所有修复
   ↓
4. eslintServer.ts:computeAllFixes()
   计算所有可修复的问题，返回 TextEdit 数组
   快速模式：只使用已知的修复（不重新运行 ESLint）
   完整模式：重新运行 ESLint 并计算所有修复
   使用 ESLint 执行修复（fix: true），然后计算差异
   执行 lint 并自动修复（output 包含修复后的内容）
   计算原始内容和修复后内容的差异，生成最小编辑
```

### 4.5 格式化链路

```
1. eslintServer.ts:connection.onDocumentFormatting()
   处理文档格式化请求（LSP textDocument/formatting）
   使用 ESLint 修复作为格式化
   ↓
2. eslintServer.ts:computeAllFixes() (AllFixesMode.format)
   （同上代码操作链路）
```

### 4.6 关键抽象层

#### ESLint 版本抽象
- **`eslint.ts:newClass()`**: 统一不同版本的 ESLint API
- **`eslint.ts:withClass()`**: 统一接口执行操作，处理工作目录切换
- **`eslint.ts:ESLintClass`**: 通过接口和命名空间声明合并，统一 CLIEngine 和 ESLint Class

#### 动态加载机制
- **`eslint.ts:resolveSettings()`**: 从工作区向上查找 `node_modules/eslint`
- **`eslintServer.ts:loadNodeModule()`**: 使用动态 `require` 加载模块
- **`path2Library` Map**: 缓存已加载的库，避免重复加载

#### LSP 通信层
- **`client.ts:LanguageClient`**: 管理客户端-服务器通信
- **`eslintServer.ts:connection`**: 处理 JSON-RPC 消息（stdin/stdout）
- **`client.ts:middleware`**: 拦截和增强 LSP 请求

---

## 5. 关键文件说明

### 5.1 客户端核心文件

| 文件 | 职责 |
|------|------|
| `client/src/extension.ts` | 扩展入口，延迟激活逻辑 |
| `client/src/client.ts` | LSP 客户端实现，配置读取，状态管理 |
| `client/src/settings.ts` | 设置类型定义和验证 |
| `client/src/tasks.ts` | 任务提供者（lint 整个工作区） |

### 5.2 服务器核心文件

| 文件 | 职责 |
|------|------|
| `server/src/eslintServer.ts` | LSP 服务器主文件，处理请求 |
| `server/src/eslint.ts` | ESLint 封装，版本抽象，验证逻辑 |
| `server/src/diff.ts` | 文本差异计算（用于修复） |
| `server/src/paths.ts` | URI 到文件系统路径转换 |

### 5.3 共享文件

| 文件 | 职责 |
|------|------|
| `$shared/settings.ts` | 设置类型定义 |
| `$shared/customMessages.ts` | 自定义 LSP 消息类型 |

---

## 6. 数据流转总结（@DATA 标记）

项目中所有标记了 `@DATA:` 注释的位置构成了扩展的核心数据流转链路。以下是按数据流向整理的关键数据流转：

### 6.1 文档同步数据流

```
客户端 → 服务器：
1. client.ts:syncedDocuments Map
   @DATA: 已同步到服务器的文档映射（URI -> TextDocument）
   ↓
2. client.ts:middleware.didOpen()
   @DATA: 将文档添加到同步映射，文档内容通过 LSP 同步到服务器
   ↓
3. LSP textDocument/didOpen 通知
   ↓
4. server:documents.get(uri) 获取文档内容
```

### 6.2 配置数据流

```
客户端 → 服务器：
1. eslint.ts:resolveSettings()
   @DATA: 通过 LSP workspace/configuration 请求从客户端获取配置（按资源作用域）
   ↓
2. connection.workspace.getConfiguration()
   ↓
3. 返回 ConfigurationSettings
   ↓
4. 解析为 TextDocumentSettings（包含 ESLint 库路径、工作目录等）
```

### 6.3 诊断数据流

```
服务器 → 客户端：
1. eslint.ts:ESLint.validate()
   @DATA: ESLint API 返回报告（包含 messages、fix、suggestions）
   ↓
2. eslint.ts:Diagnostics.create()
   @DATA: 将 ESLint 问题转换为 LSP 诊断：ESLintProblem → Diagnostic
   @DATA: 转换位置信息：ESLint 的 1-based line/column → LSP 的 0-based Position
   @DATA: ESLint severity (1/2) → LSP DiagnosticSeverity
   ↓
3. eslint.ts:validate() 返回
   @DATA: 返回诊断数组，通过 LSP 发送给客户端
   ↓
4. eslintServer.ts:connection.languages.diagnostics.on()
   @DATA: 返回诊断报告（LSP DocumentDiagnosticReport），包含诊断数组
   ↓
5. client.ts:middleware.provideCodeActions()
   @DATA: 过滤出 ESLint 诊断（从服务器返回的诊断中筛选 source === 'eslint' 的）
   @DATA: 构建新的代码操作上下文，只包含 ESLint 诊断
```

### 6.4 修复信息存储与检索

```
验证阶段（存储）：
1. eslint.ts:ESLint.validate()
   @DATA: ESLint API 返回报告（包含 fix、suggestions）
   ↓
2. eslint.ts:CodeActions.record()
   @DATA: 存储修复信息：problem.fix + diagnostic → CodeActions Map
   @DATA: 使用诊断的 key 作为内层 Map 的 key，存储包含 fix、suggestions 的 Problem 对象
   ↓
3. eslint.ts:CodeActions namespace
   @DATA: 修复信息存储（URI -> Map<DiagnosticKey, Problem>）
   @DATA: 两层 Map 结构：外层 key 是文档 URI，内层 key 是诊断的 key，value 是修复信息

代码操作阶段（检索）：
1. eslintServer.ts:connection.onCodeAction()
   @DATA: 从 CodeActions 存储中获取该文档的修复信息（在验证阶段通过 CodeActions.record() 存储）
   ↓
2. eslintServer.ts:computeAllFixes() (快速模式)
   @DATA: 从 CodeActions 存储中获取修复信息，直接转换为 TextEdit（不重新运行 ESLint）
   @DATA: 修复信息（Problem）→ TextEdit[]
```

### 6.5 修复数据转换流

```
1. eslint.ts:CodeActions.record()
   @DATA: ESLint 的 fix 对象（包含 range 和 text）
   ↓
2. eslint.ts:FixableProblem.createTextEdit()
   @DATA: 将修复信息转换为 TextEdit：ESLint fix 对象（range + text）→ LSP TextEdit（Range + newText）
   ↓
3. eslint.ts:SuggestionsProblem.createTextEdit()
   @DATA: 将建议信息转换为 TextEdit：ESLint suggestion.fix → LSP TextEdit
   ↓
4. eslintServer.ts:connection.onCodeAction()
   @DATA: 生成单个修复操作：修复信息 → TextEdit → WorkspaceChange → 存储到 changes Map
   @DATA: 将修复信息转换为 TextEdit 并添加到 WorkspaceChange
   @DATA: 存储 WorkspaceChange 到 changes Map，供 executeCommand 时使用
   ↓
5. eslintServer.ts:computeAllFixes() (完整模式)
   @DATA: 执行 lint 并自动修复，ESLint 返回修复后的完整内容（output）
   @DATA: 获取修复后的完整内容
   @DATA: 计算原始内容和修复后内容的差异（diff），生成最小编辑集合
   @DATA: 将 diff 结果转换为 TextEdit[]（包含 range 和 newText）
```

### 6.6 工作区编辑应用流

```
1. eslintServer.ts:connection.onExecuteCommand()
   @DATA: 从 changes Map 获取 WorkspaceChange
   @DATA: 将 TextEdit[] 组装成 WorkspaceChange（包含多个 TextEdit）
   ↓
2. connection.workspace.applyEdit()
   @DATA: 将 WorkspaceChange 转换为 WorkspaceEdit 并通过 LSP 发送给客户端应用
   ↓
3. 客户端应用编辑到文档
```

### 6.7 状态数据流

```
服务器 → 客户端：
1. eslintServer.ts:connection.sendNotification(StatusNotification)
   @DATA: 发送状态通知给客户端（包含验证时间）
   ↓
2. client.ts:client.onNotification(StatusNotification)
   @DATA: 更新文档状态映射（URI -> StatusInfo）
   @DATA: 更新性能状态映射（LanguageId -> PerformanceStatus）
```

### 6.8 关键数据结构

| 数据结构 | 位置 | 说明 |
|---------|------|------|
| `syncedDocuments` | `client.ts` | 已同步到服务器的文档映射（URI -> TextDocument） |
| `CodeActions` Map | `eslint.ts` | 修复信息存储（URI -> Map<DiagnosticKey, Problem>） |
| `changes` Map | `eslintServer.ts` | 代码操作对应的 WorkspaceChange（key: commandId:ruleId） |
| `documentStatus` | `client.ts` | 文档状态映射（URI -> StatusInfo） |
| `performanceStatus` | `client.ts` | 性能状态映射（LanguageId -> PerformanceStatus） |

### 6.9 数据转换映射

| 源数据 | 目标数据 | 转换函数/位置 |
|--------|---------|--------------|
| ESLintProblem | Diagnostic | `Diagnostics.create()` |
| ESLint fix 对象 | TextEdit | `FixableProblem.createTextEdit()` |
| ESLint suggestion | TextEdit | `SuggestionsProblem.createTextEdit()` |
| TextEdit[] | WorkspaceChange | `WorkspaceChange.getTextEditChange().add()` |
| WorkspaceChange | WorkspaceEdit | `workspaceChange.edit` |
| ESLint severity (1/2) | DiagnosticSeverity | `convertSeverityToDiagnosticWithOverride()` |
| ESLint 位置 (1-based) | LSP Position (0-based) | `Diagnostics.create()` 中转换 |

---

## 7. 开发建议

### 5.1 添加新功能
1. 确定功能属于客户端还是服务器
2. 如果是设置相关，遵循设置管理流程
3. 如果是 LSP 功能，可能需要扩展协议消息
4. 更新相应的 `agents.md` 文档

### 7.2 调试新功能
1. 使用 `npm run watch` 启动增量编译
2. 在 VS Code 中按 F5 启动调试
3. 在新窗口中打开测试文件
4. 观察输出通道和断点

### 7.3 性能优化
1. 监控时间预算警告
2. 使用诊断拉取模式而非推送
3. 缓存设置和 ESLint 实例
4. 避免不必要的重新验证

---

## 8. 参考资料

- [Language Server Protocol](https://microsoft.github.io/language-server-protocol/)
- [VS Code Extension API](https://code.visualstudio.com/api)
- [ESLint Node.js API](https://eslint.org/docs/latest/developer-guide/nodejs-api)
- 项目 README.md：用户使用文档
- agents.md：各模块的设计概述
