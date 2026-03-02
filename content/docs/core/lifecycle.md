---
weight: 400
date: "2026-03-02T00:00:00+00:00"
draft: false
author: "GestGo"
title: "Lifecycle"
icon: "cycle"
toc: true
description: "Adapter lifecycle and dynamic controller registration for Gest applications"
publishdate: "2026-03-02T00:00:00+00:00"
tags: ["Lifecycle", "Controllers"]
---

## Overview

The `lifecycle` package manages the startup and shutdown of adapters (HTTP, gRPC, Kafka, etc.) and provides a dynamic route registration system for controllers — all integrated with the [Uber Fx](https://github.com/uber-go/fx) dependency injection container.

**Two core responsibilities:**

| Concern | API |
|---------|-----|
| Adapter lifecycle (start/stop) | `AdapterLifecycle`, `BaseAdapter[T]`, `BaseTemplate` |
| Dynamic controller registration | `ICoreController`, `RegisterRouter`, `AsRoute` |

## Installation

```bash
go get github.com/gestgo/gest/package/core/lifecycle
```

## Adapter Lifecycle

### AdapterLifecycle Interface

Every adapter implements this interface to hook into the Fx application lifecycle:

```go
type AdapterLifecycle interface {
    OnStart(ctx context.Context) error
    OnStop(ctx context.Context) error
}
```

- `OnStart` is called when the application starts — open connections, start listeners, register routes.
- `OnStop` is called when the application shuts down — close connections, drain requests gracefully.

### BaseAdapter[T]

`BaseAdapter[T]` is a generic helper that holds a typed config and provides a `RegisterLifecycle` method:

```go
type BaseAdapter[T any] struct {
    Config T
}

func (b *BaseAdapter[T]) RegisterLifecycle(lc fx.Lifecycle, impl AdapterLifecycle)
```

Embed it in your adapter to avoid boilerplate:

```go
type HttpAdapter struct {
    lifecycle.BaseAdapter[HttpConfig]
    server *http.Server
}
```

### Registering Lifecycle with Fx

Call `RegisterLifecycle` (or the standalone `BaseTemplate`) inside your adapter's constructor to attach `OnStart`/`OnStop` to the Fx lifecycle:

```go
func NewHttpAdapter(lc fx.Lifecycle, cfg HttpConfig) *HttpAdapter {
    adapter := &HttpAdapter{}
    adapter.Config = cfg

    adapter.RegisterLifecycle(lc, adapter)
    return adapter
}

func (h *HttpAdapter) OnStart(ctx context.Context) error {
    h.server = &http.Server{Addr: fmt.Sprintf(":%d", h.Config.Port)}
    go h.server.ListenAndServe()
    return nil
}

func (h *HttpAdapter) OnStop(ctx context.Context) error {
    return h.server.Shutdown(ctx)
}
```

### Using BaseTemplate Directly

If you don't need `BaseAdapter`, call `BaseTemplate` directly:

```go
func NewGrpcAdapter(lc fx.Lifecycle, cfg GrpcConfig) *GrpcAdapter {
    adapter := &GrpcAdapter{cfg: cfg}
    lifecycle.BaseTemplate(lc, adapter)
    return adapter
}

func (g *GrpcAdapter) OnStart(ctx context.Context) error {
    // start gRPC server
    return nil
}

func (g *GrpcAdapter) OnStop(ctx context.Context) error {
    g.server.GracefulStop()
    return nil
}
```

## Dynamic Controller Registration

Instead of manually registering each route, Gest can auto-discover all route-registration methods on a controller using reflection.

### ICoreController

A **marker interface** — no methods required. Any struct that implements it is eligible for dynamic registration:

```go
type ICoreController interface{}
```

```go
// Declare your controller implements ICoreController
var _ lifecycle.ICoreController = (*UserController)(nil)

type UserController struct {
    router *echo.Echo
}
```

### Writing Controller Methods

Any exported method with the signature `func(context.Context)` is automatically called during registration. Use these methods to attach route handlers:

```go
func (u *UserController) GetUsers(ctx context.Context) {
    u.router.GET("/users", func(c echo.Context) error {
        return c.JSON(200, []string{"alice", "bob"})
    })
}

func (u *UserController) CreateUser(ctx context.Context) {
    u.router.POST("/users", func(c echo.Context) error {
        return c.JSON(201, map[string]string{"status": "created"})
    })
}

// This method is ignored — no context.Context parameter
func (u *UserController) HelperMethod() {}

// This method is ignored — has a return value
func (u *UserController) GetName(ctx context.Context) string { return "user" }
```

> Methods are called in **alphabetical order** by reflection. Only methods matching `func(context.Context)` exactly are invoked — no parameters, no return values.

### RegisterRouter

Register a single controller — calls all matching methods on it:

```go
err := lifecycle.RegisterRouter(controller, ctx)
if err != nil {
    log.Fatalf("failed to register routes: %v", err)
}
```

- If `ctx` is `nil`, `context.Background()` is used automatically.
- Stops immediately on the first method that panics (**fail-fast**).

### RegisterRouters

Register multiple controllers in one call:

```go
func (h *HttpAdapter) OnStart(ctx context.Context) error {
    return lifecycle.RegisterRouters(h.Config.Controllers, ctx)
}
```

- Processes controllers in order.
- Stops at the first failing controller with an error like `controller[1]: method UserController.CreateUser panicked: ...`.

## Fx Integration

### AsRoute

`AsRoute` is an Fx annotation helper that registers a controller constructor into an Fx **group**, making it injectable as `[]ICoreController` in your adapter:

```go
fx.Provide(
    lifecycle.AsRoute(NewUserController, "httpControllers"),
    lifecycle.AsRoute(NewOrderController, "httpControllers"),
    lifecycle.AsRoute(NewProductController, "httpControllers"),
)
```

### Collecting Controllers in the Adapter

Use `fx.In` with a group tag to receive all controllers registered under the same group:

```go
type HttpAdapterParams struct {
    fx.In
    Lifecycle   fx.Lifecycle
    Config      HttpConfig
    Controllers []lifecycle.ICoreController `group:"httpControllers"`
}

func NewHttpAdapter(p HttpAdapterParams) *HttpAdapter {
    adapter := &HttpAdapter{controllers: p.Controllers}
    adapter.Config = p.Config
    adapter.RegisterLifecycle(p.Lifecycle, adapter)
    return adapter
}

func (h *HttpAdapter) OnStart(ctx context.Context) error {
    return lifecycle.RegisterRouters(h.controllers, ctx)
}
```

## Complete Example

```go
package main

import (
    "context"
    "fmt"
    "net/http"

    "github.com/labstack/echo/v4"
    "go.uber.org/fx"

    "github.com/gestgo/gest/package/core/lifecycle"
)

// --- Config ---

type HttpConfig struct {
    Port int
}

// --- Adapter ---

type HttpAdapter struct {
    lifecycle.BaseAdapter[HttpConfig]
    echo        *echo.Echo
    controllers []lifecycle.ICoreController
}

type HttpAdapterParams struct {
    fx.In
    Lc          fx.Lifecycle
    Config      HttpConfig
    Controllers []lifecycle.ICoreController `group:"httpControllers"`
}

func NewHttpAdapter(p HttpAdapterParams) *HttpAdapter {
    adapter := &HttpAdapter{
        echo:        echo.New(),
        controllers: p.Controllers,
    }
    adapter.Config = p.Config
    adapter.RegisterLifecycle(p.Lc, adapter)
    return adapter
}

func (h *HttpAdapter) OnStart(ctx context.Context) error {
    if err := lifecycle.RegisterRouters(h.controllers, ctx); err != nil {
        return err
    }
    go h.echo.Start(fmt.Sprintf(":%d", h.Config.Port))
    return nil
}

func (h *HttpAdapter) OnStop(ctx context.Context) error {
    return h.echo.Shutdown(ctx)
}

// --- Controllers ---

type UserController struct {
    echo *echo.Echo
}

var _ lifecycle.ICoreController = (*UserController)(nil)

func NewUserController(e *echo.Echo) *UserController {
    return &UserController{echo: e}
}

func (u *UserController) GetUsers(ctx context.Context) {
    u.echo.GET("/users", func(c echo.Context) error {
        return c.JSON(http.StatusOK, []string{"alice", "bob"})
    })
}

func (u *UserController) CreateUser(ctx context.Context) {
    u.echo.POST("/users", func(c echo.Context) error {
        return c.JSON(http.StatusCreated, map[string]string{"status": "created"})
    })
}

// --- Fx App ---

func main() {
    app := fx.New(
        fx.Provide(
            func() HttpConfig { return HttpConfig{Port: 8080} },
            func() *echo.Echo { return echo.New() },
            NewHttpAdapter,
            lifecycle.AsRoute(NewUserController, "httpControllers"),
        ),
        fx.Invoke(func(*HttpAdapter) {}),
    )

    app.Run()
}
```

## API Reference

### Adapter Lifecycle

| Symbol | Description |
|--------|-------------|
| `AdapterLifecycle` | Interface with `OnStart(ctx) error` and `OnStop(ctx) error` |
| `BaseAdapter[T]` | Generic struct embedding typed config, provides `RegisterLifecycle` |
| `BaseTemplate(lc, impl)` | Registers `OnStart`/`OnStop` hooks directly with `fx.Lifecycle` |

### Controller Registration

| Symbol | Description |
|--------|-------------|
| `ICoreController` | Marker interface — no methods required |
| `RegisterRouter(ctrl, ctx)` | Calls all `func(context.Context)` methods on a single controller |
| `RegisterRouters(ctrls, ctx)` | Calls `RegisterRouter` for each controller in a slice |
| `AsRoute(f, groupTag)` | Annotates a constructor for use in an Fx group |

### Method Matching Rules for RegisterRouter

| Condition | Result |
|-----------|--------|
| Exported, takes exactly `context.Context`, no return values | Called |
| Unexported | Skipped |
| No parameters | Skipped |
| Has return values | Skipped |
| Multiple parameters | Skipped |
| Method panics | Error returned immediately (fail-fast) |
