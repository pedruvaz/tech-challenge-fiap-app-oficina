# RFC-001 · Escolha da nuvem: AWS

> [← Voltar ao índice de RFCs](./README.md)

| | |
| --- | --- |
| **Status** | Aceita — decisão já implementada; RFC escrita a posteriori |
| **Data do registro** | 19/10/2026 |
| **Escopo** | Provedor de nuvem de todo o sistema (`infra-k8s`, `infra-db`, `lambda-auth`, `app-oficina`) |
| **Relacionadas** | [RFC-002](./rfc-002-banco-gerenciado.md) · [RFC-003](./rfc-003-estrategia-de-autenticacao.md) · [Contrato da Fase 3](../14-fase3-contrato.md) |

> **Leia isto antes do resto.** Esta RFC documenta uma decisão que **já está no código**. Não existe registro da comparação entre provedores feita na época: o histórico dos quatro repositórios começa em 13/10/2026, na extração do monorepo (`infra-k8s`: *"infraestrutura Kubernetes extraída do monorepo"*), e não carrega a discussão original. Por isso a seção *Opções consideradas* é uma **avaliação a posteriori**, escrita depois do fato, e o que ela diz sobre GCP e Azure vem de conhecimento geral dos serviços — **não** foi medido, orçado nem verificado contra preços e limites vigentes. O que é fato verificável está sempre acompanhado de arquivo e linha.

## Contexto

O sistema é uma API de gestão de oficina (NestJS + Prisma sobre PostgreSQL) que, a partir da Fase 2, roda em Kubernetes e, na Fase 3, foi dividida em quatro repositórios com pipelines independentes ([contrato](../14-fase3-contrato.md), seção 1). Lendo o código, o que a infraestrutura precisa entregar é:

| Necessidade | Onde aparece |
| --- | --- |
| Kubernetes gerenciado, sem operar nós | `infra-k8s/terraform/cluster/eks.tf:1-12` |
| PostgreSQL gerenciado, inacessível pela internet | `infra-db/terraform/rds.tf:88-92` |
| Cofre de segredos lido em runtime por três consumidores | `infra-db/terraform/secrets.tf:1-7` |
| Registry de imagens com varredura de vulnerabilidade | `infra-k8s/terraform/base/ecr.tf:9-13` |
| Gateway HTTP público com authorizer customizado | `lambda-auth/terraform/api_gateway.tf:40-43,119-133` |
| Função serverless dentro da VPC, alcançando o banco | `lambda-auth/terraform/main.tf:76-82` |
| CI/CD sem chave de acesso estática guardada no GitHub | `infra-k8s/terraform/base/iam-github-oidc.tf:1-5` |
| Stacks que sobem e descem para não gerar custo contínuo | `app-oficina/README.md:35,67-69,171` |

O último item pesa nas escolhas: a stack `cluster/` ligada 24/7 custaria ~US$ 160/mês; uma sessão de demonstração de 4 h sai por ~US$ 1 (`app-oficina/README.md:35,171`). Esses números são os do README do projeto — não foram recalculados aqui.

## Opções consideradas

Três provedores. A coluna **AWS** descreve o que está implementado (fato, com evidência); as colunas **GCP** e **Azure** são o equivalente conceitual de cada peça (avaliação a posteriori, sem evidência no repositório).

| Peça | AWS (implementado) | GCP (a posteriori) | Azure (a posteriori) |
| --- | --- | --- | --- |
| Kubernetes gerenciado | EKS em **Auto Mode**: compute, storage e load balancing geridos pela AWS, sem node group (`eks.tf:1-8,27-30`) | GKE, em particular o modo Autopilot, que tem a mesma proposta de "sem node pool" | AKS (há também o modo Automatic) |
| PostgreSQL gerenciado | RDS PostgreSQL 16 (`rds.tf:63-102`) | Cloud SQL for PostgreSQL | Azure Database for PostgreSQL (Flexible Server) |
| Cofre de segredos | Secrets Manager (`secrets.tf:19-28`) | Secret Manager | Key Vault |
| Registry | ECR (`ecr.tf:1-18`) | Artifact Registry | Azure Container Registry |
| Gateway HTTP + authorizer | API Gateway **HTTP API** + Lambda authorizer (`api_gateway.tf:40-43,119-133`) | API Gateway / Cloud Endpoints; authorizer próprio atrás | API Management; authorizer próprio atrás |
| Função na VPC | Lambda `nodejs20.x` com `vpc_config` (`main.tf:68-82`) | Cloud Run functions com conector de VPC | Azure Functions com integração de VNet |
| Federação OIDC do GitHub | IAM OIDC provider + role com `AssumeRoleWithWebIdentity` (`iam-github-oidc.tf:6-69`) | Workload Identity Federation | Credenciais federadas no Entra ID |
| Load balancer do Service | NLB via `loadBalancerClass: eks.amazonaws.com/nlb` (`app-oficina/k8s/41-api-service.yaml:13-19`) | LB nativo do GKE | Azure Load Balancer |

Contraste honesto, por critério:

| Critério | Leitura |
| --- | --- |
| **Cobertura de serviços** | Os três provedores têm um equivalente para cada peça da tabela. Nenhuma peça do sistema é exclusiva da AWS; a decisão **não** foi forçada por falta de serviço nos demais. |
| **Custo de baseline** | O repositório declara dois custos fixos da AWS: o NAT Gateway (~US$ 33/mês, `vpc.tf:24-27`) e a taxa de gerenciamento do Auto Mode sobre as instâncias EC2 (`eks.tf:4-6`). De conhecimento geral, e **a conferir na tabela de preços vigente**: o control plane do EKS é cobrado por hora, enquanto o AKS tem camada gratuita e o GKE oferece crédito para um cluster por conta de faturamento. Se isso se mantém, é plausível que GCP ou Azure reduzissem o baseline. **Nunca foi orçado.** |
| **Tempo de entrega** | O código justifica o Auto Mode pelo prazo: "o tempo economizado compensa" (`eks.tf:7-8`). Esse argumento vale igualmente para GKE Autopilot e AKS Automatic; não distingue AWS. |
| **Segurança de pipeline** | OIDC sem chave estática é possível nos três. Na AWS, o que o código entrega é a trust policy restrita aos repositórios do projeto (`iam-github-oidc.tf:36-60`). |
| **Portabilidade dos manifestos** | Parcial (ver Consequências): o Service do `app-oficina` já embute a classe de LB do EKS. |
| **Familiaridade da equipe** | **Não é possível afirmar a partir do código.** Se foi um fator, não está registrado. |

O repositório **não registra** por que a AWS foi escolhida em vez de GCP ou Azure. As razões que o código declara são para escolhas *dentro* da AWS (Auto Mode por prazo, NAT único por custo, OIDC por segurança, `db.t4g.micro` por free tier — `infra-db/terraform/variables.tf:14`).

## Decisão

**AWS, região `us-east-1`, conta única.** A região está fixa nos defaults de variável e nos `backend "s3"` (`infra-k8s/terraform/base/versions.tf:18-24`; `infra-db/terraform/versions.tf:17-23`; `lambda-auth/terraform/versions.tf:15-21`). Provisionamento por Terraform com `aws ~> 6.0`, state em S3 com lock nativo (`use_lockfile`, dispensa DynamoDB — `base/versions.tf:16-17`).

Inventário do que está efetivamente em uso:

| Serviço | Papel | Evidência |
| --- | --- | --- |
| **EKS** (Auto Mode, Kubernetes 1.33) | Roda a API; node pool `general-purpose` | `cluster/eks.tf:13-30`; `cluster/variables.tf:19-23` |
| **VPC** (2 AZs, NAT único) | Rede; subnets privadas para EKS, RDS e Lambda | `cluster/vpc.tf:13-31`; `cluster/variables.tf:31-40` |
| **ECR** | Imagem da API, `scan_on_push`, lifecycle policy | `base/ecr.tf:1-18` |
| **IAM OIDC + 2 roles** | `github_actions` (deploy) e `github_terraform` (provisionamento) | `base/iam-github-oidc.tf:6-69`; `base/iam-github-terraform.tf:41-46` |
| **RDS PostgreSQL 16** | Banco da aplicação (ver [RFC-002](./rfc-002-banco-gerenciado.md)) | `infra-db/terraform/rds.tf:63-102` |
| **Secrets Manager** | Segredo único `tech-challenge-fiap/api` | `infra-db/terraform/secrets.tf:19-56` |
| **API Gateway HTTP API** | Porta de entrada pública | `lambda-auth/terraform/api_gateway.tf:40-43` |
| **Lambda** (2 funções) | Login por CPF e authorizer (ver [RFC-003](./rfc-003-estrategia-de-autenticacao.md)) | `main.tf:61-98`; `authorizer.tf:56-85` |
| **NLB** (via EKS Auto Mode) | Entrada da API no cluster, `internet-facing` | `app-oficina/k8s/41-api-service.yaml:9-19` |
| **S3** | State do Terraform e site estático do frontend | `base/versions.tf:18-24`; `base/s3-frontend.tf:7-15` |
| **CloudWatch Logs** | Logs das Lambdas e do access log do gateway, com retenção | `api_gateway.tf:56-61`; `main.tf:53-59` |

Divisão em stacks do `infra-k8s`: **`terraform/base`** (ECR, provider OIDC, roles IAM, bucket do frontend) é permanente, ~US$ 0/mês; **`terraform/cluster`** (VPC, EKS) é efêmera (`app-oficina/README.md:67-69`). RDS e Secrets Manager ficam no `infra-db`, também efêmero.

Como o `app-oficina` se autentica na AWS: o CI troca o token OIDC do GitHub por credenciais temporárias e assume a role de deploy (`app-oficina/.github/workflows/ci.yml:183-187` e `deploy.yml:92-96`). **Sem** `terraform_remote_state`: o `deploy.yml` lê *repository variables* (`AWS_REGION`, `EKS_CLUSTER_NAME`, `API_SECRET_NAME`, `ECR_REPOSITORY`) e o secret `AWS_DEPLOY_ROLE_ARN` (`deploy.yml:3-8`); entre as stacks Terraform, sim, o acoplamento é por `terraform_remote_state` ([contrato](../14-fase3-contrato.md), seção 2).

## Consequências

### Positivas

- **Sem credencial estática.** Nenhuma access key mora no GitHub; a trust policy casa apenas os repositórios do projeto, nos dois formatos de `sub` que o GitHub emite (clássico e imutável) — `iam-github-oidc.tf:36-60`. Sem essa condição o provider OIDC confiaria no GitHub inteiro.
- **Blast radius separado por pipeline.** Duas roles: a de deploy é estreita (push no ECR, `eks:DescribeCluster`, leitura de um segredo, sync de um bucket — `iam-github-oidc.tf:71-137`); a de provisionamento só é assumida pelos três repositórios de infra, não pelo `app-oficina` (`iam-github-terraform.tf:1-11`).
- **Stacks descartáveis.** Os ajustes que garantem `destroy` limpo estão no código: `force_delete` no ECR (`ecr.tf:15-17`), `force_destroy` no bucket (`s3-frontend.tf:14`), RDS sem deletion protection e secret com recovery window zero ([RFC-002](./rfc-002-banco-gerenciado.md)).
- **Pouca superfície operacional.** Auto Mode elimina node group, AMI e patch de nós (`eks.tf:1-8`).

### Negativas e custos assumidos

- **Custo fixo mesmo ocioso.** NAT Gateway único (~US$ 33/mês, `vpc.tf:24-28`) e a taxa do Auto Mode sobre as instâncias (`eks.tf:4-6`). O NAT único é ponto único de falha para saída de internet — "aceitável aqui, inaceitável em produção" (`vpc.tf:26-27`). A Lambda de login na VPC depende dessa saída para falar com o Secrets Manager (`main.tf:76-78`).
- **Visibilidade reduzida sobre o data plane.** É o preço declarado do Auto Mode (`eks.tf:5-6`).
- **Superfície pública ampliada por conveniência.** Endpoint do EKS público, "em produção isso seria privado com acesso via VPN" (`eks.tf:23-25`); NLB `internet-facing` sem VPC Link, o que deixa o NLB alcançável por fora do gateway (`api_gateway.tf:148-151`; impacto na [RFC-003](./rfc-003-estrategia-de-autenticacao.md)); site do frontend em bucket S3 público, só HTTP (`s3-frontend.tf:33-42`; `outputs.tf:26-29`).
- **Permissões amplas assumidas.** A role de provisionamento é `PowerUserAccess` mais um bloco de IAM, "ampla e assumida como tal", com o controle no trust (só repos de infra, só OIDC) e na proteção da branch `main` (`iam-github-terraform.tf:48-60`). A role de deploy tem `AmazonEKSClusterAdminPolicy` em escopo de cluster, porque o deploy cria o próprio Namespace (`eks.tf:36-57`).
- **Homologação e produção dividem o cluster**, separadas por namespace, para não dobrar o custo (`deploy.yml:60-64`).
- **Acoplamento ao provedor em lugares que não são só Terraform.** O Service da API declara `loadBalancerClass: eks.amazonaws.com/nlb` e a annotation de scheme (`app-oficina/k8s/41-api-service.yaml:9-19`); em kind o Service fica `Pending`. O ingress do RDS referencia o security group primário do cluster, que no Auto Mode é o das ENIs dos pods (`cluster/outputs.tf:44-50`; `infra-db/terraform/versions.tf:58-61`). Migrar de nuvem seria reescrever as quatro stacks e ajustar esse Service.
- **Bootstrap manual.** O bucket de state é criado à mão uma vez, porque blocos `backend` não aceitam variáveis (`base/versions.tf:11-17`).
- **Configuração fora do Terraform.** As *repository variables* e secrets do `app-oficina` são preenchidas à mão a partir dos outputs; nada detecta divergência. Em 19/10/2026 **nenhuma** das variables/secrets exigidas está configurada em nenhum dos quatro repositórios, e por isso os pipelines de infra falham em `Input required and not supplied: aws-region` ([contrato](../14-fase3-contrato.md), seção 5).

### Documentação anterior à extração

`docs/01-arquitetura.md:94` e `docs/12-infra-deploy.md` ainda descrevem `infra/terraform/` dentro do `app-oficina`, RDS na stack `cluster/` e, em `docs/13-cicd-kubernetes.md`, push para GHCR com `KUBE_CONFIG` em base64. O código vigente é o dos repositórios `infra-k8s` e `infra-db` e do `deploy.yml` (ECR + OIDC). Em divergência, vale o código.

### Pendências desta decisão

- Não há comparação de custo real entre provedores. Se a decisão for reaberta, o primeiro passo é orçar o baseline (control plane, NAT, LB, banco) nos três.
- O motivo original da escolha da AWS não está registrado; se alguém da equipe lembrar, deve ser acrescentado aqui.

## Referências

**infra-k8s**
- `terraform/cluster/eks.tf:1-12` — EKS Auto Mode: o que entrega e o que custa.
- `terraform/cluster/eks.tf:23-25` — endpoint público do control plane.
- `terraform/cluster/eks.tf:36-57` — access entry do GitHub Actions (ClusterAdmin).
- `terraform/cluster/vpc.tf:24-28` — NAT único e o trade-off.
- `terraform/cluster/outputs.tf:44-50` — SG primário do cluster no Auto Mode.
- `terraform/cluster/variables.tf:19-23,31-40` — versão do Kubernetes e número de AZs.
- `terraform/base/ecr.tf:1-18` — ECR.
- `terraform/base/iam-github-oidc.tf:1-69,71-137` — provider OIDC, trust policy e policy de deploy.
- `terraform/base/iam-github-terraform.tf:1-11,48-60` — role de provisionamento e a permissão ampla.
- `terraform/base/s3-frontend.tf:7-42` — bucket do frontend.
- `terraform/base/versions.tf:11-24` — backend S3 e bootstrap.

**infra-db**
- `terraform/rds.tf:63-102` — RDS.
- `terraform/secrets.tf:1-7,19-56` — segredo compartilhado.
- `terraform/variables.tf:13-17` — `db.t4g.micro` e free tier.
- `terraform/versions.tf:15-23,38-62` — backend e `terraform_remote_state` do cluster.

**lambda-auth**
- `terraform/api_gateway.tf:40-61,119-133,148-151` — HTTP API, authorizer, integração com o NLB.
- `terraform/main.tf:53-98` — Lambda na VPC.
- `terraform/authorizer.tf:56-85` — authorizer.

**app-oficina**
- `README.md:35,65-69,171` — custo, divisão em stacks, sessão de demonstração.
- `.github/workflows/ci.yml:166-205` — push no ECR via OIDC.
- `.github/workflows/deploy.yml:3-8,60-64,92-103` — variables, ambientes, OIDC e kubeconfig.
- `k8s/41-api-service.yaml:9-19` — NLB e `loadBalancerClass`.
- `docs/14-fase3-contrato.md` — contrato entre repositórios.
