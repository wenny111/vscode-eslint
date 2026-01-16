# Code Action（自动修复）处理机制详解

## 概述

Code Action 是 VS Code 提供的代码操作功能，ESLint 扩展通过 LSP 协议提供多种修复操作：
- **自动修复**：修复 ESLint 规则问题
- **建议应用**：应用 ESLint 的建议
- **禁用规则**：添加 `eslint-disable` 注释
- **批量修复**：修复所有可修复的问题

## 1. 完整流程概览

```
用户触发 Code Action（点击灯泡图标或快捷键）
  ↓
Client: middleware.provideCodeActions 拦截
  ↓
Client → Server: textDocument/codeAction (Request)
  ↓
Server: connection.onCodeAction 处理
  ↓
Server: 从 CodeActions.get(uri) 获取问题信息
  ↓
Server: 生成 CodeAction 列表
  - 单个修复（FixableProblem）
  - 建议（SuggestionsProblem）
  - 禁用规则（Disable Line/File）
  - 批量修复（Fix All）
  ↓
Server → Client: 返回 CodeAction[] (Response)
  ↓
Client: 显示 Code Action 菜单
  ↓
用户选择操作
  ↓
Client → Server: workspace/executeCommand (Request)
  ↓
Server: connection.onExecuteCommand 处理
  ↓
Server: 从 changes Map 获取预计算的编辑
  ↓
Server → Client: 返回 WorkspaceEdit (Response)
  ↓
Client: 应用编辑到文档
```

## 2. 问题信息存储

### 2.1 在验证时记录修复信息

```typescript
// server/src/eslint.ts:1249-1293
export async function validate(document: TextDocument, settings: ...): Promise<Diagnostic[]> {
    return withClass(async (eslintClass) => {
        CodeActions.remove(uri);  // 清除旧的修复信息

        // 执行 lint
        const reportResults = await eslintClass.lintText(content, { filePath });

        const diagnostics: Diagnostic[] = [];
        docReport.messages.forEach((problem) => {
            // 创建诊断
            const [diagnostic, override] = Diagnostics.create(settings, problem, document);
            diagnostics.push(diagnostic);

            // 记录修复信息（如果有 fix 或 suggestions）
            CodeActions.record(document, diagnostic, problem);
        });

        return diagnostics;
    }, settings);
}
```

### 2.2 CodeActions.record 存储修复信息

```typescript
// server/src/eslint.ts:768-787
export namespace CodeActions {
    const codeActions: Map<string, Map<string, Problem>> = new Map();

    export function record(document: TextDocument, diagnostic: Diagnostic, problem: ESLintProblem): void {
        if (!problem.ruleId) {
            return;
        }
        const uri = document.uri;
        let edits: Map<string, Problem> | undefined = CodeActions.get(uri);
        if (edits === undefined) {
            edits = new Map<string, Problem>();
            CodeActions.set(uri, edits);
        }

        // 使用 diagnostic 的 key 作为 Map 的 key
        edits.set(Diagnostics.computeKey(diagnostic), {
            label: `Fix this ${problem.ruleId} problem`,
            documentVersion: document.version,
            ruleId: problem.ruleId,
            line: problem.line,
            diagnostic: diagnostic,
            edit: problem.fix,        // ESLint 的修复信息
            suggestions: problem.suggestions  // ESLint 的建议
        });
    }
}
```

**数据结构：**
```
codeActions Map:
  uri → Map<diagnosticKey, Problem>
    diagnosticKey → {
      label: "Fix this no-unused-vars problem",
      documentVersion: 5,
      ruleId: "no-unused-vars",
      line: 10,
      diagnostic: Diagnostic,
      edit: { range: [100, 110], text: "" },  // 可修复
      suggestions: [...]  // 建议
    }
```

## 3. 生成 Code Action

### 3.1 处理 Code Action 请求

```typescript
// server/src/eslintServer.ts:449
connection.onCodeAction(async (params) => {
    const result: CodeActionResult = new CodeActionResult();
    const textDocument = documents.get(params.textDocument.uri);

    // 获取存储的问题信息
    const problems = CodeActions.get(uri);

    // 创建 Fixes 对象处理修复
    const fixes = new Fixes(problems);

    // 遍历每个问题，生成 Code Action
    for (const editInfo of fixes.getScoped(params.context.diagnostics)) {
        // 生成不同类型的 Code Action
    }

    return result.all();
});
```

### 3.2 生成单个修复（Auto Fix）

```typescript
// server/src/eslintServer.ts:635-647
if (Problem.isFixable(editInfo)) {
    // 1. 创建 WorkspaceChange（包含 TextEdit）
    const workspaceChange = new WorkspaceChange();
    workspaceChange.getTextEditChange({ uri, version: documentVersion })
        .add(FixableProblem.createTextEdit(textDocument, editInfo));

    // 2. 缓存编辑（用于后续执行）
    changes.set(`${CommandIds.applySingleFix}:${ruleId}`, workspaceChange);

    // 3. 创建 CodeAction
    const action = createCodeAction(
        editInfo.label,                    // "Fix this no-unused-vars problem"
        kind,                               // CodeActionKind.QuickFix
        CommandIds.applySingleFix,          // "eslint.applySingleFix"
        CommandParams.create(textDocument, ruleId),
        editInfo.diagnostic
    );
    action.isPreferred = true;  // 标记为推荐操作
    result.get(ruleId).fixes.push(action);
}
```

**FixableProblem.createTextEdit：**
```typescript
// server/src/eslint.ts:237-239
export namespace FixableProblem {
    export function createTextEdit(document: TextDocument, editInfo: FixableProblem): TextEdit {
        return TextEdit.replace(
            Range.create(
                document.positionAt(editInfo.edit.range[0]),  // 起始位置
                document.positionAt(editInfo.edit.range[1])   // 结束位置
            ),
            editInfo.edit.text || ''  // 替换文本
        );
    }
}
```

### 3.3 生成建议（Suggestions）

```typescript
// server/src/eslintServer.ts:649-662
if (Problem.hasSuggestions(editInfo)) {
    editInfo.suggestions.forEach((suggestion, suggestionSequence) => {
        // 1. 创建 WorkspaceChange
        const workspaceChange = new WorkspaceChange();
        workspaceChange.getTextEditChange({ uri, version: documentVersion })
            .add(SuggestionsProblem.createTextEdit(textDocument, suggestion));

        // 2. 缓存编辑
        changes.set(`${CommandIds.applySuggestion}:${ruleId}:${suggestionSequence}`, workspaceChange);

        // 3. 创建 CodeAction
        const action = createCodeAction(
            `${suggestion.desc} (${editInfo.ruleId})`,
            CodeActionKind.QuickFix,
            CommandIds.applySuggestion,
            CommandParams.create(textDocument, ruleId, suggestionSequence),
            editInfo.diagnostic
        );
        result.get(ruleId).suggestions.push(action);
    });
}
```

### 3.4 生成禁用规则操作

```typescript
// server/src/eslintServer.ts:665-693
if (settings.codeAction.disableRuleComment.enable) {
    // 禁用当前行
    let workspaceChange = new WorkspaceChange();
    if (settings.codeAction.disableRuleComment.location === 'sameLine') {
        workspaceChange.getTextEditChange({ uri, version: documentVersion })
            .add(createDisableSameLineTextEdit(textDocument, editInfo));
    } else {
        workspaceChange.getTextEditChange({ uri, version: documentVersion })
            .add(createDisableLineTextEdit(textDocument, editInfo, indentationText));
    }
    changes.set(`${CommandIds.applyDisableLine}:${ruleId}`, workspaceChange);
    result.get(ruleId).disable = createCodeAction(...);

    // 禁用整个文件（每个规则只生成一次）
    if (result.get(ruleId).disableFile === undefined) {
        workspaceChange = new WorkspaceChange();
        workspaceChange.getTextEditChange({ uri, version: documentVersion })
            .add(createDisableFileTextEdit(textDocument, editInfo));
        changes.set(`${CommandIds.applyDisableFile}:${ruleId}`, workspaceChange);
        result.get(ruleId).disableFile = createCodeAction(...);
    }
}
```

**禁用注释生成：**
```typescript
// server/src/eslintServer.ts:500-533
function createDisableLineTextEdit(textDocument: TextDocument, editInfo: Problem, indentationText: string): TextEdit {
    const lineComment = LanguageDefaults.getLineComment(textDocument.languageId);  // "//" 或 "#"
    const blockComment = LanguageDefaults.getBlockComment(textDocument.languageId);  // ["/*", "*/"]

    // 检查上一行是否已有 eslint-disable-next-line
    const prevLine = textDocument.getText(Range.create(...));
    if (matchedLineDisable) {
        // 追加到现有注释
        return TextEdit.insert(Position.create(editInfo.line - 2, insertionIndex), `, ${editInfo.ruleId}`);
    }

    // 创建新注释
    const commentStyle = settings.codeAction.disableRuleComment.commentStyle;
    if (commentStyle === 'block') {
        disableRuleContent = `${indentationText}${blockComment[0]} eslint-disable-next-line ${editInfo.ruleId} ${blockComment[1]}${EOL}`;
    } else {
        disableRuleContent = `${indentationText}${lineComment} eslint-disable-next-line ${editInfo.ruleId}${EOL}`;
    }

    return TextEdit.insert(Position.create(editInfo.line - 1, 0), disableRuleContent);
}
```

### 3.5 生成批量修复（Fix All）

```typescript
// server/src/eslintServer.ts:592-614
const isSourceFixAll = (only === ESLintSourceFixAll || only === CodeActionKind.SourceFixAll);

if (isSourceFixAll) {
    // 模式 1：直接计算所有修复（用于保存时）
    const textDocumentIdentifier = { uri: textDocument.uri, version: textDocument.version };
    const edits = await computeAllFixes(textDocumentIdentifier, AllFixesMode.onSave);
    if (edits !== undefined) {
        result.fixAll.push(CodeAction.create(
            `Fix all fixable ESLint issues`,
            { documentChanges: [TextDocumentEdit.create(textDocumentIdentifier, edits)] },
            ESLintSourceFixAll
        ));
    }
} else if (isSource) {
    // 模式 2：通过命令执行（用于手动触发）
    result.fixAll.push(createCodeAction(
        `Fix all fixable ESLint issues`,
        CodeActionKind.Source,
        CommandIds.applyAllFixes,
        CommandParams.create(textDocument)
    ));
}
```

### 3.6 生成同规则批量修复

```typescript
// server/src/eslintServer.ts:707-735
if (result.length > 0) {
    const sameProblems: Map<string, FixableProblem[]> = new Map();

    // 收集同一规则的所有问题
    for (const editInfo of fixes.getAllSorted()) {
        if (sameProblems.has(editInfo.ruleId)) {
            const same = sameProblems.get(editInfo.ruleId)!;
            if (!Fixes.overlaps(getLastEdit(same), editInfo)) {  // 检查是否重叠
                same.push(editInfo);
            }
        }
    }

    // 如果同一规则有多个问题，生成批量修复
    sameProblems.forEach((same, ruleId) => {
        if (same.length > 1) {
            const sameFixes: WorkspaceChange = new WorkspaceChange();
            const sameTextChange = sameFixes.getTextEditChange({ uri, version: documentVersion });
            same.map(fix => FixableProblem.createTextEdit(textDocument, fix))
                .forEach(edit => sameTextChange.add(edit));
            changes.set(CommandIds.applySameFixes, sameFixes);
            result.get(ruleId).fixAll = createCodeAction(
                `Fix all ${ruleId} problems`,
                kind,
                CommandIds.applySameFixes,
                CommandParams.create(textDocument)
            );
        }
    });
}
```

## 4. 执行 Code Action

### 4.1 客户端发送执行命令

```typescript
// client/src/client.ts:366-383
commands.registerCommand('eslint.executeAutofix', async () => {
    const textEditor = Window.activeTextEditor;
    const textDocument: VersionedTextDocumentIdentifier = {
        uri: textEditor.document.uri.toString(),
        version: textEditor.document.version
    };
    const params: ExecuteCommandParams = {
        command: 'eslint.applyAllFixes',
        arguments: [textDocument]
    };
    await client.start();
    client.sendRequest(ExecuteCommandRequest.type, params);
});
```

### 4.2 服务器端执行命令

```typescript
// server/src/eslintServer.ts:819-855
connection.onExecuteCommand(async (params) => {
    const commandParams: CommandParams = params.arguments![0] as CommandParams;

    let workspaceChange: WorkspaceChange | undefined;

    if (params.command === CommandIds.applyAllFixes) {
        // 批量修复：重新计算所有修复
        const edits = await computeAllFixes(commandParams, AllFixesMode.command);
        if (edits !== undefined && edits.length > 0) {
            workspaceChange = new WorkspaceChange();
            const textChange = workspaceChange.getTextEditChange(commandParams);
            edits.forEach(edit => textChange.add(edit));
        }
    } else {
        // 单个修复：从缓存获取
        if ([CommandIds.applySingleFix, CommandIds.applyDisableLine, CommandIds.applyDisableFile].indexOf(params.command) !== -1) {
            workspaceChange = changes.get(`${params.command}:${commandParams.ruleId}`);
        } else if ([CommandIds.applySuggestion].indexOf(params.command) !== -1) {
            workspaceChange = changes.get(`${params.command}:${commandParams.ruleId}:${commandParams.sequence}`);
        } else {
            workspaceChange = changes.get(params.command);
        }
    }

    if (workspaceChange === undefined) {
        return null;
    }

    // 应用编辑
    return connection.workspace.applyEdit(workspaceChange.edit);
});
```

## 5. 批量修复计算

### 5.1 两种模式

#### 模式 1：使用已知修复（快速模式）

```typescript
// server/src/eslintServer.ts:770-775
if (mode === AllFixesMode.onSave && settings.codeActionOnSave.mode === CodeActionsOnSaveMode.problems) {
    // 只使用已经计算好的修复（从 CodeActions.get(uri)）
    const result = problems !== undefined && problems.size > 0
        ? new Fixes(problems).getApplicable().map(fix => FixableProblem.createTextEdit(textDocument, fix))
        : [];
    return result;
}
```

**特点：**
- ✅ **快速**：不需要重新执行 ESLint
- ✅ **安全**：只修复已知的非重叠问题
- ❌ **不完整**：可能遗漏一些修复

#### 模式 2：重新执行 ESLint（完整模式）

```typescript
// server/src/eslintServer.ts:776-815
else {
    // 重新执行 ESLint 获取所有修复
    return ESLint.withClass(async (eslintClass) => {
        const result: TextEdit[] = [];

        // 执行 lint（带 fix: true）
        const reportResults = await eslintClass.lintText(originalContent, { filePath });

        if (reportResults[0].output !== undefined) {
            const fixedContent = reportResults[0].output;

            // 计算最小差异编辑
            const diffs = stringDiff(originalContent, fixedContent, false);

            // 转换为 TextEdit[]
            for (const diff of diffs) {
                result.push({
                    range: {
                        start: textDocument.positionAt(diff.originalStart),
                        end: textDocument.positionAt(diff.originalStart + diff.originalLength)
                    },
                    newText: fixedContent.substr(diff.modifiedStart, diff.modifiedLength)
                });
            }
        }
        return result;
    }, settings, { fix: true });
}
```

**特点：**
- ✅ **完整**：修复所有可修复的问题
- ✅ **准确**：ESLint 处理修复冲突
- ❌ **较慢**：需要重新执行 ESLint

### 5.2 差异计算（stringDiff）

```typescript
// server/src/diff.ts
export function stringDiff(original: string, modified: string, ...): Diff[] {
    // 计算两个字符串的差异
    // 返回最小编辑集
}
```

**为什么需要差异计算？**
- ESLint 返回完整的修复后内容
- VS Code 需要增量编辑（TextEdit[]）
- 差异算法计算最小编辑集

## 6. 客户端中间件处理

### 6.1 过滤和增强

```typescript
// client/src/client.ts:555-588
middleware: {
    provideCodeActions: async (document, range, context, token, next) => {
        // 1. 检查文档是否同步
        if (!syncedDocuments.has(document.uri.toString())) {
            return [];
        }

        // 2. 过滤：只处理 ESLint 诊断
        const eslintDiagnostics: Diagnostic[] = [];
        for (const diagnostic of context.diagnostics) {
            if (diagnostic.source === 'eslint') {
                eslintDiagnostics.push(diagnostic);
            }
        }

        // 3. 如果没有 ESLint 诊断，返回空
        if (eslintDiagnostics.length === 0) {
            return [];
        }

        // 4. 创建新上下文（只包含 ESLint 诊断）
        const newContext: CodeActionContext = {
            ...context,
            diagnostics: eslintDiagnostics
        };

        // 5. 调用服务器
        const start = Date.now();
        const result = await next(document, range, newContext, token);

        // 6. 性能监控
        if (context.only?.value.startsWith('source.fixAll')) {
            performanceInfo.fixTime = Date.now() - start;
            updateStatusBar(document);
        }

        return result;
    }
}
```

## 7. Code Action 类型

### 7.1 类型定义

```typescript
// server/src/eslintServer.ts:327-335
type RuleCodeActions = {
    fixes: CodeAction[];           // 自动修复
    suggestions: CodeAction[];     // 建议
    disable?: CodeAction;          // 禁用当前行
    fixAll?: CodeAction;           // 修复所有同规则问题
    disableFile?: CodeAction;      // 禁用整个文件
    showDocumentation?: CodeAction; // 显示文档
};
```

### 7.2 CodeActionKind

```typescript
CodeActionKind.QuickFix              // 快速修复
CodeActionKind.Source                // 源代码操作
CodeActionKind.SourceFixAll          // 修复所有
`${CodeActionKind.SourceFixAll}.eslint`  // ESLint 专用
```

## 8. 关键数据结构

### 8.1 Problem 类型

```typescript
// server/src/eslint.ts:220-250
export type Problem = {
    label: string;
    documentVersion: number;
    ruleId: string;
    line: number;
    diagnostic: Diagnostic;
    edit?: ESLintAutoFixEdit;      // 可修复
    suggestions?: ESLintSuggestionResult[];  // 建议
};

export type FixableProblem = Problem & {
    edit: ESLintAutoFixEdit;  // 必须有 edit
};

export type SuggestionsProblem = Problem & {
    suggestions: ESLintSuggestionResult[];  // 必须有 suggestions
};
```

### 8.2 ESLintAutoFixEdit

```typescript
// ESLint 返回的修复格式
{
    range: [number, number],  // [startOffset, endOffset]
    text: string              // 替换文本
}
```

### 8.3 TextEdit（LSP 格式）

```typescript
// LSP 的编辑格式
{
    range: {
        start: { line: number, character: number },
        end: { line: number, character: number }
    },
    newText: string
}
```

## 9. 完整示例流程

### 9.1 单个修复示例

**场景：** 修复 `var unused = 1;` 中的未使用变量

```
1. 验证阶段
   ESLint.validate() → 发现问题
   CodeActions.record() → 存储修复信息
   {
     ruleId: "no-unused-vars",
     edit: { range: [0, 15], text: "" }  // 删除 "var unused = 1;"
   }

2. Code Action 请求
   Client → Server: textDocument/codeAction
   Server: 从 CodeActions.get(uri) 获取问题
   Server: 生成 CodeAction
   {
     title: "Fix this no-unused-vars problem",
     kind: "quickfix",
     command: "eslint.applySingleFix",
     arguments: [{ uri, version, ruleId: "no-unused-vars" }]
   }

3. 执行修复
   Client → Server: workspace/executeCommand
   Server: changes.get("eslint.applySingleFix:no-unused-vars")
   Server: 返回 WorkspaceEdit
   {
     changes: {