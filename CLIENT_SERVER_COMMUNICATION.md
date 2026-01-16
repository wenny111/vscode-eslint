# Client-Server 通信机制详解

## 概述

VS Code ESLint 扩展采用 **Language Server Protocol (LSP)** 架构，Client 和 Server 通过 **IPC (Inter-Process Communication)** 进行通信。

## 1. 连接建立流程

### 1.1 初始化阶段

```typescript
// client/src/extension.ts
function realActivate(context: ExtensionContext) {
    // 创建 LanguageClient
    [client, acknowledgePerformanceStatus] = ESLintClient.create(context, validator);

    // 启动客户端，这会触发服务器进程的启动
    client.start();
}
```

### 1.2 LanguageClient 创建

```typescript
// client/src/client.ts:149
const client: LanguageClient = new LanguageClient(
    'ESLint',                           // 客户端名称
    createServerOptions(context.extensionUri),  // 服务器选项
    createClientOptions()               // 客户端选项
);
```

**关键点：**
- `LanguageClient` 是 `vscode-languageclient` 库提供的类
- 它负责管理整个 LSP 通信生命周期
- 创建时传入 `ServerOptions` 和 `ClientOptions`

## 2. ServerOptions - 服务器进程配置

### 2.1 创建 ServerOptions

```typescript
// client/src/client.ts:403
function createServerOptions(extensionUri: Uri): ServerOptions {
    const serverModule = Uri.joinPath(extensionUri, 'server', 'out', 'eslintServer.js').fsPath;

    return {
        run: {
            module: serverModule,           // 服务器入口文件
            transport: TransportKind.ipc,  // 使用 IPC 传输
            runtime: undefined,            // Node.js 运行时（可选）
            options: {
                execArgv: undefined,       // Node.js 执行参数
                cwd: getCurrentServerWorkingDirectory(),  // 工作目录
                env: { ... }               // 环境变量
            }
        },
        debug: {                           // 调试模式配置
            module: serverModule,
            transport: TransportKind.ipc,
            options: {
                execArgv: ['--nolazy', '--inspect=6011'],  // 调试参数
                cwd: getCurrentServerWorkingDirectory(),
                env: { ... }
            }
        }
    };
}
```

**ServerOptions 的作用：**
1. **指定服务器入口**：`server/out/eslintServer.js`
2. **选择传输方式**：`TransportKind.ipc`（进程间通信）
3. **配置进程环境**：工作目录、环境变量、Node.js 参数

### 2.2 IPC 传输机制

`TransportKind.ipc` 使用 Node.js 的 IPC 通道：
- Client 和 Server 运行在**不同的 Node.js 进程**中
- 通过 **stdin/stdout** 或 **命名管道** 进行通信
- 消息格式：**JSON-RPC 2.0**

## 3. ClientOptions - 客户端配置

### 3.1 创建 ClientOptions

```typescript
// client/src/client.ts:439
function createClientOptions(): LanguageClientOptions {
    return {
        documentSelector: [{ scheme: 'file' }, { scheme: 'untitled' }],
        revealOutputChannelOn: RevealOutputChannelOn.Never,
        initializationOptions: {},
        progressOnInitialization: true,

        // 文件系统监听
        synchronize: {
            fileEvents: [
                Workspace.createFileSystemWatcher('**/.eslintrc*'),
                Workspace.createFileSystemWatcher('**/eslint.config.*'),
                Workspace.createFileSystemWatcher('**/.eslintignore'),
                Workspace.createFileSystemWatcher('**/package.json')
            ]
        },

        // 诊断拉取配置
        diagnosticPullOptions: {
            onChange: true,
            onSave: true,
            onFocus: true,
            filter: (document, mode) => {
                // 过滤逻辑
            }
        },

        // 中间件（拦截和增强请求）
        middleware: {
            didOpen: async (document, next) => { ... },
            didChange: async (event, next) => { ... },
            provideCodeActions: async (document, range, context, token, next) => { ... },
            // ...
        }
    };
}
```

**ClientOptions 的作用：**
1. **文档选择器**：指定哪些文档需要处理
2. **文件监听**：自动同步配置文件变化
3. **诊断配置**：控制何时拉取诊断
4. **中间件**：拦截和增强 LSP 请求

## 4. Connection 创建（服务器端）

### 4.1 createConnection 调用

```typescript
// server/src/eslintServer.ts:35
const connection: ProposedFeatures.Connection = createConnection(
    ProposedFeatures.all,  // 启用所有提案特性
    {
        connectionStrategy: {
            cancelUndispatched: (message) => {
                // 取消未分发的请求策略
            }
        },
        maxParallelism: 1  // 最大并行度
    }
);
```

**createConnection 的作用：**
- 这是 `vscode-languageserver` 库提供的函数
- 创建 LSP 服务器端的连接对象
- **内部实现是黑盒**，但我们可以理解其工作原理

### 4.2 Connection 内部机制（推测）

虽然 `createConnection` 是黑盒，但根据 LSP 规范，它内部会：

1. **监听 stdin**：接收来自客户端的 JSON-RPC 消息
2. **写入 stdout**：发送响应和通知到客户端
3. **解析 JSON-RPC**：解析请求/响应/通知
4. **路由消息**：根据 method 路由到对应的处理器

```javascript
// 伪代码：createConnection 内部逻辑（简化）
function createConnection() {
    const connection = {
        // 注册处理器
        onInitialize: (handler) => { ... },
        onCodeAction: (handler) => { ... },
        // ...

        // 发送消息
        sendRequest: (method, params) => { ... },
        sendNotification: (method, params) => { ... },

        // 监听 stdin
        listen: () => {
            process.stdin.on('data', (data) => {
                const message = JSON.parse(data);
                routeMessage(message);
            });
        }
    };

    return connection;
}
```

## 5. 通信协议：LSP (Language Server Protocol)

### 5.1 LSP 消息类型

LSP 使用 **JSON-RPC 2.0** 协议，有三种消息类型：

#### 1. Request（请求-响应）
```json
// 客户端 → 服务器
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "textDocument/codeAction",
  "params": { ... }
}

// 服务器 → 客户端（响应）
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": [ ... ]
}
```

#### 2. Notification（通知，无响应）
```json
// 客户端 → 服务器
{
  "jsonrpc": "2.0",
  "method": "textDocument/didOpen",
  "params": { ... }
}

// 服务器 → 客户端
{
  "jsonrpc": "2.0",
  "method": "eslint/status",
  "params": { ... }
}
```

#### 3. Response（响应）
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": { ... }
}
```

### 5.2 标准 LSP 方法

| 方法 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `initialize` | Request | C→S | 初始化服务器 |
| `initialized` | Notification | C→S | 初始化完成 |
| `textDocument/didOpen` | Notification | C→S | 文档打开 |
| `textDocument/didChange` | Notification | C→S | 文档变更 |
| `textDocument/didClose` | Notification | C→S | 文档关闭 |
| `textDocument/codeAction` | Request | C→S | 获取代码操作 |
| `textDocument/formatting` | Request | C→S | 格式化文档 |
| `workspace/executeCommand` | Request | C→S | 执行命令 |
| `workspace/didChangeConfiguration` | Notification | C→S | 配置变更 |

### 5.3 自定义消息

ESLint 扩展还定义了自定义消息（在 `$shared/customMessages.ts`）：

```typescript
// 自定义通知
export namespace StatusNotification {
    export const type = new NotificationType<StatusParams>('eslint/status');
}

// 自定义请求
export namespace NoConfigRequest {
    export const type = new RequestType<NoConfigParams, {}, void>('eslint/noConfig');
}
```

## 6. 事件监听机制

### 6.1 服务器端监听

服务器使用 `connection.on*` 方法注册处理器：

```typescript
// server/src/eslintServer.ts

// 1. 初始化请求
connection.onInitialize((params, _cancel, progress) => {
    // 返回服务器能力
    return { capabilities: { ... } };
});

// 2. 诊断拉取
connection.languages.diagnostics.on(async (params) => {
    const diagnostics = await ESLint.validate(document, settings);
    return { kind: DocumentDiagnosticReportKind.Full, items: diagnostics };
});

// 3. 代码操作
connection.onCodeAction(async (params) => {
    // 生成代码操作
    return codeActions;
});

// 4. 执行命令
connection.onExecuteCommand(async (params) => {
    // 执行命令
    return null;
});

// 5. 文档格式化
connection.onDocumentFormatting((params) => {
    return edits;
});

// 6. 配置文件变化
connection.onDidChangeWatchedFiles(async (params) => {
    // 重新验证
    connection.languages.diagnostics.refresh();
});

// 7. 启动监听
connection.listen();  // 开始监听 stdin
```

### 6.2 客户端监听

客户端使用 `client.on*` 方法注册处理器：

```typescript
// client/src/client.ts

// 1. 服务器通知
client.onNotification(StatusNotification.type, (params) => {
    updateDocumentStatus(params);
});

client.onNotification(ExitCalled.type, (params) => {
    // 服务器进程退出
    client.error(`Server process exited with code ${params[0]}`);
});

// 2. 服务器请求
client.onRequest(NoConfigRequest.type, (params) => {
    // 处理无配置请求
    client.warn('No ESLint configuration found');
    return {};
});

client.onRequest(NoESLintLibraryRequest.type, async (params) => {
    // 处理无 ESLint 库请求
    return {};
});

// 3. 客户端状态变化
client.onDidChangeState((event) => {
    if (event.newState === State.Running) {
        client.info('ESLint server is running.');
    }
});
```

### 6.3 客户端发送请求

客户端使用 `client.sendRequest` 发送请求：

```typescript
// client/src/client.ts:380
await client.sendRequest(ExecuteCommandRequest.type, {
    command: 'eslint.applyAllFixes',
    arguments: [textDocument]
});
```

## 7. 完整通信流程示例

### 7.1 文档验证流程

```
1. 用户打开文件 app.js
   ↓
2. Client: 检测到文档打开
   ↓
3. Client: 通过 middleware.didOpen 过滤
   ↓
4. Client → Server: textDocument/didOpen (Notification)
   ↓
5. Server: 接收通知，更新文档管理器
   ↓
6. Client: 触发诊断拉取（onChange/onSave/onFocus）
   ↓
7. Client → Server: textDocument/diagnostic (Request)
   ↓
8. Server: ESLint.resolveSettings() 解析设置
   ↓
9. Server: ESLint.validate() 执行 lint
   ↓
10. Server → Client: 返回诊断结果 (Response)
    {
      "id": 1,
      "result": {
        "kind": "full",
        "items": [Diagnostic, ...]
      }
    }
   ↓
11. Client: 显示诊断（错误/警告）
```

### 7.2 代码操作流程

```
1. 用户点击 Quick Fix
   ↓
2. Client → Server: textDocument/codeAction (Request)
   ↓
3. Server: connection.onCodeAction 处理
   ↓
4. Server: 分析问题，生成代码操作
   ↓
5. Server → Client: 返回代码操作列表 (Response)
   ↓
6. Client: 显示代码操作菜单
   ↓
7. 用户选择操作
   ↓
8. Client → Server: workspace/executeCommand (Request)
   ↓
9. Server: connection.onExecuteCommand 处理
   ↓
10. Server: 执行操作，返回 WorkspaceEdit
   ↓
11. Client: 应用编辑到文档
```

## 8. 中间件（Middleware）机制

中间件允许客户端拦截和增强 LSP 请求：

```typescript
// client/src/client.ts:487
middleware: {
    // 拦截 didOpen
    didOpen: async (document, next) => {
        // 前置处理：检查是否需要验证
        if (validator.check(document) !== Validate.off) {
            // 调用下一个处理器（发送到服务器）
            const result = next(document);
            // 记录同步的文档
            syncedDocuments.set(document.uri.toString(), document);
            return result;
        }
        // 不发送到服务器
    },

    // 拦截代码操作
    provideCodeActions: async (document, range, context, token, next) => {
        // 过滤诊断：只处理 ESLint 诊断
        const eslintDiagnostics = context.diagnostics.filter(d => d.source === 'eslint');
        const newContext = { ...context, diagnostics: eslintDiagnostics };

        // 调用服务器
        const result = await next(document, range, newContext, token);

        // 后置处理：记录性能
        const start = Date.now();
        // ...

        return result;
    }
}
```

**中间件的作用：**
- **过滤**：只处理需要的文档/诊断
- **增强**：添加额外信息
- **性能监控**：记录请求时间
- **缓存**：避免重复请求

## 9. 进程间通信（IPC）详解

### 9.1 IPC 传输方式

`TransportKind.ipc` 使用 Node.js 的 IPC：

```typescript
// LanguageClient 内部（简化）
class LanguageClient {
    start() {
        // 启动服务器进程
        const serverProcess = spawn('node', [
            serverModule,
            '--stdio'  // 使用标准输入输出
        ], {
            stdio: ['pipe', 'pipe', 'pipe']  // stdin, stdout, stderr
        });

        // 连接 stdin/stdout
        this.writer = serverProcess.stdin;
        this.reader = serverProcess.stdout;

        // 监听消息
        this.reader.on('data', (data) => {
            const message = JSON.parse(data);
            this.handleMessage(message);
        });
    }

    sendRequest(method, params) {
        const message = {
            jsonrpc: '2.0',
            id: this.nextId++,
            method,
            params
        };
        this.writer.write(JSON.stringify(message) + '\n');
    }
}
```

### 9.2 消息格式

每条消息以换行符 `\n` 分隔：

```
Content-Length: 123\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"textDocument/codeAction","params":{...}}
```

## 10. 关键代码位置总结

### 10.1 客户端

| 文件 | 关键代码 | 说明 |
|------|---------|------|
| `client/src/extension.ts:136` | `ESLintClient.create()` | 创建 LanguageClient |
| `client/src/client.ts:149` | `new LanguageClient()` | 实例化客户端 |
| `client/src/client.ts:403` | `createServerOptions()` | 配置服务器进程 |
| `client/src/client.ts:439` | `createClientOptions()` | 配置客户端选项 |
| `client/src/client.ts:200` | `client.onNotification()` | 监听服务器通知 |
| `client/src/client.ts:218` | `client.onRequest()` | 处理服务器请求 |
| `client/src/client.ts:380` | `client.sendRequest()` | 发送请求到服务器 |

### 10.2 服务器端

| 文件 | 关键代码 | 说明 |
|------|---------|------|
| `server/src/eslintServer.ts:35` | `createConnection()` | 创建连接对象 |
| `server/src/eslintServer.ts:187` | `connection.onInitialize()` | 处理初始化 |
| `server/src/eslintServer.ts:254` | `connection.languages.diagnostics.on()` | 处理诊断请求 |
| `server/src/eslintServer.ts:448` | `connection.onCodeAction()` | 处理代码操作 |
| `server/src/eslintServer.ts:818` | `connection.onExecuteCommand()` | 处理命令执行 |
| `server/src/eslintServer.ts:868` | `connection.listen()` | 开始监听消息 |

## 11. 调试技巧

### 11.1 启用 LSP 跟踪

```json
{
  "eslint.trace.server": "verbose"
}
```

这会显示所有 LSP 消息的详细日志。

### 11.2 查看 IPC 通信

在输出通道中可以看到：
- `[Trace] Sending request 'textDocument/codeAction'`
- `[Trace] Received response 'textDocument/codeAction' in 45ms`

### 11.3 断点调试

**客户端断点：**
- `client/src/client.ts:200` - 接收通知
- `client/src/client.ts:380` - 发送请求

**服务器端断点：**
- `server/src/eslintServer.ts:254` - 诊断请求
- `server/src/eslintServer.ts:448` - 代码操作请求

## 12. 总结

### 12.1 通信架构

```
┌─────────────────────────────────────────────────┐
│  VS Code Extension Host (主进程)                │
│  ┌───────────────────────────────────────────┐  │
│  │  Client (client/src/)                     │  │
│  │  - LanguageClient                         │  │
│  │  - 管理 LSP 通信                          │  │
│  └───────────────────────────────────────────┘  │
│            ↕ IPC (stdin/stdout)                  │
│  ┌───────────────────────────────────────────┐  │
│  │  Server (独立 Node.js 进程)                │  │
│  │  - Connection                             │  │
│  │  - 处理 LSP 请求                          │  │
│  │  - 执行 ESLint                            │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### 12.2 关键要点

1. **LanguageClient** 负责启动服务器进程和管理通信
2. **createConnection** 创建服务器端连接（内部使用 stdin/stdout）
3. **IPC 传输** 通过 JSON-RPC 2.0 协议
4. **事件驱动** 使用 `on*` 方法注册处理器
5. **中间件** 允许客户端拦截和增强请求
6. **双向通信** 客户端和服务器都可以发送请求和通知

### 12.3 通信模式

- **Request-Response**：客户端请求，服务器响应
- **Notification**：单向通知，无需响应
- **自定义消息**：扩展标准 LSP 协议
