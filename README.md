# platform-cluster-config

Configuração do cluster, mantida pelo time de plataforma: Argo CD, políticas e onboarding
de produtos. Só a branch `main`, só por PR.

Os manifestos dos serviços ficam em outro repo, o
[platform-gitops](https://github.com/gerffeson-lab/platform-gitops), escrito pela esteira.
A separação é uma fronteira de permissão: o bot da esteira escreve manifestos, nunca
política nem quota.

```
bootstrap/
  argocd/        instala o Argo CD (no OpenShift: Subscription do OpenShift GitOps)
  root.yaml      app-of-apps: o Argo CD passa a gerenciar cluster/
cluster/
  politicas/     ValidatingAdmissionPolicy (nativas, sem Kyverno)
  produtos/      1 arquivo por produto: namespaces por ambiente, quota, AppProject, ApplicationSet
kind/            cluster local para testar
```

## Como um serviço chega ao cluster

1. Produto novo entra no catálogo do self-service, e a plataforma adiciona
   `cluster/produtos/<produto>.yaml`.
2. A esteira do serviço renderiza o chart da plataforma e grava em
   `platform-gitops`, na branch do ambiente, a pasta `manifests/<produto>/<serviço>/`.
3. O `ApplicationSet` do produto acha a pasta e cria a Application
   `<serviço>-<ambiente>` no namespace `<produto>-<ambiente>`.

| Branch do serviço | Branch em platform-gitops | Ambiente |
|---|---|---|
| `develop` | `develop` | dev |
| `release/*` | `release` | hml |
| `main` | `main` | prd |

O Argo CD consulta o Git a cada 3 minutos. Com um webhook do GitHub para o Argo CD, a
pasta nova vira Application em segundos.

Uma pasta que some não apaga nada (`applicationsSync: create-update`): descomissionar um
serviço é um passo explícito, não um efeito colateral de um commit errado.

## Guardrails

| Regra | Onde |
|---|---|
| Container com limits, até 2 CPU / 4Gi | VAP `recursos-por-container` (barra o Deployment, o erro aparece no sync do Argo CD) e `LimitRange` (barra o pod) |
| Imagem do registry da plataforma, com versão fixa | VAP `imagem-confiavel` |
| Sem root, sem privilégios | Pod Security `restricted` no namespace (no OpenShift: SCC `restricted-v2`) |
| Consumo total do produto por ambiente | `ResourceQuota` |
| Produto só implanta no que é dele | `AppProject`: só `<produto>-*`, sem recursos de cluster, sem mexer em quota |

As políticas valem para namespaces com o label `plataforma.acme/produto`.

## Testar localmente (kind)

```bash
kind create cluster --config kind/cluster.yaml
kubectl apply -k bootstrap/argocd --server-side
kubectl -n argocd wait --for=condition=Available deploy --all --timeout=300s
kubectl apply -f bootstrap/root.yaml

# console do Argo CD em https://localhost:8443 (usuário admin)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
kubectl -n argocd port-forward svc/argocd-server 8443:443
```

Cluster de 1 nó com o Argo CD: cerca de 1,5–3 GB de RAM. `kind delete cluster --name plataforma` para liberar.

## No OpenShift do banco

| Aqui | No banco |
|---|---|
| Argo CD da comunidade (`bootstrap/argocd`) | Operator OpenShift GitOps |
| Pod Security `restricted` | SCC `restricted-v2` (já é o padrão) |
| 3 ambientes num cluster, namespace por ambiente | Cluster por ambiente: o ApplicationSet de cada um lista só o seu ambiente (ou o ACM distribui) |
| Repo público no GitHub | Repo `internal`/privado; o Argo CD lê com um GitHub App só de leitura |
