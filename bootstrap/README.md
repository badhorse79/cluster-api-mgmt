# Cluster api management

## Bootstrap

kind create cluster

export HCLOUD_TOKEN=<token>
kubectl create secret generic hetzner --from-literal=hcloud=$HCLOUD_TOKEN
kubectl patch secret hetzner -p '{"metadata":{"labels":{"clusterctl.cluster.x-k8s.io/move":""}}}'

helm repo add capi-operator https://kubernetes-sigs.github.io/cluster-api-operator
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update

helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
helm install capi-operator capi-operator/cluster-api-operator --create-namespace -n capi-operator-system -f capi-values.yaml --wait --timeout 90s

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

argocd admin initial-password -n argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443

kubectl apply -n argocd -f talos-cluster-application.yaml





