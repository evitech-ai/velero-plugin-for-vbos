# Velero plugins for VBOS

## Overview

This repository contains these plugins to support running Velero on evitech VBOS:

- An object store plugin for persisting and retrieving backups on VBOS S3. Content of backup is log files, warning/error files, restore logs.

## Setup

## Run vbos S3 proxy server

Velero requires an object storage bucket to store backups in, preferably unique to a single Kubernetes cluster. Create an S3 bucket, replacing placeholders appropriately:

```bash
kubectl create ns velero
kubectl run s3server --env="AUTH_KEY=111" --env="AUTH_SECRET=222" --port=9000 --image=registry.vizion.ai/evi/s3proxy:v1 -n velero
kubectl expose deployment/s3server -n velero
kubectl get svc s3server -n velero
```

Get the s3proxy service ip address from output

```bash
NAME       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
s3server   ClusterIP   10.15.13.35   <none>        9000/TCP   4m32s
```

Create a Velero-specific credentials file (`vbos-credentials`) in your local directory:

```bash
[default]
aws_access_key_id=<AWS_ACCESS_KEY_ID>
aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>
```

where the access key id and secret are the values returned from the `create-access-key` request.

## Install and start Velero

Install Velero and enable CSI Plugin support 
, including all prerequisites, into the cluster and start the deployment. This will create a namespace called `velero`, and place a deployment named `velero` in it.

```bash
velero install \
    --features=EnableCSI \
    --provider vbos \
    --plugins registry.vizion.ai/evi/velero-plugin-for-vbos:v1.0,velero/velero-plugin-for-csi:v0.1.2 \
    --bucket default \
    --use-volume-snapshots="false" \
    --backup-location-config region=us-east-1,s3Url=http://10.4.10.1:9000,s3ForcePathStyle=true \
    --secret-file ./vbos-credentials
```

Uninstall 

```base
kubectl delete namespace/velero clusterrolebinding/velero
kubectl delete crds -l component=velero
```

Additionally, you can specify `--use-restic` to enable restic support, and `--wait` to wait for the deployment to be ready.

(Optional) Specify [additional configurable parameters][1] for the `--backup-location-config` flag.

(Optional) [Customize the Velero installation][2] further to meet your needs.

For more complex installation needs, use either the Helm chart, or add `--dry-run -o yaml` options for generating the YAML representation for the installation.

## Debug backup

Use the example to create backup job and verify it:
- https://velero.io/docs/v1.5/examples/

```base
velero backup create nginx-backup --include-namespaces nginx-example
kubectl logs deployment/velero -n velero
velero backup logs nginx-backup
velero backup describe nginx-backup

kubectl delete namespaces nginx-example
velero restore create --from-backup nginx-backup

velero backup delete nginx-backup
```

Some errors:

```bash
#1 VolumeSnapshotClass don't has velero.io/csi-volumesnapshot-class label
time="2020-10-28T03:08:05Z" level=error msg="Error backing up item" backup=velero/nginx-backup-csi error="error executing custom action (groupResource=persistentvolumeclaims, namespace=nginx-example, name=nginx-logs): rpc error: code = Unknown desc = failed to get volumesnapshotclass for storageclass csi-ext4-sc-vset1: failed to get volumesnapshotclass for provisioner ext4.vbos.evitech.ai, ensure that the desired volumesnapshot class has the velero.io/csi-volumesnapshot-class label" logSource="pkg/backup/backup.go:455" name=nginx-deployment-f96b7fd86-9mp8p

#2 freezed by CSI and conflict with velero annotation: pre.hook.backup.velero.io/command: '["/sbin/fsfreeze", "--freeze", "/var/log/nginx"]'
time="2020-10-28T03:46:45Z" level=info msg="Waiting for volumesnapshotcontents snapcontent-052791aa-95ec-4b5f-acc8-11d1900bd09b to have snapshot handle. Retrying in 5s" backup=velero/nginx-backup-csi1 cmd=/plugins/velero-plugin-for-csi logSource="/go/src/velero-plugin-for-csi/internal/util/util.go:179" pluginName=velero-plugin-for-csi
time="2020-10-28T03:46:50Z" level=info msg="Waiting for volumesnapshotcontents snapcontent-052791aa-95ec-4b5f-acc8-11d1900bd09b to have snapshot handle. Retrying in 5s" backup=velero/nginx-backup-csi1 cmd=/plugins/velero-plugin-for-csi logSource="/go/src/velero-plugin-for-csi/internal/util/util.go:179" pluginName=velero-plugin-for-csi

#3 install velero use arg:   --use-volume-snapshots="false" 
time="2020-10-28T08:52:49Z" level=error msg="Error getting volume snapshotter for volume snapshot location" backup=velero/nginx-backup2 error="unable to locate VolumeSnapshotter plugin named velero.io/vbos" logSource="pkg/backup/item_backupper.go:449" name=vbos-a1b22b6b-4ffb-453d-bc81-84e5313260bf namespace= persistentVolume=vbos-a1b22b6b-4ffb-453d-bc81-84e5313260bf resource=persistentvolumes volumeSnapshotLocation=default
```

[1]: backupstoragelocation.md
[2]: https://velero.io/docs/customize-installation/