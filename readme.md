# OpenShift API for Data Protection Demo

Scaffolding to automatically deploy and exercise OADP for demonstration purposes.

OADP is downstream of Velero.

# How to

* Install the operator

```bash
$ oc apply -k operator/base
```

* Configure for use with noobaa storage

```bash
$ oc apply -k instance/base
serviceaccount/bucket-job created
role.rbac.authorization.k8s.io/bucket-job created
rolebinding.rbac.authorization.k8s.io/bucket-job created
configmap/bucket-job created
secret/cloud-credentials created
job.batch/bucket-job created
dataprotectionapplication.oadp.openshift.io/velero-sample created
objectbucketclaim.objectbucket.io/oadp created
```

* Create a backup

```bash
$ oc apply -k backup
backup.velero.io/backup-demo created
```

* Interacting with velero command using pod

```bash
$ source .bashrc
$ velero backup-location get

NAME              PROVIDER   BUCKET/PREFIX                                      PHASE       LAST VALIDATED                  ACCESS MODE   DEFAULT
velero-sample-1   aws        oadp-5a513c53-4b09-45c4-8a13-57979f844e1a/velero   Available   2023-03-18 00:08:47 +0000 UTC   ReadWrite     true

$ velero describe backups backup-demo
...
```