## https://kargo.akuity.io/quickstart

```shell
./kind.sh

## Cleaning up
kind delete cluster --name kargo-quickstart

https://localhost:31443
https://localhost:31444

arch=$(uname -m)
echo $arch
[ "$arch" = "x86_64" ] && arch=amd64
curl -L -o kargo https://github.com/akuity/kargo/releases/latest/download/kargo-$(uname -s | tr '[:upper:]' '[:lower:]')-${arch}
curl -L -o kargo https://github.com/akuity/kargo/releases/latest/download/kargo-darwin-arm64
chmod +x kargo
sudo mv kargo /usr/local/bin

kargo version

export GITOPS_REPO_URL=https://github.com/safe2008/kargo-demo
export GITHUB_USERNAME=safe2008
export GITHUB_PAT=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

kargo login https://localhost:31444 \
  --admin \
  --password admin \
  --insecure-skip-tls-verify

cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: kargo-demo
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - stage: dev
      - stage: qa
      - stage: uat
      - stage: prod
  template:
    metadata:
      name: kargo-demo-{{stage}}
      annotations:
        kargo.akuity.io/authorized-stage: kargo-demo:{{stage}}
    spec:
      project: default
      source:
        repoURL: ${GITOPS_REPO_URL}
        targetRevision: stage/{{stage}}
        path: stages/{{stage}}
      destination:
        server: https://kubernetes.default.svc
        namespace: kargo-demo-{{stage}}
      syncPolicy:
        syncOptions:
        - CreateNamespace=true
EOF

kargo create project kargo-demo

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: kargo-demo-repo
  namespace: kargo-demo
  labels:
    kargo.akuity.io/secret-type: repository
stringData:
  type: git
  url: ${GITOPS_REPO_URL}
  username: ${GITHUB_USERNAME}
  password: ${GITHUB_PAT}
EOF


cat <<EOF | kargo apply -f -
apiVersion: kargo.akuity.io/v1alpha1
kind: Warehouse
metadata:
  name: kargo-demo
  namespace: kargo-demo
spec:
  subscriptions:
  - image:
      repoURL: nginx
      semverConstraint: ^1.24.0
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: dev
  namespace: kargo-demo
spec:
  subscriptions:
    warehouse: kargo-demo
  promotionMechanisms:
    gitRepoUpdates:
    - repoURL: ${GITOPS_REPO_URL}
      writeBranch: stage/dev
      kustomize:
        images:
        - image: nginx
          path: stages/dev
    argoCDAppUpdates:
    - appName: kargo-demo-dev
      appNamespace: argocd
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: qa
  namespace: kargo-demo
spec:
  subscriptions:
    upstreamStages:
    - name: dev
  promotionMechanisms:
    gitRepoUpdates:
    - repoURL: ${GITOPS_REPO_URL}
      writeBranch: stage/qa
      kustomize:
        images:
        - image: nginx
          path: stages/qa
    argoCDAppUpdates:
    - appName: kargo-demo-qa
      appNamespace: argocd
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: uat
  namespace: kargo-demo
spec:
  subscriptions:
    upstreamStages:
    - name: qa
  promotionMechanisms:
    gitRepoUpdates:
    - repoURL: ${GITOPS_REPO_URL}
      writeBranch: stage/uat
      kustomize:
        images:
        - image: nginx
          path: stages/uat
    argoCDAppUpdates:
    - appName: kargo-demo-uat
      appNamespace: argocd
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: prod
  namespace: kargo-demo
spec:
  subscriptions:
    upstreamStages:
    - name: uat
  promotionMechanisms:
    gitRepoUpdates:
    - repoURL: ${GITOPS_REPO_URL}
      writeBranch: stage/prod
      kustomize:
        images:
        - image: nginx
          path: stages/prod
    argoCDAppUpdates:
    - appName: kargo-demo-prod
      appNamespace: argocd
EOF

kargo get warehouses --project kargo-demo

kargo get stages --project kargo-demo

kargo get freight --project kargo-demo

export FREIGHT_ID=$(kargo get freight --project kargo-demo --output jsonpath={.id})
kargo stage promote --project kargo-demo dev --freight $FREIGHT_ID
kargo get promotions --project kargo-demo
kargo get stages --project kargo-demo
kargo get stage dev --project kargo-demo --output jsonpath-as-json={.status}
kargo get freight $FREIGHT_ID --project kargo-demo --output jsonpath-as-json={.status}

kargo stage promote --project kargo-demo qa --freight $FREIGHT_ID
kargo get promotions --project kargo-demo

kargo stage promote --project kargo-demo uat --freight $FREIGHT_ID
kargo get promotions --project kargo-demo

kargo stage promote --project kargo-demo prod --freight $FREIGHT_ID
kargo get promotions --project kargo-demo

uat: localhost:30082
prod: localhost:30083

kind delete cluster --name kargo-quickstart


```