# SAGA Pattern

## Problema

Diversos microserviços que podem apresentar inconsistências nos dados.

**Exemplo:** o usuário faz o pedido e possui estoque, porém ao acionar posteriormente o microserviço de estoque, a quantidade já não está mais disponível.

## Solução

Cada microserviço processa uma solicitação e grava no seu banco de dados e em seguida passa essa transação adiante para os outros. Se houver alguma falha, ações compensatórias (ex: rollback) são executadas em todos que já foram executados.

## Abordagens

### 1. Orquestração
Um serviço central (orquestrador) coordena todos os passos da saga e decide o que fazer em caso de falha.

```
Orquestrador → Serviço A → Serviço B → Serviço C
                  ↑                        |
                  └──── Compensação ←──────┘ (se falhar)
```

**Vantagens:** fácil de entender, lógica centralizada
**Desvantagens:** ponto único de falha, acoplamento ao orquestrador

### 2. Coreografia
Cada serviço sabe o que fazer quando recebe um evento. Não há coordenador central.

```
Serviço A → Evento → Serviço B → Evento → Serviço C
                                               |
Serviço A ← Evento compensatório ←────────────┘ (se falhar)
```

**Vantagens:** desacoplado, cada serviço é autônomo
**Desvantagens:** fluxo mais difícil de rastrear, complexidade distribuída

## Quando usar

- Transações distribuídas entre múltiplos microserviços
- Quando não é possível usar transações ACID tradicionais (banco único)
- Operações que precisam de rollback em caso de falha parcial

---

[← Anterior: Tell, Don't Ask](05-tell-dont-ask.md) | [Voltar ao índice](README.md)
