# Documentation: start-http.sh

## File Metadata

- **Path**: `start-http.sh`
- **Size**: 729 bytes
- **Extension**: .sh
- **Lines**: 21
- **Generated**: 2025-11-15T09:16:49.613851

## Original Source

```sh
#!/bin/bash

echo "╔════════════════════════════════════════╗"
echo "║  TrendRadar MCP Server (HTTP 模式)    ║"
echo "╚════════════════════════════════════════╝"
echo ""

# 检查虚拟环境
if [ ! -d ".venv" ]; then
    echo "❌ [错误] 虚拟环境未找到"
    echo "请先运行 ./setup-mac.sh 进行部署"
    echo ""
    exit 1
fi

echo "[模式] HTTP (适合远程访问)"
echo "[地址] http://localhost:3333/mcp"
echo "[提示] 按 Ctrl+C 停止服务"
echo ""

uv run python -m mcp_server.server --transport http --host 0.0.0.0 --port 3333

```

## High-Level Overview

This is a text file with 21 lines.

### Preview (first 20 lines)

```
#!/bin/bash

echo "╔════════════════════════════════════════╗"
echo "║  TrendRadar MCP Server (HTTP 模式)    ║"
echo "╚════════════════════════════════════════╝"
echo ""

# 检查虚拟环境
if [ ! -d ".venv" ]; then
    echo "❌ [错误] 虚拟环境未找到"
    echo "请先运行 ./setup-mac.sh 进行部署"
    echo ""
    exit 1
fi

echo "[模式] HTTP (适合远程访问)"
echo "[地址] http://localhost:3333/mcp"
echo "[提示] 按 Ctrl+C 停止服务"
echo ""

```

... and 1 more lines


## Performance & Security Notes

✅ No obvious security or performance concerns detected.

## Related Files

No obvious file references detected.
