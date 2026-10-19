# RotaExpress — Análise das medições com 1, 2 e 3 Pods

Entregáveis: `k8s/` (manifestos), `resultados/` (logs do k6 e evidências), `resultados_modelo.csv` (tabela preenchida) e este arquivo.

## 1. Cenário

| Item | Valor |
|---|---|
| Computador | MacBook com Apple M2 (8 núcleos, arm64), 8 GB de RAM, macOS |
| Docker | Docker Desktop 29.7.2 (VM com 8 CPUs e ~3,8 GiB) |
| Cluster | Minikube v1.39.0, perfil `ops-lab`, driver docker, `--cpus=3 --memory=3072`, Kubernetes v1.37.0, containerd |
| Recursos por Pod | requests = limits de CPU em 500m; memória 64Mi (request) / 128Mi (limite); QoS Burstable |
| Gerador | Job k6 `grafana/k6:1.0.0` no mesmo cluster (250m request / 500m limite) |
| Demanda | `TAXA=6` pedidos/s, `PARADAS=9`, `DURACAO=60` s (+10 s de aquecimento) |

**Por que TAXA = 6.** Sem limite de CPU, um cálculo levou 184 ms (Grupo 2). Com meia CPU, levou de 280 a 340 ms, ou seja, um Pod atende no máximo cerca de 3 pedidos/s e três Pods cerca de 9. O piloto com 1 Pod e TAXA 6 teve 55% de falhas, 162 sucessos e 0 iterações descartadas. A taxa é alta o bastante para saturar 1 Pod e cabe na capacidade teórica de 3 Pods, então foi mantida em todas as 9 medições.

## 2. Evidências

Medianas das três rodadas (A, B, C) de cada configuração:

| Pods | Sucessos/s (`rotas_ok/60`) | Falhas | p95 `latencia_ok` | Iterações descartadas | CPU reservada |
|---|---|---|---|---|---|
| 1 | **2,73** | **54,6%** | 360 ms | 0 | 0,5 CPU |
| 2 | **3,75** | **37,7%** | 378 ms | 0 | 1,0 CPU |
| 3 | **4,40** | **26,9%** | 408 ms | 0 | 1,5 CPU |

- **Repetibilidade.** As três rodadas de cada configuração ficaram próximas. Sucessos/s variaram de 2,60 a 2,88 com 1 Pod, de 3,68 a 3,80 com 2 e de 4,38 a 4,47 com 3. Os valores individuais estão no CSV.
- **Metas.** p95 < 800 ms e 0 iterações descartadas foram cumpridos em todas as execuções. A meta de falhas < 1% não foi cumprida em nenhuma: o k6 saiu com código 99 e todos os Jobs aparecem como `Failed`, mas com o resumo completo nos logs.
- **Distribuição entre Pods.** Pela diferença dos contadores `total_aceitos` antes e depois de cada teste, a carga ficou equilibrada. Exemplos: com 3 Pods, 96 / 101 / 106 e 103 / 103 / 100; com 2 Pods, 130 / 134 e 127 / 142.
- **CPU (`kubectl top`).** Os Pods de aplicação chegaram a picos de 400 a 480m com 1 Pod, perto do limite de 500m. Com 2 e 3 Pods ficaram tipicamente entre 200 e 350m cada. As amostras do metrics-server chegam com atraso e repetidas, por isso servem só como indício.

## 3. Explicação

**Observado.**
- Mais réplicas aumentaram os sucessos e reduziram as falhas, sem piorar a latência dos sucessos (p95 estável entre 340 e 480 ms).
- O ganho diminui a cada réplica. De 1 para 2 Pods foram +1,02 sucessos/s com +0,5 CPU; de 2 para 3, só +0,65 sucessos/s com o mesmo +0,5 CPU.
- Mesmo com 3 Pods, cuja capacidade teórica (~9/s) é maior que a demanda (6/s), 27% dos pedidos falharam.
- O p95 baixo esconde as falhas: os 503 são respostas instantâneas e não entram em `latencia_ok`.

**Hipótese (coerente com os dados, não testada isoladamente).** As falhas não vêm de falta de CPU total. Elas vêm do desenho da aplicação: um cálculo por vez, sem fila, e um Service que escolhe o Pod sem saber se ele está ocupado. Um pedido que cai num Pod que está calculando recebe 503 na hora, mesmo havendo outro Pod livre. Com cada Pod ocupado cerca de 0,31 s por pedido, a chance de encontrar o Pod ocupado cai devagar conforme réplicas são adicionadas. É um sistema "com perda", não com espera. Por isso replicar ajuda, mas com retorno cada vez menor.

**Outra observação do ambiente.** Com o driver docker no macOS, o nó informa 8 CPUs alocáveis (os núcleos da VM do Docker), embora o contêiner do nó esteja limitado a 3 CPUs pelo Docker (`NanoCpus=3e9`). Por isso o passo 8.1 com 6 réplicas não gerou Pods `Pending`: todas couberam. Foi preciso pedir 16 réplicas para ver `0/1 nodes are available: 1 Insufficient cpu`, com os requests do nó em 99% (evidência em `resultados/limite-no.txt`). Consequência: o scheduler aceitaria mais Pods do que o nó de fato consegue executar. Nas medições, 3 Pods mais o k6 somam no máximo 2 CPUs de limite, dentro das 3 reais.

## 4. Decisão

**Configuração adotada para esta demanda: 3 Pods (1,5 CPU reservada).** Foi a melhor das testadas: +61% de sucessos em relação a 1 Pod, p95 dentro da meta e carga equilibrada. **Nenhuma configuração atingiu a meta de falhas < 1%**, então 3 Pods é uma redução de dano, não o atendimento das metas.

**Próxima investigação.** Aumentar réplicas não parece ser o caminho. Pela tendência medida, chegar a < 1% de falhas exigiria muito mais Pods do que cabem no nó. O que deve mudar o resultado:
1. **Fila ou espera curta na aplicação**, em vez de 503 imediato. Também é possível aceitar mais de um cálculo por Pod e usar CPU adicional.
2. **Retentativa no cliente** para 503, com espera curta. É preciso medir de novo para ver o efeito no p95.
3. **Balanceamento que considere a ocupação** (por exemplo least-connections via Ingress ou service mesh), em vez da escolha aleatória por conexão.
4. **Algoritmo mais eficiente** que força bruta, o que reduz o tempo ocupado por pedido.
5. Só depois disso, um HPA por CPU com `min`/`max` baseados nestas medições.

**Limite da conclusão.** Tudo roda em um único nó. Três Pods aqui não protegem contra a queda do nó. O laboratório mede atendimento sob carga, não recuperação de desastre.

## Registro das etapas de operação

- **Grupo 3.** Deployment aplicado em duas versões. O `kubectl delete pod` mostrou o ReplicaSet recriando o Pod com outro IP (10.244.0.4 → 10.244.0.5). A segunda versão gerou novo ReplicaSet (rolling update) e QoS Burstable.
- **Grupo 4.** EndpointSlice listou o IP do Pod. `wget http://rotas:8000/healthz` de dentro do cluster e o port-forward funcionaram. O cálculo pelo port-forward levou 341 ms com meia CPU.
- **Grupo 5.** `rotas:1.1` ficou em `ErrImagePull`/`ImagePullBackOff` (`pull access denied`). O Pod antigo continuou atendendo durante a falha. O `rollout undo` restaurou `rotas:1.0` e o histórico passou a ser 1, 3, 4.
