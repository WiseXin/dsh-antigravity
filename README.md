# dsh-antigravity

适用于 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（DSH）的 Google Antigravity / Cloud Code Assist 模型提供商插件。

**本仓库是基于 [LiZhenNet/dsh-antigravity](https://github.com/LiZhenNet/dsh-antigravity)（`v0.0.4`，commit `9495776`）的兼容适配分支**，仅对原代码做了最小改动，使其能在当前 DSH Desktop Beta 内置的 `@deepseek-ai/dsh-llm` 版本上正常加载。

> 非官方集成。本项目与 Google 无关，亦未获得 Google 认可。请仅在您有权访问的账号和服务中使用。

## 为什么需要这个分支

上游 `dsh-antigravity@0.0.4` 从 `@deepseek-ai/dsh-llm` 具名导入 `CallId`，而 DSH Desktop Beta 内置的 `@deepseek-ai/dsh-llm@0.1.2-rc.1` **没有导出 `CallId`**（该版本中工具调用 id 的品牌函数名为 `ToolCallId`）。ESM 具名导入在链接期即校验导出是否存在，缺失会直接抛错并导致整棵插件树加载失败：

```
SyntaxError: The requested module '@deepseek-ai/dsh-llm' does not provide an export named 'CallId'
    at file:///C:/Users/.../.dsh/profiles/desktop/node_modules/dsh-antigravity/lib/index.js:15
```

详情见下方“改动内容”。上游若在后续版本改用 `CallId` 命名（npm 上较新的 `dsh-llm` alpha 版本已调整），此差异即可消除。

## 改动内容（与上游的差异）

与上游 commit `9495776` 相比，本分支**仅**将 `CallId` 替换为语义等价的 `ToolCallId`，共 3 处，全部位于 `lib/index.js`：

1. 具名导入：

   ```diff
    CONTEXT_WINDOW_EXCEEDED_CODE,
   -CallId,
    EMPTY_RESPONSE_CODE,
   +ToolCallId,
   ```

2. 工具调用 block：

   ```diff
   -const block = { type: "tool-call", id: CallId(toolId), name: toolName, arguments: argsText };
   +const block = { type: "tool-call", id: ToolCallId(toolId), name: toolName, arguments: argsText };
   ```

3. `tool-call-delta`：

   ```diff
   -        id: CallId(toolId),
   +        id: ToolCallId(toolId),
   ```

其余逻辑、模型列表、OAuth 流程、设置页 i18n 等均与上游保持一致。

## 使用方式

### DSH Desktop（以 profile 依赖方式安装）

在 `~/.dsh/profiles/desktop/package.json` 中将依赖指向本仓库：

```json
"dependencies": {
  "dsh-antigravity": "github:WiseXin/dsh-antigravity"
}
```

重新安装依赖后完全退出并重启 DSH Desktop。

### 手动替换插件目录（临时覆盖）

将 `lib/`、`bin/`、`assets/`、`cordis.patch.yml`、`package.json` 等复制到：

```text
~/.dsh/profiles/desktop/node_modules/dsh-antigravity/
```

完全退出（含托盘）并重启 DSH Desktop。

> 说明：DSH Desktop 内置的 `dsh-llm` 版本由应用安装包决定，profile 目录无法覆盖；
> 因此直接替换插件目录（而非其依赖）即可让本分支生效。任何一次插件重装 / `pnpm install` 都会恢复成依赖声明中的版本。

## 完整使用说明

插件的安装、Google OAuth 登录、模型与配额说明见：

- 中文：[`README.zh.md`](./README.zh.md)
- 英文：[`README.md`](./README.md)（上游原始英文说明，本文件已替换为仓库说明，可查看上游仓库）

上游仓库：<https://github.com/LiZhenNet/dsh-antigravity>

## 许可证

[MIT](./LICENSE)。上游版权归 `dsh-antigravity contributors` 所有；本分支保留原许可证与版权声明，仅做兼容性适配。
