# tinia-file-mcp（bundle）

本目录 `index.js` 是 [Tinia_Local_MCP/tinia-file-mcp](https://github.com/bestfunc/Tinia_Local_MCP)
仓库 esbuild bundle 后的产物（含所有 npm 依赖单文件）。**不要直接编辑**，源码改动请去
Tinia_Local_MCP，跑：

```bash
cd Tinia_Local_MCP/tinia-file-mcp
npm run deploy   # build + 复制到本目录
```

每个 plugin 变体（tinia / tinia-onprem / tinia-local）的根目录有 symlink
`local-mcp -> ../_shared/local-mcp`，跟 `skills` 共享机制一致。

plugin.json 启动语法：

```json
"tinia-file": {
  "type": "stdio",
  "command": "node",
  "args": ["${CLAUDE_PLUGIN_ROOT}/local-mcp/tinia-file-mcp/index.js"],
  "env": { "TINIA_ENDPOINT": "..." }
}
```
