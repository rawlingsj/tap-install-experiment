```
kubectl create ns tap-install
```
```
tanzu imagepullsecret add tap-registry \
  --username <TANZU_NET_USER> --password <TANZU_NET_PASSWORD> \
  --registry registry.tanzu.vmware.com \
  --export-to-all-namespaces
```
```
tanzu imagepullsecret add registry-credentials  \
  --username <REGISTRY_USERNAME> --password <REGISTRY_PASSWORD> \
  --registry <REGISTRY_SERVER> \
  --export-to-all-namespaces
```
#kubectl apply -f versionstream/tap-install/values/local/

#kubectl apply -f versionstream/tap-install/packages/

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



# valdate

kubectl get PackageInstall -n tap-install