# ADR-002 · Comunicação entre repositórios

> [← Voltar aos ADRs](./README.md)

## Status

Aceito, 2026-10-19.

## Contexto

Com a divisão em quatro repositórios ([ADR-001](./adr-001-quatro-repositorios.md)), valores criados por um repositório precisam chegar a outro: IDs de VPC e subnets, o security group do cluster, o ARN do segredo, o nome do cluster, a URL do ECR. Existem dois tipos de consumidor, e eles não têm o mesmo meio de leitura:

- **Stacks Terraform** (`infra-k8s`, `infra-db`, `lambda-auth`), que podem declarar um data source e ler o state de outra stack.
- **O `app-oficina`**, cujo pipeline não roda Terraform: o `deploy.yml` roda `aws` e `kubectl`.

A decisão tem, portanto, **dois mecanismos diferentes**, e a distinção entre eles é o ponto central deste ADR.

## Decisão

### Mecanismo 1: entre stacks Terraform, `terraform_remote_state`

Cada stack publica `outputs` e a seguinte os lê com `data "terraform_remote_state"` apontando para a chave do state da anterior no mesmo bucket S3 (`tf-state-tech-challenge-fiap-<ACCOUNT_ID>`, `use_lockfile` para o lock). O consumidor expõe os valores em `locals` no `versions.tf`.

| Produtor (state) | Output | Consumidor | Uso |
| ---------------- | ------ | ---------- | --- |
| `infra-k8s/base` | `ecr_repository_url` | `infra-k8s/cluster` | reexportado no output do cluster |
| `infra-k8s/base` | `github_actions_role_arn` | `infra-k8s/cluster` | EKS access entry (`ClusterAdmin`) da role de deploy |
| `infra-k8s/cluster` | `vpc_id` | `infra-db` | SGs do RDS e da Lambda |
| `infra-k8s/cluster` | `private_subnet_ids` | `infra-db`, `lambda-auth` | subnet group do RDS; VPC config da Lambda |
| `infra-k8s/cluster` | `cluster_primary_security_group_id` | `infra-db` | ingress 5432 do RDS a partir dos pods (no Auto Mode as ENIs dos pods recebem este SG) |
| `infra-db` | `lambda_security_group_id` | `lambda-auth` | SG anexado à função `auth` |
| `infra-db` | `api_secret_arn` | `lambda-auth` | policy de leitura do segredo |
| `infra-db` | `api_secret_name` | `lambda-auth` | nome do segredo passado à função em runtime |

Os demais outputs de cada stack existem, mas nenhuma outra stack os lê por remote state:

- `infra-k8s/base`: `ecr_repository_arn`, `github_actions_role_name`, `oidc_provider_arn`, `aws_account_id`, `github_terraform_role_arn`, `frontend_bucket_name`, `frontend_website_endpoint`.
- `infra-k8s/cluster`: `cluster_name`, `cluster_endpoint`, `configure_kubectl`, `vpc_cidr_block`, `node_security_group_id` (alternativa documentada ao SG primário), `aws_region`, `project_name`.
- `infra-db`: `db_endpoint` (`sensitive`), `db_name`, `db_security_group_id`, `create_k8s_secret_command`.
- `lambda-auth`: `lambda_function_name`, `lambda_function_arn`, `lambda_invoke_arn`, `lambda_role_arn`, `log_group_name`, `api_id`, `api_endpoint`, `authorizer_id`, `authorizer_function_name`. Nenhuma stack lê o state do `lambda-auth`.

A ordem `base` → `cluster` → `infra-db` → `lambda-auth` é a ordem de leitura dos states. Os `outputs.tf` são tratados como API pública: o comentário em `cluster/outputs.tf` avisa que renomear um output quebra o `apply` dos outros repositórios.

### Mecanismo 2: do Terraform para o `app-oficina`, variables e secrets do GitHub preenchidos à mão

**Não há `terraform_remote_state` no `app-oficina`.** O `deploy.yml` (e o job de push no ECR do `ci.yml`) lê configuração de `vars.*` e `secrets.*` do repositório do GitHub, que alguém preenche manualmente a partir dos outputs do Terraform:

| Variable / secret (`app-oficina`) | Origem (output) | Usado em |
| --------------------------------- | --------------- | -------- |
| `secrets.AWS_DEPLOY_ROLE_ARN` | `infra-k8s/base`: `github_actions_role_arn` | OIDC no `ci.yml` (push) e no `deploy.yml` |
| `vars.AWS_REGION` | `infra-k8s/cluster`: `aws_region` (valor literal `us-east-1`) | OIDC, `update-kubeconfig`, ECR |
| `vars.EKS_CLUSTER_NAME` | `infra-k8s/cluster`: `cluster_name` | `aws eks update-kubeconfig` |
| `vars.API_SECRET_NAME` | `infra-db`: `api_secret_name` | `aws secretsmanager get-secret-value` que alimenta o Secret `api-secrets` |
| `vars.ECR_REPOSITORY` | nome do repositório ECR (`project_name`, derivável de `ecr_repository_url`) | tag da imagem no push e no `set image`/Job |
| `vars.FRONTEND_URL` | `infra-k8s/base`: `frontend_website_endpoint` | `FRONTEND_URL` do ConfigMap. Opcional: se ausente, o deploy deriva `http://<ECR_REPOSITORY>-front-<conta>.s3-website-<região>.amazonaws.com` |

Os três repositórios de infra têm o mesmo padrão em escala menor: `secrets.AWS_TERRAFORM_ROLE_ARN` (output `github_terraform_role_arn`) e `vars.AWS_REGION`. O passo está descrito como snippet de `gh secret set` / `gh variable set` na seção 5 de `infra-k8s/docs/00-setup-inicial-aws.md`, executado por uma pessoa depois do `apply` da `base`. O cabeçalho do `deploy.yml` repete a lista de pré-requisitos.

Há um terceiro caminho manual, no sentido inverso: o hostname do NLB criado pelo Service `api` do `app-oficina` entra no `lambda-auth` por `terraform apply -var api_nlb_hostname=…`.

### Alternativas e contraste

O código não registra a deliberação que levou aos mecanismos acima; o contraste a seguir é análise de trade-off sobre o que o repositório faz hoje, não histórico de decisão.

| Alternativa | O que mudaria | Por que o desenho atual é o que está no código |
| ----------- | ------------- | ---------------------------------------------- |
| **SSM Parameter Store** | Cada stack publicaria `aws_ssm_parameter` e o consumidor leria por `data "aws_ssm_parameter"`. O `deploy.yml` também poderia ler (`aws ssm get-parameter`), eliminando o handoff manual do mecanismo 2. | Exige recursos extras por valor, um esquema de paths e serialização das listas (subnets). O `terraform_remote_state` não cria recurso nenhum e já tem a semântica desejada: a ausência do state falha no plan e define a ordem de deploy (comentário em `infra-db/terraform/versions.tf`). O custo é que ele só serve a quem roda Terraform, e é por isso que o `app-oficina` ficou de fora. |
| **Data sources por tag** (`data "aws_vpc"`, `data "aws_subnets"` com filtro) | O consumidor procuraria recursos por tag (as stacks já aplicam `Project` e `Stack` via `default_tags`). | Acopla o consumidor à convenção de tags e de nomes em vez de a um contrato explícito (`outputs.tf`); um filtro que casa zero ou vários recursos falha de forma menos legível do que um output ausente, e os outputs documentam o que é contrato. |
| **Monorepo** | Um único root module ou um grafo de stacks no mesmo repositório: referências diretas, sem remote state, uma única ordem imposta pelo Terraform. | É o estado anterior do projeto (ver nota de Fase 3 em `docs/README.md`). A entrega da Fase 3 é composta por quatro repositórios, cada um com pipeline e permissão próprios (ADR-001); o remote state é o preço de manter os states e as permissões separados. |

## Consequências

### Positivas

- **Mecanismo 1 sem recurso extra nem passo manual.** O contrato é o `outputs.tf` do produtor; o consumidor lê o valor tipado (`list(string)`, `string`) direto do state. A ordem de deploy aparece como erro explícito (`Unable to find remote state`) e não como valor vazio.
- **Fonte única do valor.** VPC, subnets e SGs vêm do state que os criou; as stacks não replicam IDs em `tfvars`.
- **Segredo fora do caminho.** Nem o remote state nem as variables carregam credencial: o `db_endpoint` é `sensitive`, a senha nunca é output, e o `app-oficina` lê o segredo em tempo de deploy do Secrets Manager (o `deploy.yml` materializa o Secret `api-secrets` e não passa valor pelo repositório).
- **O pipeline da aplicação não depende de Terraform.** O mecanismo 2 não exige que o runner do `app-oficina` tenha acesso ao bucket de state nem versão de Terraform.

### Negativas

- **O mecanismo 2 é um passo manual, não versionado e sem verificação.** As variables e secrets do `app-oficina` são preenchidos à mão a partir de outputs; nada no código garante que o valor digitado bate com o output atual, e uma mudança no produtor (por exemplo, recriar o cluster com outro nome) não chega ao `app-oficina`. O snippet de `gh variable set` existe só como documentação.
- **Hoje o passo não foi executado.** Em 2026-10-19, `gh variable list` e `gh secret list` nos quatro repositórios retornam vazio: nenhuma variable ou secret está configurada. Consequência direta: todo job que assume uma role AWS falha em `aws-actions/configure-aws-credentials` com `Input required and not supplied: aws-region` (`plan` e `apply` dos três repositórios de infra, push no ECR e deploy do `app-oficina`). Só os jobs sem credencial (`validate`, testes, kind) passam.
- **Duas fontes de verdade para o mesmo dado.** `ECR_REPOSITORY`, `EKS_CLUSTER_NAME` e `API_SECRET_NAME` existem no Terraform (derivados de `project_name` e `cluster_name`) e de novo como strings no GitHub; o default do `FRONTEND_URL` ainda reconstrói o nome do bucket a partir de `ECR_REPOSITORY`, acoplando o deploy à convenção `<project>-front-<conta>` de `s3-frontend.tf`.
- **O caminho inverso `app-oficina` → `lambda-auth` também é manual.** O hostname do NLB entra por `-var`, fora do `ci.yml` do `lambda-auth` (ver ADR-001).
- **`terraform_remote_state` expõe o state inteiro ao consumidor, não só os outputs.** Quem lê precisa de `s3:GetObject` no arquivo de state, que contém todos os atributos do produtor (no `infra-db`, o state inclui os `random_password` das chaves JWT). Na prática, as três stacks já rodam sob a mesma role de provisionamento (PowerUserAccess + IAM), então isso não amplia acesso hoje, mas elimina qualquer isolamento por stack no bucket.
- **Acoplamento ao layout do state.** O bucket e a chave de cada stack estão literais em cada `terraform_remote_state` e em cada bloco `backend` (este não aceita variável); renomear uma chave ou o bucket exige editar vários repositórios.
- **Contrato frágil a renomes.** Sem teste de contrato, renomear `cluster_primary_security_group_id` ou `private_subnet_ids` só aparece quando o `infra-db` ou o `lambda-auth` rodam o plan.
- **Descrições divergentes do código.** `github_actions_role_name` diz ser usado pelo `cluster/` para o access entry, mas o código usa `github_actions_role_arn`; o README do `infra-k8s` lista `vpc_id` como lido pelo `lambda-auth`, que na verdade lê só `private_subnet_ids` do cluster.

## Referências

- `infra-k8s/terraform/base/outputs.tf`, `infra-k8s/terraform/cluster/outputs.tf`, `infra-k8s/terraform/cluster/versions.tf` (`data "terraform_remote_state" "base"`) e `eks.tf` (access entry).
- `infra-db/terraform/versions.tf` (`data "terraform_remote_state" "k8s"`, `locals`), `outputs.tf`, `secrets.tf`.
- `lambda-auth/terraform/versions.tf` (`terraform_remote_state` `k8s` e `db`), `outputs.tf`, `variables.tf` (`api_nlb_hostname`).
- `app-oficina/.github/workflows/deploy.yml` (cabeçalho com os pré-requisitos, passos "Autentica na AWS", "Configura o kubectl", "Namespace e Secret da API", "Resolve as URLs públicas") e `ci.yml` (job `push`).
- `infra-k8s/docs/00-setup-inicial-aws.md`, seções 5 e 6.
- [ADR-001](./adr-001-quatro-repositorios.md).
