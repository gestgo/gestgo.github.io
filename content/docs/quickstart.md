---
weight: 100
date: "2023-05-03T22:37:22+01:00"
draft: false
author: "GestGo"
title: "Introduction"
icon: "rocket_launch"
toc: true
description: "An introduction to the Gest framework for building scalable server-side applications in Go"
publishdate: "2023-05-03T22:37:22+01:00"
tags: ["Beginners"]
---

## What is Gest?

Gest is a framework for building efficient, high-performance, and scalable server-side applications in Golang. It is designed to provide a clear and structured development experience, similar to how NestJS structures Node.js applications and how Spring Boot organizes the Java ecosystem.

Gest embraces the following principles:

- **Dependency Injection (DI)**
- **Modular Architecture**
- **Inversion of Control (IoC)**
- **Layered & Clean Architecture**
- **OOP-oriented thinking** while remaining idiomatic to Go

## What Gest Is Not

Unlike frameworks that primarily focus on HTTP routing, Gest is **not** an HTTP wrapper around `net/http`.

Instead, Gest provides a generalized application core layer that enables multiple transport mechanisms to coexist within a unified architecture, including:

| Transport | Description |
|---|---|
| RESTful APIs | HTTP |
| gRPC services | Protocol Buffers |
| Message queue consumers | Event-driven messaging |
| Event-driven handlers | Pub/Sub |
| Background workers | Scheduled tasks |

In Gest, transport mechanisms are treated as **adapters**. Business logic and modules remain fully decoupled from external communication layers.

## Philosophy

Over the years, Golang has become a powerful choice for backend systems due to:

- High performance
- Strong concurrency model (goroutines, channels)
- Small binary size
- Simple deployment

However, the traditional Go ecosystem tends to favor micro-libraries (routers, middleware, standalone DI tools) and lacks an opinionated application-level architecture for large-scale systems.

Gest was created to address this gap by providing:

- A **centralized application container**
- **Managed application lifecycle**
- Clear separation between:
  - Core domain
  - Infrastructure
  - Transport layer
- The ability to run **multiple transports simultaneously** within a single application:
  - HTTP server
  - gRPC server
  - Message consumers
  - Scheduled workers

## Inspiration

The architecture of Gest is heavily inspired by:

- The **modular system and dependency injection** model of [NestJS](https://nestjs.com/)

Gest uses [uber-fx](https://github.com/uber-go/fx) as its underlying dependency injection engine, leveraging its lifecycle management and constructor-based DI to power the application container.
