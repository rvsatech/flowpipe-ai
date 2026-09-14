# FlowPipe AI — Documento de Visão

> **Status:** Documento de visão inicial (pode e vai mudar)
> **Próxima etapa:** Definir system design e arquitetura detalhada

**Nome do projeto:** FlowPipe AI *(provisório — pode mudar)*

---

## 1. O que é o FlowPipe AI

FlowPipe AI é um **SaaS backend-first, multi-tenant**, que permite a empresas:

- Definir **pipelines de processamento** (ex.: "processar nota fiscal", "detectar anomalia de billing", "enviar e-mail em massa")
- Disparar tarefas desses pipelines via **API**
- Acompanhar em **tempo real** o status de cada tarefa (`PENDING` → `PROCESSING` → `COMPLETED` / `FAILED`)
- Receber **alertas e recomendações geradas por IA**

Cada cliente (**tenant**) tem seus próprios pipelines, tarefas, dados e quotas de uso — isolados dos demais.

> Projeto com fins de **aprendizado**, mas conduzido com arquitetura, decisões e processo **como se fosse um produto real**.

---

## 2. Problema e valor para o mercado

Empresas de todos os tamanhos precisam rodar pipelines de processamento para fins diversos: processar documentos (notas fiscais, contratos, currículos), analisar billing de cloud, rodar integrações (ERP, e-mails em massa, relatórios).

Hoje isso costuma ser resolvido com:

- Scripts ad-hoc rodando na mão
- Cron jobs frágeis, sem retry adequado
- Nenhuma visibilidade do que está rodando
- Nenhuma priorização inteligente
- Nenhuma medição de custo por cliente/processo

### O que o FlowPipe AI oferece

- Uma **API única** para definir e disparar pipelines
- **Filas e workers** para execução assíncrona, com retry e dead-letter queue
- **IA integrada** para:
  - Extrair dados de documentos não estruturados
  - Detectar anomalias de custo (ex.: picos de gasto em cloud)
  - Priorizar tarefas mais críticas/valiosas
- **Observabilidade** (métricas, logs, traces)
- **Multi-tenant** com billing simulado (Stripe em modo teste)

### Impacto esperado

- Reduz custo de desenvolvimento interno (não precisa construir tudo do zero)
- Aumenta confiabilidade dos processos (retry, observabilidade, alertas)
- Ajuda a identificar desperdícios e anomalias antes que virem problema grande
- Permite que uma pessoa cuide de um volume muito maior de tarefas

---

## 3. Pipelines suportados (exemplos)

### 3.1 Document Processing

Upload de documento → OCR + extração de dados com IA → validação de regras de negócio → persistência → notificação.

### 3.2 Billing Anomaly Detection

Upload de CSV de billing (AWS/GCP/Azure) → normalização e agregação → detecção de anomalias (picos de gasto, serviços novos) → geração de alertas e recomendações de economia.

### 3.3 Generic Job Runner

Jobs genéricos (enviar e-mails, sincronizar com ERP, gerar relatórios) → execução assíncrona com retry e observabilidade.

---

## 4. Arquitetura em alto nível

```
                                ┌────────────────────────┐
                                │        Cliente          │
                                │ (dashboard / API caller) │
                                └────────────┬─────────────┘
                                             │  REST/GraphQL (JWT/OAuth)
                                             ▼
                                ┌────────────────────────┐
                                │      API (NestJS)       │
                                │  Auth · Tenants · Billing│
                                │  simulado · Webhooks     │
                                └────────────┬─────────────┘
                                             │ enfileira tarefa
                                             ▼
                                ┌────────────────────────┐
                                │   Fila (Redis + BullMQ)  │
                                │   retry + dead-letter    │
                                └────────────┬─────────────┘
                                             │
                                             ▼
                                ┌────────────────────────┐
                                │  Orchestrator (Golang)  │
                                │  alta concorrência,      │
                                │  distribui etapas do     │
                                │  pipeline entre workers   │
                                └──────┬───────────┬───────┘
                                       │           │
                         ┌─────────────┘           └─────────────┐
                         ▼                                       ▼
              ┌────────────────────┐                  ┌────────────────────┐
              │  Workers Python     │                  │  Workers Golang     │
              │  OCR · extração IA  │                  │  validação de regras │
              │  detecção de anomalia│                 │  processamento       │
              │  (scikit-learn/pyod)│                  │  CPU-intensivo       │
              └──────────┬──────────┘                  └──────────┬──────────┘
                         │                                        │
                         └───────────────┬────────────────────────┘
                                         ▼
                            ┌────────────────────────┐
                            │  PostgreSQL (multi-tenant)│
                            │  dados, tarefas, resultados│
                            └────────────┬─────────────┘
                                         │
                                         ▼
                            ┌────────────────────────┐
                            │  Notificação / Webhook   │
                            │  (via API NestJS)         │
                            └────────────────────────┘

        ┌──────────────────────────────────────────────────────────┐
        │   Observabilidade transversal: Prometheus + Grafana        │
        │   Logs estruturados por task_id · pipeline_id · tenant_id  │
        └──────────────────────────────────────────────────────────┘
```

### Papel de cada camada

| Camada | Responsabilidade |
|---|---|
| **API (NestJS)** | Ponto único de entrada: autenticação, definição de pipelines, disparo de tarefas, billing simulado, notificações |
| **Fila (Redis + BullMQ)** | Desacopla o disparo da execução; garante retry e dead-letter queue para falhas |
| **Orchestrator (Golang)** | Coordena a execução das etapas de cada pipeline com alta concorrência |
| **Workers Python** | Tarefas de IA: OCR, extração de dados, detecção de anomalias |
| **Workers Golang** | Validação de regras de negócio e processamento CPU-intensivo |
| **PostgreSQL** | Persistência multi-tenant de tarefas, resultados e configuração |
| **Observabilidade** | Métricas, logs e traces transversais a todo o sistema |

---

## 5. Fluxo de uma tarefa (exemplo: Document Processing)

```
Cliente               API (NestJS)          Fila            Orchestrator (Go)         Workers
  │                        │                  │                     │                    │
  │  POST /tasks           │                  │                     │                    │
  │  (pipeline, arquivo)   │                  │                     │                    │
  ├───────────────────────►│                  │                     │                    │
  │                        │  enfileira task   │                     │                    │
  │                        ├─────────────────►│                     │                    │
  │  task_id + PENDING     │                  │                     │                    │
  │◄───────────────────────┤                  │                     │                    │
  │                        │                  │  consome task        │                    │
  │                        │                  ├────────────────────►│                    │
  │                        │                  │                     │  1. OCR + extração  │
  │                        │                  │                     ├───────────────────►│
  │                        │                  │                     │◄───────────────────┤
  │                        │                  │                     │  2. validação regras │
  │                        │                  │                     ├───────────────────►│
  │                        │                  │                     │◄───────────────────┤
  │                        │                  │                     │  3. persistência     │
  │                        │                  │                     │  4. notificação       │
  │  status via WS/API      │                  │                     │                    │
  │  PENDING→PROCESSING→COMPLETED              │                     │                    │
  │◄────────────────────────────────────────────────────────────────┤                    │
```

---

## 6. Stack tecnológico

| Tecnologia | Papel no sistema |
|---|---|
| **NestJS** | API principal (REST/GraphQL), auth (JWT/OAuth), billing simulado (Stripe teste), notificações, orquestração de alto nível |
| **Python** | IA e automação: OCR, extração de dados, detecção de anomalias (scikit-learn, statsmodels, pyod, ou LLM via API) |
| **Golang** | Orchestrator de alta performance (muitas tarefas concorrentes) e workers de validação/processamento CPU-intensivo |
| **PostgreSQL** | Banco multi-tenant (Supabase, Neon ou AWS RDS conforme disponibilidade de créditos) |
| **Redis + BullMQ** | Filas e mensageria (local via Docker ou Upstash Redis free) |
| **Storage** | Volume Docker local, ou Cloudflare R2 / AWS S3 |
| **Prometheus + Grafana** | Observabilidade — métricas, logs estruturados por `task_id`/`pipeline_id`/`tenant_id`, traces via OpenTelemetry (opcional) |
| **Docker / Docker Compose** | Todo o sistema rodando localmente (API, workers, Redis, PostgreSQL, Prometheus, Grafana) |
| **Stripe (modo teste)** | Billing simulado — mede uso por tenant sem processar dinheiro real |

---

## 7. Conceitos e práticas a estudar e aplicar

**Arquitetura e design**
- Microsserviços (ou modular monolith bem dividido)
- DDD (Domain-Driven Design) aplicado de forma prática
- Spec-driven development (especificação antes do código)
- System design (load balancer, escalabilidade, resiliência, limites de serviço)

**Conceitos técnicos**
- Filas e processamento assíncrono
- Pipelines de processamento (dados, tarefas, workflows)
- Eventos e mensageria (event-driven architecture)
- Real-time (WebSockets, SSE)
- Observabilidade (métricas, logs, traces)
- TDD e pirâmide de testes (unitários, integração, E2E)
- Multi-tenancy: isolamento lógico por `tenant_id` (ou schema por tenant), quotas e limites por plano

---

## 8. Restrições de infraestrutura e custos

**Premissa principal:** o projeto deve ser **totalmente viável sem gastar dinheiro**. Tudo é simulado quando necessário (billing, e-mails, integrações externas) — mas as decisões de arquitetura devem ser sérias, como em um produto real.

**AWS:** ideia inicial é usar 1 ou 2 serviços (ex.: SQS, S3, EC2) para agregar valor real ao projeto ("usei SQS para filas, S3 para storage"). Limitação: US$ 100 de crédito, parcialmente usados em outro projeto de curso. Por isso, cada componente tem uma alternativa free como plano B.

| Componente | Ideal (AWS) | Alternativa free |
|---|---|---|
| Banco de dados | RDS (PostgreSQL) | Supabase / Neon |
| Filas | SQS | Redis + BullMQ (local ou Upstash) |
| Compute | EC2 / ECS | Docker local, Render, Railway |
| Storage | S3 | Cloudflare R2 / volume Docker |
| Observabilidade | CloudWatch | Prometheus + Grafana (local ou Grafana Cloud) |

Se os créditos permitirem, 1–2 serviços AWS agregam valor ao portfólio. Caso contrário, o projeto roda 100% local/free sem problema.

---

## 9. Mini fluxo de uso — persona

**Persona:** CTO de uma startup de e-commerce

**Contexto:** a startup recebe centenas de notas fiscais por mês e quer automatizar a extração de dados. Também quer monitorar custos de AWS e receber alertas de anomalias.

1. **Cadastro e configuração inicial** — CTO acessa o dashboard, cria o tenant da empresa, configura o plano (ex.: "Startup" — 1.000 tasks/mês).
2. **Criação de pipelines** — via API/dashboard, define `DOCUMENT_PROCESSING` (notas fiscais) e `BILLING_ANOMALY_DETECTION` (billing AWS).
3. **Disparo de tarefa de documento** — upload de uma nota fiscal (PDF) via API com `pipeline_id = DOCUMENT_PROCESSING`. API retorna `task_id` com status `PENDING`.
4. **Processamento assíncrono** — a task entra na fila; o orchestrator (Golang) consome e executa: OCR + extração (Python) → validação de regras (Golang) → persistência (PostgreSQL) → notificação (webhook via NestJS).
5. **Acompanhamento em tempo real** — CTO consulta o status via API/WebSocket: `PENDING` → `PROCESSING` → `COMPLETED`.
6. **Upload de billing e detecção de anomalia** — CTO envia CSV de billing da AWS; o pipeline `BILLING_ANOMALY_DETECTION` roda e o Python detecta, por exemplo, "EC2 está 180% acima da média dos últimos 14 dias". O sistema registra a anomalia, gera uma recomendação ("verifique instâncias EC2 esquecidas ligadas") e dispara alerta (e-mail simulado + webhook).
7. **Billing simulado** — Stripe (modo teste) mede o uso do tenant (ex.: 150 tasks de `DOCUMENT_PROCESSING`, 10 de `BILLING_ANOMALY_DETECTION`). No fim do mês, o CTO vê no dashboard: "Você usou 16% do plano Startup", com simulação de cobrança de US$ 29,00.

---

## 10. Próximos passos (o que ainda falta definir)

Este documento é intencionalmente de alto nível. Os próximos passos:

- **System design** — arquitetura detalhada de serviços, bancos, filas e eventos; diagramas de sequência e fluxo de dados mais aprofundados
- **Engenharia e ADRs** — decisões de arquitetura documentadas (ADRs), trade-offs discutidos (ex.: monolito modular vs. microsserviços)
- **Requisitos funcionais e não funcionais** — histórias de usuário, SLAs, SLOs, métricas de sucesso
- **Plano de implementação** — divisão em fases (MVP, v1, v2), critérios de aceite por fase

> Um documento separado — **"Plano de Desenvolvimento"** — vai detalhar por onde começar, o que configurar primeiro, boas práticas, organização do GitHub (issues, project, branches, actions) e o fluxo de trabalho simulando um time real.
