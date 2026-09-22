# Contribuindo com o FlowPipe AI

Projeto solo, mas o fluxo simula um time real — ver [docs/development-plan.md](docs/development-plan.md) pra entender o porquê de cada prática.

## Fluxo de branches

- `main` é protegida: sem push direto, todo mudança entra via Pull Request.
- Branches curtas, nomeadas por tipo: `feature/`, `fix/`, `chore/`, `docs/`, `refactor/`, `hotfix/` + descrição curta.
  Ex.: `feature/document-processing-ocr`, `chore/repo-bootstrap`.

## Convenção de commits

[Conventional Commits](https://www.conventionalcommits.org/):

| Tipo | Quando usar |
|---|---|
| `feat:` | nova funcionalidade |
| `fix:` | correção de bug |
| `docs:` | mudança só de documentação |
| `refactor:` | mudança de código sem alterar comportamento |
| `test:` | adição/ajuste de testes |
| `chore:` | tarefas de manutenção (deps, config) |
| `ci:` | mudanças em workflows/CI |

## Abrindo um Pull Request

1. Abra a branch a partir da `main` atualizada.
2. Siga o [template de PR](.github/PULL_REQUEST_TEMPLATE.md) — o que mudou, por quê, como testar, checklist.
3. Linke a issue relacionada (`Closes #<numero>`).
4. Espere os checks de CI passarem antes de pedir merge.

## Rodando localmente

Ainda não disponível — chega na **Fase 4 (Infraestrutura e ambientes)** do [Plano de Desenvolvimento](docs/development-plan.md#fase-4--infraestrutura-e-ambientes), via `docker-compose`.

## Abrindo uma issue

Use o template certo — [Bug Report, Feature Request, Task/Chore ou ADR Proposal](.github/ISSUE_TEMPLATE/) — e marque a área afetada (`area: api`, `area: worker-python`, `area: orchestrator-go`, `area: infra`).
