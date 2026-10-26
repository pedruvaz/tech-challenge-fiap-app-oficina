# 16 · ADRs — Decisões arquiteturais

> [← Voltar ao índice](../README.md)

Registro das decisões arquiteturais **já tomadas** na Fase 3. Cada ADR descreve o que está no código dos quatro repositórios da entrega (`app-oficina`, `infra-k8s`, `infra-db`, `lambda-auth`), com contexto, decisão, consequências positivas e negativas e as fontes verificáveis. Onde o código não registra o motivo de uma escolha, o ADR diz isso em vez de reconstruir uma justificativa.

Formato: Status · Contexto · Decisão · Consequências · Referências.

| # | ADR | Status | Resumo |
| - | --- | ------ | ------ |
| 001 | [Quatro repositórios, um pipeline por repositório](./adr-001-quatro-repositorios.md) | Aceito (2026-10-19) | `infra-k8s` → `infra-db` → `lambda-auth` → `app-oficina`; o SG da Lambda nasce no `infra-db` para evitar dependência circular; a ordem não é imposta por nenhum workflow |
| 002 | [Comunicação entre repositórios](./adr-002-comunicacao-entre-repositorios.md) | Aceito (2026-10-19) | `terraform_remote_state` entre stacks Terraform; repository variables/secrets do GitHub, preenchidas à mão, do Terraform para o `app-oficina` |
| 003 | [Escalabilidade com HPA](./adr-003-escalabilidade-com-hpa.md) | Aceito (2026-10-19) | HPA por CPU (70%, 2–6 réplicas) sobre metrics-server instalado pelo `infra-k8s`; nós por conta do EKS Auto Mode; `replicas` fixo no Deployment conflita com o HPA a cada apply |

## Repositórios citados

| Repositório | Papel |
| ----------- | ----- |
| [`infra-k8s`](https://github.com/pedruvaz/tech-challenge-fiap-infra-k8s) | VPC, EKS Auto Mode, metrics-server, ECR, OIDC e roles do GitHub Actions (stacks `base/` e `cluster/`) |
| [`infra-db`](https://github.com/pedruvaz/tech-challenge-fiap-infra-db) | RDS PostgreSQL, security groups e segredo no Secrets Manager |
| [`lambda-auth`](https://github.com/pedruvaz/tech-challenge-fiap-lambda-auth) | API Gateway, Lambda de login por CPF e authorizer JWT |
| [`app-oficina`](https://github.com/pedruvaz/tech-challenge-fiap-app-oficina) | API NestJS, manifestos `k8s/` e workflow de deploy no EKS |
