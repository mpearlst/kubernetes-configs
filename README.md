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
export TALOS_IP="10.0.0.73"
export TALOS_API_PORT="6443"
export CLUSTER_NAME="makko"
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

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/experimental/gateway.networking.k8s.io_gateways.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml \
  -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.1/config/crd/experimental/gateway.networking.k8s.io_grpcroutes.yaml
```
Installing Cilium is pretty straight forward, but I have not been able to get it to work on another namespace. Be sure to update the version to whatever you want.

```sh
helm upgrade \
    cilium \
    cilium/cilium \
    --version 1.17.2 \
    --namespace kube-system \
    --set ipam.mode=kubernetes \
    --set kubeProxyReplacement=true \
    --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
    --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
    --set cgroup.autoMount.enabled=false \
    --set cgroup.hostRoot=/sys/fs/cgroup \
    --set k8sServiceHost=localhost \
    --set k8sServicePort=7445 \
    --set l2announcements.enabled=true \
    --set gatewayAPI.enabled=true \
    --set gatewayAPI.externalTrafficPolicy=Local \
    --set gatewayAPI.secretsNamespace.create=false \
    --set gatewayAPI.secretsNamespace.name=certificate \
    --install
```

## Argocd Setup

This is grabbing all the relevant info from the git repo, and then applying all of it to the cluster.
```sh
export argocd_applicationyaml=$(curl -sL "https://raw.githubusercontent.com/Senk02/kubernetes-configs/refs/heads/main/makko/argocd/argocd/application.yaml" | yq eval-all '. | select(.metadata.name == "argocd-application" and .kind == "Application")' -)
export argocd_name=$(echo "$argocd_applicationyaml" | yq eval '.metadata.name' -)
export argocd_chart=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.chart' -)
export argocd_repo=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.repoURL' -)
export argocd_namespace=$(echo "$argocd_applicationyaml" | yq eval '.spec.destination.namespace' -)
export argocd_version=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.targetRevision' -)
export argocd_values=$(echo "$argocd_applicationyaml" | yq eval '.spec.source.helm.valuesObject' - | yq eval 'del(.configs.cm)' -)
export argocd_config=$(curl -sL "https://raw.githubusercontent.com/Senk02/kubernetes-configs/refs/heads/main/makko/argocd/argocd/appset.yaml" | yq eval-all '. | select(.kind == "AppProject" or .kind == "ApplicationSet")' -)

echo "$argocd_values" | helm template $argocd_name $argocd_chart --repo $argocd_repo --version $argocd_version --namespace $argocd_namespace --values - | kubectl apply --namespace $argocd_namespace --filename -

echo "$argocd_config" | kubectl apply --filename -
```