# Documentation: pyproject.toml

## File Metadata

- **Path**: `pyproject.toml`
- **Size**: 539 bytes
- **Extension**: .toml
- **Lines**: 25
- **Generated**: 2025-11-15T09:16:49.622229

## Original Source

```toml
[project]
name = "trendradar-mcp"
version = "1.0.1"
description = "TrendRadar MCP Server - 新闻热点聚合工具"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.32.5,<3.0.0",
    "pytz>=2025.2,<2026.0",
    "PyYAML>=6.0.3,<7.0.0",
    "fastmcp>=2.12.0,<2.14.0",
    "websockets>=13.0,<14.0",
]

[project.scripts]
trendradar = "mcp_server.server:run_server"

[dependency-groups]
dev = []

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["mcp_server"]

```

## High-Level Overview

This is a text file with 25 lines.

### Preview (first 20 lines)

```
[project]
name = "trendradar-mcp"
version = "1.0.1"
description = "TrendRadar MCP Server - 新闻热点聚合工具"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.32.5,<3.0.0",
    "pytz>=2025.2,<2026.0",
    "PyYAML>=6.0.3,<7.0.0",
    "fastmcp>=2.12.0,<2.14.0",
    "websockets>=13.0,<14.0",
]

[project.scripts]
trendradar = "mcp_server.server:run_server"

[dependency-groups]
dev = []

[build-system]
```

... and 5 more lines


## Performance & Security Notes

✅ No obvious security or performance concerns detected.

## Related Files

No obvious file references detected.
