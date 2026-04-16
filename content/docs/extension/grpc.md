---
weight: 210
date: "2026-04-16T00:00:00+00:00"
draft: false
author: "GestGo"
title: "gRPC Extension"
icon: "extension"
toc: true
description: "gRPC server adapter for exposing services to clients using the gRPC protocol"
publishdate: "2026-04-16T00:00:00+00:00"
tags: ["Extension"]
---

## Overview

The `grpc` extension wraps [google.golang.org/grpc](https://pkg.go.dev/google.golang.org/grpc) to run a gRPC server inside a Gest application. It integrates with the Fx lifecycle so the server starts and stops automatically with your app.

## Installation

```bash
go get github.com/gestgo/gest/package/extension/grpc
```

## Core Concepts

### IGRPCHandler

The only interface you need to implement. Group related services into a single handler:

```go
type IGRPCHandler interface {
    Register(s *grpc.Server)
}
```

Create one handler per domain (e.g. `UserServiceHandler`, `OrderServiceHandler`) and call the generated `pb.Register*Server` inside `Register`.

### Config

```go
type Config struct {
    Port          int                // listening port
    ServerOptions []grpc.ServerOption // optional: interceptors, TLS, etc.
}
```

## Quick Start

### 1. Define a proto and generate code

```proto
// proto/greeter/greeter.proto
syntax = "proto3";
package greeter;
option go_package = "myapp/proto/greeter";

service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest { string name = 1; }
message HelloReply   { string message = 1; }
```

```bash
protoc --go_out=. --go-grpc_out=. proto/greeter/greeter.proto
```

### 2. Implement a Handler

```go
package service

import (
    "context"
    "fmt"

    pb "myapp/proto/greeter"
    "google.golang.org/grpc"
)

type GreeterHandler struct {
    pb.UnimplementedGreeterServer
}

func NewGreeterHandler() *GreeterHandler {
    return &GreeterHandler{}
}

// Register implements grpc.IGRPCHandler
func (h *GreeterHandler) Register(s *grpc.Server) {
    pb.RegisterGreeterServer(s, h)
}

func (h *GreeterHandler) SayHello(_ context.Context, req *pb.HelloRequest) (*pb.HelloReply, error) {
    return &pb.HelloReply{Message: fmt.Sprintf("Hello, %s!", req.Name)}, nil
}
```

### 3. Wire with Fx

```go
package main

import (
    "myapp/service"

    grpcext "github.com/gestgo/gest/package/extension/grpc"
    "go.uber.org/fx"
    "google.golang.org/grpc"
)

func main() {
    app := fx.New(
        fx.Supply(grpcext.Config{
            Port: 50051,
        }),

        fx.Provide(grpcext.AsHandler(service.NewGreeterHandler)),

        grpcext.Module(),

        fx.Invoke(func(*grpc.Server) {}), // keep server alive
    )

    app.Run()
}
```

## Registering Multiple Handlers

Group related services into separate handlers and register them all with `AsHandler`:

```go
fx.Provide(
    grpcext.AsHandler(service.NewUserServiceHandler),
    grpcext.AsHandler(service.NewOrderServiceHandler),
    grpcext.AsHandler(service.NewPaymentServiceHandler),
),
grpcext.Module(),
```

Each handler's `Register` method is called once during `OnStart`, in the order they are provided.

## Server Options

Pass `grpc.ServerOption` values via `Config.ServerOptions` to configure interceptors, TLS, max message size, etc.

### Unary Interceptor

```go
fx.Supply(grpcext.Config{
    Port: 50051,
    ServerOptions: []grpc.ServerOption{
        grpc.UnaryInterceptor(func(
            ctx context.Context,
            req any,
            info *grpc.UnaryServerInfo,
            handler grpc.UnaryHandler,
        ) (any, error) {
            log.Printf("RPC: %s", info.FullMethod)
            return handler(ctx, req)
        }),
    },
}),
```

### TLS

```go
import "google.golang.org/grpc/credentials"

creds, err := credentials.NewServerTLSFromFile("cert.pem", "key.pem")
if err != nil {
    log.Fatal(err)
}

fx.Supply(grpcext.Config{
    Port: 50051,
    ServerOptions: []grpc.ServerOption{
        grpc.Creds(creds),
    },
}),
```

### Max Message Size

```go
fx.Supply(grpcext.Config{
    Port: 50051,
    ServerOptions: []grpc.ServerOption{
        grpc.MaxRecvMsgSize(16 * 1024 * 1024), // 16 MB
    },
}),
```

## Complete Example: User Service

**Project layout:**

```
myapp/
├── main.go
├── proto/user/user.proto
└── src/
    ├── module/app.go
    └── service/user.go
```

**`main.go`**

```go
package main

import "myapp/src/module"

func main() {
    app := module.NewApp()
    app.Run()
}
```

**`src/module/app.go`**

```go
package module

import (
    "myapp/src/service"

    grpcext "github.com/gestgo/gest/package/extension/grpc"
    "go.uber.org/fx"
    "google.golang.org/grpc"
)

func NewApp() *fx.App {
    return fx.New(
        fx.Supply(grpcext.Config{
            Port: 50051,
        }),

        fx.Provide(grpcext.AsHandler(service.NewUserServiceHandler)),

        grpcext.Module(),

        fx.Invoke(func(*grpc.Server) {}),
    )
}
```

**`src/service/user.go`**

```go
package service

import (
    "context"

    pb "myapp/proto/user"
    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

type UserServiceHandler struct {
    pb.UnimplementedUserServiceServer
}

func NewUserServiceHandler() *UserServiceHandler {
    return &UserServiceHandler{}
}

func (h *UserServiceHandler) Register(s *grpc.Server) {
    pb.RegisterUserServiceServer(s, h)
}

func (h *UserServiceHandler) GetUser(_ context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    if req.Id == "" {
        return nil, status.Error(codes.InvalidArgument, "id is required")
    }
    return &pb.User{Id: req.Id, Name: "John Doe"}, nil
}
```

**Build and run:**

```bash
go build -o myapp .
./myapp
# grpc: starting gRPC server on [::]:50051
```

## Accessing the Raw grpc.Server

`grpc.Module()` and `RegisterGRPCHooks` both provide `*grpc.Server` in the Fx graph. Inject it if you need to register services outside of handlers:

```go
fx.Invoke(func(s *grpc.Server) {
    pb.RegisterHealthServer(s, health.NewServer())
}),
```

## API Reference

### Config

| Field | Type | Description |
|-------|------|-------------|
| `Port` | `int` | Listening port for the gRPC server |
| `ServerOptions` | `[]grpc.ServerOption` | Optional server options (interceptors, TLS, etc.) |

### Functions

| Function | Description |
|----------|-------------|
| `grpc.Module()` | Returns Fx module that provides `*grpc.Server` |
| `grpc.AsHandler(f)` | Annotates a constructor into the `grpcHandlers` Fx group |
| `grpc.RegisterGRPCHooks(lc, params)` | Low-level: creates adapter and registers lifecycle manually |
| `grpc.New(cfg, handlers...)` | Creates `GRPCAdapter` without Fx |

### GRPCAdapter Methods

| Method | Description |
|--------|-------------|
| `OnStart(ctx) error` | Registers all handlers and starts listening |
| `OnStop(ctx) error` | Gracefully stops the gRPC server |
| `Server() *grpc.Server` | Returns the underlying `grpc.Server` |
