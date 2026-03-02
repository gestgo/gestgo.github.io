---
weight: 300
date: "2026-03-02T00:00:00+00:00"
draft: false
author: "GestGo"
title: "Log"
icon: "terminal"
toc: true
description: "Structured, leveled logging for Gest applications powered by Uber Zap"
publishdate: "2026-03-02T00:00:00+00:00"
tags: ["Logging"]
---

## Overview

The `log` package provides a structured, leveled logging system for Gest applications. It is powered by [Uber Zap](https://github.com/uber-go/zap) and exposes a clean, interface-driven API that supports multiple logging styles:

| Style | Methods | Use case |
|-------|---------|----------|
| Basic | `Info()`, `Error()`, ... | Simple string messages |
| Formatted | `Infof()`, `Errorf()`, ... | Printf-style templates |
| Structured | `Infow()`, `Errorw()`, ... | Key-value pairs (recommended for production) |
| Line | `Infoln()`, `Errorln()`, ... | Messages with explicit newline |

## Installation

```bash
go get github.com/gestgo/gest/package/common/log
go get github.com/gestgo/gest/package/common/log/adapter/zap
```

## Quick Start

```go
package main

import (
    "log"

    zapAdapter "github.com/gestgo/gest/package/common/log/adapter/zap"
)

func main() {
    logger, err := zapAdapter.NewDevelopment()
    if err != nil {
        log.Fatal(err)
    }
    defer logger.Sync()

    logger.Info("server started")
    logger.Infof("listening on port %d", 8080)
    logger.Infow("request received", "method", "GET", "path", "/api/users")
}
```

## Log Levels

Levels are defined in `core.Level` and ordered from lowest to highest severity:

| Level | Constant | Description |
|-------|----------|-------------|
| `-1` | `DebugLevel` | Detailed debug information |
| `0` | `InfoLevel` | General informational messages (default) |
| `1` | `WarnLevel` | Non-critical warnings |
| `2` | `ErrorLevel` | Errors that need attention |
| `3` | `DPanicLevel` | Panics in development mode only |
| `4` | `PanicLevel` | Always panics after logging |
| `5` | `FatalLevel` | Calls `os.Exit(1)` after logging |

Only messages at or above the configured level are output. For example, setting `InfoLevel` suppresses all `Debug` messages.

## Creating a Logger

### Development Logger

Human-friendly console output with color and full stack traces. Level defaults to `Debug`.

```go
import zapAdapter "github.com/gestgo/gest/package/common/log/adapter/zap"

logger, err := zapAdapter.NewDevelopment()
```

**Output example:**
```
2026-03-02T10:00:00.000+0700    INFO    main.go:12    server started
```

### Production Logger

Structured JSON output optimized for log aggregation systems. Level defaults to `Info`.

```go
logger, err := zapAdapter.NewProduction()
```

**Output example:**
```json
{"timestamp":"2026-03-02T10:00:00.000+0700","level":"info","caller":"main.go:12","msg":"server started"}
```

### Example Logger

Minimal logger for tests and examples — no timestamps, no caller info.

```go
logger := zapAdapter.NewExample()
```

### No-op Logger

Discards all output. Useful in tests to suppress logs.

```go
logger := zapAdapter.NewNop()
```

### Custom Logger with Config

Full control over level, encoding, and output paths:

```go
import (
    "github.com/gestgo/gest/package/common/log/core"
    zapAdapter "github.com/gestgo/gest/package/common/log/adapter/zap"
)

logger, err := zapAdapter.NewWithConfig(zapAdapter.Config{
    Level:            core.InfoLevel,
    Development:      false,
    Encoding:         "json",        // "json" or "console"
    OutputPaths:      []string{"stdout", "/var/log/app.log"},
    ErrorOutputPaths: []string{"stderr"},
})
```

### Custom Logger with Options

Functional options provide a fluent API for configuration:

```go
logger, err := zapAdapter.NewWithOptions(
    zapAdapter.WithLevel(core.DebugLevel),
    zapAdapter.WithConsoleEncoding(),
    zapAdapter.WithOutputPaths("stdout", "/var/log/app.log"),
)

// Development preset + options
logger, err := zapAdapter.NewDevelopmentWithOptions(
    zapAdapter.WithLevel(core.WarnLevel),
)

// Production preset + options
logger, err := zapAdapter.NewProductionWithOptions(
    zapAdapter.WithLevel(core.ErrorLevel),
    zapAdapter.WithJSONEncoding(),
)
```

**Available options:**

| Option | Description |
|--------|-------------|
| `WithLevel(level)` | Set minimum log level |
| `WithDevelopment(bool)` | Enable/disable development mode |
| `WithEncoding(enc)` | Set encoding: `"json"` or `"console"` |
| `WithJSONEncoding()` | Shorthand for JSON encoding |
| `WithConsoleEncoding()` | Shorthand for console encoding |
| `WithOutputPaths(paths...)` | Set output destinations |
| `WithErrorOutputPaths(paths...)` | Set error output destinations |

## Logging Styles

All styles are available on `ISugaredLogger`. Use whichever fits the context.

### Basic

Pass values directly — they are concatenated with spaces:

```go
logger.Debug("connecting to database")
logger.Info("server started")
logger.Warn("disk usage above 80%")
logger.Error("failed to process request")
```

### Formatted (Printf-style)

Use a format string with `%` verbs:

```go
logger.Debugf("loaded %d records from %s", count, tableName)
logger.Infof("server listening on %s:%d", host, port)
logger.Errorf("query failed after %dms: %v", elapsed, err)
```

### Structured (recommended for production)

Pass key-value pairs after the message — best for log querying and aggregation:

```go
logger.Infow("request completed",
    "method", "POST",
    "path", "/api/orders",
    "status", 201,
    "duration_ms", 42,
)

logger.Errorw("database error",
    "error", err,
    "query", sql,
    "user_id", userID,
)
```

**JSON output:**
```json
{"level":"info","msg":"request completed","method":"POST","path":"/api/orders","status":201,"duration_ms":42}
```

### Line (explicit newline)

Similar to basic but appends a newline — behaves like `fmt.Println`:

```go
logger.Infoln("server started")
logger.Errorln("unexpected shutdown")
```

### Dynamic level

Log at a level determined at runtime:

```go
import "github.com/gestgo/gest/package/common/log/core"

// Formatted
logger.Logf(core.WarnLevel, "retry attempt %d of %d", attempt, maxRetries)

// Structured
logger.Logw(core.ErrorLevel, "operation failed", "error", err)

// Line
logger.Logln(core.InfoLevel, "background job completed")
```

## Contextual Logging

### Adding Persistent Fields with `With`

Create a child logger that always includes a set of fields. Ideal for request-scoped logging:

```go
// All logs from requestLogger will include request_id and user_id
requestLogger := logger.With(
    "request_id", requestID,
    "user_id", userID,
)

requestLogger.Info("processing request")
requestLogger.Infow("order created", "order_id", orderID)
```

### Lazy Field Evaluation with `WithLazy`

Like `With`, but fields are evaluated only when the log is actually written. Useful when building the value is expensive:

```go
logger = logger.WithLazy("expensive_field", computeExpensiveValue())
```

### Named Loggers

Attach a name to the logger to indicate the component or subsystem:

```go
dbLogger := logger.Named("database")
dbLogger.Info("connection pool initialized")
// Output: ... logger=database msg="connection pool initialized"

httpLogger := logger.Named("http")
httpLogger.Infow("request received", "path", "/api/users")
```

### Context-aware Logging

Pass a `context.Context` to attach tracing information (e.g., OpenTelemetry trace IDs):

```go
ctxLogger := logger.WithContext(ctx)
ctxLogger.Infow("processing", "order_id", orderID)
```

## Using with Fx (Gest DI)

The `log` package ships an `fx` module for seamless integration with the Gest dependency injection container.

### Register the Module

```go
import (
    "github.com/gestgo/gest/package/common/log"
    "go.uber.org/fx"
)

app := fx.New(
    log.Module(),
    // ... other modules
)
```

`log.Module()` provides a `core.ISugaredLogger` that can be injected anywhere in your application. The log level defaults to `INFO` but can be overridden by providing a `string` tagged `Level` in the fx graph.

### Inject into a Service

```go
import "github.com/gestgo/gest/package/common/log/core"

type UserService struct {
    logger core.ISugaredLogger
}

func NewUserService(logger core.ISugaredLogger) *UserService {
    return &UserService{
        logger: logger.Named("user-service"),
    }
}

func (s *UserService) CreateUser(ctx context.Context, name string) error {
    s.logger.Infow("creating user", "name", name)
    // ...
    s.logger.Infow("user created", "name", name, "id", newUserID)
    return nil
}
```

## Logger Control

### Flush Buffered Logs

Call `Sync()` before the application exits to ensure all buffered log entries are written:

```go
defer logger.Sync()
```

### Get Current Level

```go
level := logger.Level()
fmt.Println(level) // "INFO"
```

### Access Underlying Zap Logger

When you need raw `*zap.Logger` access (e.g., to pass to a third-party library):

```go
raw := logger.Desugar().(*zap.Logger)
```

## Interface Reference

The `log/core` package defines composable interfaces. You can declare dependencies against the minimal interface you need:

| Interface | Methods included |
|-----------|-----------------|
| `IBasicLogger` | `Debug`, `Info`, `Warn`, `Error`, `DPanic`, `Panic`, `Fatal` |
| `IFormattedLogger` | `Debugf`, `Infof`, `Warnf`, `Errorf`, `DPanicf`, `Panicf`, `Fatalf`, `Logf` |
| `IStructuredLogger` | `Debugw`, `Infow`, `Warnw`, `Errorw`, `DPanicw`, `Panicw`, `Fatalw`, `Logw` |
| `ILineLogger` | `Debugln`, `Infoln`, `Warnln`, `Errorln`, `DPanicln`, `Panicln`, `Fatalln`, `Logln` |
| `IContextualLogger` | `With`, `WithLazy`, `Named` |
| `IContextLogger` | `WithContext` |
| `ILoggerControl` | `Desugar`, `Level`, `Sync` |
| `ILogger` | Basic + Formatted + Control |
| `IFullLogger` | Basic + Formatted + Structured + Control |
| `ISugaredLogger` | All of the above |

Use the narrowest interface in your code to keep dependencies minimal and tests easy to mock:

```go
// Only needs to log errors — use a narrow interface
type Repo struct {
    logger core.IBasicLogger
}

// Needs structured logging — use IFullLogger
type Service struct {
    logger core.IFullLogger
}

// Full capabilities — use ISugaredLogger
type Handler struct {
    logger core.ISugaredLogger
}
```

## Custom Log Adapters

The `log/core` package defines `ISugaredLogger` as a pure interface. Any struct that implements all its sub-interfaces is a valid adapter and can be used wherever `ISugaredLogger` is expected — including with `log.Module()` in the Gest DI container.

### Interface to Implement

A full adapter must satisfy all seven sub-interfaces:

```go
type ISugaredLogger interface {
    IBasicLogger      // Debug/Info/Warn/Error/DPanic/Panic/Fatal
    IFormattedLogger  // Debugf/Infof/.../Logf
    IStructuredLogger // Debugw/Infow/.../Logw
    ILineLogger       // Debugln/Infoln/.../Logln
    IContextualLogger // With/WithLazy/Named
    IContextLogger    // WithContext
    ILoggerControl    // Desugar/Level/Sync
}
```

### Example: slog Adapter

Wrap Go's standard `log/slog` package as a Gest logger:

```go
package slogadapter

import (
    "context"
    "log/slog"

    "github.com/gestgo/gest/package/common/log/core"
)

type SlogAdapter struct {
    logger *slog.Logger
    level  core.Level
}

func New(logger *slog.Logger, level core.Level) core.ISugaredLogger {
    return &SlogAdapter{logger: logger, level: level}
}

// IBasicLogger
func (s *SlogAdapter) Debug(args ...any) { s.logger.Debug(fmt.Sprint(args...)) }
func (s *SlogAdapter) Info(args ...any)  { s.logger.Info(fmt.Sprint(args...)) }
func (s *SlogAdapter) Warn(args ...any)  { s.logger.Warn(fmt.Sprint(args...)) }
func (s *SlogAdapter) Error(args ...any) { s.logger.Error(fmt.Sprint(args...)) }
func (s *SlogAdapter) DPanic(args ...any) { s.logger.Error(fmt.Sprint(args...)) }
func (s *SlogAdapter) Panic(args ...any) {
    s.logger.Error(fmt.Sprint(args...))
    panic(fmt.Sprint(args...))
}
func (s *SlogAdapter) Fatal(args ...any) {
    s.logger.Error(fmt.Sprint(args...))
    os.Exit(1)
}

// IFormattedLogger
func (s *SlogAdapter) Debugf(tmpl string, args ...any) { s.logger.Debug(fmt.Sprintf(tmpl, args...)) }
func (s *SlogAdapter) Infof(tmpl string, args ...any)  { s.logger.Info(fmt.Sprintf(tmpl, args...)) }
func (s *SlogAdapter) Warnf(tmpl string, args ...any)  { s.logger.Warn(fmt.Sprintf(tmpl, args...)) }
func (s *SlogAdapter) Errorf(tmpl string, args ...any) { s.logger.Error(fmt.Sprintf(tmpl, args...)) }
func (s *SlogAdapter) DPanicf(tmpl string, args ...any) { s.Errorf(tmpl, args...) }
func (s *SlogAdapter) Panicf(tmpl string, args ...any) {
    msg := fmt.Sprintf(tmpl, args...)
    s.logger.Error(msg)
    panic(msg)
}
func (s *SlogAdapter) Fatalf(tmpl string, args ...any) {
    s.logger.Error(fmt.Sprintf(tmpl, args...))
    os.Exit(1)
}
func (s *SlogAdapter) Logf(level core.Level, tmpl string, args ...any) {
    s.logger.Log(context.Background(), slog.Level(level), fmt.Sprintf(tmpl, args...))
}

// IStructuredLogger
func (s *SlogAdapter) Debugw(msg string, kv ...any) { s.logger.Debug(msg, kv...) }
func (s *SlogAdapter) Infow(msg string, kv ...any)  { s.logger.Info(msg, kv...) }
func (s *SlogAdapter) Warnw(msg string, kv ...any)  { s.logger.Warn(msg, kv...) }
func (s *SlogAdapter) Errorw(msg string, kv ...any) { s.logger.Error(msg, kv...) }
func (s *SlogAdapter) DPanicw(msg string, kv ...any) { s.logger.Error(msg, kv...) }
func (s *SlogAdapter) Panicw(msg string, kv ...any) {
    s.logger.Error(msg, kv...)
    panic(msg)
}
func (s *SlogAdapter) Fatalw(msg string, kv ...any) {
    s.logger.Error(msg, kv...)
    os.Exit(1)
}
func (s *SlogAdapter) Logw(level core.Level, msg string, kv ...any) {
    s.logger.Log(context.Background(), slog.Level(level), msg, kv...)
}

// ILineLogger
func (s *SlogAdapter) Debugln(args ...any) { s.Debug(args...) }
func (s *SlogAdapter) Infoln(args ...any)  { s.Info(args...) }
func (s *SlogAdapter) Warnln(args ...any)  { s.Warn(args...) }
func (s *SlogAdapter) Errorln(args ...any) { s.Error(args...) }
func (s *SlogAdapter) DPanicln(args ...any) { s.DPanic(args...) }
func (s *SlogAdapter) Panicln(args ...any) { s.Panic(args...) }
func (s *SlogAdapter) Fatalln(args ...any) { s.Fatal(args...) }
func (s *SlogAdapter) Logln(level core.Level, args ...any) {
    s.Logf(level, "%s", fmt.Sprint(args...))
}

// IContextualLogger
func (s *SlogAdapter) With(args ...any) core.ISugaredLogger {
    return &SlogAdapter{logger: s.logger.With(args...), level: s.level}
}
func (s *SlogAdapter) WithLazy(args ...any) core.ISugaredLogger {
    return s.With(args...)
}
func (s *SlogAdapter) Named(name string) core.ISugaredLogger {
    return &SlogAdapter{logger: s.logger.WithGroup(name), level: s.level}
}

// IContextLogger
func (s *SlogAdapter) WithContext(_ any) core.ISugaredLogger { return s }

// ILoggerControl
func (s *SlogAdapter) Desugar() any       { return s.logger }
func (s *SlogAdapter) Level() core.Level  { return s.level }
func (s *SlogAdapter) Sync() error        { return nil }
```

### Example: Logrus Adapter (minimal)

Wrap [logrus](https://github.com/sirupsen/logrus) for teams already using it:

```go
package logrusadapter

import (
    "github.com/sirupsen/logrus"
    "github.com/gestgo/gest/package/common/log/core"
)

type LogrusAdapter struct {
    entry *logrus.Entry
    level core.Level
}

func New(logger *logrus.Logger, level core.Level) core.ISugaredLogger {
    return &LogrusAdapter{entry: logrus.NewEntry(logger), level: level}
}

func (l *LogrusAdapter) Info(args ...any)  { l.entry.Info(args...) }
func (l *LogrusAdapter) Infof(tmpl string, args ...any) { l.entry.Infof(tmpl, args...) }
func (l *LogrusAdapter) Infow(msg string, kv ...any) {
    l.entry.WithFields(kvToFields(kv)).Info(msg)
}

func (l *LogrusAdapter) With(args ...any) core.ISugaredLogger {
    return &LogrusAdapter{entry: l.entry.WithFields(kvToFields(args)), level: l.level}
}

func (l *LogrusAdapter) Named(name string) core.ISugaredLogger {
    return &LogrusAdapter{entry: l.entry.WithField("logger", name), level: l.level}
}

// ... implement remaining methods similarly

func kvToFields(kv []any) logrus.Fields {
    fields := logrus.Fields{}
    for i := 0; i+1 < len(kv); i += 2 {
        if key, ok := kv[i].(string); ok {
            fields[key] = kv[i+1]
        }
    }
    return fields
}
```

### Replacing the Default Adapter in Fx

To use your custom adapter with `log.Module()`, provide a replacement in the fx graph:

```go
import (
    "go.uber.org/fx"
    "log/slog"

    "github.com/gestgo/gest/package/common/log/core"
    slogadapter "your/project/internal/adapter/slog"
)

func ProvideSlogLogger() (core.ISugaredLogger, error) {
    handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    })
    return slogadapter.New(slog.New(handler), core.InfoLevel), nil
}

app := fx.New(
    fx.Provide(ProvideSlogLogger), // replaces log.Module()
    // ... other modules
)
```

### No-op Adapter for Testing

Implement a silent logger for unit tests without any dependency on Zap:

```go
type NopLogger struct{}

func (n *NopLogger) Debug(args ...any)                              {}
func (n *NopLogger) Info(args ...any)                               {}
func (n *NopLogger) Warn(args ...any)                               {}
func (n *NopLogger) Error(args ...any)                              {}
func (n *NopLogger) DPanic(args ...any)                             {}
func (n *NopLogger) Panic(args ...any)                              {}
func (n *NopLogger) Fatal(args ...any)                              {}
func (n *NopLogger) Debugf(t string, a ...any)                      {}
func (n *NopLogger) Infof(t string, a ...any)                       {}
func (n *NopLogger) Warnf(t string, a ...any)                       {}
func (n *NopLogger) Errorf(t string, a ...any)                      {}
func (n *NopLogger) DPanicf(t string, a ...any)                     {}
func (n *NopLogger) Panicf(t string, a ...any)                      {}
func (n *NopLogger) Fatalf(t string, a ...any)                      {}
func (n *NopLogger) Logf(l core.Level, t string, a ...any)          {}
func (n *NopLogger) Debugw(msg string, kv ...any)                   {}
func (n *NopLogger) Infow(msg string, kv ...any)                    {}
func (n *NopLogger) Warnw(msg string, kv ...any)                    {}
func (n *NopLogger) Errorw(msg string, kv ...any)                   {}
func (n *NopLogger) DPanicw(msg string, kv ...any)                  {}
func (n *NopLogger) Panicw(msg string, kv ...any)                   {}
func (n *NopLogger) Fatalw(msg string, kv ...any)                   {}
func (n *NopLogger) Logw(l core.Level, msg string, kv ...any)       {}
func (n *NopLogger) Debugln(args ...any)                            {}
func (n *NopLogger) Infoln(args ...any)                             {}
func (n *NopLogger) Warnln(args ...any)                             {}
func (n *NopLogger) Errorln(args ...any)                            {}
func (n *NopLogger) DPanicln(args ...any)                           {}
func (n *NopLogger) Panicln(args ...any)                            {}
func (n *NopLogger) Fatalln(args ...any)                            {}
func (n *NopLogger) Logln(l core.Level, args ...any)                {}
func (n *NopLogger) With(args ...any) core.ISugaredLogger           { return n }
func (n *NopLogger) WithLazy(args ...any) core.ISugaredLogger       { return n }
func (n *NopLogger) Named(name string) core.ISugaredLogger          { return n }
func (n *NopLogger) WithContext(_ any) core.ISugaredLogger          { return n }
func (n *NopLogger) Desugar() any                                   { return n }
func (n *NopLogger) Level() core.Level                              { return core.FatalLevel }
func (n *NopLogger) Sync() error                                    { return nil }
```

**Usage in tests:**

```go
func TestUserService(t *testing.T) {
    svc := NewUserService(&NopLogger{})
    // ...
}
```

### Adapter Checklist

| Requirement | Notes |
|-------------|-------|
| Implement all 7 sub-interfaces | Compile error if any method is missing |
| `With`/`WithLazy`/`Named` return a new adapter | Never mutate the existing adapter |
| `Sync()` flushes pending writes | Return `nil` if your backend doesn't buffer |
| `Desugar()` returns the underlying logger | Return `self` if there is no underlying logger |
| `Level()` returns current minimum level | Used by callers to skip expensive formatting |
| `Panic`/`Panicf`/`Panicw` must call `panic()` | Contract expected by the interface |
| `Fatal`/`Fatalf`/`Fatalw` must call `os.Exit(1)` | Contract expected by the interface |

## Output to File

Write logs to a file in addition to (or instead of) stdout:

```go
logger, err := zapAdapter.NewWithOptions(
    zapAdapter.WithJSONEncoding(),
    zapAdapter.WithOutputPaths("stdout", "/var/log/myapp.log"),
    zapAdapter.WithErrorOutputPaths("stderr", "/var/log/myapp-error.log"),
)
```
