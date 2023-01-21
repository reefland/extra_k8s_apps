# Velero - backup and migrate Kubernetes resources and persistent volumes

Velero is a tool to safely backup and restore, perform disaster recovery, and migrate Kubernetes cluster resources and persistent volumes.  Project Site:  <https://velero.io/>

[Return to Application List](../)

## Installation Notes

* Helm based ArgoCD application deployment
* Deployed as a Statefulset pod and a DaemonSet for [File System](https://velero.io/docs/v1.10/file-system-backup/) level backups
* Velero requires you to select a [Storage Provider](https://velero.io/docs/v1.10/supported-providers/) to store the backups.  This configuration uses Amazon S3 compatible provider such as [MinIO](https://min.io/). [I use MinIO hosted on my TrueNAS server to allow data to be backed-up external to the cluster.]

* Velero optionally supports Kubernetes [CSI Snaphotter](https://velero.io/docs/v1.10/csi/) to take a [volume snapshot](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) as part of the backup process.
  * You can consider the [CSI Snapshot ArgoCD / Kustomize](../external-snapshotter-argocd-kustomize/) installer I use
  * Can be used with [Democratic CSI for TrueNAS](https://github.com/democratic-csi/democratic-csi), [Longhorn](https://longhorn.io/), [Rook-Ceph](https://rook.io/), or many other CSI providers
    You can consider my [Democratic CSI installer via Ansible](https://github.com/reefland/ansible-k3s-argocd-renovate/blob/master/docs/democratic-csi-settings.md), [Longhorn Installer via Ansible](https://github.com/reefland/ansible-k3s-argocd-renovate/blob/master/docs/longhorn-settings.md), or [Rook-Ceph ArgoCD / Helm installer](../rook-ceph-argocd-helm/)

---

## Installation

NOTE: You will need to create a secret prior to installation such as:

  ```yaml
  ---
  apiVersion: v1
  kind: Secret
  metadata:
    name: velero-secrets-store
  namespace: velero
  type: Opaque
  stringData:
    cloud: |
      [default]
      aws_access_key_id=<KEY_ID_HERE>
      aws_secret_access_key=<PASSWORD_HERE>
  ```

* If you name the secret differently, you will have to update the configuration below `existingSecret: velero-secrets-store` which alterative secret name
* Where `aws_access_key_id` is the account or User ID to connect to the S3 compatible storage
* Where `aws_secret_access_key` is the password to be used for the above account.

Apply this directly, or convert to a [Sealed-Secret](https://github.com/reefland/ansible-k3s-argocd-renovate/blob/master/docs/sealed-secrets-settings.md) and commit the encrypted secret to GitHub for deployment.

Review `velero-argocd-helm/applications/velero.yaml`

* Define the ArgoCD project to assign this application to
* ArgoCD uses `default` project by default

  ```yaml
  spec:
    project: default
  ```

* Define initial Helm chart location and version of Velero to install:

  ```yaml
  chart: velero
  repoURL: https://vmware-tanzu.github.io/helm-charts
  targetRevision: 3.1.0
  ```

Review the example [values.yaml](https://github.com/vmware-tanzu/helm-charts/blob/main/charts/velero/values.yaml) provided by the project. These are applied below and customized to your environment.

* Define plugins to be installed as InitContainers within the Velero application pod.  This installation installs support for CSI snapshots and AWS compatible S3 storage:
  * NOTE: Renovate BOT hints are included to keep these image image versions upt-to-date.

  ```yaml
  initContainers:
    - name: velero-plugin-for-csi
      # renovate: datasource=docker depName=velero/velero-plugin-for-csi
      image: velero/velero-plugin-for-csi:v0.3.2
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: /target
          name: plugins

    - name: velero-plugin-for-aws
      # renovate: datasource=docker depName=velero/velero-plugin-for-aws
      image: velero/velero-plugin-for-aws:v1.5.2
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - mountPath: /target
          name: plugins
  ```

* Enable Prometheus Integration - [See GitHub Issue 388](https://github.com/vmware-tanzu/helm-charts/issues/388) which prevents it from working under ArgoCD.

  ```yaml
    metrics:
      enabled: true

    serviceMonitor:
      enabled: true

    prometheusRule:
      enabled: true
  ```

---

## Define Velero Configuration

* This configuration enables CSI snapshotting and AWS S3 compatible storage:

  ```yaml
    configuration:
      features: EnableCSI
      provider: aws
  ```

* Define the storage location. This installation uses MinIO, an S3 compatible object store.  This defines a storage location named `minio`:

  ```yaml
    # Parameters for the `minio` BackupStorageLocation. See
    # https://velero.io/docs/v1.10/api-types/backupstoragelocation/
    backupStorageLocation:
      name: minio
      provider: aws
  ```

* Define the S3 storage bucket name (required) and prefix (optional) to store the backup:

  ```yaml
      bucket: velero
      prefix:
  ```

* The S3 compatible storage can be defined with a region name (AWS uses "us‑east‑1" as an example), this can be "default" or "minio" whatever you defined in the S3 storage configuration.
  * Define the URL to access your S3 bucket:

  ```yaml
      config:
        region: minio
        s3ForcePathStyle: true
        s3Url: "https://minio.example.com:9000"
  ```

* Define the location where snapshots will be stored:

  ```yaml
    # Parameters for the `minio` VolumeSnapshotLocation. See
    # https://velero.io/docs/v1.10/api-types/volumesnapshotlocation/
    volumeSnapshotLocation:
      name: minio
      provider: aws
      config:
        region: minio
  ```

* Define the name of the secret to use which has the User ID and Password to connect to the S3 bucket:

  ```yaml
    # Info about the secret to be used by the Velero deployment, which should contain credentials for
    # the cloud provider IAM account you've set up for Velero.
    credentials:
      # Whether a secret should be used. Set to false if, for examples:
      useSecret: true
      # Name of a pre-existing secret (if any) in the Velero namespace that should be used to
      # get IAM account credentials. Optional.
      existingSecret: velero-secrets-store
  ```

* If only using CSI snapshotter, a daemonSet deployment is not needed.  If you want a copy of the data stored within the Persistent Volume to also be copied to S3 compatible storage then the daemonSet is needed for the File System backup.

  ```yaml
    # Whether to deploy the node-agent daemonset. (Enabled restic/kopia backups)
    deployNodeAgent: true
  ```

* Define the data uploader type such as [restic](https://restic.net/) or [kopia](https://kopia.io/):

  ```yaml
    uploaderType: kopia
  ```

* Define if you want this to be used by default or would you rather it only apply to annotated pods:

  ```yaml
    # Set true for backup all pod volumes without having to apply annotation on the pod when used file system backup Default: false.
    defaultVolumesToFsBackup: true
  ```

---

### Installing Velero Client

The client is used to interact with the Velero deployment.  This can be run remotely if you have kubectl configured to work remotely as well.

```shell
$ cd ~/Downloads

$ VERSION="v1.10.0"

$ wget https://github.com/vmware-tanzu/velero/releases/download/${VERSION}/velero-${VERSION}-linux-amd64.tar.gz

$ tar -xvzf velero-${VERSION}-linux-amd64.tar.gz

$ sudo install velero-${VERSION}-linux-amd64/velero /usr/local/bin/


$ velero version                                                                                         
Client:
        Version: v1.10.0
        Git commit: 367f563072659f0bcd809bc33507fd75cd722344
Server:
        Version: v1.10.0
```

### Example Manual Backup

```shell
  velero backup create <backupName> --selector <label>=<value> [options]
```

Example of making a Velero backup of [Apt-Cacher NG](../apt-cacher-ng-argocd-helm/) to Minio S3 compatiable storage with included upload of PVC data to S3:

```shell
$ velero backup create apt-cacher-ng-backup-02 --selector "app.kubernetes.io/name=apt-cacher-ng" --default-volumes-to-fs-backup true

Backup request "apt-cacher-ng-backup-02" submitted successfully.
```

* `apt-cacher-ng-backup-02` is the name of the backup
* `app.kubernetes.io/name=apt-cacher-ng` is the label applied to the pod to backup (defines what to backup)
* `--default-volumes-to-fs-backup true` option passed to the backup engine (do File System backup)

Monitor Progress:

```shell
$ velero backup describe apt-cacher-ng-backup-02 --details

....
Estimated total items to be backed up:  12
Items backed up so far:                 0

Resource List:  <backup resource list not found>

Velero-Native Snapshots: <none included>

restic Backups:
  In Progress:
    apt-cacher-ng/apt-cacher-ng-0: cache-vol (55.10%)
....
```

```shell
....
restic Backups:
  In Progress:
    apt-cacher-ng/apt-cacher-ng-0: cache-vol (85.43%)
....
```

View Backup Details:

```shell
$ velero backup describe apt-cacher-ng-backup-02 --details

Name:         apt-cacher-ng-backup-02
Namespace:    velero
Labels:       velero.io/storage-location=minio
Annotations:  velero.io/source-cluster-k8s-gitversion=v1.25.4+k3s1
              velero.io/source-cluster-k8s-major-version=1
              velero.io/source-cluster-k8s-minor-version=25

Phase:  Completed

Errors:    0
Warnings:  0

Namespaces:
  Included:  *
  Excluded:  <none>

Resources:
  Included:        *
  Excluded:        <none>
  Cluster-scoped:  auto

Label selector:  app.kubernetes.io/name=apt-cacher-ng

Storage Location:  minio

Velero-Native Snapshot PVs:  auto

TTL:  720h0m0s

CSISnapshotTimeout:  10m0s

Hooks:  <none>

Backup Format Version:  1.1.0

Started:    2023-01-20 12:21:09 -0500 EST
Completed:  2023-01-20 12:22:01 -0500 EST

Expiration:  2023-02-19 12:21:09 -0500 EST

Total items to be backed up:  16
Items backed up:              16

Resource List:
  apps/v1/ControllerRevision:
    - apt-cacher-ng/apt-cacher-ng-585bbf8bd4
    - apt-cacher-ng/apt-cacher-ng-85c8587fbb
    - apt-cacher-ng/apt-cacher-ng-bd969788b
    - apt-cacher-ng/apt-cacher-ng-c7f7c48c7
  apps/v1/StatefulSet:
    - apt-cacher-ng/apt-cacher-ng
  discovery.k8s.io/v1/EndpointSlice:
    - apt-cacher-ng/apt-cacher-ng-tqctk
  networking.k8s.io/v1/Ingress:
    - apt-cacher-ng/apt-cacher-ng
  snapshot.storage.k8s.io/v1/VolumeSnapshot:
    - apt-cacher-ng/velero-apt-cacher-ng-cache-vol-67xpm
  snapshot.storage.k8s.io/v1/VolumeSnapshotClass:
    - ceph-block
  snapshot.storage.k8s.io/v1/VolumeSnapshotContent:
    - snapcontent-b4a931de-dfab-462f-b1fb-88d83d76749a
  v1/ConfigMap:
    - apt-cacher-ng/apt-cacher-ng-acng-conf-file
  v1/Endpoints:
    - apt-cacher-ng/apt-cacher-ng
  v1/PersistentVolume:
    - pvc-3b63c083-37f8-4a02-b702-440d628a931c
  v1/PersistentVolumeClaim:
    - apt-cacher-ng/apt-cacher-ng-cache-vol
  v1/Pod:
    - apt-cacher-ng/apt-cacher-ng-0
  v1/Service:
    - apt-cacher-ng/apt-cacher-ng

Velero-Native Snapshots: <none included>

restic Backups:
  Completed:
    apt-cacher-ng/apt-cacher-ng-0: cache-vol
```

* Backup can be restored on:
  * Same cluster / same or different namespace
  * Different cluster / same or different namespace

[Return to Application List](../)
