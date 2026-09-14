# FlowPipe AI — Plano de Desenvolvimento

> Documento de apoio à [Doc de Visão](vision.md). Define a ordem lógica de execução: o que configurar antes de escrever qualquer código de negócio, e quais práticas manter do início ao fim.
> Este documento **não contém código de implementação** — apenas processo, estrutura e templates. Código fica na doc de setup técnico (à parte).

---

## Visão geral das fases

| Fase | Nome | Foco |
|---|---|---|
| 0 | Fundamentos do repositório | GitHub: branches, commits, Dependabot, issues, project, PR, CI |
| 1 | Engenharia e System Design | Modelagem de dados, diagramas, contratos, ADRs iniciais |
| 2 | Estrutura de pastas | Monorepo + DDD por serviço |
| 3 | Setup inicial por tecnologia | NestJS, Python, Golang — checklist de inicialização |
| 4 | Infraestrutura e ambientes | Docker, local vs. prod, segredos |
| 5 | Documentação | README, CONTRIBUTING, ADRs, diagramas |
| 6 | Testes | Pirâmide de testes, CI gating |

---

## Fase 0 — Fundamentos do repositório

### 0.1 Estratégia de branches
- Modelo: `main` (produção/estável) + `develop` (integração) + branches de trabalho — ou trunk-based com `main` protegida + feature branches curtas.
- Convenção de nome: `feature/`, `fix/`, `chore/`, `docs/`, `refactor/`, `hotfix/` + descrição curta (ex.: `feature/document-processing-ocr`).
- Proteção da(s) branch(es) principal(is): PR obrigatório, status checks obrigatórios, revisão obrigatória (mesmo solo — simula processo real), push direto proibido.

### 0.2 Convenção de commits
- Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`, `ci:`).
- Gera changelog automático depois e obriga pensar no "tipo" de cada mudança.

### 0.3 Dependabot
- Atualização automática por ecossistema: npm (NestJS), pip (Python), gomod (Golang), GitHub Actions.
- Frequência semanal, com agrupamento de PRs para não gerar spam.

### 0.4 Issues — tipos e labels
- Templates: `Bug Report`, `Feature Request`, `Task/Chore`, `ADR Proposal` (ver seção de Templates).
- Labels: por tipo (`bug`, `feature`, `task`, `docs`, `adr`), por prioridade (`priority: high/medium/low`), por área (`area: api`, `area: worker-python`, `area: orchestrator-go`, `area: infra`).

### 0.5 GitHub Projects (board)
- Colunas: `Backlog` → `Ready` → `In Progress` → `In Review` → `Done`.
- Automação: mover card ao abrir/mergear PR linkado à issue.
- Milestones por fase do produto (MVP, v1, v2).

### 0.6 Pull Request template
- Seções: o que mudou, por quê, como testar, checklist (ver Templates).

### 0.7 GitHub Actions / Workflows (CI)
- Pipeline separado por serviço (monorepo — só roda o que mudou).
- Etapas: lint → build → testes unitários → testes de integração (quando existirem) → build de imagem Docker (futuro).
- Branch protection exigindo esses checks para merge.

### 0.8 CODEOWNERS
- Simula revisão obrigatória por área, mesmo trabalhando sozinho — bom exercício de processo de time.

---

## Fase 1 — Engenharia e System Design

Pensar o sistema no papel antes de criar qualquer pasta ou arquivo de código.

- **Modelagem de dados** — diagrama ER das entidades centrais: `Tenant`, `User`, `Plan/Subscription`, `Pipeline`, `Task`, `Anomaly`, `Alert/Recommendation`, `AuditLog`. Definir relacionamentos e o que pertence a qual tenant.
- **Diagramas de arquitetura (modelo C4)** — nível de Contexto (sistema x atores externos), nível de Containers (API, fila, orchestrator, workers, banco), e nível de Componentes onde fizer sentido.
- **Fluxogramas / diagramas de sequência** — por pipeline (Document Processing, Billing Anomaly Detection, Generic Job Runner), detalhando decisões e pontos de falha/retry.
- **Contratos entre serviços** — formato dos eventos/mensagens trafegando na fila: campos de uma task, campos de um resultado, o que caracteriza uma falha.
- **Requisitos não funcionais (mesmo simulados)** — tempo de resposta esperado, taxa de erro aceitável, throughput alvo, política de retry (tentativas, backoff).
- **Primeiras ADRs nascem aqui** — ex.: monorepo vs. multi-repo, estratégia multi-tenant no Postgres, escolha da fila.
- Ferramentas sugeridas: Excalidraw, draw.io, ou Mermaid.

---

## Fase 2 — Estrutura de pastas (monorepo + DDD)

```
flowpipe-ai/
├── apps/
│   ├── api-nestjs/
│   │   └── src/
│   │       ├── modules/
│   │       │   └── <dominio>/         # ex.: tenants, pipelines, tasks, billing
│   │       │       ├── domain/         # entidades e regras de negócio puras
│   │       │       ├── application/    # casos de uso (orquestra o domínio)
│   │       │       ├── infrastructure/ # Postgres, clients de fila/storage
│   │       │       └── interface/      # controllers REST/GraphQL, DTOs
│   │       └── shared/                 # utilitários transversais da API
│   ├── worker-python/
│   │   └── src/
│   │       ├── domain/                 # regras: o que é uma anomalia, regras de extração válidas
│   │       ├── application/            # casos de uso: "processar documento", "detectar anomalia"
│   │       ├── infrastructure/         # consumo de fila, storage, modelos de IA
│   │       └── interface/              # entrypoint do worker (listener da fila)
│   └── orchestrator-go/
│       └── internal/                   # convenção Go: pacote não exportável
│           ├── domain/                 # regras de orquestração e validação
│           ├── application/            # casos de uso: "orquestrar pipeline X"
│           ├── infrastructure/         # fila, banco, clients externos
│           └── interface/              # entrypoints (consumers, CLI)
├── packages/
│   └── contracts/                      # schemas dos eventos trafegados na fila (contrato entre serviços)
├── infra/
│   ├── docker/                         # Dockerfile de cada serviço
│   ├── docker-compose.yml              # sobe tudo junto localmente
│   └── observability/                  # configs Prometheus/Grafana
├── docs/
│   ├── vision.md                       # doc de visão
│   ├── development-plan.md             # este documento
│   ├── adr/                            # Architecture Decision Records
│   └── diagrams/                       # C4, ER, sequência
├── .github/
│   ├── ISSUE_TEMPLATE/
│   ├── workflows/
│   └── dependabot.yml
├── CONTRIBUTING.md
├── README.md
└── .env.example
```

### Responsabilidade de cada camada DDD (vale para os 3 serviços)

| Camada | Responsabilidade |
|---|---|
| `domain/` | Regras de negócio puras — entidades, value objects. Não sabe nada sobre banco, fila ou HTTP. É o "coração" do serviço. |
| `application/` | Casos de uso — orquestra entidades de domínio pra cumprir uma ação (ex.: "processar nota fiscal"). Não sabe *como* os dados são salvos, só *que* precisam ser. |
| `infrastructure/` | Implementações concretas — acesso a Postgres, fila, storage, chamadas externas. É a camada que conversa com o mundo real. |
| `interface/` | Porta de entrada — controllers HTTP, consumers de fila, CLI. Traduz uma requisição externa em uma chamada para `application/`. |

**`packages/contracts/`** — numa arquitetura poliglota, é onde fica definido (documentado/versionado) o formato dos eventos que Nest, Python e Go trocam entre si via fila, garantindo que os três "falem a mesma língua" mesmo sendo linguagens diferentes.

---

## Fase 3 — Setup inicial por tecnologia (checklist)

**NestJS (API)**
- Inicialização do projeto, linter/formatter, módulos organizados por domínio (não por tipo técnico), config de ambiente (`.env` + validação de schema), setup de auth base, testes (unit + e2e).

**Python (workers de IA)**
- Gerenciador de dependências (poetry ou pip-tools), estrutura de projeto, linter/formatter (ruff/black), testes (pytest), separação entre lógica de IA e "worker runner" (consumo de fila).

**Golang (orchestrator + workers)**
- Estrutura de módulo (`go.mod`), convenções idiomáticas de projeto Go, linter (golangci-lint), testes nativos, separação entre orchestrator e workers de validação.

---

## Fase 4 — Infraestrutura e ambientes

### 4.1 Docker
- Um `Dockerfile` por serviço + `docker-compose.yml` unificado (API, workers, Redis, Postgres, Prometheus, Grafana).
- Healthchecks por container.

### 4.2 Ambientes
- `local` — docker-compose, dados de teste, billing/e-mail sempre simulados.
- `prod` (simulado/free-tier) — Supabase/Neon, Upstash, Render/Railway. Mesmo sendo "produção de mentira", manter variáveis e segredos separados do local.

### 4.3 Configuração e segredos
- `.env.example` versionado; `.env` real nunca commitado.
- Estratégia única de variáveis entre os três serviços (prefixos por serviço: `API_`, `WORKER_PY_`, `ORCHESTRATOR_GO_`).

---

## Fase 5 — Documentação

- **README** — já feito (template próprio).
- **CONTRIBUTING.md** — como rodar local, como abrir PR, convenções de commit/branch.
- **ADRs** (`docs/adr/000X-titulo.md`) — registradas conforme as decisões forem tomadas.
- **Diagramas** — `docs/diagrams/`, alimentados a partir da Fase 1.

---

## Fase 6 — Testes desde o início

- Pirâmide definida antes de codar: unitários (maioria) → integração (fluxo entre camadas) → E2E (fluxo completo de um pipeline).
- CI exige testes passando para mergear, mesmo com poucos testes no início.
- Meta de cobertura simbólica (ex.: 70%) só para criar o hábito, sem virar obsessão.

---

## Templates

### Templates de Issue (`.github/ISSUE_TEMPLATE/`)

**1. Bug Report**
```
---
name: Bug Report
about: Reportar um comportamento inesperado ou incorreto
title: "[BUG] "
labels: bug
---

## Descrição
Descreva o problema de forma clara e objetiva.

## Passos para reproduzir
1. ...
2. ...

## Comportamento esperado
O que deveria acontecer.

## Comportamento atual
O que está acontecendo.

## Contexto adicional
Logs, prints, ambiente (local/prod), serviço afetado (api-nestjs / worker-python / orchestrator-go / infra).
```

**2. Feature Request**
```
---
name: Feature Request
about: Propor uma nova funcionalidade
title: "[FEATURE] "
labels: feature
---

## Problema que motiva essa feature
Que necessidade real essa funcionalidade resolve?

## Solução proposta
Descrição da funcionalidade.

## Alternativas consideradas
Outras formas de resolver o mesmo problema (se houver).

## Critérios de aceite
- [ ] ...
- [ ] ...

## Serviço(s) afetado(s)
api-nestjs / worker-python / orchestrator-go / infra / docs
```

**3. Task / Chore**
```
---
name: Task / Chore
about: Tarefa técnica (setup, refatoração, dependência) — não é bug nem feature
title: "[TASK] "
labels: task
---

## O que precisa ser feito
Descrição objetiva.

## Motivação
Por que essa tarefa é necessária agora.

## Definição de pronto
- [ ] ...
- [ ] ...
```

**4. ADR Proposal**
```
---
name: ADR Proposal
about: Propor uma decisão de arquitetura para discussão
title: "[ADR] "
labels: adr, discussion
---

## Contexto
Qual problema/decisão precisa ser resolvida?

## Opções consideradas
1. Opção A — prós/contras
2. Opção B — prós/contras

## Decisão proposta
Qual opção e por quê.

## Consequências
Implicações positivas e negativas para o sistema.
```

### Pull Request template (`.github/PULL_REQUEST_TEMPLATE.md`)
```
## O que mudou
Resumo objetivo.

## Por que
Contexto/motivação (Closes #<numero>).

## Como testar
Passos para validar localmente.

## Checklist
- [ ] Testes passando
- [ ] Documentação atualizada (se aplicável)
- [ ] Nenhum segredo/credencial commitado
- [ ] Branch segue a convenção de nome
```

### ADR template (`docs/adr/0001-titulo.md`)
```
# ADR 0001: <Título da decisão>

**Status:** Proposto | Aceito | Substituído por ADR-000X

## Contexto
...

## Decisão
...

## Consequências
...
```

### Convenção de commits (Conventional Commits)

| Tipo | Quando usar |
|---|---|
| `feat:` | nova funcionalidade |
| `fix:` | correção de bug |
| `docs:` | mudança só de documentação |
| `refactor:` | mudança de código sem alterar comportamento |
| `test:` | adição/ajuste de testes |
| `chore:` | tarefas de manutenção (deps, config) |
| `ci:` | mudanças em workflows/CI |

---

## 10 issues iniciais sugeridas

1. **[TASK]** Configurar branch protection rules na `main` (PR obrigatório, checks obrigatórios)
2. **[TASK]** Criar templates de issue (Bug, Feature, Task, ADR) e template de PR
3. **[TASK]** Configurar Dependabot (npm, pip, gomod, GitHub Actions)
4. **[TASK]** Criar GitHub Project (board) com colunas e automações básicas
5. **[TASK]** Configurar pipeline de CI base (lint) para os 3 serviços via GitHub Actions
6. **[ADR]** Monorepo vs. multi-repo — decisão e justificativa
7. **[ADR]** Modelagem inicial de dados — entidades centrais (`Tenant`, `Pipeline`, `Task`, `Plan`) e estratégia de isolamento multi-tenant
8. **[TASK]** Desenhar diagrama de arquitetura C4 (nível Contexto + Containers)
9. **[TASK]** Inicializar esqueleto do `api-nestjs` (estrutura DDD vazia, lint, testes configurados, sem lógica de negócio)
10. **[TASK]** Subir `docker-compose` local com Postgres + Redis (infra básica, sem lógica de negócio ainda)

---

**Status:** Plano inicial de desenvolvimento
**Próxima etapa:** Executar Fase 0 e Fase 1 (issues #1–#8 da lista acima)
