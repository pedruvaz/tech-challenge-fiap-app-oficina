# ADR-001 · Quatro repositórios, um pipeline por repositório

> [← Voltar aos ADRs](./README.md)

## Status

Aceito, 2026-10-19.

## Contexto

Até a Fase 2 o projeto era um repositório único: a API, os manifestos Kubernetes e o Terraform conviviam em `infra/terraform/` (a nota no topo de [`docs/README.md`](../README.md) registra a mudança). A Fase 3 acrescenta peças com ciclos de vida e permissões diferentes:

- **Rede, cluster e identidade do CI** (VPC, EKS Auto Mode, ECR, provider OIDC do GitHub e roles). A stack `base/` é permanente (~US$ 0/mês); a `cluster/` sobe e desce a cada demonstração (~US$ 130–160/mês ligada).
- **Banco e segredo** (RDS PostgreSQL 16, security groups, segredo `tech-challenge-fiap/api` no Secrets Manager). Também efêmero: o RDS vive nas subnets da VPC do cluster e precisa ser destruído antes dela.
- **Autenticação serverless** (API Gateway, Lambda `auth` e authorizer), publicada com `terraform apply` e um bundle Node.
- **A API** (NestJS no EKS), implantada por `kubectl` e não por Terraform.

Os quatro componentes dependem uns dos outros em uma direção só no grafo de infraestrutura: o banco precisa da VPC e do security group do cluster; a Lambda precisa das subnets, do security group e do segredo; a API precisa do cluster, do ECR e do segredo.

## Decisão

Um repositório por componente, cada um com o próprio pipeline do GitHub Actions, aplicados nesta ordem:

```
infra-k8s  ->  infra-db  ->  lambda-auth  ->  app-oficina
```

| Repositório | Conteúdo | Pipeline | Credencial AWS |
| ----------- | -------- | -------- | -------------- |
| `infra-k8s` | `terraform/base/` (ECR, OIDC, 2 roles, bucket do frontend) e `terraform/cluster/` (VPC, EKS, metrics-server) | `terraform.yml`: `fmt`/`validate` sem credencial; `plan` no PR; `apply` de `base` e depois `cluster`, em sequência, no push em `main` | role `…-github-terraform` |
| `infra-db` | `terraform/` (RDS, SGs, segredo) | `terraform.yml`: mesmo desenho, uma stack | role `…-github-terraform` |
| `lambda-auth` | `src/` (handlers) e `terraform/` (API Gateway, funções) | `ci.yml`: typecheck, testes com Postgres real, bundle, `plan` (PR) ou `apply` (`main`) | role `…-github-terraform` |
| `app-oficina` | `src/`, `k8s/` | `ci.yml` (build, lint, testes, kind, push no ECR) e `deploy.yml` (disparado por `workflow_run` do CI) | role `…-github-actions` |

Pontos que fazem parte da decisão:

- **Cada stack Terraform tem o próprio state** no mesmo bucket S3, com chave distinta (`base/terraform.tfstate`, `cluster/terraform.tfstate`, `db/terraform.tfstate` e `lambda-auth/terraform.tfstate`) e lock nativo (`use_lockfile`). Dentro do `infra-k8s` as duas stacks (`base` e `cluster`) são separadas por ciclo de vida, não por repositório. O mecanismo pelo qual uma stack lê a outra está no [ADR-002](./adr-002-comunicacao-entre-repositorios.md).
- **Duas roles, de propósito.** `…-github-terraform` (PowerUserAccess + IAM) só é assumível pelos três repositórios de infra; `…-github-actions` (push no ECR, `eks:DescribeCluster`, leitura do segredo da API, sync no S3) é assumida pelos quatro repositórios e pelo frontend. O pipeline do `app-oficina` nunca recebe permissão para criar ou destruir infraestrutura. O trust usa OIDC, sem access key em secret.
- **O security group da Lambda nasce no `infra-db`, não no `lambda-auth`.** O ingress 5432 do RDS precisa referenciar o SG da Lambda, e um SG só pode ser referenciado depois de existir. Se ele nascesse no `lambda-auth`, haveria dependência circular entre as duas stacks (Lambda esperando o RDS, RDS esperando o SG da Lambda). Criado no `infra-db` e exportado como `lambda_security_group_id`, o grafo fica em uma direção: `infra-db` cria e exporta, `lambda-auth` apenas consome. O README do `infra-db` documenta o mesmo raciocínio, e a chave JWT compartilhada pela Lambda e pelo authorizer mora no mesmo repositório pelo mesmo motivo de direção única (é onde o segredo é composto).
- **Sem ambiente de homologação duplicado.** Homologação e produção se separam por namespace do Kubernetes dentro do mesmo cluster (`tech-challenge` e `tech-challenge-homolog`, via `sed` no `deploy.yml`), não por uma segunda cópia da infraestrutura: os nomes de role, ECR e cluster derivam de `project_name` fixo e colidiriam, e o custo dobraria.
- **Aprovação antes do `apply`.** Os jobs `apply` dos repositórios de infra declaram `environment: prod`, o que permite exigir aprovação manual em Settings > Environments sem alterar o workflow.

## Consequências

### Positivas

- **Permissões separadas por repositório.** Quem desenvolve a API não tem, pelo pipeline, poder de destruir o cluster ou o banco. A role ampla fica restrita a três repositórios e à proteção da `main`.
- **Blast radius e state por componente.** Um `apply` no `lambda-auth` não planeja nem toca VPC, EKS ou RDS. O lock é por state, e cada repositório tem `concurrency` próprio (`terraform-infra-k8s`, `terraform-infra-db`), então applies de repositórios diferentes não se bloqueiam.
- **Ciclos de vida diferentes ficam separáveis.** O que fica de pé (`base`) é desacoplado do que sobe e desce (`cluster`, RDS), e o `destroy` segue a ordem inversa documentada nos READMEs (RDS antes da VPC).
- **Grafo de dependência acíclico na infraestrutura.** A decisão do SG da Lambda elimina o único ciclo que o desenho teria entre `infra-db` e `lambda-auth`.
- **Falha rápida quando a ordem é violada.** Aplicar uma stack sem o state da anterior falha no plan com "Unable to find remote state"; os workflows tratam isso como esperado (`continue-on-error` no `plan`) e documentam que é ordem de execução, não bug.
- **Pipeline de aplicação independente da infra.** O `deploy.yml` só faz `kubectl` e `aws` (Secrets Manager, ECR); não roda Terraform.

### Negativas

- **A ordem é acoplamento operacional e nenhum workflow a impõe.** Verificado nos quatro repositórios: o único `workflow_run` existente é o do `app-oficina` (`Deploy` depois de `CI`, no mesmo repositório); não há `repository_dispatch`, `workflow_call` entre repositórios nem qualquer gatilho de um pipeline de infra a outro. A ordem `infra-k8s` → `infra-db` → `lambda-auth` → `app-oficina` é seguida por convenção (README de cada repositório) e protegida apenas pelo efeito colateral da ausência de state.
- **A proteção por ausência de state não cobre state desatualizado.** Como o `cluster` e o RDS são efêmeros e recriados, um `apply` no `infra-k8s` que troque VPC, subnets ou security group não dispara o `infra-db` nem o `lambda-auth`; o `plan` seguinte só pega a divergência se alguém o rodar. O `infra-k8s` apenas imprime os outputs no summary do job para o operador levar adiante.
- **A cadeia não é um DAG puro de ponta a ponta.** O API Gateway do `lambda-auth` precisa do hostname do NLB que o Service `api` do `app-oficina` cria, e esse valor entra por `-var api_nlb_hostname=…` à mão. Com o default (vazio), nenhuma rota da API é criada, e o `ci.yml` do `lambda-auth` não passa a variável: como as rotas da API saem de um `for_each` sobre mapa vazio, um `apply` automático em `main` planeja a remoção das rotas criadas antes à mão com o hostname. O `lambda-auth` precede o `app-oficina` na ordem de deploy, mas depende dele para ficar completo.
- **Handoff manual entre o Terraform e o GitHub.** Os secrets e variables de cada repositório são preenchidos à mão a partir de outputs (passo 5 de [`infra-k8s/docs/00-setup-inicial-aws.md`](https://github.com/pedruvaz/tech-challenge-fiap-infra-k8s)). Verificado em 2026-10-19 com `gh variable list` e `gh secret list`: **nenhum dos quatro repositórios tem variable ou secret configurado**. Por isso os jobs que assumem a role AWS (`plan`, `apply`, push no ECR, deploy) falham em `aws-actions/configure-aws-credentials` com "Input required and not supplied: aws-region". Os jobs `validate`, que rodam sem credencial, não são afetados. Detalhes e a tabela completa no [ADR-002](./adr-002-comunicacao-entre-repositorios.md).
- **Bootstrap fora do pipeline.** O bucket de state (etapa T0) e o primeiro `apply` da `base` são feitos à mão com credencial de pessoa: a role que o pipeline assume só passa a existir depois desse `apply`.
- **Contrato entre repositórios sem verificação automática.** Os `outputs.tf` são tratados como API pública por convenção (comentário no `cluster/outputs.tf` e README do `infra-k8s`). Renomear um output quebra o `apply` do consumidor, e isso só aparece quando o consumidor roda. Não há teste de contrato nem versionamento dos outputs.
- **Repetição entre repositórios.** O nome do bucket de state aparece literal no `backend` e em cada `terraform_remote_state` (blocos `backend` não aceitam variável); a versão do Terraform (`1.13.3`) é repetida em três workflows; a configuração de OIDC/região é repetida em cada repositório e precisa ser feita quatro vezes. Mudanças transversais viram quatro PRs.
- **O README do `app-oficina` descreve a cadeia de forma simplificada demais.** Ele afirma que cada repositório "lê os outputs do anterior via `terraform_remote_state`"; isso vale entre stacks Terraform, mas não para o `app-oficina`, que não usa remote state (ver ADR-002).

## Referências

- `infra-k8s/README.md` (propósito, ordem, duas roles, formato do `sub` OIDC, "por que não há homologação aqui") e `infra-k8s/docs/00-setup-inicial-aws.md`.
- `infra-db/README.md`, seção "O SG da Lambda nasce aqui, não no repositório da Lambda"; `infra-db/terraform/rds.tf` (`aws_security_group.lambda_auth`, `rds_from_lambda`).
- `infra-k8s/.github/workflows/terraform.yml`, `infra-db/.github/workflows/terraform.yml`, `lambda-auth/.github/workflows/ci.yml`, `app-oficina/.github/workflows/{ci,deploy}.yml`.
- `infra-k8s/terraform/base/iam-github-oidc.tf` e `iam-github-terraform.tf` (trust e policies das duas roles).
- `lambda-auth/terraform/api_gateway.tf` e `variables.tf` (`api_nlb_hostname`).
- [ADR-002](./adr-002-comunicacao-entre-repositorios.md) · [`docs/01-arquitetura.md`](../01-arquitetura.md).
