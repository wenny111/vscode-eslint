# Playgrounds 测试项目说明

## ESLint 版本解析机制

### 问题：为什么 playground 使用了项目根目录的 ESLint 9.28.0？

**扩展的 ESLint 查找逻辑：**

1. **从文件所在目录向上查找** `node_modules/eslint`
   ```
   playgrounds/flat-config/app.js
   ↓ 查找 node_modules/eslint
   playgrounds/flat-config/          (没有 node_modules)
   ↓ 继续向上
   playgrounds/                       (没有 node_modules)
   ↓ 继续向上
   /home/guowentao/program/vscode-eslint/  (找到！有 node_modules/eslint@9.28.0)
   ```

2. **如果 playground 目录没有自己的 `node_modules`**
   - 会向上查找到项目根目录
   - 使用项目根目录的 ESLint 版本（9.28.0）

3. **如果 playground 目录有自己的 `node_modules`**
   - 优先使用 playground 目录的 ESLint 版本
   - 例如：`playgrounds/flat-config/node_modules/eslint` 中的版本

### 解决方案

#### 方案 1：在 playground 目录安装依赖（推荐）

```bash
cd playgrounds/flat-config
npm install
```

这会：
- 创建 `playgrounds/flat-config/node_modules/`
- 安装 `package.json` 中指定的 ESLint 版本（如 8.27.0）
- 扩展会优先使用这个版本

#### 方案 2：更新配置文件以兼容 ESLint 9.x

如果 playground 没有自己的 `node_modules`，需要更新配置文件：

**ESLint 8.x 格式（不兼容 9.x）：**
```javascript
module.exports = [
  "eslint:recommended",  // ❌ ESLint 9.x 不支持字符串
  { files: ["**/*.js"], ... }
];
```

**ESLint 9.x 格式：**
```javascript
const js = require('@eslint/js');

module.exports = [
  js.configs.recommended,  // ✅ 使用对象
  { files: ["**/*.js"], ... }
];
```

### 各 Playground 说明

#### `playgrounds/7.0/`
- ESLint 7.x
- 使用 `.eslintrc.json`（legacy config）
- 需要安装：`cd playgrounds/7.0 && npm install`

#### `playgrounds/8.0/`
- ESLint 8.x
- 使用 `.eslintrc.json`（legacy config）
- 需要安装：`cd playgrounds/8.0 && npm install`

#### `playgrounds/9.0/flat/`
- ESLint 9.x
- 使用 `eslint.config.js`（flat config）
- 需要安装：`cd playgrounds/9.0/flat && npm install`

#### `playgrounds/flat-config/`
- **问题：** 配置为 ESLint 8.x 格式，但可能使用项目根目录的 ESLint 9.x
- **解决：**
  1. 安装依赖：`cd playgrounds/flat-config && npm install`
  2. 或更新配置文件（已修复为 ESLint 9.x 格式）

### 验证 ESLint 版本

1. **查看输出通道**
   - `View > Output > ESLint`
   - 查找：`ESLint library loaded from: <path>`
   - 路径会显示实际使用的 ESLint 位置

2. **检查 node_modules**
   ```bash
   # 检查 playground 目录
   ls playgrounds/flat-config/node_modules/eslint/package.json

   # 检查项目根目录
   cat package.json | grep eslint
   ```

### 调试技巧

1. **启用调试模式**
   ```json
   {
     "eslint.debug": true,
     "eslint.trace.server": "verbose"
   }
   ```

2. **查看完整路径**
   - 输出通道会显示 ESLint 库的完整路径
   - 确认是否使用了预期的版本

3. **强制使用特定版本**
   - 在 playground 目录运行 `npm install`
   - 确保 `package.json` 中指定了正确的版本

### 常见错误

#### 错误：`Unexpected non-object config at user-defined index 0`

**原因：** ESLint 9.x 不支持字符串配置

**解决：**
- 在 playground 安装 ESLint 8.x：`npm install`
- 或更新配置文件为 ESLint 9.x 格式

#### 错误：`ESLint library not found`

**原因：** 找不到 ESLint 库

**解决：**
- 在 playground 目录运行 `npm install`
- 或检查全局安装：`npm list -g eslint`

### 最佳实践

1. **每个 playground 应该有自己的依赖**
   ```bash
   cd playgrounds/<name>
   npm install
   ```

2. **使用多根工作区测试**
   - 打开 `playgrounds/testing.code-workspace`
   - 可以同时测试多个 playground

3. **检查配置兼容性**
   - ESLint 8.x 和 9.x 的 flat config 格式不同
   - 确保配置文件与 ESLint 版本匹配
