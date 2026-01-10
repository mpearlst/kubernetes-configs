# Kubernetes-configs

## Talos setup

On your host machine, the one where you will be issuing commands from, you'll want to make a `patch.yaml`:
```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true
```

You'll then want to set a few variables to make the following commands work with your personal configuration:
```sh
export TALOS_IP="192.168.10.30"
export TALOS_API_PORT="6443"
export CLUSTER_NAME="funland"
```

You'll want to let this run for a while, should take about 5ish minutes to fully finish, then move onto Cilium
```sh
talosctl gen config "${CLUSTER_NAME}" "https://${TALOS_IP}:${TALOS_API_PORT}" --config-patch @patch.yaml

talosctl apply-config --insecure -n "${TALOS_IP}" --file controlplane.yaml

talosctl bootstrap --nodes "${TALOS_IP}" --endpoints "${TALOS_IP}" --talosconfig=./talosconfig

talosctl kubeconfig --nodes "${TALOS_IP}" --endpoints "${TALOS_IP}" --talosconfig=./talosconfig
```

## Cilium install

The cluster WILL NOT show as ready for the control plane, so just run the following commands when you see the node added, and can `kubectl get nodes`.  

Install the CRDs, add the helm repo, and create the namespace:
```sh
kubectl create namespace certificate

helm repo add cilium https://helm.cilium.io/

kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.1/standard-install.yaml
kubectl apply --server-side -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.4.1/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
```

Installing Cilium is pretty straight forward, but I have not been able to get it to work on another namespace.
```sh
export cilium_applicationyaml=$(curl -sL "https://raw.githubusercontent.com/mpearlst/kubernetes-configs/refs/heads/main/funland/kube-system/cilium/application.yaml" | yq eval-all '. | select(.metadata.name == "cilium-application" and .kind == "Application")' -)
export cilium_name=$(echo "$cilium_applicationyaml" | yq eval '.metadata.name' -)
export cilium_chart=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.chart' -)
export cilium_repo=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.repoURL' -)
export cilium_namespace=$(echo "$cilium_applicationyaml" | yq eval '.spec.destination.namespace' -)
export cilium_version=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.targetRevision' -)
export cilium_values=$(echo "$cilium_applicationyaml" | yq eval '.spec.source.helm.valuesObject' -)

echo "$cilium_values" | helm template $cilium_name $cilium_chart --repo $cilium_repo --version $cilium_version --namespace $cilium_namespace --values - | kubectl apply --namespace $cilium_namespace --filename -
```

## Argocd Setup
```sh
export argocd_applicationyaml=$(curl -sL "https://raw.githubusercontent.com/mpearlst/kubernetes-configs/refs/heads/main/funland/argocd/argocd/application.yaml" | yq eval-all '. | select(.metadata.name == "argocd-application" and .kind == "Application")' -)
export argocd_name=$(echo "$argocd_applicationyaml" | yq eval '.metadata.name' -)
export argocd_chart=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.chart' -)
export argocd_repo=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.repoURL' -)
export argocd_namespace=$(echo "$argocd_applicationyaml" | yq eval '.spec.destination.namespace' -)
export argocd_version=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.targetRevision' -)
export argocd_values=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.helm.valuesObject' - | yq eval 'del(.configs.cm)' -)
export argocd_config=$(curl -sL "https://raw.githubusercontent.com/mpearlst/kubernetes-configs/refs/heads/main/funland/argocd/argocd/appset.yaml" | yq eval-all '. | select(.kind == "AppProject" or .kind == "ApplicationSet")' -)


echo "$argocd_values" | helm template $argocd_name $argocd_chart --repo $argocd_repo --version $argocd_version --namespace $argocd_namespace --values - | kubectl apply --namespace $argocd_namespace --filename -

echo "$argocd_config" | kubectl apply --filename -
```