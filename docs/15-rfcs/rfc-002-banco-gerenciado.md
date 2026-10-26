# RFC-002 · Banco de dados: RDS PostgreSQL gerenciado

> [← Voltar ao índice de RFCs](./README.md)

| | |
| --- | --- |
| **Status** | Aceita — decisão já implementada; RFC escrita a posteriori |
| **Data do registro** | 19/10/2026 |
| **Escopo** | Onde e como o PostgreSQL da aplicação roda em ambiente AWS (`infra-db`) |
| **Relacionadas** | [RFC-001](./rfc-001-escolha-da-nuvem.md) · [RFC-003](./rfc-003-estrategia-de-autenticacao.md) · [Contrato da Fase 3](../14-fase3-contrato.md) |

> **Sobre as fontes.** As seções *Decisão* e a tabela de postura de custo descrevem o que está nos arquivos Terraform do `infra-db`, linha a linha. A comparação com Aurora Serverless v2 é **avaliação a posteriori**: não há registro dela no repositório, e os atributos de Aurora vêm de conhecimento geral, sem verificação contra a documentação e os preços vigentes. O `README.md` do `infra-db` menciona um ADR de banco de dados "no repositório de documentação" com a justificativa formal; esse ADR não está entre as fontes lidas para esta RFC.

## Contexto

O domínio é fortemente relacional (cliente, veículo, ordem de serviço, peças, insumos) e exige integridade referencial e transações ACID para abertura de OS e baixa de estoque (`docs/01-arquitetura.md:22-29`). A escolha de PostgreSQL com Prisma é anterior a esta RFC; aqui se decide **onde ele roda** em ambiente AWS.

Quem conecta no banco:

| Cliente | Driver | Como obtém a conexão | Evidência |
| --- | --- | --- | --- |
| API (pods no EKS) | Prisma | `DATABASE_URL` materializada do Secrets Manager em `Secret/api-secrets` no deploy | `app-oficina/.github/workflows/deploy.yml:105-131`; `k8s/40-api-deployment.yaml:98-102` |
| Job `migrate` | Prisma (`migrate deploy`) | mesmo `api-secrets` | `k8s/jobs/30-migrate-job.yaml:29-60` |
| Lambda de login por CPF (na VPC) | `pg`, SQL cru na tabela `cliente` | lê o segredo em runtime; `Pool` com `max: 1` | `lambda-auth/src/cliente.ts:22-37,62-90` |

Restrições que decorrem do código:

1. **Dois tipos de compute precisam alcançar o banco**: pods do EKS e uma Lambda dentro da VPC.
2. **O ciclo de vida é efêmero**: a stack sobe para demonstrar e desce em seguida, e o `terraform destroy` precisa terminar limpo (`infra-db/README.md:77-84,134-140`).
3. **Um único segredo** carrega `DATABASE_URL` e as chaves JWT (`secrets.tf:1-7`), então o banco e o segredo nascem na mesma stack.
4. **O custo é o critério** declarado (`infra-db/README.md:134-137`).

## Opções consideradas

| Critério | A. RDS PostgreSQL, instância única (**escolhida**) | B. Aurora PostgreSQL Serverless v2 | C. PostgreSQL no cluster (StatefulSet) |
| --- | --- | --- | --- |
| Operação | Gerenciada pela AWS: patch de engine, storage, criptografia em repouso | Gerenciada pela AWS, com camada de cluster (writer, storage compartilhado) | Por conta da equipe: imagem, PVC, upgrade, backup |
| Alcance pela Lambda | Sim: endpoint na VPC, SG referenciando o SG da Lambda (`rds.tf:52-61`) | Sim, mesmo modelo de rede | **Não, pelo que está no repositório**: o Service é headless (`clusterIP: None`) e só resolve dentro do cluster (`k8s/local/20-postgres.yaml:1-15`) |
| Ciclo de vida do dado | Independente do cluster, mas atrelado à stack `infra-db` | Idem | Atrelado ao PVC (2Gi) dentro do cluster (`docs/01-arquitetura.md:221`) |
| Custo de baseline | `db.t4g.micro`, 20 GB; elegível ao free tier nos 12 primeiros meses (`variables.tf:13-23`) | **A conferir.** Cobrança por capacidade (ACU-hora) e por storage; sem equivalência com o free tier da instância micro. Não orçado | Sem custo extra de serviço; usa a capacidade do cluster |
| Destroy limpo | Exige `deletion_protection = false` e `skip_final_snapshot = true` (`rds.tf:94-98`) | Exigiria o mesmo tipo de ajuste, em recurso de cluster e de instância | Some junto com o namespace/PVC |
| Aderência ao código existente | Total: Prisma e `pg` falam com RDS sem mudança | Total no nível de protocolo | Total para a API; **quebra a Lambda** |
| Papel no projeto | Alvo de AWS (`infra-db`) | Não existe no repositório | Existe, mas **só para desenvolvimento local em kind** (`k8s/local/20-postgres.yaml`; `app-oficina/README.md:71,127`) |

Notas sobre as opções:

- **C não é hipotética: foi a Fase 2.** A arquitetura original tinha o StatefulSet `postgres` com PVC de 2Gi dentro do namespace (`docs/01-arquitetura.md:204,220-221`). Hoje ele sobrevive em `k8s/local/` e é explicitamente "só dev local; no EKS o banco é RDS" (`k8s/README.md:21`). A razão estrutural contra C em AWS é a Lambda: o banco precisa ser alcançável de fora do cluster, e um Service headless não é. Isso é inferência a partir dos manifestos, não uma justificativa registrada.
- **B é avaliação a posteriori.** O que se pode afirmar pelo código: o projeto não tem requisito de escala variável de banco — o consumo é uma API com 2 a 6 réplicas (`k8s/50-hpa.yaml`) e uma Lambda com pool de uma conexão (`cliente.ts:11-19`) — e o critério explícito é custo mínimo. Aurora Serverless v2 resolve variação de carga e alta disponibilidade, que o projeto não pede e que o custo de baseline provavelmente não pagaria nesta escala. **Não foi medido.**
- **Variante de A, Multi-AZ**, está explicitamente desligada (ver postura de custo abaixo), então não é uma quarta opção, e sim uma configuração recusada.

## Decisão

**RDS PostgreSQL 16, instância única, em subnets privadas, definido no repositório `infra-db`**, via módulo `terraform-aws-modules/rds/aws ~> 6.0`. Valores reais, de `infra-db/terraform/`:

| Parâmetro | Valor | Onde |
| --- | --- | --- |
| Engine / versão | `postgres` / `"16"` (família `postgres16`) | `rds.tf:69-72` |
| Classe da instância | `db.t4g.micro` (default da variável) | `variables.tf:13-17`; `rds.tf:73` |
| Storage alocado | 20 GB (mínimo do Postgres) | `variables.tf:19-23`; `rds.tf:75` |
| Criptografia em repouso | `storage_encrypted = true` | `rds.tf:76` |
| Banco / usuário master | `oficina` / `oficina_admin` | `variables.tf:25-35` |
| Senha | `random_password` de 32 chars, alfabeto restrito `!#%*-_=+` | `rds.tf:1-9,80` |
| Gestão da senha | `manage_master_user_password = false`: a senha vai para o nosso segredo, para montar a `DATABASE_URL` completa num JSON só | `rds.tf:83-86` |
| Multi-AZ | `false` | `rds.tf:88` |
| Acesso público | `publicly_accessible = false`; subnets privadas | `rds.tf:89-91` |
| Security group do RDS | 5432 apenas a partir do SG primário do EKS e do SG da Lambda | `rds.tf:11-26,52-61` |
| Deletion protection | `false` | `rds.tf:97` |
| Snapshot final | `skip_final_snapshot = true` | `rds.tf:98` |
| Retenção de backup | `backup_retention_period = 0` | `rds.tf:100` |
| Monitoramento estendido | `create_monitoring_role = false` | `rds.tf:101` |
| Segredo | `tech-challenge-fiap/api`, `recovery_window_in_days = 0` | `secrets.tf:19-28` |

### O security group da Lambda nasce aqui

O ingress do RDS precisa referenciar o SG da Lambda, e um SG não pode ser referenciado antes de existir. Se ele nascesse no `lambda-auth`, o grafo teria ciclo: a Lambda esperando o RDS e o RDS esperando o SG da Lambda. Criado no `infra-db` e exportado como `lambda_security_group_id`, o grafo fica numa direção só: `infra-k8s -> infra-db -> lambda-auth` (`rds.tf:28-40`; `outputs.tf:23-29`; `infra-db/README.md:103-109`; consumo em `lambda-auth/terraform/versions.tf:47-66`).

### A connection string e o TLS

```
postgresql://<user>:<senha url-encoded>@<endpoint>/oficina?schema=public&sslmode=require&uselibpqcompat=true
```

(`secrets.tf:42-50`). O raciocínio, que tem nuance de driver:

1. O RDS PostgreSQL ≥ 15 vem com `rds.force_ssl=1`: sem TLS a conexão é recusada (`no pg_hba.conf entry ... no encryption`). Por isso `sslmode=require` é obrigatório (`secrets.tf:33-35`). O `rds.tf` não declara `parameters` nem parameter group com ajustes: o TLS obrigatório é o default do engine.
2. No driver `pg` do Node, `sslmode=require` **sozinho** se comporta como `verify-full`: valida a cadeia de certificados. A CA do RDS não está no trust store do runtime, e a conexão cai com *self-signed certificate in chain* (`secrets.tf:35-38`).
3. `uselibpqcompat=true` troca para a semântica do libpq: `require` **criptografa sem exigir a CA**, o mesmo comportamento do `psql` (`secrets.tf:38-39`; `infra-db/README.md:119-126`).
4. O engine do Prisma (usado por `migrate deploy` e pela API) aceita e ignora o parâmetro extra (`secrets.tf:39-40`). A Lambda, que usa `pg` direto, é quem de fato depende dele (`cliente.ts:25-34`).
5. A senha entra em `urlencode(...)` (`secrets.tf:46`), e o alfabeto da senha já exclui `/`, `@`, `"` e espaço (`rds.tf:4-8`).

Efeito de segurança, dito sem rodeio: a conexão é **cifrada**, mas o cliente **não verifica a identidade do servidor**. Isso protege contra escuta passiva; não protege contra um intermediário ativo dentro da VPC. É um trade-off consciente do README (`criptografa sem exigir a CA`), e o caminho para fechá-lo é distribuir a CA do RDS ao trust store e voltar a `verify-full`.

### Postura de custo: o que o README assume como "não é produção"

O README do `infra-db` é explícito: *"Sem Multi-AZ, sem backup, sem deletion protection ... Não é configuração de produção"* (`infra-db/README.md:134-140`). A tabela registra o que cada escolha significa na prática.

| Escolha | Valor | Consequência real | Por que foi aceita (segundo o código) |
| --- | --- | --- | --- |
| Sem Multi-AZ | `multi_az = false` (`rds.tf:88`) | Uma única instância, em uma AZ. Falha de AZ ou manutenção significa indisponibilidade; não há failover. O subnet group usa 2 AZs só porque o RDS exige (`cluster/variables.tf:31-37`) | Custo |
| Sem backup | `backup_retention_period = 0` (`rds.tf:100`) | Backups automáticos desligados: **não há restauração a um ponto no tempo**. Perda da instância = perda dos dados | Stack feita para subir e descer; os dados são seed e demonstração |
| Sem snapshot final | `skip_final_snapshot = true` (`rds.tf:98`) | `terraform destroy` apaga os dados sem deixar cópia | Snapshot obrigatório faria o `destroy` falhar e "sobra instância rodando — exatamente o custo que queremos evitar" (`rds.tf:94-97`) |
| Sem deletion protection | `deletion_protection = false` (`rds.tf:97`) | Um `destroy` (ou um `apply` errado) remove a instância sem barreira | Mesma razão: `destroy` limpo |
| Segredo sem janela de recuperação | `recovery_window_in_days = 0` (`secrets.tf:23-27`) | Exclusão imediata e irreversível do segredo | Sem isso o próximo `apply` falha com *already scheduled for deletion*; o nome fica ocupado por 7 a 30 dias |

Itens adicionais da postura, **observados no código e não declarados como decisão** pelo README:

- **A aplicação conecta como usuário master.** A `DATABASE_URL` usa `var.db_username` (`oficina_admin`), o mesmo da criação da instância (`secrets.tf:44-49`). Não há usuário de menor privilégio para a API nem para a Lambda, que só precisa de `SELECT` em `cliente`.
- **Sem rotação de segredo.** Não há `aws_secretsmanager_secret_rotation` no repositório. A senha e as chaves JWT são geradas por `random_password` e só mudam se a stack for recriada.
- **A senha vive no state do Terraform**, que é criptografado no S3 (`versions.tf:21`) mas contém o valor em claro, como é inerente a `random_password`.
- **Recriar a stack invalida tudo que dependia dela**: nova senha de banco, **novas chaves JWT** (todo token emitido deixa de valer) e banco vazio, exigindo `migrate` e seed de novo.
- **Sem RDS Proxy.** A Lambda limita o próprio pool a uma conexão porque "com `db.t4g.micro` o limite de conexões é baixo e a concorrência da Lambda pode ser alta" (`cliente.ts:11-19`). A proteção é do lado do cliente; não há nada entre a Lambda e o banco.
- **Sem autoscaling de storage** declarado (`max_allocated_storage` ausente) e sem Performance Insights ou monitoramento estendido (`rds.tf:101`).
- **Versão de engine `"16"` sem minor fixada** (`rds.tf:70`): o minor é o que a AWS entregar na criação.

O que mudaria num ambiente real, por oposição direta a cada item acima: Multi-AZ, retenção de backup maior que zero, deletion protection ligada e snapshot final obrigatório, usuário de aplicação com privilégio mínimo, rotação de segredo, CA do RDS no trust store. **Nada disso está implementado nem planejado aqui**; é a lista de lacunas, não um backlog aprovado.

## Consequências

### Positivas

- **Sem operar banco.** Patch de engine, storage e criptografia em repouso ficam com a AWS; a equipe não mantém StatefulSet, PVC nem rotina de upgrade.
- **Banco inacessível pela internet.** Subnet privada, sem IP público, ingress só de dois security groups referenciados por ID (`rds.tf:11-26,52-61,89-92`).
- **Grafo de dependência acíclico** entre as stacks, graças ao SG da Lambda no `infra-db`.
- **Um único segredo, composto num lugar.** `DATABASE_URL` e chaves JWT saem de `secrets.tf` e são lidos por três consumidores (Lambda, authorizer, API), sem que nenhum valor passe pelo repositório (`deploy.yml:105-107`).
- **`destroy` previsível** na ordem inversa: `infra-db` antes do `infra-k8s`, porque o RDS vive nas subnets daquela VPC e segura o `destroy` dela (`infra-db/README.md:77-84`).

### Negativas e riscos

- **Dados descartáveis por construção.** Sem backup e sem snapshot final, o RDS **não deve guardar nada que importe**. Isso é compatível com demonstração e incompatível com uso real.
- **Disponibilidade de instância única.** Ver tabela de postura.
- **Acoplamento de schema entre repositórios.** A Lambda consulta `cliente` por SQL cru, com `regexp_replace` na coluna (`cliente.ts:62-90`). Renomear `num_documento` ou `deletado_em` no Prisma do `app-oficina` faz a Lambda devolver 401 para todo cliente, sem erro explícito. A fixture `lambda-auth/test/fixtures/schema.sql` é uma **cópia** do subconjunto lido, e só denuncia divergência se alguém a atualizar.
- **Seq scan no login.** O `regexp_replace` na coluna impede o uso do índice único de `num_documento` (`cliente.ts:56-60`); irrelevante no volume de uma oficina, registrado pelo próprio código.
- **Ordem de deploy exigida.** `infra-db` lê o state do cluster; aplicado antes, falha no `plan` com *Unable to find remote state* (`versions.tf:38-44`).
- **O secret não é recuperável após `destroy`** (`recovery_window_in_days = 0`): o preço da escolha que evita o travamento do `apply` seguinte.

### Pendências

- O ADR de banco de dados citado pelo README não foi lido; se ele registra justificativas além das aqui transcritas, esta RFC deve apontá-lo.
- Nenhuma medição de custo de Aurora Serverless v2 foi feita.

## Referências

**infra-db**
- `terraform/rds.tf:1-9` — senha aleatória e alfabeto restrito.
- `terraform/rds.tf:11-26` — SG do RDS e ingress do EKS.
- `terraform/rds.tf:28-61` — SG da Lambda criado aqui; egress; ingress do RDS a partir dele.
- `terraform/rds.tf:63-102` — módulo RDS: engine, classe, storage, rede, flags de custo.
- `terraform/secrets.tf:1-7,19-28` — segredo compartilhado e recovery window.
- `terraform/secrets.tf:30-56` — composição da `DATABASE_URL` e comentário de TLS.
- `terraform/variables.tf:13-35` — classe, storage, banco, usuário.
- `terraform/outputs.tf:23-29` — `lambda_security_group_id`.
- `terraform/versions.tf:15-23,38-62` — backend e state do cluster.
- `README.md:77-84,103-140` — destruir, SG da Lambda, chave JWT, TLS, recovery window, postura de custo.

**infra-k8s**
- `terraform/cluster/outputs.tf:29-55` — saídas consumidas pelo `infra-db`.
- `terraform/cluster/variables.tf:31-40` — 2 AZs como mínimo para EKS e subnet group.

**lambda-auth**
- `src/cliente.ts:11-37,49-90` — pool `max: 1`, SQL cru, `regexp_replace`.
- `terraform/versions.tf:47-66` — consumo do state do `infra-db`.
- `test/fixtures/schema.sql:1-12` — subconjunto de schema para teste.

**app-oficina**
- `docs/01-arquitetura.md:22-29,204,220-221` — por que PostgreSQL; StatefulSet da Fase 2.
- `.github/workflows/deploy.yml:105-131` — segredo para `api-secrets`.
- `k8s/jobs/30-migrate-job.yaml:29-60` — `DATABASE_URL` do Job de migration.
- `k8s/local/20-postgres.yaml:1-15` — Service headless do Postgres local.
- `prisma/schema.prisma:59-74` — tabela `cliente` lida pela Lambda.
