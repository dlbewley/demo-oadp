#  Testing Backups

## Test HTTP

* Cleanup

```bash
oc delete -k backup/base
oc delete backups -n openshift-adp --all
velero backup delete --all
```

* Check settings

```bash
oc get dpa/velero-sample -n openshift-adp -o json | jq '.spec.backupLocations[].velero.config.s3Url'
"http://s3.openshift-storage.svc"

oc get bsl/velero-sample-1 -n openshift-adp -o json | jq '.spec.config.s3Url'
"http://s3.openshift-storage.svc"
```

* Create backup 

```bash
oc apply -k backup/base
kustomize build backup/base | kfilt -k backup
```
```yaml
---
apiVersion: velero.io/v1
kind: Backup
metadata:
  labels:
    velero.io/storage-location: default
  name: backup-demo
  namespace: openshift-adp
spec:
  excludedResources: []
  hooks: {}
  includedNamespaces:
  - dale
  - demo
  - static
  includedResources:
  - deployments
  - configmaps
  - secrets
  storageLocation: velero-sample-1
  ttl: 720h0m0s
```

* Check backup results

```bash
velero backup get
NAME          STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
backup-demo   Completed   0        0          2023-03-20 19:36:04 +0000 UTC   29d       velero-sample-1    <none>
```

## Test HTTPS


* Cleanup

```bash
oc delete -k backup/base
oc delete backups -n openshift-adp --all
velero backup delete --all
```

* Set to https

```bash
PATCH='[
{ "op": "replace", "path": "/spec/backupLocations/0/velero/config/s3Url", "value": "https://s3.openshift-storage.svc:443"}
]'

oc patch dpa/velero-sample -n openshift-adp --type=json --patch="$PATCH"
```

* Check settings

```bash
oc get dpa/velero-sample -n openshift-adp -o json | jq '.spec.backupLocations[].velero.config.s3Url'
"https://s3.openshift-storage.svc:443"

oc get bsl/velero-sample-1 -n openshift-adp -o json | jq '.spec.config.s3Url'
"https://s3.openshift-storage.svc:443"
```

* Create backup

```bash
kustomize build backup | kfilt -k backup
oc apply -k backup/base
```

* Check backup results

```bash
velero backup get backup-demo
```

## Test HTTPS + PVC

* Cleanup

```bash
oc delete -k backup/base
oc delete backups -n openshift-adp --all
velero backup delete --all
```

* Check settings

```bash
oc get dpa/velero-sample -n openshift-adp -o json | jq '.spec.backupLocations[].velero.config.s3Url'
"https://s3.openshift-storage.svc:443"

oc get bsl/velero-sample-1 -n openshift-adp -o json | jq '.spec.config.s3Url'
"https://s3.openshift-storage.svc:443"
```

* Create backup including PVC

```bash
kustomize build backup/overlays/pv
```
```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  labels:
    velero.io/storage-location: default
  name: backup-demo
  namespace: openshift-adp
spec:
  excludedResources: []
  hooks: {}
  includedNamespaces:
  - dale
  - demo
  - static
  includedResources:
  - deployments
  - configmaps
  - secrets
  - pvc
  storageLocation: velero-sample-1
  ttl: 720h0m0s
```

```bash
oc apply -k backup/overlays/pv
```

* Check results

```bash
velero backup logs backup-demo
I0320 20:10:42.542196  159645 request.go:601] Waited for 1.172218782s due to client-side throttling, not priority and fairness, request: GET:https://172.30.0.1:443/apis/kubevirt.io/v1?timeout=32s
I0320 20:10:52.741835  159645 request.go:601] Waited for 11.371612066s due to client-side throttling, not priority and fairness, request: GET:https://172.30.0.1:443/apis/apps.openshift.io/v1?timeout=32s
An error occurred: Get "https://s3.openshift-storage.svc:443/oadp-5a513c53-4b09-45c4-8a13-57979f844e1a/velero/backups/backup-demo/backup-demo-logs.gz?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=nXV41mLu7p7xBE1eXsSn%!F(MISSING)20230320%!F(MISSING)us-east-1%!F(MISSING)s3%!F(MISSING)aws4_request&X-Amz-Date=20230320T201055Z&X-Amz-Expires=600&X-Amz-SignedHeaders=host&X-Amz-Signature=f182986c2bf4277ed8520cf67f029ee2a519148b47527da906fdcc6f47991af0": x509: certificate signed by unknown authority

The --insecure-skip-tls-verify flag can also be used to accept any TLS certificate for the download, but it is susceptible to man-in-the-middle attacks.
command terminated with exit code 1
```

**Adding PVC causes x509 error.**