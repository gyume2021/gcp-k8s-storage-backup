# gcp-k8s-storage-backup

## [Install the CLI](https://github.com/vmware-tanzu/velero/releases/tag/v1.8.1)

```bash
tar -xvf <RELEASE-TARBALL-NAME>.tar.gz
cd velero-v1.8.1-linux-amd64
mv velero /usr/local/bin
```

## [Install the Velero server](https://github.com/vmware-tanzu/helm-charts)

- add helm repo
```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts

# docker image url: https://hub.docker.com/r/velero/velero-plugin-for-gcp/tags
helm upgrade --cleanup-on-fail \
--install velero vmware-tanzu/velero \
--reuse-values \
--namespace=velero \
--create-namespace \
--set cleanUpCRDs=true \
--set-file credentials.secretContents.cloud=cred.json \
--set configuration.provider=gcp \
--set configuration.backupStorageLocation.name=backup-velereo-location-name \
--set configuration.backupStorageLocation.bucket=velereo-bucket \
--set configuration.backupStorageLocation.config.region=asia-east1 \
--set configuration.volumeSnapshotLocation.name=snapshot-velereo-location-name \
--set configuration.volumeSnapshotLocation.config.region=asia-east1 \
--set initContainers[0].name=velero-plugin-for-gcp \
--set initContainers[0].image=velero/velero-plugin-for-gcp:v1.4.1 \
--set initContainers[0].volumeMounts[0].mountPath=/target \
--set initContainers[0].volumeMounts[0].name=plugins
#--dry-run -o yaml
```

- Grant access to Velero
    - Create a service account key, specifying an output file (`credentials-velero`) in your local directory

```bash
gcloud iam service-accounts keys create credentials-velero \
    --iam-account $SERVICE_ACCOUNT_EMAIL
```

- Uninstall Velero
```bash
helm uninstall velero -n velero
```

## [velero plugin for gcp](https://github.com/vmware-tanzu/velero-plugin-for-gcp#setup)

```bash
BUCKET=velereo-bucket
gsutil mb gs://$BUCKET/
```

- Create Google Service Account

```bash
gcloud config list

PROJECT_ID=$(gcloud config get-value project)

GSA_NAME=velero
gcloud iam service-accounts create $GSA_NAME \
    --display-name "Velero service account"

SERVICE_ACCOUNT_EMAIL=$(gcloud iam service-accounts list \
  --filter="displayName:Velero service account" \
  --format 'value(email)')
```

- Create Custom Role with Permissions for the Velero GSA

```bash
ROLE_PERMISSIONS=(
    compute.disks.get
    compute.disks.create
    compute.disks.createSnapshot
    compute.snapshots.get
    compute.snapshots.create
    compute.snapshots.useReadOnly
    compute.snapshots.delete
    compute.zones.get
    storage.objects.create
    storage.objects.delete
    storage.objects.get
    storage.objects.list

    # iam.serviceAccounts.signBlob
)

gcloud iam roles create velero.server \
    --project $PROJECT_ID \
    --title "Velero Server" \
    --permissions "$(IFS=","; echo "${ROLE_PERMISSIONS[*]}")"

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:$SERVICE_ACCOUNT_EMAIL \
    --role projects/$PROJECT_ID/roles/velero.server

gsutil iam ch serviceAccount:$SERVICE_ACCOUNT_EMAIL:objectAdmin gs://${BUCKET}
```