# 01 — Microsserviços: quando faz sentido e quando é over-engineering

## A pergunta clássica de entrevista sênior

"Quando você escolheria microsserviços em vez de um monolito?" — essa pergunta aparece em quase toda entrevista de sênior porque **a resposta revela julgamento**, não conhecimento técnico. Qualquer um pode listar as vantagens dos microsserviços. O sênior sabe quando **não** usá-los.

---

## O que microsserviços resolvem (de verdade)

### 1. Escala independente

No eventhub, o serviço de reservas recebe 1000 req/s durante a venda de ingressos, mas o serviço de notificações processa 10 msg/s. Com monolito, você escala tudo junto — desperdiçando recursos. Com microsserviços, escala só o serviço de reservas.

### 2. Deploy independente

Você pode fazer deploy do serviço de notificações sem risco de quebrar o fluxo de reserva. No monolito, qualquer deploy mexe em tudo.

### 3. Isolamento de falhas

Se o serviço de notificações cair, as reservas continuam funcionando. No monolito, um OutOfMemoryException no módulo de relatórios derruba a API inteira.

### 4. Times autônomos

Em empresas com 50+ devs, times diferentes podem ser donos de serviços diferentes, com stacks, cadências de deploy e até linguagens diferentes. Isso é o principal argumento do Conway's Law — a arquitetura do software reflete a estrutura de comunicação da organização.

---

## O que microsserviços criam (os custos reais)

### 1. Complexidade de rede

Chamadas entre serviços falham. A rede é não-confiável por definição. Você precisa de retry, circuit breaker, timeout, fallback — patterns que um monolito não precisa.

### 2. Consistência eventual

Num monolito com banco único, `BEGIN; INSERT; INSERT; COMMIT` garante atomicidade. Com microsserviços e bancos separados, você precisa lidar com estados intermediários, compensação e idempotência.

### 3. Debugging distribuído

Um erro no monolito gera um stack trace. Com microsserviços, um erro pode ter começado no serviço A, passado pelo B, falhado no C — você precisa de tracing distribuído só pra entender o que aconteceu.

### 4. Duplicação de código

Validação de email, formatação de data, middleware de autenticação — tudo isso tende a ser duplicado entre serviços (ou movido para uma lib compartilhada, que é um monolito disfarçado).

### 5. Operação mais cara

Cada serviço precisa de CI/CD, monitoramento, logging, secrets, e possivelmente seu próprio banco. 3 microsserviços = 3 pipelines de CI/CD = 3 bancos = 3x a complexidade operacional.

---

## A régua de decisão

| Contexto | Recomendação |
|---|---|
| Startup com 2-5 devs, produto em validação | **Monolito modular**. Microsserviços vão te atrasar. |
| Empresa com 20+ devs, domínio complexo | **Microsserviços por domínio**. Separe por bounded context, não por camada técnica. |
| Sistema com picos de carga sazonais (eventhub) | **Híbrido**: monolito principal + workers isolados para jobs pesados (notificações, geração de relatórios). |
| Time sem experiência em distributed systems | **Monolito**. O custo de aprendizado de distributed systems é maior que o benefício dos microsserviços. |
| Você está na Fase 7 do roadmap | **Microsserviços para aprendizado**. Implementar mesmo que o projeto não justifique — o objetivo é entender os patterns. |

---

## O approach do eventhub: modular monolith com mensageria

Vamos implementar **dois serviços** no eventhub:

```
┌─────────────────────┐         RabbitMQ          ┌──────────────────────┐
│   EventHub.Api      │ ──── ReservaConfirmada ───▶│  EventHub.Notifier   │
│  (serviço principal) │                            │  (worker service)    │
│                     │                            │                      │
│  POST /events       │                            │  Consome evento      │
│  POST /reservations │                            │  Simula envio email  │
│  GET  /events       │                            │                      │
│                     │                            │                      │
│  PostgreSQL + Redis │                            │  (log apenas)        │
└─────────────────────┘                            └──────────────────────┘
```

### Por que não 5 microsserviços

Poderíamos separar: Eventos, Reservas, Pagamentos, Notificações, Usuários. Mas:
- O eventhub é um projeto de portfólio — o objetivo é demonstrar conhecimento, não simular um sistema de produção da Netflix
- Cada serviço adicional = overhead de projeto, Dockerfile, configuração
- Dois serviços são suficientes para demonstrar: publicação/consumo de mensagens, consistência eventual, resiliência

### O que o serviço de notificações faz

1. Consome mensagens `ReservaConfirmada` da fila RabbitMQ
2. Simula envio de email de confirmação (log estruturado)
3. Simula envio de SMS (log estruturado)
4. Implementa retry com Polly (se o "envio" falhar)
5. Dead letter queue para mensagens que falharam após N tentativas

---

## Checklist: você entendeu o tópico quando conseguir...

- [ ] Listar 3 vantagens e 3 desvantagens reais de microsserviços (além do clichê "escala" e "complexidade")
- [ ] Explicar por que "modular monolith" é o ponto de partida correto para a maioria dos projetos
- [ ] Argumentar que microsserviços resolvem problemas organizacionais (times), não só técnicos
- [ ] Desenhar o fluxo de comunicação entre os serviços do eventhub
- [ ] Saber que cada microsserviço deve ter seu próprio banco (não compartilhar banco entre serviços)
