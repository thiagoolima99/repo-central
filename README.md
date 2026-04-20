# repo-central — Centralized Security Pipelines

Repositório central de workflows reutilizáveis de segurança para pipelines DevSecOps.

## Arquitetura

```
repo-central/
└── .github/workflows/
    ├── reusable-nodejs-security.yml     # Node.js/TypeScript
    ├── reusable-python-security.yml     # Python/Flask
    ├── reusable-terraform-security.yml  # Terraform/IaC
    └── reusable-android-security.yml    # Java/Android
```

Cada repositório de aplicação chama o workflow central via `workflow_call`, mantendo toda a lógica de segurança em um único lugar. Para adicionar um novo repositório, basta criar um `security.yml` de 10 linhas apontando para o workflow correspondente.

---

## Repositórios cobertos

| Repositório | Tecnologia | Workflow |
|---|---|---|
| [juice-shop](https://github.com/thiagoolima99/juice-shop) | Node.js/TypeScript | reusable-nodejs-security.yml |
| [VAmPI](https://github.com/thiagoolima99/VAmPI) | Python/Flask | reusable-python-security.yml |
| [terragoat](https://github.com/thiagoolima99/terragoat) | Terraform | reusable-terraform-security.yml |
| [diva-android](https://github.com/thiagoolima99/diva-android) | Java/Android | reusable-android-security.yml |

---

## Ferramentas utilizadas

| Ferramenta | Tipo | Repositórios | Comportamento |
|---|---|---|---|
| Gitleaks | Secret Scanning | Todos | Bloqueante |
| Snyk SCA | Análise de dependências | Juice Shop, VAmPI | Bloqueante em HIGH/CRITICAL |
| Snyk Code | SAST | Juice Shop, VAmPI, diva-android | Bloqueante em HIGH/CRITICAL |
| Snyk IaC | IaC Misconfiguration | Terragoat | Bloqueante em HIGH/CRITICAL |

---

## Decisões técnicas

### Por que Snyk ao invés de ferramentas separadas?

Optamos por centralizar em Snyk (SCA + Code + IaC) para simplificar a stack e reduzir o número de tokens e integrações necessárias. Abaixo os comparativos realizados durante a implementação.

### SAST Python — Snyk Code vs Bandit

O Bandit é uma ferramenta consolidada para análise estática de Python, especializada em padrões inseguros como uso de `eval`, `subprocess`, `pickle` e configurações Flask inseguras. Durante o teste no VAmPI, o Bandit encontrou poucos findings relevantes — o mesmo resultado do Snyk Code, com configuração mais complexa. Decisão: **Snyk Code**, por menor fricção operacional e token já disponível.

### IaC — Snyk vs Checkov

O Checkov verificou o Terragoat com **358 falhas** contra **16 do Snyk IaC**. O Checkov cobre centenas de regras de CIS Benchmarks, NIST e PCI-DSS, enquanto o Snyk foca nas misconfigurations mais críticas e exploráveis. Bloquear o pipeline por 358 issues causaria ruído excessivo e incentivaria o uso do bypass. Decisão: **Snyk IaC no pipeline de PR** para bloqueio cirúrgico; **Checkov recomendado para auditorias periódicas** agendadas fora do PR.

### Secret Scanning — Gitleaks vs TruffleHog

O TruffleHog vai além do Gitleaks ao validar ativamente se o secret encontrado ainda está em uso (ex: tenta uma chamada na AWS para confirmar se a chave é válida), reduzindo falsos positivos. O Gitleaks é mais simples, rápido e possui GitHub Action oficial pronta. Decisão: **Gitleaks**, suficiente para o contexto do desafio e mais fácil de manter.

---

## Resultados dos scans

### Juice Shop (Node.js)
- **Gitleaks**: 0 findings — nenhum secret exposto
- **Snyk SCA**: 40 vulnerabilidades (incluindo RCE em vm2, Authentication Bypass em jsonwebtoken, Sandbox Bypass) — pipeline bloqueado
- **Snyk Code**: 25 findings HIGH (SQL Injection, NoSQL Injection, XSS, SSRF, Path Traversal, Hardcoded Secrets) — pipeline bloqueado

### VAmPI (Python/Flask)
- **Gitleaks**: 0 findings
- **Snyk SCA**: vulnerabilidades encontradas — pipeline bloqueado
- **Snyk Code**: findings encontrados — pipeline bloqueado
- **Observação**: vulnerabilidades de lógica (IDOR, broken access control) não são detectáveis por SAST — requer DAST

### Terragoat (Terraform)
- **Gitleaks**: 0 findings
- **Snyk IaC**: 16 misconfigurations HIGH/CRITICAL — pipeline bloqueado
- **Checkov (comparativo local)**: 358 falhas no total

### diva-android (Java/Android)
- **Gitleaks**: 0 findings
- **Snyk Code**: 0 findings HIGH/CRITICAL — pipeline passou
- **Snyk SCA**: não suportado (ver limitações abaixo)

---

## Limitações identificadas

### diva-android — SCA não suportado
O projeto usa Gradle 2.4 (2015), incompatível com o Snyk SCA. O Gradle desatualizado é em si um risco de segurança — versões antigas não recebem patches e podem conter vulnerabilidades conhecidas. Atualizar exigiria refatoração do projeto.

### VAmPI e diva-android — vulnerabilidades de lógica
Ferramentas de SAST não detectam vulnerabilidades comportamentais como IDOR, broken access control e armazenamento inseguro. Para esses casos, recomenda-se:
- **OWASP ZAP** para APIs REST (DAST)
- **MobSF** para apps Android (análise estática mobile-específica)

### Gitleaks — 0 findings em todos os repositórios
Esperado — os projetos são open source e não contêm secrets reais. Em repositórios corporativos, o Gitleaks teria maior impacto detectando tokens e credenciais commitados acidentalmente.

---

## Como funciona o Security Gate

Cada workflow possui um job `Security Gate` que é o ponto de controle central do pipeline:

- Se **bypass ativo** → gate passa, merge liberado
- Se **vulnerabilidades encontradas** → gate falha, merge bloqueado
- Se **todos os scans passam** → gate passa, merge liberado

O Security Gate está configurado como **status check obrigatório** nos rulesets de todos os repositórios, impedindo o merge sem sua aprovação.

---

## Proteção de branches

Cada repositório possui ruleset configurado no branch principal (`master`) com:

- **Require status checks to pass** — o `Security Gate` precisa passar antes do merge
- **Block force pushes** — impede sobrescrever o histórico

Em produção, recomenda-se adicionar:
- **Require approvals** — mínimo 1 aprovação do time de segurança antes do merge

---

## Bypass de segurança

Em situações excepcionais, o pipeline pode ser bypassado de duas formas:

1. **Label no PR** — aplica a label `security-bypass` no PR antes de abrir ou via re-run
2. **Mensagem no commit** — inclui `[skip security]` ou `[security bypass]` na mensagem

A detecção da label usa a **API do GitHub em tempo real** (não o payload do evento), garantindo que labels adicionadas após o início do pipeline sejam detectadas corretamente no re-run.

A label `security-bypass` foi criada em todos os repositórios com cor vermelha para indicar uso restrito.

### Controle de acesso ao bypass

| Ambiente | Quem pode bypassar |
|---|---|
| Repositório pessoal | Owner do repositório |
| GitHub Organization | Admins e Maintainers (via permissão de labels) |
| Com Teams | Apenas membros do time definido em "Require review from specific teams" |
| Com CODEOWNERS | Apenas membros definidos em `.github/CODEOWNERS` |

> ⚠️ Todo bypass fica registrado no histórico de Actions do GitHub, garantindo rastreabilidade.

---

## Como escalar para novos repositórios

### Opção 1 — GitHub CLI (recomendado para poucos repos)
```bash
gh api repos/thiagoolima99/NOVO-REPO/contents/.github/workflows/security.yml \
  --method PUT \
  --field message="feat: add security pipeline" \
  --field content="$(base64 < security-template.yml)"
```

### Opção 2 — Script bash (múltiplos repos)
Cria um script que itera sobre uma lista de repositórios e aplica o `security.yml` via GitHub CLI automaticamente.

### Opção 3 — GitHub Actions no repo-central
Um workflow que detecta novos repositórios na organização e abre PRs automaticamente com o `security.yml`.

### Opção 4 — Terraform + GitHub Provider
Define branch protection rules, secrets e workflows de todos os repos como infraestrutura como código e aplica com `terraform apply`.

---

## Como adicionar um novo repositório manualmente

1. Cria `.github/workflows/security.yml` no repositório:

```yaml
name: Security CI

on:
  push:
    branches: [main, master, develop]
  pull_request:
    branches: [main, master, develop]

jobs:
  security:
    name: <Tecnologia> Security
    uses: thiagoolima99/repo-central/.github/workflows/reusable-<tecnologia>-security.yml@main
    with:
      fail-on-high: true
    secrets: inherit
    permissions:
      contents: read
```

2. Adiciona o secret `SNYK_TOKEN` em `Settings → Secrets → Actions`
3. Configura o ruleset em `Settings → Rules`:
   - Target branch: `master` ou `main`
   - Ativa "Require status checks to pass"
   - Adiciona `Security Gate` como check obrigatório
4. Cria a label `security-bypass` em `Settings → Labels` (cor vermelha, acesso restrito)
