# Documentation: start-http.bat

## File Metadata

- **Path**: `start-http.bat`
- **Size**: 786 bytes
- **Extension**: .bat
- **Lines**: 25
- **Generated**: 2025-11-15T09:16:49.611877

## Original Source

```bat
@echo off
chcp 65001 >nul

echo ╔════════════════════════════════════════╗
echo ║  TrendRadar MCP Server (HTTP 模式)    ║
echo ╚════════════════════════════════════════╝
echo.

REM 检查虚拟环境
if not exist ".venv\Scripts\python.exe" (
    echo ❌ [错误] 虚拟环境未找到
    echo 请先运行 setup-windows.bat 或 setup-windows-en.bat 进行部署
    echo.
    pause
    exit /b 1
)

echo [模式] HTTP (适合远程访问)
echo [地址] http://localhost:3333/mcp
echo [提示] 按 Ctrl+C 停止服务
echo.

uv run python -m mcp_server.server --transport http --host 0.0.0.0 --port 3333

pause

```

## High-Level Overview

This is a text file with 25 lines.

### Preview (first 20 lines)

```
@echo off
chcp 65001 >nul

echo ╔════════════════════════════════════════╗
echo ║  TrendRadar MCP Server (HTTP 模式)    ║
echo ╚════════════════════════════════════════╝
echo.

REM 检查虚拟环境
if not exist ".venv\Scripts\python.exe" (
    echo ❌ [错误] 虚拟环境未找到
    echo 请先运行 setup-windows.bat 或 setup-windows-en.bat 进行部署
    echo.
    pause
    exit /b 1
)

echo [模式] HTTP (适合远程访问)
echo [地址] http://localhost:3333/mcp
echo [提示] 按 Ctrl+C 停止服务
```

... and 5 more lines


## Performance & Security Notes

✅ No obvious security or performance concerns detected.

## Related Files

No obvious file references detected.
