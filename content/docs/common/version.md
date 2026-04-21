---
weight: 430
date: "2026-04-21T00:00:00+00:00"
draft: false
author: "GestGo"
title: "Version"
icon: "info"
toc: true
description: "Build metadata injection and pretty startup banner for Gest applications"
publishdate: "2026-04-21T00:00:00+00:00"
tags: ["Common"]
---

## Overview

The `version` package provides build metadata (version, commit, branch, extra) to your application at runtime. `Extra` is **generic** — you define your own typed struct and inject its fields via `ldflags`, giving you full type safety with zero runtime JSON parsing.

| Source | How | When to use |
|--------|-----|-------------|
| Framework vars | `-X '...version.Version=1.2.3'` | Version, commit, branch, build time |
| App-defined vars | `-X 'main.Env=prod'` | Your own typed metadata |

**Output example:**

```
┌─────────────────────────────────────────────┐
│             billing-svc  1.2.3              │
├─────────────────────────────────────────────┤
│  Version     1.2.3                          │
│  Branch      main                           │
│  Commit      abc1234                        │
│  Built       2026-04-21T10:00:00Z           │
│  Go          go1.24.2                       │
├─────────────────── extra ───────────────────┤
│  cluster     green                          │
│  env         prod                           │
│  service     billing                        │
└─────────────────────────────────────────────┘
```

## Installation

```bash
go get github.com/gestgo/gest/package/common/version
```

## Core Concepts

### Info[T]

Generic struct — `T` is the type of your extra metadata:

```go
type Info[T any] struct {
    AppName   string
    Version   string
    Commit    string
    Branch    string
    BuildTime string
    GoVersion string  // populated from runtime.Version()
    Extra     T
}
```

### vars.go — framework ldflags targets

```go
var (
    AppName   = ""
    Version   = ""
    Commit    = ""
    Branch    = ""
    BuildTime = ""
)
```

Fields with an empty value are omitted from the banner automatically.

Override at build time:

```bash
go build -ldflags "\
  -X 'github.com/gestgo/gest/package/common/version.AppName=billing-svc' \
  -X 'github.com/gestgo/gest/package/common/version.Version=1.2.3' \
  -X 'github.com/gestgo/gest/package/common/version.Branch=main' \
  -X 'github.com/gestgo/gest/package/common/version.Commit=abc1234' \
  -X 'github.com/gestgo/gest/package/common/version.BuildTime=2026-04-21T10:00:00Z'"
```

### New[T]

```go
func New[T any](extra T) *Info[T]
```

No error return — no JSON parsing, no failure modes.

## Quick Start

### 1. No extra metadata

```go
version.New[any](nil).Print()
```

### 2. Typed struct extra

Define your own struct and ldflags vars:

```go
// vars injected by CI:
//   -X 'main.env=prod'
//   -X 'main.cluster=blue'
var (
    env     = "local"
    cluster = "none"
)

type AppMeta struct {
    Env     string `json:"env"`
    Cluster string `json:"cluster"`
}

func main() {
    version.New(AppMeta{Env: env, Cluster: cluster}).Print()
}
```

### 3. Map extra (dynamic keys)

```go
version.New(map[string]any{
    "service": "billing",
    "tenant":  "acme",
}).Print()
```

## Display

### Print — centered terminal banner

```go
info.Print()
```

Detects terminal width via `ioctl TIOCGWINSZ` (Linux/macOS) or `$COLUMNS` env var, then centers the box. Falls back to 80 columns if not a TTY.

**Extra rendering rules:**
- struct / map → sorted key-value rows in the `extra` section
- primitive (string, int, …) → single line
- `nil` / empty → extra section hidden

### Log — structured logger

```go
info.Log(logger)
```

Writes the box as a single `Info` log message. Compatible with any type implementing:

```go
type Logger interface {
    Info(msg string, args ...any)
}
```

Works out of the box with `log/slog` and Uber Zap sugar:

```go
// slog
info.Log(slog.Default())

// zap sugar
info.Log(zapLogger.Sugar())
```

### String — raw box string

```go
box := info.String()
```

Returns the raw box without terminal centering padding. Useful for embedding in larger output or tests.

### JSON — structured / machine-readable

`Info[T]` has full JSON tags so you can serialize it directly:

```go
// marshal manually
b, _ := json.Marshal(info)
fmt.Println(string(b))
// {"app_name":"billing-svc","version":"1.2.3","commit":"abc1234","branch":"main","build_time":"2026-04-21T10:00:00Z","go_version":"go1.24.2","extra":{"env":"prod","cluster":"blue"}}

// pass to a structured logger — zap/slog serialize automatically
logger.Infow("startup", "version", info)

// expose as HTTP endpoint
c.JSON(http.StatusOK, info)
```

Empty fields are omitted (`omitempty`) — consistent with the box display.

## API Reference

### Functions

| Function | Description |
|----------|-------------|
| `New[T any](extra T) *Info[T]` | Creates Info with typed Extra; no error return |

### Info[T] Methods

| Method | Description |
|--------|-------------|
| `Print()` | Centered box to `stdout`; detects terminal width |
| `Log(Logger)` | Box via structured logger |
| `String() string` | Raw box string, no centering |
| `json.Marshal(info)` | JSON output via stdlib; empty fields omitted |

### Package-level Variables (ldflags targets)

| Variable | Default | Description |
|----------|---------|-------------|
| `AppName` | `""` | Application name shown in the banner title |
| `Version` | `""` | Semantic version |
| `Commit` | `""` | Git commit SHA |
| `Branch` | `""` | Git branch |
| `BuildTime` | `""` | Build timestamp |

## Makefile recipe

A common pattern for injecting all fields in a `Makefile`:

```makefile
VERSION   := $(shell git describe --tags --always --dirty)
COMMIT    := $(shell git rev-parse --short HEAD)
BRANCH    := $(shell git rev-parse --abbrev-ref HEAD)
BUILDTIME := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)
MODULE    := github.com/gestgo/gest/package/common/version

APP_MODULE := your-module/main  # or wherever your vars live

LDFLAGS := \
  -X '$(MODULE).Version=$(VERSION)' \
  -X '$(MODULE).Commit=$(COMMIT)' \
  -X '$(MODULE).Branch=$(BRANCH)' \
  -X '$(MODULE).BuildTime=$(BUILDTIME)' \
  -X '$(APP_MODULE).env=prod' \
  -X '$(APP_MODULE).cluster=blue'

build:
	go build -ldflags "$(LDFLAGS)" -o bin/app .
```
