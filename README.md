## https://kargo.akuity.io/quickstart

```shell
curl -L https://raw.githubusercontent.com/akuity/kargo/main/hack/quickstart/k3d.sh | sh

localhost:31443
localhost:31444

arch=$(uname -m)
echo $arch
[ "$arch" = "x86_64" ] && arch=amd64
curl -L -o kargo https://github.com/akuity/kargo/releases/latest/download/kargo-$(uname -s | tr '[:upper:]' '[:lower:]')-${arch}
curl -L -o kargo https://github.com/akuity/kargo/releases/latest/download/kargo-darwin-arm64
chmod +x kargo
sudo mv kargo /usr/local/bin

kargo version

export GITOPS_REPO_URL=https://github.com/safe2008/kargo-demo.git
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
      - stage: test
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


```