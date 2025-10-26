# Cluster api management

Requirements:
 - helm
 - clusterctl
 - talosctl
 - kind
 - kubectl
 - cilium cli
 - hubble cli
 
Also:
 - hetzner project with api token and ssh key
 - talos snapshot in your project


## Bootstrap

```bash
# create kind cluster
kind create cluster

# prep helm repositories
helm repo add capi-operator https://kubernetes-sigs.github.io/cluster-api-operator
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update

# install cert-manager and capi-operator on local cluster
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set installCRDs=true
helm install capi-operator capi-operator/cluster-api-operator --create-namespace -n capi-operator-system -f capi-values.yaml --wait --timeout 90s

# This is where I should create a management template cluster and move mgmt cluster to that
...

# Deploy argo on mgmt cluster
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

## you can port forward to argo ui
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0
## get argo secret from local cluster
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d


## Apply workload cluster application on cluster
kubectl apply -n argocd -f talos-cluster-application.yaml
## Create hetzner secret after cluster namespace have been created
## really should be part of argo deploy
export HCLOUD_TOKEN=<token>
kubectl create secret generic hetzner --from-literal=hcloud=$HCLOUD_TOKEN -n talos-cluster
kubectl patch secret hetzner -p '{"metadata":{"labels":{"clusterctl.cluster.x-k8s.io/move":""}}}' -n talos-cluster

# cluster should know be created
kubectl get cluster -n talos-cluster
clusterctl describe cluster talos-cluster -n talos-cluster

# we can also get kubeconfig for workload cluster
clusterctl get kubeconfig -n talos-cluster talos-cluster > kubeconfig

# and talos config
kubectl get secret -n talos-cluster talos-cluster-talosconfig -o jsonpath='{.data.talosconfig}' | base64 -d > cluster-talosconfig
talosctl config merge cluster-talosconfig
export TALOS_CONTROL_PLANE_IP=`kubectl --kubeconfig kubeconfig get nodes -o wide | grep control-plane | awk -F' ' '{print $6}'`
talosctl -n $TALOS_CONTROL_PLANE_IP version
talosctl -n $TALOS_CONTROL_PLANE_IP dashboard

# when cluster is up connect to it
k9s --kubeconfig kubeconfig

# get argo secret from workload cluster
kubectl --kubeconfig kubeconfig get secret -n argocd argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d

# portforward to argo on workload cluster
kubectl --kubeconfig kubeconfig port-forward svc/argocd-server -n argocd 8081:443 --address=0.0.0.0

# get load balancer ip
#old nginx: kubectl --kubeconfig kubeconfig get service ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
#cilium: 
kubectl --kubeconfig kubeconfig get service cilium-ingress -n kube-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# test connection
curl -H 'Host: hello.local' http://91.98.13.171

# cilium
cilium --kubeconfig kubeconfig status --wait # get status

# hubble
hubble --kubeconfig kubeconfig status
hubble --kubeconfig kubeconfig observe

# auto portforward to hubble ui
cilium --kubeconfig kubeconfig hubble ui

# cleanup
kubectl delete cluster talos-cluster -n talos-cluster
kind delete cluster
# manually delete ingress loadbalancer since it doesn't seem to be deleted
``` 

## Update machine templates

Machine templates are mostly immutable, so changes like, talos upgrades, should be done by creating new template and pointing machines to the new template.

Example update worker template:

1. Create new TalosConfigTemplate with another name eg. talos-cluster-worker-2
2. Update worker MachineDeployment with the new bootstrap template, ie. look to `kind: TalosConfigTemplate` and change `name:` to `name: talos-cluster-worker-2`
3. New machines should be rolled out
