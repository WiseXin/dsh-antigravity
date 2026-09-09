# dsh-antigravity 插件加载失败：CallId 导出错误分析与修复

> 记录 DSH（DeepSeek Harness）Desktop Beta 下插件 `dsh-antigravity` 因
> `@deepseek-ai/dsh-llm` API 版本错配导致整棵插件树加载失败的根因、本地修复与根治建议。

## 报错全文

```
dsh-plugin-desktop-beta: plugin tree failed to load:
failed to apply loader entry include (cordis:include):
failed to import loader entry dsh-antigravity (dsh-antigravity):
The requested module '@deepseek-ai/dsh-llm' does not provide an export named 'CallId'

file:///C:/Users/xinrui/.dsh/profiles/desktop/node_modules/dsh-antigravity/lib/index.js:15
  CallId,
  ^^^^^^
SyntaxError: The requested module '@deepseek-ai/dsh-llm' does not provide an export named 'CallId'
    at #asyncInstantiate (node:internal/modules/esm/module_job:327:21)
    at async ModuleJob.run (node:internal/modules/esm/module_job:431:5)
    at async node:internal/modules/esm/loader:643:26
    at async file:///D:/Program%20Files/Soft/DSH%20Desktop%20Beta/resources/app.asar/node_modules/@deepseek-ai/cordis-plugin-loader/lib/index.js:279:16
    ...
```

## 环境信息

| 项 | 值 |
| --- | --- |
| 系统 | Windows |
| 应用 | DSH Desktop Beta（DeepSeek Harness 桌面版） |
| 插件 | `dsh-antigravity@0.0.4`（来源 `github:LiZhenNet/dsh-antigravity#9495776...`） |
| 插件安装路径 | `C:/Users/<user>/.dsh/profiles/desktop/node_modules/dsh-antigravity/` |
| 内置 `@deepseek-ai/dsh-llm` | `0.1.2-rc.1`（随应用打包，位于 `resources/app.asar/node_modules/`） |

## 根因分析

1. 插件 `dsh-antigravity@0.0.4` 在 `lib/index.js` 中从 `@deepseek-ai/dsh-llm` 具名导入了 `CallId`，
   并用它给工具调用（tool-call）id 打品牌：

   ```js
   // lib/index.js 顶部 import 块（节选）
   import {
     CONTEXT_WINDOW_EXCEEDED_CODE,
     CallId,          // ← 该导出在当前内置版本中不存在
     EMPTY_RESPONSE_CODE,
     LlmAdapter,
     LlmError,
     QUOTA_EXCEEDED_CODE,
     ReasoningEffortId,
     attributionHeaders,
     contentHasImage,
     isContextWindowExceededError,
     isQuotaExceededError,
   } from "@deepseek-ai/dsh-llm";
   ```

2. 但 DSH Desktop Beta 内置的 `@deepseek-ai/dsh-llm@0.1.2-rc.1` 的导出列表中**没有 `CallId`**，
   工具调用 id 的品牌函数叫 **`ToolCallId`**（同类导出还有 `MessageId`、`ProviderRequestId`、`ReasoningEffortId`）。

3. ESM 具名导入在模块链接阶段即校验导出是否存在，导出缺失会直接抛 `SyntaxError`，
   导致 `dsh-antigravity` 无法加载，进而整棵 plugin tree 加载失败。

4. 结论：**版本错配**。插件按较新的 `dsh-llm`（已加入 `CallId` 的版本）开发；
   当前 Desktop Beta 内置的 `0.1.2-rc.1` 仍使用 `ToolCallId`。
   npm 上 `@deepseek-ai/dsh-llm` 后续已发布 `0.1.3-alpha.2`、`0.1.5-alpha.1` 等版本。

## 本地修复（临时补丁）

将插件中 `CallId` 替换为内置版本存在的等价 API `ToolCallId`，共 3 处：

### 修改点

文件：`C:/Users/<user>/.dsh/profiles/desktop/node_modules/dsh-antigravity/lib/index.js`

1. **导入语句**（约第 15 行）：

   ```diff
    CONTEXT_WINDOW_EXCEEDED_CODE,
   -CallId,
    EMPTY_RESPONSE_CODE,
   +ToolCallId,
   ```

2. **工具调用 block**（约第 1940 行）：

   ```diff
   -const block = { type: "tool-call", id: CallId(toolId), name: toolName, arguments: argsText };
   +const block = { type: "tool-call", id: ToolCallId(toolId), name: toolName, arguments: argsText };
   ```

3. **tool-call-delta**（约第 1952 行）：

   ```diff
   -        id: CallId(toolId),
   +        id: ToolCallId(toolId),
   ```

### 一键补丁（PowerShell）

```powershell
$p = "$env:USERPROFILE\.dsh\profiles\desktop\node_modules\dsh-antigravity\lib\index.js"
$c = Get-Content $p -Raw
$c = $c.Replace("  CallId,`r`n", "")       # 移除导入行
$c = $c.Replace("CallId(toolId)", "ToolCallId(toolId)")
Set-Content -Path $p -Value $c -NoNewline
```

> 说明：导入行移除后需自行把 `ToolCallId` 补进 import 列表（或直接在上方脚本前
> 把 `EMPTY_RESPONSE_CODE,` 后面插入一行 `  ToolCallId,`）。更稳妥的做法是手改三处。

### 验证

```bash
node --check "C:\Users\<user>\.dsh\profiles\desktop\node_modules\dsh-antigravity\lib\index.js"
```

通过后**完全退出并重启 DSH Desktop Beta**（从托盘退出），插件树即可正常加载。

## 注意事项

- 这是对 `node_modules` 的直接修改，后续任何一次插件重装 / `pnpm install` / 应用更新都会**覆盖还原**。
- 仅替换 API 名，语义对应（工具调用 id 品牌），不影响功能逻辑。

## 根治建议

- **升级 DSH Desktop Beta**，等待内置 `dsh-llm` 更新到包含 `CallId` 的版本；
- 或到插件仓库 [LiZhenNet/dsh-antigravity](https://github.com/LiZhenNet/dsh-antigravity) 提交 Issue，
  请作者发布与当前 Desktop Beta 内置 `dsh-llm@0.1.2-rc.1` 兼容的版本。

## 参考

- 插件仓库：[LiZhenNet/dsh-antigravity](https://github.com/LiZhenNet/dsh-antigravity)
- `@deepseek-ai/dsh-llm` npm 版本列表：<https://www.npmjs.com/package/@deepseek-ai/dsh-llm>
- 本机版本证据：`resources/app.asar/node_modules/@deepseek-ai/dsh-llm/package.json` → `0.1.2-rc.1`，
  其 `lib/index.js` 末尾 `export { ... }` 列表含 `ToolCallId` 但**不含** `CallId`
