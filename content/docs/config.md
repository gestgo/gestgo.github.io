---
weight: 200
date: "2026-03-02T00:00:00+00:00"
draft: false
author: "GestGo"
title: "Config"
icon: "settings"
toc: true
description: "Type-safe, multi-source configuration management for Gest applications"
publishdate: "2026-03-02T00:00:00+00:00"
tags: ["Configuration"]
---

## Overview

The `config` package provides a type-safe, generic configuration system that can load and merge settings from multiple sources: files, environment variables, and command-line flags.

**Key features:**

- **Generic & type-safe** — define your config as a plain Go struct
- **Multi-source** — combine file, env, and flag loaders in any order
- **Priority-based merging** — loaders listed last have the highest priority
- **Custom merge strategies** — deep merge (default) or shallow merge
- **Validation** — validate config after loading

## Installation

```bash
go get github.com/gestgo/gest/package/common/config
go get github.com/gestgo/gest/package/common/config/loader
```

## Quick Start

```go
package main

import (
    "fmt"
    "log"

    "github.com/gestgo/gest/package/common/config"
    "github.com/gestgo/gest/package/common/config/loader"
)

type AppConfig struct {
    Server struct {
        Host string `mapstructure:"host"`
        Port int    `mapstructure:"port"`
    } `mapstructure:"server"`
    Database struct {
        DSN string `mapstructure:"dsn"`
    } `mapstructure:"database"`
}

func main() {
    cfg := config.New[AppConfig](
        loader.NewFileLoader("config.yaml", "yaml"), // lowest priority
        loader.NewEnvLoader("APP").WithAutoKeys(AppConfig{}), // highest priority
    )

    if err := cfg.Load(); err != nil {
        log.Fatal(err)
    }

    appCfg := cfg.Get()
    fmt.Printf("Server: %s:%d\n", appCfg.Server.Host, appCfg.Server.Port)
}
```

## Defining Your Config Struct

Use `mapstructure` tags to map config keys to struct fields:

```go
type AppConfig struct {
    Server struct {
        Host string `mapstructure:"host"`
        Port int    `mapstructure:"port"`
    } `mapstructure:"server"`

    Database struct {
        Host     string `mapstructure:"host"`
        Port     int    `mapstructure:"port"`
        Name     string `mapstructure:"name"`
        Password string `mapstructure:"password"`
    } `mapstructure:"database"`

    Features map[string]bool `mapstructure:"features"`
}
```

> If a field has no `mapstructure` tag, the lowercased field name is used automatically.

## Loaders

Loaders are the sources from which configuration is read. You can combine multiple loaders — they are merged in the order provided, with **the last loader having the highest priority**.

### File Loader

Loads configuration from a file. Supported formats: **JSON, YAML, TOML, Properties, HCL**.

```go
import "github.com/gestgo/gest/package/common/config/loader"

// YAML
fileLoader := loader.NewFileLoader("config.yaml", "yaml")

// JSON
fileLoader := loader.NewFileLoader("config.json", "json")

// TOML
fileLoader := loader.NewFileLoader("config.toml", "toml")
```

**Example `config.yaml`:**

```yaml
server:
  host: localhost
  port: 8080
database:
  host: localhost
  port: 5432
  name: mydb
```

### Environment Variable Loader

Loads configuration from environment variables. Variables are matched by a prefix and converted automatically:

- `APP_SERVER_HOST` → `server.host`
- `APP_DATABASE_PORT` → `database.port`

```go
// With prefix "APP", reads APP_* env vars
envLoader := loader.NewEnvLoader("APP")

// No prefix — reads all env vars
envLoader := loader.NewEnvLoader("")
```

**Bind specific keys manually:**

```go
envLoader := loader.NewEnvLoader("APP").
    WithKeys("server.host", "server.port", "database.password")
```

**Auto-extract keys from your struct (recommended):**

```go
envLoader := loader.NewEnvLoader("APP").
    WithAutoKeys(AppConfig{})
```

`WithAutoKeys` uses reflection to extract all keys from the struct, so you don't need to list them manually.

**Setting env vars:**

```bash
export APP_SERVER_HOST=0.0.0.0
export APP_SERVER_PORT=9090
export APP_DATABASE_PASSWORD=secret
```

### Flag Loader

Loads configuration from command-line flags using `pflag` (POSIX-compliant).

**Using the global flag set:**

```go
import (
    "github.com/spf13/pflag"
    "github.com/gestgo/gest/package/common/config/loader"
)

pflag.String("server.host", "localhost", "Server host")
pflag.Int("server.port", 8080, "Server port")
pflag.Parse()

flagLoader := loader.NewFlagLoader(nil) // nil = use pflag.CommandLine
```

**Using a custom flag set:**

```go
flags := pflag.NewFlagSet("app", pflag.ExitOnError)
flags.String("server.host", "localhost", "Server host")
flags.Int("server.port", 8080, "Server port")
flags.Parse(os.Args[1:])

flagLoader := loader.NewFlagLoader(flags)
```

**Running the app:**

```bash
./myapp --server.host=0.0.0.0 --server.port=9090
```

> Flags with dots (`.`) automatically create nested structures:
> `--server.port=9090` → `Server.Port = 9090`

## Combining Multiple Loaders

You can stack loaders from multiple sources. **Loaders listed last override those listed first.**

```go
cfg := config.New[AppConfig](
    loader.NewFileLoader("config.yaml", "yaml"),          // 1. lowest priority
    loader.NewEnvLoader("APP").WithAutoKeys(AppConfig{}), // 2. overrides file
    loader.NewFlagLoader(nil),                            // 3. highest priority
)

if err := cfg.Load(); err != nil {
    log.Fatal(err)
}
```

**Priority order (highest → lowest):**

| Source | Priority |
|--------|----------|
| CLI Flags | Highest |
| Environment Variables | Medium |
| Config File | Lowest |

## Merge Strategies

### DefaultMerge (default)

Deep merge using reflection. Only **non-zero** values from the source override the destination.

```go
// dst = {Server: {Host: "localhost", Port: 8080}}
// src = {Server: {Port: 9090}}
// result = {Server: {Host: "localhost", Port: 9090}}  ← Host preserved, Port overridden
```

This is the default and works correctly with the multi-loader pattern.

### ShallowMerge

Replaces the entire struct if the source is non-zero. Useful when you want each loader to fully replace the previous config.

```go
cfg := config.New[AppConfig](loaders...).
    WithMerge(config.ShallowMerge[AppConfig])
```

### Custom Merge

Implement your own merge logic:

```go
customMerge := func(dst, src *AppConfig) error {
    if src.Server.Port != 0 {
        dst.Server.Port = src.Server.Port
    }
    // ... custom rules
    return nil
}

cfg := config.New[AppConfig](loaders...).
    WithMerge(customMerge)
```

## Validation

### Using ValidatorFunc

The simplest way to add validation — pass a function directly:

```go
import "github.com/gestgo/gest/package/common/config/core"

cfg := config.New[AppConfig](loaders...).
    WithValidator(core.ValidatorFunc[AppConfig](func(c *AppConfig) error {
        if c.Server.Port < 1024 {
            return fmt.Errorf("server.port must be >= 1024, got %d", c.Server.Port)
        }
        if c.Database.DSN == "" {
            return fmt.Errorf("database.dsn is required")
        }
        return nil
    }))
```

### Using the Validator Interface

For reusable validators, implement the `Validator[T]` interface:

```go
type AppConfigValidator struct{}

func (v *AppConfigValidator) Validate(cfg *AppConfig) error {
    if cfg.Server.Port < 1024 {
        return fmt.Errorf("server.port must be >= 1024")
    }
    return nil
}

cfg := config.New[AppConfig](loaders...).
    WithValidator(&AppConfigValidator{})
```

### CompositeValidator

Combine multiple validators — all must pass:

```go
portValidator := core.ValidatorFunc[AppConfig](func(c *AppConfig) error {
    if c.Server.Port < 1024 {
        return fmt.Errorf("server.port must be >= 1024")
    }
    return nil
})

dbValidator := core.ValidatorFunc[AppConfig](func(c *AppConfig) error {
    if c.Database.DSN == "" {
        return fmt.Errorf("database.dsn is required")
    }
    return nil
})

validator := config.NewCompositeValidator(portValidator, dbValidator)

cfg := config.New[AppConfig](loaders...).
    WithValidator(validator)
```

## Accessing Config Data

After calling `Load()`, retrieve your config with `Get()` or `GetPtr()`:

```go
if err := cfg.Load(); err != nil {
    log.Fatal(err)
}

// Value copy
appCfg := cfg.Get()
fmt.Println(appCfg.Server.Port)

// Pointer (for passing by reference)
appCfgPtr := cfg.GetPtr()
fmt.Println(appCfgPtr.Server.Host)
```

> Always call `Load()` before `Get()`. Calling `Get()` without `Load()` returns the zero value of `T`.

## Complete Example

```go
package main

import (
    "fmt"
    "log"

    "github.com/spf13/pflag"
    "github.com/gestgo/gest/package/common/config"
    "github.com/gestgo/gest/package/common/config/core"
    "github.com/gestgo/gest/package/common/config/loader"
)

type AppConfig struct {
    Server struct {
        Host string `mapstructure:"host"`
        Port int    `mapstructure:"port"`
    } `mapstructure:"server"`
    Database struct {
        DSN string `mapstructure:"dsn"`
    } `mapstructure:"database"`
}

func main() {
    // Define flags
    pflag.String("server.host", "localhost", "Server host")
    pflag.Int("server.port", 8080, "Server port")
    pflag.Parse()

    cfg := config.New[AppConfig](
        loader.NewFileLoader("config.yaml", "yaml"),
        loader.NewEnvLoader("APP").WithAutoKeys(AppConfig{}),
        loader.NewFlagLoader(nil),
    ).WithValidator(core.ValidatorFunc[AppConfig](func(c *AppConfig) error {
        if c.Server.Port <= 0 {
            return fmt.Errorf("server.port must be > 0")
        }
        return nil
    }))

    if err := cfg.Load(); err != nil {
        log.Fatalf("failed to load config: %v", err)
    }

    appCfg := cfg.Get()
    fmt.Printf("Starting server on %s:%d\n", appCfg.Server.Host, appCfg.Server.Port)
}
```

## API Reference

### `config.New[T]`

```go
func New[T any](loaders ...Loader[*T]) *Config[T]
```

Creates a new `Config` with the given loaders. Uses `DefaultMerge` by default.

### `Config[T]` Methods

| Method | Description |
|--------|-------------|
| `WithMerge(fn MergeFunc[T]) *Config[T]` | Set a custom merge strategy |
| `WithValidator(v Validator[T]) *Config[T]` | Set a validator |
| `Load() error` | Load and merge all sources, then validate |
| `Get() T` | Return the loaded config as a value |
| `GetPtr() *T` | Return a pointer to the loaded config |

### Loaders

| Constructor | Description |
|-------------|-------------|
| `loader.NewFileLoader(path, fileType string)` | Load from file (json, yaml, toml, hcl, properties) |
| `loader.NewEnvLoader(prefix string)` | Load from environment variables |
| `loader.NewFlagLoader(flagSet *pflag.FlagSet)` | Load from CLI flags |

### Merge Functions

| Function | Description |
|----------|-------------|
| `config.DefaultMerge[T]` | Deep merge — non-zero values override (default) |
| `config.ShallowMerge[T]` | Shallow merge — replaces entire struct if non-zero |

### Validators

| Type | Description |
|------|-------------|
| `core.ValidatorFunc[T]` | Function adapter implementing `Validator[T]` |
| `config.NewCompositeValidator[T]` | Combines multiple validators |
