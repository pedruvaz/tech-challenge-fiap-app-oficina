# 15 · RFCs

> [← Voltar ao índice da documentação](../README.md)

Decisões técnicas da Fase 3 sobre nuvem, banco e autenticação. Cada RFC registra uma decisão **que já foi tomada e está implementada**: o objetivo é deixar o raciocínio, as alternativas e as consequências escritos, não propor mudança.

## Índice

| RFC | Título | Decisão em uma linha | Status |
| --- | --- | --- | --- |
| [RFC-001](./rfc-001-escolha-da-nuvem.md) | Escolha da nuvem | AWS (`us-east-1`): EKS Auto Mode, RDS, Secrets Manager, ECR, API Gateway HTTP API, Lambda e OIDC do GitHub | Aceita, registro a posteriori |
| [RFC-002](./rfc-002-banco-gerenciado.md) | Banco gerenciado | RDS PostgreSQL 16, instância única `db.t4g.micro`, sem Multi-AZ, sem backup, sem deletion protection | Aceita, registro a posteriori |
| [RFC-003](./rfc-003-estrategia-de-autenticacao.md) | Estratégia de autenticação | CPF -> Lambda -> JWT HS256, com authorizer Lambda no API Gateway e segredo compartilhado; **com falha de autorização conhecida na API** | Aceita, registro a posteriori |

## Como ler

- **Registro a posteriori.** As RFCs foram escritas depois das decisões, a partir do código. Onde o repositório não guarda o motivo original (por exemplo, por que AWS e não GCP ou Azure), a RFC diz isso e marca a comparação como avaliação a posteriori. Atributos de serviços que não estão no código vêm de conhecimento geral e **não foram verificados** contra preços ou limites vigentes.
- **Estrutura fixa.** Contexto · Opções consideradas · Decisão · Consequências · Referências.
- **Referências.** Formato `caminho:linha`, relativo à raiz do repositório indicado em negrito na seção. Os quatro repositórios da entrega são `infra-k8s`, `infra-db`, `lambda-auth` e `app-oficina` (ver o [contrato da Fase 3](../14-fase3-contrato.md)). As linhas valem para o estado dos repositórios em 19/10/2026.
- **Fato, inferência, lacuna.** Quando algo é lido do código, a RFC cita arquivo e linha. Quando é inferência de comportamento em runtime, está dito e marcado como não testado. Quando é lacuna, aparece em *Pendências*.

## Em caso de divergência

`docs/01-arquitetura.md` (menção a `infra/terraform/`), `docs/12-infra-deploy.md` e `docs/13-cicd-kubernetes.md` são anteriores à extração em quatro repositórios e ainda descrevem RDS na stack `cluster/`, GHCR e `KUBE_CONFIG`. O código vigente está em `infra-k8s`, `infra-db`, `lambda-auth` e no `deploy.yml`/`ci.yml` do `app-oficina`, com ECR e OIDC. Vale o código.

## Relacionados

- [Contrato entre os repositórios da Fase 3](../14-fase3-contrato.md)
- [ADRs](../16-adrs/): decisões arquiteturais sobre os quatro repositórios, a comunicação entre eles e a escalabilidade com HPA.
