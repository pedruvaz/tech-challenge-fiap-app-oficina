# Contrato entre os repositórios da Fase 3

A Fase 3 quebrou o projeto em quatro repositórios, cada um com pipeline próprio
e ciclo de vida independente. Independência de deploy só funciona com uma
fronteira explícita: este documento é essa fronteira. Ele define o que cada
repositório publica, o que cada um consome e por qual mecanismo — e mudar
qualquer nome listado aqui é *breaking change* para quem está a jusante.

## 1. Os quatro repositórios

| Repositório | Papel |
|---|---|
| [infra-k8s](https://github.com/pedruvaz/tech-challenge-fiap-infra-k8s) | VPC, cluster EKS, ECR, OIDC do GitHub e as roles IAM |
| [infra-db](https://github.com/pedruvaz/tech-challenge-fiap-infra-db) | RDS PostgreSQL gerenciado, Secrets Manager e os security groups |
| [lambda-auth](https://github.com/pedruvaz/tech-challenge-fiap-lambda-auth) | autenticação por CPF, emissão do JWT, API Gateway e authorizer |
| [app-oficina](https://github.com/pedruvaz/tech-challenge-fiap-app-oficina) | a API da oficina rodando no cluster |

### Ordem de deploy

```
infra-k8s  ──►  infra-db  ──►  lambda-auth  ──►  app-oficina
```

A ordem não é convenção: é dependência de dados. Cada etapa lê o state da
anterior, e aplicar fora de ordem falha no `plan`, não em runtime.

**Nenhum workflow impõe essa ordem automaticamente.** Não há `workflow_run`
encadeando os repositórios — a sequência é responsabilidade de quem dispara os
pipelines.

## 2. Dois mecanismos de comunicação, não um

Esta é a parte do contrato que mais confunde, porque as fronteiras não são
todas do mesmo tipo.

### 2.1 Entre stacks Terraform: `terraform_remote_state`

`infra-k8s`, `infra-db` e `lambda-auth` conversam lendo o state S3 da stack
anterior. É uma dependência versionada, resolvida no `plan`: se um output
sumir, o `plan` quebra na hora.

### 2.2 Do Terraform para o app: *repository variables*

O `app-oficina` **não** lê state de ninguém. O `deploy.yml` consome variáveis e
segredos do próprio repositório no GitHub, preenchidos **à mão** a partir dos
outputs do Terraform.

Isso é um passo manual e não versionado, e é a fraqueza conhecida do contrato:
não existe nada que detecte divergência entre o que o Terraform produziu e o
que está gravado nas variables. Quando divergem, a falha aparece no deploy, não
no `plan`.

## 3. Outputs publicados

### 3.1 `infra-k8s`, stack `terraform/base`

| Output | Consumido por |
|---|---|
| `github_terraform_role_arn` | secret `AWS_TERRAFORM_ROLE_ARN` dos três repos de infra |
| `github_actions_role_arn` | secret `AWS_DEPLOY_ROLE_ARN` do `app-oficina` |
| `ecr_repository_url` · `ecr_repository_arn` | `app-oficina` (push da imagem) |
| `oidc_provider_arn` · `aws_account_id` | diagnóstico e composição de policies |
| `frontend_bucket_name` · `frontend_website_endpoint` | front-end estático |
| `github_actions_role_name` | diagnóstico |

### 3.2 `infra-k8s`, stack `terraform/cluster`

| Output | Consumido por |
|---|---|
| `vpc_id` | `infra-db` |
| `private_subnet_ids` | `infra-db`, `lambda-auth` |
| `cluster_primary_security_group_id` | `infra-db` (ingress do RDS) |
| `cluster_name` | variable `EKS_CLUSTER_NAME` do `app-oficina` |
| `node_security_group_id` · `vpc_cidr_block` | regras de rede |
| `cluster_endpoint` · `configure_kubectl` | acesso ao cluster |
| `ecr_repository_url` · `aws_region` · `project_name` | conveniência |

### 3.3 `infra-db`

| Output | Consumido por |
|---|---|
| `lambda_security_group_id` | `lambda-auth` (VPC config da função) |
| `api_secret_name` | `lambda-auth` e variable `API_SECRET_NAME` do `app-oficina` |
| `api_secret_arn` | `lambda-auth` (policy de leitura do segredo) |
| `db_security_group_id` | regras de rede |
| `db_endpoint` · `db_name` | diagnóstico (`db_endpoint` é `sensitive`) |
| `create_k8s_secret_command` | conveniência operacional |

**O security group da Lambda nasce aqui, não no `lambda-auth`.** O ingress do
RDS precisa referenciar esse SG, e um security group não pode ser referenciado
antes de existir. Se ele nascesse no repositório da Lambda, haveria ciclo: a
Lambda esperando o RDS e o RDS esperando o SG da Lambda. Criando-o no
`infra-db`, o grafo fica numa direção só.

### 3.4 `lambda-auth`

| Output | Consumido por |
|---|---|
| `api_endpoint` · `api_id` | URL pública da API; registro de rotas |
| `authorizer_id` · `authorizer_function_name` | proteção das rotas |
| `lambda_function_name` · `lambda_function_arn` · `lambda_invoke_arn` | integração do gateway |
| `lambda_role_arn` · `log_group_name` | IAM e observabilidade |

## 4. O segredo compartilhado

O `infra-db` compõe **um único** segredo no Secrets Manager,
`tech-challenge-fiap/api`, com três chaves:

| Chave | Quem usa |
|---|---|
| `DATABASE_URL` | `app-oficina` (Prisma) |
| `JWT_ACCESS_SECRET` | `lambda-auth` assina · `app-oficina` e o authorizer validam |
| `JWT_REFRESH_SECRET` | renovação de token |

As chaves JWT moram junto da `DATABASE_URL` por um motivo estrutural: a Lambda
**assina** o token e o authorizer do API Gateway o **valida**. Se cada lado
gerasse a própria chave, todo token emitido seria recusado na porta de entrada.
Uma chave só, lida pelos dois.

A `DATABASE_URL` usa `sslmode=require&uselibpqcompat=true`. O módulo
`terraform-aws-modules/rds` cria um parameter group na family `postgres16`,
mas **sem declarar nenhum parâmetro customizado** — ou seja, `rds.force_ssl`
fica no default do engine, que para PostgreSQL ≥ 15 já é `1`. TLS é
obrigatório sem que ninguém precise configurar.

O `uselibpqcompat=true` existe porque no driver `pg` do Node o `require`
sozinho se comporta como `verify-full` e derruba a conexão por falta da CA do
RDS no trust store. O efeito prático é **conexão cifrada sem verificação da
identidade do servidor**: protege contra escuta passiva, não contra um
intermediário ativo dentro da VPC.

## 5. Configuração obrigatória por repositório

Os pipelines autenticam por OIDC; nenhum usa chave de acesso estática. Cada
repositório precisa de:

| Repositório | Secrets | Variables |
|---|---|---|
| `infra-k8s` | `AWS_TERRAFORM_ROLE_ARN` | `AWS_REGION` |
| `infra-db` | `AWS_TERRAFORM_ROLE_ARN` | `AWS_REGION` |
| `lambda-auth` | `AWS_TERRAFORM_ROLE_ARN` | `AWS_REGION` |
| `app-oficina` | `AWS_DEPLOY_ROLE_ARN` | `AWS_REGION`, `EKS_CLUSTER_NAME`, `API_SECRET_NAME`, `ECR_REPOSITORY`, `FRONTEND_URL` |

> **Estado em 19/10/2026: nenhuma dessas variables ou secrets está configurada
> em nenhum dos quatro repositórios.** Essa é a causa única das falhas de CI
> nos três repositórios de infraestrutura, que morrem em
> `Input required and not supplied: aws-region` antes mesmo de tentar assumir a
> role. Enquanto isso não for preenchido, nenhum pipeline tem como ficar verde.

## 6. Regras de mudança

1. **Renomear output é breaking change.** Quem publica avisa quem consome, e as
   duas mudanças entram juntas.
2. **Acrescentar output é seguro.** Remover, nunca — deprecie antes.
3. **Mudar o formato do segredo** exige coordenação tripla: `infra-db` compõe,
   `lambda-auth` assina e o `app-oficina` consome.
4. **Variables do `app-oficina` seguem os outputs.** Um `terraform apply` que
   mude `cluster_name`, `api_secret_name` ou `ecr_repository_url` obriga a
   atualizar a variable correspondente à mão, no mesmo momento.

## 7. Documentos relacionados

- [RFCs](15-rfcs/) — decisões técnicas: nuvem, banco e autenticação.
- [ADRs](16-adrs/) — decisões arquiteturais: os quatro repositórios, a
  comunicação entre eles e a escalabilidade com HPA.
