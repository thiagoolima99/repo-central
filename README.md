# repo-central — Centralized Security Pipelines

Repositório central de workflows reutilizáveis de segurança para pipelines DevSecOps.

## Arquitetura
repo-central/
└── .github/workflows/
└── reusable-nodejs-security.yml   # Workflow reutilizável para Node.js/TypeScript
Cada repositório de aplicação chama o workflow central via `workflow_call`, mantendo toda a lógica de segurança em um único lugar.

## Ferramentas utilizadas

| Ferramenta | Tipo | Comportamento |
|---|---|---|
| Gitleaks | Secret Scanning | Bloqueante |
| Snyk SCA | Análise de dependências | Bloqueante em HIGH/CRITICAL |
| Snyk Code | SAST — análise estática | Bloqueante em HIGH/CRITICAL |

## Repositórios cobertos

| Repositório | Tecnologia | Workflow |
|---|---|---|
| [juice-shop](https://github.com/thiagoolima99/juice-shop) | Node.js/TypeScript | reusable-nodejs-security.yml |

## Proteção de branches

Cada repositório possui as seguintes regras no branch principal (`master`/`main`):

- **Require status checks to pass** — o job `Security Gate` precisa passar antes do merge
- **Require approvals** — em produção, exige aprovação do time de segurança antes do merge

Em uma GitHub Organization, o "Require approvals" pode ser restrito ao time de segurança via:
- **Teams** — cria um time `security-team` e configura em "Require review from specific teams"
- **CODEOWNERS** — arquivo `.github/CODEOWNERS` com `* @org/security-team` + "Require review from Code Owners"

## Bypass de segurança

Em situações excepcionais, o pipeline pode ser bypassado de duas formas:

1. **Label no PR** — aplica a label `security-bypass` no PR
2. **Mensagem no commit** — inclui `[skip security]` ou `[security bypass]` na mensagem

### Controle de acesso ao bypass

| Ambiente | Quem pode bypassar |
|---|---|
| Repositório pessoal | Owner do repositório |
| GitHub Organization | Admins e Maintainers (via permissão de labels) |
| Com CODEOWNERS | Apenas membros do `security-team` |

> ⚠️ O bypass deve ser usado apenas em situações excepcionais e auditadas. Todo bypass fica registrado no histórico de Actions do GitHub.

## Como funciona o Security Gate

O job `Security Gate` é o ponto de controle central:

- Se **bypass ativo** → gate passa, merge liberado
- Se **vulnerabilidades encontradas** → gate falha, merge bloqueado
- Se **todos os scans passam** → gate passa, merge liberado

## Como adicionar um novo repositório

1. Crie o arquivo `.github/workflows/security.yml` no repositório:

```yaml
name: Security CI

on:
  push:
    branches: [main, master, develop]
  pull_request:
    branches: [main, master, develop]

jobs:
  security:
    name: Node.js Security
    uses: thiagoolima99/repo-central/.github/workflows/reusable-nodejs-security.yml@main
    with:
      fail-on-high: true
    secrets: inherit
    permissions:
      contents: read
```

2. Adicione o secret `SNYK_TOKEN` em `Settings → Secrets → Actions`
3. Configure a Branch Protection Rule apontando o `Security Gate` como status check obrigatório
4. Crie a label `security-bypass` em `Settings → Labels`
