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




=========

now add some secrets that contain sensitive data:

(following can to be replaced using some templating approach)

1. tbs registry details

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  annotations:
    tkg.tanzu.vmware.com/tanzu-package: tbs-tap-install
  name: tbs-tap-install-values
  namespace: tap-install
stringData:
  tbs-values.yaml: |
    ---
    kp_default_repository: $REGISTRY_SERVER
    kp_default_repository_username: $REGISTRY_USERNAME
    kp_default_repository_password: $REGISTRY_PASSWORD
    tanzunet_username: $TANZU_NET_USER
    tanzunet_password: $TANZU_NET_PASSWORD
type: Opaque
EOF
```

2. metadata cert

run:
```bash
export METADATA_STORE_TOKEN=$(kubectl get secrets -n tap-install -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='metadata-store-tap-install-sa')].data.token}" | base64 -d)
```
```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: metadata-store-secret
  namespace: scan-link-system
type: kubernetes.io/opaque
stringData:
  token: $METADATA_STORE_TOKEN
EOF
```

3. run:
```bash
kubectl get secret app-tls-cert -n metadata-store -o json | jq -r '.data."ca.crt"' | base64 -d > DELETEME
```
```yaml
cat <<EOF |  kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  annotations:
    tkg.tanzu.vmware.com/tanzu-package: scan-controller-tap-install
  name: scan-controller-tap-install-values
  namespace: tap-install
stringData:
  scst-scan-controller-values.yaml: |
    ---
    metadataStoreUrl: https://metadata-store-app.metadata-store.svc.cluster.local:8443
    metadataStoreCa: |-
$(awk '{printf "      %s\n", $0}' < DELETEME)
    metadataStoreTokenSecret: metadata-store-secret
type: Opaque
EOF
```
```bash
rm DELETEME
```

watch until package reconciles:
```bash
kubectl get packagerepositories.packaging.carvel.dev -n tap-install --watch
```
watch until installs reconcile:
```bash
kubectl get packageinstalls.packaging.carvel.dev -A
```
####

Caveats

- defaulted to local (kind) installs
- no Cosign secrets setup