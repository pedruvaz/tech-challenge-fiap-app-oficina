# ADR-003 · Escalabilidade com HPA

> [← Voltar aos ADRs](./README.md)

## Status

Aceito, 2026-10-19.

## Contexto

A API roda no EKS como um Deployment (`k8s/40-api-deployment.yaml`) atrás de um Service `LoadBalancer` (NLB). A autenticação é por JWT, sem sessão no servidor, e o schema do banco é aplicado por um Job separado, então réplicas adicionais não carregam estado local. Escalabilidade automática é um requisito explícito da fase (comentários em `infra-k8s/terraform/cluster/addons.tf` e `variables.tf`).

O cluster é EKS em Auto Mode (`compute_config` com `node_pools = ["general-purpose"]`, sem node group configurado). Há dois níveis de escala a resolver: o número de pods e o número de nós.

## Decisão

### Pods: HorizontalPodAutoscaler por CPU

`k8s/50-hpa.yaml` (`autoscaling/v2`, namespace `tech-challenge`, alvo `Deployment/api`):

| Parâmetro | Valor |
| --------- | ----- |
| `minReplicas` | 2 |
| `maxReplicas` | 6 |
| Métrica | `Resource` `cpu`, `Utilization`, `averageUtilization: 70` |
| `behavior.scaleDown.stabilizationWindowSeconds` | 120 (o default do Kubernetes é 300) |
| `behavior.scaleUp` | não configurado (default do Kubernetes) |

Memória não é métrica de escala. A utilização é calculada sobre o `request` do container (`100m` de CPU; `limits` de `500m`, `256Mi`/`512Mi` de memória), então o alvo é ~70m de CPU médios por pod.

Elementos do Deployment e do PDB que acompanham o HPA:

- `replicas: 2` fixo, igual ao `minReplicas`.
- `RollingUpdate` com `maxUnavailable: 0` e `maxSurge: 1`; `topologySpreadConstraints` por zona com `ScheduleAnyway` (preferência, não exigência, para não deixar pod `Pending` se houver nós em uma só AZ).
- `PodDisruptionBudget` `api` com `minAvailable: 1`, para que o drain de um nó nunca derrube todas as réplicas.
- `preStop` com `sleep 5`, `terminationGracePeriodSeconds: 30` e `enableShutdownHooks()` no Nest, para que o scale-down não corte requisições em voo.
- O `deploy.yml` aplica `k8s/50-hpa.yaml` e `k8s/60-pdb.yaml` junto com o Deployment e o Service, depois da migration.

### Fonte das métricas: metrics-server instalado pelo `infra-k8s`

A stack `infra-k8s/terraform/cluster/addons.tf` instala o metrics-server por `helm_release` (chart `3.12.2`, namespace `kube-system`), controlado por `var.install_metrics_server`, com default `true`. O comentário no código registra que **não está confirmado** se o EKS Auto Mode já entrega o metrics-server; o teste prescrito é `kubectl top nodes` depois do primeiro `apply`, e, se responder, trocar a variável para `false`. Esse teste não está registrado no repositório.

Sem a API `metrics.k8s.io` o HPA fica com o alvo `<unknown>` (condição `FailedGetResourceMetric`) e nunca escala: o Deployment permanece nas 2 réplicas do manifesto. O README raiz, o `docs/04-execucao.md` e o `docs/07-ci-cd.md` descrevem esse estado como esperado em kind, onde o metrics-server não é instalado.

### Nós: gerenciados pelo EKS Auto Mode

O `eks.tf` descreve o Auto Mode como compute gerenciado pela AWS "com Karpenter, Bottlerocket e IMDSv2 por padrão", e o README do `infra-k8s` fala em "escalabilidade de nós gerenciada pela AWS". **Nenhum repositório instala Cluster Autoscaler ou Karpenter próprio**, nem declara `NodePool` ou `NodeClass` customizado: o único pool habilitado é o `general-purpose` embutido. Quando o HPA cria pods que não cabem nos nós existentes, a provisão de nós fica a cargo do Auto Mode.

## Consequências

### Positivas

- **Disponibilidade mínima sem custo de capacidade ociosa grande.** O piso de 2 réplicas, somado ao PDB, ao `maxUnavailable: 0` e à distribuição por zona, mantém a API servindo durante rollout e drain.
- **Sem componente de nó para operar.** Não há Cluster Autoscaler, Karpenter nem node group no código: a escala de nós é do Auto Mode, e o `infra-k8s` só precisa instalar o metrics-server.
- **Teto de custo explícito.** `maxReplicas: 6` limita a escala horizontal e o pedido total de CPU (6 × 100m) a um valor previsível, coerente com a stack efêmera de ~US$ 130–160/mês.
- **O requisito de infraestrutura é versionado.** O metrics-server é provisionado pelo Terraform, não por passo manual esquecido, e o README do `infra-k8s` o lista como propósito da stack.
- **Scale-down mais ágil que o default.** A janela de 120 s (contra 300 s) devolve capacidade mais rápido, e o `preStop`/graceful shutdown evita perda de requisições na redução.

### Negativas

- **`replicas: 2` fixo no Deployment conflita com o HPA a cada reaplicação do manifesto.** O `deploy.yml` roda `kubectl apply -f k8s/40-api-deployment.yaml` em todo release, e o manifesto declara `spec.replicas: 2`. Pelo comportamento padrão do `kubectl apply`, um campo presente no manifesto e diferente do estado vivo é reescrito: se o HPA tinha levado o Deployment a, por exemplo, 5 réplicas sob carga, o deploy o reduz a 2 e o HPA só volta a escalar no próximo ciclo de reconciliação. Durante o rollout sob carga, isso é uma queda momentânea de capacidade provocada pelo próprio deploy. O mesmo vale para a reaplicação manual no kind. O dado vem do comportamento documentado do Kubernetes e do manifesto; **não foi observado neste cluster**. A correção usual é remover `spec.replicas` do Deployment e deixar o HPA como único dono do campo, mas isso muda o primeiro apply e o ambiente kind (sem metrics-server o Deployment cairia para 1 réplica, abaixo das duas que o comentário do PDB assume). Decisão em aberto, não tomada neste ADR.
- **A necessidade do `helm_release` do metrics-server não está confirmada.** O código não registra se o Auto Mode já fornece a API `metrics.k8s.io`; se fornecer, o `helm_release` é redundante e a variável deveria ser `false`. Sem nenhum dos dois, o HPA é inerte.
- **A escala nunca foi exercitada por teste.** O job `kind` do CI aplica o HPA sem metrics-server, então só valida que o manifesto é aceito, e o repositório não contém teste de carga. O que o ADR descreve do comportamento sob carga vem da configuração, não de medição.
- **Escalar a 70% do `request` é agressivo.** Com `request` de 100m e `limit` de 500m, o HPA reage a ~70m de uso médio por pod, bem abaixo do teto que o container poderia consumir. É uma consequência dos valores de `resources`, e o par `requests`/`limits` não tem justificativa de dimensionamento registrada.
- **Só CPU.** Pressão de memória (`limits: 512Mi`) não aumenta réplicas; um pod que estoure memória é reiniciado (OOMKill), não compensado pelo HPA.
- **Escalar a API não escala o banco.** O RDS é uma `db.t4g.micro` sem Multi-AZ. O `PrismaPg` é criado sem `max` em `src/prisma/prisma.service.ts`, então cada pod usa o tamanho de pool padrão do `pg` (10); em 6 réplicas isso são até 60 conexões, fora a migration e a Lambda. O limite de conexões da instância não está configurado nem verificado no repositório. O README do `lambda-auth` já aponta o limite baixo da instância como motivo para `max: 1` no pool da função.
- **Os nós novos levam tempo.** Como pods adicionais que não cabem nos nós atuais dependem do Auto Mode provisionar capacidade, o tempo de scale-out em carga real é maior do que o ciclo do HPA; não há medição desse tempo.
- **Documentação divergente do código.** `docs/13-cicd-kubernetes.md` descreve `maxReplicas: 10` e alvo de memória de 80%, o que não corresponde ao `k8s/50-hpa.yaml` (6 réplicas, só CPU).

## Referências

- `k8s/50-hpa.yaml`, `k8s/40-api-deployment.yaml`, `k8s/60-pdb.yaml`, `k8s/README.md` (seção "No EKS").
- `.github/workflows/deploy.yml`, passo "Aplica Deployment, Service, HPA e PDB".
- `infra-k8s/terraform/cluster/addons.tf` (`helm_release.metrics_server`), `variables.tf` (`install_metrics_server`) e `eks.tf` (Auto Mode, `node_pools`).
- `infra-k8s/README.md` (propósito da stack); `docs/07-ci-cd.md` (job `kind`); `src/prisma/prisma.service.ts`.
- [ADR-001](./adr-001-quatro-repositorios.md) para a organização dos repositórios e do cluster compartilhado entre homologação e produção.
