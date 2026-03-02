---
weight: 200
date: "2026-03-02T00:00:00+00:00"
draft: false
author: "GestGo"
title: "MCP Extension"
icon: "extension"
toc: true
description: "Model Context Protocol (MCP) server adapter for exposing tools to LLM clients"
publishdate: "2026-03-02T00:00:00+00:00"
tags: ["Extension"]
---

## Overview

The `mcp` extension wraps [mcp-go](https://github.com/mark3labs/mcp-go) to expose an MCP server inside a Gest application. It integrates with the Fx lifecycle so the server starts and stops automatically with your app.

**Two transport modes:**

| Transport | Constant | Use case |
|-----------|----------|----------|
| stdin/stdout | `TransportStdio` | Claude Desktop, CLI tools, local LLM agents |
| HTTP SSE | `TransportSSE` | Web clients, remote agents, browser-based tools |

## Installation

```bash
go get github.com/gestgo/gest/package/extension/mcp
```

## Core Concepts

### IMCPHandler

The only interface you need to implement. Group related tools into a single handler:

```go
type IMCPHandler interface {
    Register(s *mcpserver.MCPServer)
}
```

Create one handler per domain (e.g. `FileTools`, `DatabaseTools`, `APITools`) and register all tools inside `Register`.

### Config

```go
type Config struct {
    Name      string        // server name shown to the LLM client
    Version   string        // server version
    Transport TransportType // TransportStdio | TransportSSE
    Port      int           // only used with TransportSSE
}
```

## Quick Start

### 1. Implement a Handler

```go
package tools

import (
    "context"

    "github.com/mark3labs/mcp-go/mcp"
    mcpserver "github.com/mark3labs/mcp-go/server"
)

type GreetTools struct{}

func NewGreetTools() *GreetTools {
    return &GreetTools{}
}

// Register implements mcp.IMCPHandler
func (g *GreetTools) Register(s *mcpserver.MCPServer) {
    s.AddTool(mcp.NewTool("greet",
        mcp.WithDescription("Say hello to a person"),
        mcp.WithString("name",
            mcp.Required(),
            mcp.Description("Name of the person"),
        ),
    ), g.greet)
}

func (g *GreetTools) greet(_ context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    name := mcp.ParseArgument(req, "name", "world").(string)
    return mcp.NewToolResultText("Hello, " + name + "!"), nil
}
```

### 2. Wire with Fx

```go
package main

import (
    "myapp/tools"

    "github.com/gestgo/gest/package/extension/mcp"
    mcpserver "github.com/mark3labs/mcp-go/server"
    "go.uber.org/fx"
)

func main() {
    app := fx.New(
        fx.NopLogger, // suppress fx logs (required for stdio transport)

        fx.Supply(mcp.Config{
            Name:      "my-mcp-server",
            Version:   "1.0.0",
            Transport: mcp.TransportStdio,
        }),

        fx.Provide(mcp.AsHandler(tools.NewGreetTools)),

        mcp.Module(),

        fx.Invoke(func(*mcpserver.MCPServer) {}), // keep server alive
    )

    app.Run()
}
```

> **Important:** Always use `fx.NopLogger` with `TransportStdio`. Fx writes startup logs to stdout, which would corrupt the MCP stdio protocol.

## Transport Modes

### Stdio (default)

Communicates over `stdin`/`stdout`. Suitable for Claude Desktop and CLI-based agents.

```go
fx.Supply(mcp.Config{
    Name:      "my-server",
    Version:   "1.0.0",
    Transport: mcp.TransportStdio,
})
```

When stdin is closed (client disconnects), the server exits automatically via `os.Exit(0)`.

**Claude Desktop config** (`~/.config/claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "my-server": {
      "command": "/path/to/my-mcp-server"
    }
  }
}
```

### SSE (HTTP Server-Sent Events)

Communicates over HTTP. Suitable for web clients and remote agents.

```go
fx.Supply(mcp.Config{
    Name:      "my-server",
    Version:   "1.0.0",
    Transport: mcp.TransportSSE,
    Port:      8080,
})
```

The server listens on the configured port and handles MCP messages over SSE.

## Registering Multiple Handlers

Group related tools into separate handlers and register them all with `AsHandler`:

```go
fx.Provide(
    mcp.AsHandler(tools.NewFileTools),
    mcp.AsHandler(tools.NewDatabaseTools),
    mcp.AsHandler(tools.NewAPITools),
),
mcp.Module(),
```

Each handler's `Register` method is called once during `OnStart`, in the order they are provided.

## Writing Tool Handlers

### Tool Parameters

Use `mcp.With*` options to define input parameters:

```go
s.AddTool(mcp.NewTool("create_file",
    mcp.WithDescription("Create a new file with content"),
    mcp.WithString("path",
        mcp.Required(),
        mcp.Description("File path"),
    ),
    mcp.WithString("content",
        mcp.Description("File content (optional)"),
    ),
    mcp.WithBoolean("overwrite",
        mcp.Description("Overwrite if exists (default false)"),
    ),
    mcp.WithNumber("permissions",
        mcp.Description("Unix permissions (default 644)"),
    ),
), handler)
```

| Option | Type | Description |
|--------|------|-------------|
| `mcp.WithString(key, opts...)` | string | Text input |
| `mcp.WithNumber(key, opts...)` | number | Numeric input |
| `mcp.WithBoolean(key, opts...)` | boolean | True/false flag |
| `mcp.WithObject(key, opts...)` | object | JSON object |
| `mcp.WithArray(key, opts...)` | array | JSON array |
| `mcp.Required()` | — | Mark parameter as required |
| `mcp.Description(text)` | — | Describe the parameter |

### Parsing Arguments

```go
func (h *MyHandler) handle(_ context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    // Parse with default fallback
    path := mcp.ParseArgument(req, "path", "").(string)
    overwrite := mcp.ParseBoolean(req, "overwrite", false)
    count := mcp.ParseNumber(req, "count", 10)

    if path == "" {
        return mcp.NewToolResultError("path is required"), nil
    }

    // ... logic

    return mcp.NewToolResultText("done"), nil
}
```

### Returning Results

| Function | Description |
|----------|-------------|
| `mcp.NewToolResultText(text)` | Return a plain text response |
| `mcp.NewToolResultError(msg)` | Return an error visible to the LLM |

Return `error` (the second return value) only for unexpected system failures — the LLM cannot handle them. For expected failures (file not found, invalid input), use `mcp.NewToolResultError`.

```go
func (h *MyHandler) handle(_ context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        // Expected failure — return as tool result, not error
        return mcp.NewToolResultError(fmt.Sprintf("cannot read file: %v", err)), nil
    }

    return mcp.NewToolResultText(string(data)), nil
}
```

## Complete Example: Filesystem Server

The full example from `example/mcp-filesystem/` — a production-ready MCP server that exposes filesystem operations:

**Project layout:**

```
mcp-filesystem/
├── main.go
└── src/
    ├── module/app.go
    └── tools/filesystem.go
```

**`main.go`**

```go
package main

import "mcp-filesystem/src/module"

func main() {
    app := module.NewApp()
    app.Run()
}
```

**`src/module/app.go`**

```go
package module

import (
    "mcp-filesystem/src/tools"

    "github.com/gestgo/gest/package/extension/mcp"
    mcpserver "github.com/mark3labs/mcp-go/server"
    "go.uber.org/fx"
)

func NewApp() *fx.App {
    return fx.New(
        fx.NopLogger,

        fx.Supply(mcp.Config{
            Name:      "filesystem",
            Version:   "1.0.0",
            Transport: mcp.TransportStdio,
        }),

        fx.Provide(mcp.AsHandler(tools.NewFileSystemTools)),

        mcp.Module(),

        fx.Invoke(func(*mcpserver.MCPServer) {}),
    )
}
```

**`src/tools/filesystem.go`** — implements `read_file`, `write_file`, `list_directory`, `create_directory`, `delete`, `move`, `file_info`:

```go
package tools

import (
    "context"
    "os"

    "github.com/mark3labs/mcp-go/mcp"
    mcpserver "github.com/mark3labs/mcp-go/server"
)

type FileSystemTools struct {
    RootDir string // restricts all operations to this directory
}

func NewFileSystemTools() *FileSystemTools {
    root, _ := os.Getwd()
    return &FileSystemTools{RootDir: root}
}

func (f *FileSystemTools) Register(s *mcpserver.MCPServer) {
    s.AddTool(mcp.NewTool("read_file",
        mcp.WithDescription("Read file contents"),
        mcp.WithString("path", mcp.Required(), mcp.Description("Path to the file")),
    ), f.readFile)

    s.AddTool(mcp.NewTool("write_file",
        mcp.WithDescription("Write content to a file"),
        mcp.WithString("path", mcp.Required(), mcp.Description("Path to the file")),
        mcp.WithString("content", mcp.Required(), mcp.Description("Content to write")),
    ), f.writeFile)

    // ... register other tools
}

func (f *FileSystemTools) readFile(_ context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
    path := mcp.ParseArgument(req, "path", "").(string)
    data, err := os.ReadFile(path)
    if err != nil {
        return mcp.NewToolResultError(err.Error()), nil
    }
    return mcp.NewToolResultText(string(data)), nil
}
```

**Build and run:**

```bash
go build -o mcp-filesystem .
./mcp-filesystem
```

## Accessing the Raw MCPServer

`mcp.Module()` and `RegisterMCPHooks` both provide `*mcpserver.MCPServer` in the Fx graph. Inject it if you need to register tools outside of handlers:

```go
fx.Invoke(func(s *mcpserver.MCPServer) {
    s.AddTool(mcp.NewTool("ping",
        mcp.WithDescription("Health check"),
    ), func(_ context.Context, _ mcp.CallToolRequest) (*mcp.CallToolResult, error) {
        return mcp.NewToolResultText("pong"), nil
    })
}),
```

## API Reference

### Config

| Field | Type | Description |
|-------|------|-------------|
| `Name` | `string` | Server name shown to the LLM |
| `Version` | `string` | Server version string |
| `Transport` | `TransportType` | `TransportStdio` or `TransportSSE` |
| `Port` | `int` | Listening port (SSE only) |

### Functions

| Function | Description |
|----------|-------------|
| `mcp.Module()` | Returns Fx module that provides `*MCPServer` |
| `mcp.AsHandler(f)` | Annotates a constructor into the `mcpHandlers` Fx group |
| `mcp.RegisterMCPHooks(lc, params)` | Low-level: creates adapter and registers lifecycle manually |
| `mcp.New(cfg, handlers...)` | Creates `MCPAdapter` without Fx |

### MCPAdapter Methods

| Method | Description |
|--------|-------------|
| `OnStart(ctx) error` | Registers all handlers and starts the transport |
| `OnStop(ctx) error` | Gracefully shuts down the transport |
| `Server() *MCPServer` | Returns the underlying `mcp-go` server |
