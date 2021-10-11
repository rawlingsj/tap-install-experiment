These are experimental docs that make up a proof of concept to enable a GitOps approach for installing packages for [TAP](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/0.2/tap-0-2/GUID-overview.html).

This experiment whilst is very generic has only been tested with a local [kind](https://kind.sigs.k8s.io/) cluster so other cloud providers are work in progress depending on the sucess of this experiment.

# Prerequisits

- Ensure you follow prerequisits 1,2 and 3 [here](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/0.2/tap-0-2/GUID-install-intro.html)
- [`kind`](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) - Minimum requirements: 8 CPUs for i9 or equivalent, 12 CPUs for i7 or equivalent, 8 GB RAM (12+ GB recommended), 120 GB disk space. If on DockerDesktop for Mac check your Resources under the Preferences section.
- [`flux cli`](https://fluxcd.io/docs/installation/#install-the-flux-cli)
- [`carvel tooling`](https://github.com/vmware-tanzu/carvel-kapp/releases/latest)
- signup to https://network.tanzu.vmware.com/

# Installation

1. Create your own cluster's GitOps repository from the GitHub template link below, the repository name you choose will be used when setting the `TAP_GITOPS_REPOSITORY` environment variable in step 2 below.  This repository contains the desired state your cluster should be in which aids tracabiliy, rollbacks and disaster recovery scenarios.

click > https://github.com/rawlingsj/tap-install-experiment/generate

2. Set local environment variables

| ENVIRONMENT VARIABLE | EXAMPLE | DESCRIPTION |
| -------------------- | ------- | ----------- |
|`TAP_GITOPS_REPOSITORY` | chucknorris | YOUR cluster Github repository name that you chose during step 1 above |
|`TANZU_NET_USER` | foo | username after signing up to https://network.tanzu.vmware.com/ in step 1 to pull Tanzu Application platform images |
|`TANZU_NET_PASSWORD` | bar123! | password after sigining up to https://network.tanzu.vmware.com/ in step 1 to pull Tanzu Application platform images
|`REGISTRY_SERVER` | docker.io | a container registry server and user / organisation to push images to
|`REGISTRY_USERNAME` | wine | a container registry username authenticatiom to push images to
|`REGISTRY_PASSWORD` | beer456! | a container registry password or token authenticatiom to push images to
|`GITHUB_USER` | chucknorris | github username to interact with github
|`GITHUB_TOKEN` | ghp_1234abcde | github personal access token used to interact with github see https://github.com/settings/tokens/new?scopes=repo,read:user,read:org,user:email,write:repo_hook,admin:repo_hook

open your terminal and set your environment variables:

```
export TAP_GITOPS_REPOSITORY=
export TANZU_NET_USER=
export TANZU_NET_PASSWORD=
export REGISTRY_SERVER=
export REGISTRY_USERNAME=
export REGISTRY_PASSWORD=
export GITHUB_USER=
export GITHUB_TOKEN=
```

4. Create a local kubernetes cluster
```
kind create cluster --name tap
```

5. Apply base set of installer tooling to the cluster (would be good to move certmanager + secrets manager into GitOps repo):
```
kapp deploy --yes -a kc -f https://github.com/vmware-tanzu/carvel-kapp-controller/releases/download/v0.27.0/release.yml

kapp deploy --yes -a sg -f https://github.com/vmware-tanzu/carvel-secretgen-controller/releases/download/v0.5.0/release.yml

kapp deploy --yes -a cert-manager -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
```

6. Create some incluster secrets that provide authentication to pull container images from the Tanzu Network and push to your personal container registry.

```
tanzu imagepullsecret add tap-registry \
  --username $TANZU_NET_USER --password $TANZU_NET_PASSWORD \
  --registry registry.tanzu.vmware.com \
  --export-to-all-namespaces

tanzu imagepullsecret add registry-credentials  \
  --username $REGISTRY_USERNAME --password $REGISTRY_PASSWORD \
  --registry $REGISTRY_SERVER \
  --namespace default
```

7. Install [Flux](https://github.com/fluxcd/] which will provision your cluster with the packages of [Tanzu Application Platform](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/0.2/tap-0-2/GUID-overview.html) via Git and [Flux](https://github.com/fluxcd/)

```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=$TAP_GITOPS_REPOSITORY \
  --branch=main \
  --path=./versionstream/tap-base-install \
  --personal
```

---

These next steps could be improved and automated but for the purpose of this experiment let's use a simple approach.

Now add three secrets that contain sensitive data so not stored in git (let's automate this soon):

(following can to be replaced using some better templating approach)

1. Tanzu Build Service secrets

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
    kp_default_repository: $REGISTRY_SERVER/$REGISTRY_USERNAME/build-service
    kp_default_repository_username: $REGISTRY_USERNAME
    kp_default_repository_password: $REGISTRY_PASSWORD
    tanzunet_username: $TANZU_NET_USER
    tanzunet_password: $TANZU_NET_PASSWORD
type: Opaque
EOF
```

2. Scan link secrets

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

3. Scan secrets

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
rm DELETEME
```

---

watch until package reconciles:
```bash
kubectl get packagerepositories.packaging.carvel.dev -n tap-install --watch
```
watch until installs reconcile:
```bash
kubectl get packageinstalls.packaging.carvel.dev -A --watch
```
# Caveats + extra work or discovery needed

- defaulted to local (kind) installs, other cloud provider setups will need to be configured and templated
- there are a number of manual steps still that relate to incluster secrets which can be automated
- no Cosign secrets setup
- github personal user setup, github organisatiom and other git provider support can be found https://fluxcd.io/docs/installation/#bootstrap
