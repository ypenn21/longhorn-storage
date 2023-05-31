# Overview

This repostiory is used to add Longhorn trait to an ACM-nabled Anthos cluster

> :warning: Verify there is NO `longhorn-system` namespace on the cluster BEFORE applying this Cluster Trait Repo

## How to use

Must apply the following `config-management-system` namespace (no async way to do this yet):

```yaml
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata:
  name: lh-trait-sync
  namespace: config-management-system
spec:
  sourceFormat: unstructured
  git:
    repo: "https://gitlab.com/gcp-solutions-public/retail-edge/available-cluster-traits/longhorn-anthos.git"
    branch: "main"
    dir: "/config"
    auth: "token"
    secretRef:
      name: longhorn-git-creds                # matches the secret below

---

apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: longhorn-git-creds-es
  namespace: config-management-system
spec:
  refreshInterval: 1m
  secretStoreRef:
    kind: ClusterSecretStore
    name: gcp-secret-store
  target:                                       # K8s secret definition
    name: longhorn-git-creds                    ############# Matches the secretRef above
    creationPolicy: Owner
  data:
  - secretKey: username                         # K8s secret key name inside secret
    remoteRef:
      key: longhorn-access-token-creds          #  GCP Secret Name
      property: username                        # field inside GCP Secret
  - secretKey: token                            # K8s secret key name inside secret
    remoteRef:
      key: longhorn-access-token-creds          #  GCP Secret Name
      property: token                           # field inside GCP Secret

```

### Create Required GCP Secret (secretRef above)

```
# One time only
gcloud secrets create longhorn-git-creds -n config-management-system --replication-policy="automatic"
export TOKEN="<token-name>"
export TOKEN_VALUE="<token-value>"
# this can be run multiple times, adds a new version each time
echo -n "{ \"username\":\"${TOKEN}\", \"token\":\"${TOKEN_VALUE}\" }" | gcloud secrets versions add longhorn-git-creds --data-file=-
```

### Create/Update Config folder (update the code to latest version)

```
nomos hydrate --source-format=unstructured --output=config --no-api-server-check
```

## Local Validation

Assuming `nomos` is installed (via `gcloud components install nomos`)

```
nomos vet --no-api-server-check --source-format=unstructured --path config/
```

### Docker method

Using this link to find the version of nomos-docker:  https://cloud.google.com/anthos-config-management/docs/how-to/updating-private-registry#expandable-1

```
docker pull gcr.io/config-management-release/nomos:latest
docker run -it -v $(pwd):/code/ gcr.io/config-management-release/nomos:latest nomos vet --no-api-server-check --source-format un structured --path /code/config/
```

### ACM Overview

See [our documentation](https://cloud.google.com/anthos-config-management/docs/repo) for how to use each subdirectory.
