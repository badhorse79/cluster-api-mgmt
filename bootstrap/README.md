# Cluster api management

## Bootstrap

kind create cluster

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

export HCLOUD_TOKEN=<token>
kubectl create secret generic hetzner --from-literal=hcloud=$HCLOUD_TOKEN -n talos-cluster
kubectl patch secret hetzner -p '{"metadata":{"labels":{"clusterctl.cluster.x-k8s.io/move":""}}}' -n talos-cluster

clusterctl get kubeconfig -n talos-cluster talos-cluster > kubeconfig

kubectl get secret -n talos-cluster talos-cluster-talosconfig -o jsonpath='{.data.talosconfig}' | base64 -d > cluster-talosconfig
talosctl config merge cluster-talosconfig

export TALOS_CONTROL_PLANE_IP=`kubectl --kubeconfig kubeconfig get nodes -o wide | grep control-plane | awk -F' ' '{print $6}'`
talosctl -n $TALOS_CONTROL_PLANE_IP version
talosctl -n $TALOS_CONTROL_PLANE_IP dashboard


kubectl --kubeconfig kubeconfig get secret -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d