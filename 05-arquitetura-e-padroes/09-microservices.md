# Microservices vs Monolith

## Monolith

A single application with all the code together:

```
┌─────────────────────────┐
│        Monolito          │
│  ┌─────┐ ┌─────┐ ┌────┐│
│  │Users│ │Order│ │Pay ││
│  └─────┘ └─────┘ └────┘│
│        [1 banco]         │
└─────────────────────────┘
```

### Advantages

- Simple to develop, test, deploy
- A single codebase
- Simple ACID transactions
- No network latency between modules

### Disadvantages

- Scales everything together (cannot scale just the payment module)
- Deploys everything together (a small change redeploys everything)
- Large teams step on each other's toes
- Single technology stack

## Microservices

Each module is an independent service with its own database:

```
┌────────┐  ┌────────┐  ┌────────┐
│ Users  │  │ Orders │  │Payment │
│ Service│  │ Service│  │Service │
│ [DB 1] │  │ [DB 2] │  │ [DB 3] │
└───┬────┘  └───┬────┘  └───┬────┘
    │           │            │
────┴───────────┴────────────┴──── Message Bus / API Gateway
```

### Advantages

- Independent scaling per service
- Independent deployment
- Each team "owns" its service
- Technology can vary per service

### Disadvantages

- High operational complexity (monitoring, tracing, deployment)
- Distributed transactions (no simple ACID -- requires SAGA)
- Network latency
- Harder debugging

## Modular Monolith (middle ground)

A monolith with **well-isolated modules**. Each module has its own entities and rules, but runs in the same process:

```
┌─────────────────────────────────┐
│         Modular Monolith         │
│  ┌─────────┐ ┌─────────┐ ┌────┐│
│  │ Users   │ │ Orders  │ │Pay ││
│  │ Module  │ │ Module  │ │Mod ││
│  │(schema1)│ │(schema2)│ │(s3)││
│  └─────────┘ └─────────┘ └────┘│
│         [1 banco, N schemas]     │
└─────────────────────────────────┘
```

- Communication between modules via **internal interfaces** or **in-process events**
- If needed in the future, extracting a module into a microservice is easier
- **Best option for the majority of projects**

## Communication between services

### Synchronous (request/response)

```
Service A  ──HTTP/gRPC──→  Service B
           ←──response───
```

- REST APIs or gRPC
- Simple, but creates **temporal coupling** (A depends on B being up)

### Asynchronous (events/messages)

```
Service A  ──publica evento──→  Message Broker  ──entrega──→  Service B
```

- RabbitMQ, Kafka, Azure Service Bus, SQS
- Decouples services in time
- More resilient, but more complex

## API Gateway

Single entry point that routes to microservices:

```
Cliente → [API Gateway] → Users Service
                        → Orders Service
                        → Payment Service
```

Responsibilities:
- Routing
- Centralized authentication
- Rate limiting
- Response aggregation
- SSL termination

Tools: YARP (.NET), Kong, Ocelot, Azure API Management

## When to choose what

| Scenario | Recommendation |
|---------|-------------|
| New project, small team | Monolith or Modular Monolith |
| Simple domain, CRUD | Monolith |
| Complex domain, medium team | Modular Monolith |
| Multiple teams, high scale | Microservices |
| Parts with very different volumes | Microservices for those parts |

> "If you can't build a well-made monolith, you won't be able to build microservices." -- Simon Brown

---

[← Previous: CQRS](08-cqrs.md) | [Next: Messaging →](10-mensageria.md) | [Back to index](README.md)
