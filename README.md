[![Build Status][101]][102]

# Velero plugins for VBOS

## Overview

This repository contains these plugins to support running Velero on evitech VBOS:

- An object store plugin for persisting and retrieving backups on VBOS S3. Content of backup is log files, warning/error files, restore logs.

## Setup

## Run vbos S3 proxy server

Velero requires an object storage bucket to store backups in, preferably unique to a single Kubernetes cluster (see the [FAQ][11] for more details). Create an S3 bucket, replacing placeholders appropriately:

```bash
kubectl create ns velero
kubectl run s3server --env="AUTH_KEY=111" --env="AUTH_SECRET=222" --port=9000 --image=registry.vizion.ai/evi/s3proxy:v1 -n velero
kubectl expose deployment/s3server -n velero
```

Get the s3proxy service ip address:

```bash
kubectl get svc s3server -n velero
```
Get the ip from output

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

[Download][4] Velero

Install Velero, including all prerequisites, into the cluster and start the deployment. This will create a namespace called `velero`, and place a deployment named `velero` in it.


```bash
velero install \
    --provider vbos \
    --plugins registry.vizion.ai/evi/velero-plugin-for-vbos:v1.0 \
    --bucket default \
    --backup-location-config region=us-east-1,s3Url=http://10.15.13.35:9000,s3ForcePathStyle=true \
    --secret-file ./vbos-credentials
```

Uninstall 

```base
kubectl delete namespace/velero clusterrolebinding/velero
kubectl delete crds -l component=velero
```

Additionally, you can specify `--use-restic` to enable restic support, and `--wait` to wait for the deployment to be ready.

(Optional) Specify [additional configurable parameters][7] for the `--backup-location-config` flag.

(Optional) Specify [additional configurable parameters][8] for the `--snapshot-location-config` flag.

(Optional) [Customize the Velero installation][9] further to meet your needs.

For more complex installation needs, use either the Helm chart, or add `--dry-run -o yaml` options for generating the YAML representation for the installation.

## Debug backup

Use the example to create backup job to verify it:
- https://velero.io/docs/v1.5/examples/

```base
velero backup create nginx-backup --include-namespaces nginx-example
kubectl logs deployment/velero -n velero
velero backup logs nginx-backup
velero backup describe nginx-backup
```