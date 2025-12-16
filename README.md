# nimio-velero-backup

This project demonstrates how to configure Velero backups for a Kubernetes cluster
using MinIO as an S3-compatible object storage.

Architecture:
- Kubernetes (Minikube)
- Velero installed via Helm
- MinIO as S3 backend
- File system backups via node-agent (no snapshots)

[Install and setup MinIO](https://docs.google.com/document/d/1SldbZLYdbFMh_JeVpa0VHhXk4ndLvdP2_xgTOV0U7Sk/edit?usp=sharing)

1. Create Velero namespace
    ```bash
    kubectl create ns velero
    ```

2. Create bucket and user for velero in minio
   ```bash
   mc alias set local http://127.0.0.1:9000 minioadmin minioadmin  ##create admin user

   mc mb local/backup-stg  ##create bucket

   mc admin user add local velero "password"  ## create user for backup

   mc admin policy create local VeleroRW ./minio/minio-velero-user-policy.json  ## add policy to minio

   mc admin policy attach ks-stg VeleroRW user=velero  ## appoint this policy added user
   ```

3. Add credentials
   ```bash
   mc admin accesskey create local velero  ## create S3 credentials for user

   insert this credentials to ./velero/velero-bucket-creds.yaml
   #  stringData:
   #   cloud: |
   #     [default]
   #     aws_access_key_id=11111
   #     aws_secret_access_key=1111

   kubectl apply -f ./velero/velero-bucket-creds.yaml
   ```

4. Install Velero
   ```bash
    helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
    helm upgrade --install velero vmware-tanzu/velero --namespace velero -f ./velero/values.yaml
   ```

5. reate Velero Schedule (cron-based backups)
   ```bash
   kubectl apply -f ./velero/schedule.yaml

   kubectl get schedules.velero.io -n velero  ## checking cron is enable
   ```
   
6. Test backup
   ```bash
   cat <<'EOF' | kubectl apply -f -
   apiVersion: velero.io/v1
   kind: Backup
   metadata:
     name: test-backup-now
     namespace: velero
   spec:
     snapshotVolumes: false
   EOF

   kubectl get backups -n velero
   mc ls local/backup-stg/backups  ## check backup in minio
   
   ```

7. Restart Velero if credentials are changed
  ```bash
  kubectl rollout restart deploy/velero -n velero
  ```

8. Restore backup
  ```bash
  velero restore create --from-backup vault-consul-backup ## need install Velero CLI

  # without Velero 
  cat <<'EOF' | kubectl apply -f -
  apiVersion: velero.io/v1
  kind: Restore
  metadata:
    name: restore-test
    namespace: velero
  spec:
    backupName: test-backup-now
  EOF
  ```

Options
  ```bash
  ## Checking secrets
    kubectl get secret -n velero velero-bucket-creds
    kubectl describe secret -n velero velero-bucket-creds

  ## If hung CRD-Job
    kubectl delete job velero-upgrade-crds -n velero

  ## Logs
    kubectl get pods -n velero
    kubectl logs -n velero pod/<velero-pod> -c velero --tail=200
    kubectl logs -n velero pod/<node-agent-pod> -c node-agent --tail=200

  ## Checking sync Velero with Minio
    kubectl describe backupstoragelocations.velero.io -n velero

    # Status Available(at the end of the output)
    Status:
    Last Synced Time:      2025-12-16T19:42:47Z
    Last Validation Time:  2025-12-16T19:43:19Z
    Phase:                 Available
  ```
   
