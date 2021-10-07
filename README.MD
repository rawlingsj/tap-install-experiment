set environment variables
```
export TANZU_NET_USER=
export TANZU_NET_PASSWORD=
export REGISTRY_SERVER=
export REGISTRY_USERNAME=
export REGISTRY_PASSWORD=
export GITHUB_USER=
export GITHUB_TOKEN=
```

```
kind create cluster --name tap
```
```
kapp deploy --yes -a kc -f https://github.com/vmware-tanzu/carvel-kapp-controller/releases/download/v0.27.0/release.yml

kapp deploy --yes -a sg -f https://github.com/vmware-tanzu/carvel-secretgen-controller/releases/download/v0.5.0/release.yml

kapp deploy --yes -a cert-manager -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
```

```
kubectl create ns tap-install
```
```
tanzu imagepullsecret add tap-registry \
  --username $TANZU_NET_USER --password $TANZU_NET_PASSWORD \
  --registry registry.tanzu.vmware.com \
  --export-to-all-namespaces

tanzu imagepullsecret add registry-credentials  \
  --username $REGISTRY_USERNAME --password $REGISTRY_PASSWORD \
  --registry $REGISTRY_SERVER \
  --export-to-all-namespaces
```

```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=tap-install-experiment \
  --branch=main \
  --path=./clusters/my-cluster \
  --personal
```

kubectl get packagerepositories.packaging.carvel.dev -A
kubectl get packageinstalls.packaging.carvel.dev -A

=========

now some customisations:


kubectl edit secret scan-controller-tap-install-values -n tap-install

The `metadataStoreCa` value can be determined by:

```
kubectl get secret app-tls-cert -n metadata-store -o json | jq -r '.data."ca.crt"' | base64 -d
```

Create a metadata-store-secret.yaml for this secret:
```
---
apiVersion: v1
kind: Secret
metadata:
  name: metadata-store-secret
  namespace: scan-link-system
type: kubernetes.io/opaque
stringData:
  token: <METADATA_STORE_TOKEN>
```

The METADATA_STORE_TOKEN value can be determined by:
```
kubectl get secrets -n tap-install -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='metadata-store-tap-install-sa')].data.token}" | base64 -d
```

kubectl apply -f metadata-store-secret.yaml

####

Caveats

no Cosign secrets setup