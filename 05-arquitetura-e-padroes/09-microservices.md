# Microservices vs Monolito

## Monolito

Uma unica aplicacao com todo o codigo junto:

```
┌─────────────────────────┐
│        Monolito          │
│  ┌─────┐ ┌─────┐ ┌────┐│
│  │Users│ │Order│ │Pay ││
│  └─────┘ └─────┘ └────┘│
│        [1 banco]         │
└─────────────────────────┘
```

### Vantagens

- Simples de desenvolver, testar, deployar
- Uma unica base de codigo
- Transacoes ACID simples
- Sem latencia de rede entre modulos

### Desvantagens

- Escala tudo junto (nao pode escalar so o modulo de pagamento)
- Deploy tudo junto (mudanca pequena redeploya tudo)
- Times grandes pisam no pe um do outro
- Tecnologia unica

## Microservices

Cada modulo e um servico independente com seu proprio banco:

```
┌────────┐  ┌────────┐  ┌────────┐
│ Users  │  │ Orders │  │Payment │
│ Service│  │ Service│  │Service │
│ [DB 1] │  │ [DB 2] │  │ [DB 3] │
└───┬────┘  └───┬────┘  └───┬────┘
    │           │            │
────┴───────────┴────────────┴──── Message Bus / API Gateway
```

### Vantagens

- Escala independente por servico
- Deploy independente
- Cada time "dona" do seu servico
- Tecnologia pode variar por servico

### Desvantagens

- Complexidade operacional alta (monitoramento, tracing, deploy)
- Transacoes distribuidas (sem ACID simples — precisa de SAGA)
- Latencia de rede
- Debugging mais dificil

## Modular Monolith (meio-termo)

Monolito com **modulos bem isolados**. Cada modulo tem suas proprias entidades e regras, mas roda no mesmo processo:

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

- Comunicacao entre modulos via **interfaces internas** ou **eventos in-process**
- Se precisar no futuro, extrair um modulo para microservice e mais facil
- **Melhor opcao para a maioria dos projetos**

## Comunicacao entre servicos

### Sincrona (request/response)

```
Service A  ──HTTP/gRPC──→  Service B
           ←──response───
```

- REST APIs ou gRPC
- Simples, mas cria **acoplamento temporal** (A depende de B estar de pe)

### Assincrona (eventos/mensagens)

```
Service A  ──publica evento──→  Message Broker  ──entrega──→  Service B
```

- RabbitMQ, Kafka, Azure Service Bus, SQS
- Desacopla servicos no tempo
- Mais resiliente, mas mais complexo

## API Gateway

Ponto unico de entrada que roteia para os microservices:

```
Cliente → [API Gateway] → Users Service
                        → Orders Service
                        → Payment Service
```

Responsabilidades:
- Roteamento
- Autenticacao centralizada
- Rate limiting
- Agregacao de respostas
- SSL termination

Ferramentas: YARP (.NET), Kong, Ocelot, Azure API Management

## Quando escolher o que

| Cenario | Recomendacao |
|---------|-------------|
| Projeto novo, time pequeno | Monolito ou Modular Monolith |
| Dominio simples, CRUD | Monolito |
| Dominio complexo, time medio | Modular Monolith |
| Multiplos times, alta escala | Microservices |
| Partes com volumes muito diferentes | Microservices para essas partes |

> "Se voce nao consegue fazer um monolito bem feito, nao vai conseguir fazer microservices." — Simon Brown

---

[← Anterior: CQRS](08-cqrs.md) | [Próximo: Mensageria →](10-mensageria.md) | [Voltar ao índice](README.md)
