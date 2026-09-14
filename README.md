# FlowPipe AI

> SaaS backend-first, multi-tenant, para empresas definirem e rodarem **pipelines de processamento** — de notas fiscais a detecção de anomalia de billing — com IA integrada.
>
> Projeto de **estudo**, conduzido com arquitetura, decisões e processo como se fosse um produto real.

**Status:** 🚧 Fase 0 — fundamentos do repositório (ver [Plano de Desenvolvimento](docs/development-plan.md))

---

## O que é

Cada cliente (tenant) pode definir pipelines (`DOCUMENT_PROCESSING`, `BILLING_ANOMALY_DETECTION`, jobs genéricos), disparar tarefas via API, acompanhar o status em tempo real (`PENDING → PROCESSING → COMPLETED/FAILED`) e receber alertas gerados por IA — tudo isolado por tenant.

Veja o [Documento de Visão](docs/vision.md) completo para o problema, o valor pro mercado e os pipelines suportados.

## Arquitetura, em uma linha

```
Cliente → Nginx → API (NestJS) → Fila (Redis+BullMQ) → Orchestrator (Go) → Workers (Python/Go) → PostgreSQL
```

Detalhes completos, com diagramas e o porquê de cada peça, no [Documento de Visão](docs/vision.md#4-arquitetura-em-alto-nível).

## Stack

| Camada | Tecnologia |
|---|---|
| API | NestJS (REST/GraphQL, JWT/OAuth) |
| IA / workers | Python (OCR, extração, detecção de anomalia) |
| Orchestrator / workers | Golang (alta concorrência, validação, CPU-intensivo) |
| Banco | PostgreSQL (multi-tenant) |
| Fila | Redis + BullMQ |
| Observabilidade | Prometheus + Grafana |
| Infra local | Docker / Docker Compose |

Lista completa e papel de cada tecnologia: [docs/vision.md § 6](docs/vision.md#6-stack-tecnológico).

## Documentação

| Documento | Conteúdo |
|---|---|
| [docs/vision.md](docs/vision.md) | O que é, problema, arquitetura, stack, restrições de custo |
| [docs/development-plan.md](docs/development-plan.md) | Fases de execução, estrutura de pastas (DDD), templates, primeiras issues |
| [docs/adr/](docs/adr/) | Decisões de arquitetura registradas |
| [docs/diagrams/](docs/diagrams/) | Diagramas C4, ER e de sequência |

Versões ilustradas (com diagramas navegáveis e banco de termos), usadas como referência de estudo:
- [FlowPipe AI — Visão Geral](https://claude.ai/code/artifact/9843861c-0ce8-4024-92bf-9678686e70bf)
- [FlowPipe AI — Plano de Desenvolvimento](https://claude.ai/code/artifact/888befe1-e180-482a-a636-194bd25b5467)
- [FlowPipe Infra Playbook](https://claude.ai/code/artifact/aae1757d-8af6-4b0c-b719-293646db3667) — Nginx, load balancer, filas/workers e a versão AWS

## Contribuindo

Fluxo de branches, convenção de commits e como abrir um PR: [CONTRIBUTING.md](CONTRIBUTING.md).
