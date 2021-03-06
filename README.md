# ansible-arangodb-operator

## Purpose
- Ansible role to deploy [ArangoDB Operator](https://github.com/arangodb/kube-arangodb) in Kubernetes.
- Multiple operators can be deployed in multiple namespaces. Default namespace is: `arangodb`
- Multiple ArangoDB clusters can be deployed by an operator.


## Features
- Downloads [manifests](https://github.com/arangodb/kube-arangodb/tree/master/manifests) from desired release
- Applies user default patches, overriding defaults in original manifests
- Loads resulting manifests in Kubernetes:
    - default: by connecting to the master K8S node via ssh
    - alternatively: provinding local kubernetes config, by setting `kubeconfig_file_path`
- Instruct operator to deploy cluster(s) and optionally local storages


## Dependencies
- [Kustomize](https://github.com/kubernetes-sigs/kustomize) - will be automatically downloaded in project `bin/` directory
- Pip3 packages: see [requirements.pip](requirements.pip) - automatically installed in preflight
- Developed and tested with ansible 2.8, may work with lower/higher versions


## Usage
1. Set up your ansible inventory with appropriate hosts and SSH keys:
    - To run remotely:
        - Set your ansible_host in `sample-inventory/host_vars/master` to point to your kubernetes master
        - Set up your ssh key for authentication to the master
    - To run locally:
        - in [sample-inventory/host_vars/master](sample-inventory/host_vars/master), uncomment: `ansible_connection: local`
        - set the playbook variable `kubeconfig_file_path` pointing to a local kubeconfig
2. Set your cluster and deployments options in a vars file (See [sample-vars.yml](sample-vars.yml))
3. Run the playbook:

    ```bash
    ANSIBLE_SSH_PIPELINING=true \
    ANSIBLE_CONFIG=sample-ansible.cfg \
    ansible playbook \
        --inventory sample-inventory \
        sample-playbook.yml \
        --extra-vars "@sample-vars.yml"`
    ```

    - To skip preflight and speed up a bit, add `--skip-tags preflight`


## Reference
- https://www.arangodb.com/docs/stable/tutorials-kubernetes.html
- https://www.arangodb.com/docs/stable/deployment-kubernetes-deployment-resource.html


## ArangoDB Backup & Restore
- 3 types of backups:
  - Physical (raw or “cold”) backups - can be done when the ArangoDB Server is not running
  - Logical backups - `arangodump` & `arangorestore`
  - Hot backups - Arangobackup & Hot Backup API - Enterprise Edition only
- Ref: https://www.arangodb.com/docs/stable/backup-restore.html
- Will explain more about the **Logical backups** method in the following sections

### Backup
- By using `arangodump`, this can be done manually or in a kubernetes CronJob as in the example below
  ```
  apiVersion: batch/v1beta1
  kind: CronJob
  metadata:
    labels: {}
    name: "arangodb-backup"
    namespace: <namespace>
  spec:
    schedule: 0 0 * * *
    jobTemplate:
      spec:
        template:
          spec:
            containers:
            - name: arangodb-backup
              image: arangodb:<image_tag>
              command:
              - /bin/sh
              - -c
              args:
              - |-
                arangodump \
                  --server.endpoint "http+tcp://<cluster-exporter-service-name>.<namespace>:8529" \
                  --server.database "<database_name>" \
                  --server.username $DB_USERNAME --server.password $DB_PASSWORD \
                  --output-directory "/backups/<database_name>-`date '+%Y-%m-%d_%H-%M-%S'`" && \
                find /backups/ -mindepth 1 -maxdepth 1 -mtime +7 -type d -exec rm -fvr {} \;
              env:
              - name: DB_USERNAME
                valueFrom:
                  secretKeyRef:
                    key: username
                    name: <arangodb-cluster-root-password-secret>
              - name: DB_PASSWORD
                valueFrom:
                  secretKeyRef:
                    key: password
                    name: <arangodb-cluster-root-password-secret>
              volumeMounts:
              - mountPath: /backups
                name: arangodb-backup-pv
            volumes:
            - name: arangodb-backup-pv
              persistentVolumeClaim:
                claimName: arangodb-backup-pvc
  ```
  - A persistent volume (`arangodb-backup-pv`) and a persistent volume claim (`arangodb-backup-pvc`) should be created for this to work
- This CronJob will perform a daily backup and cleanup, leaving the backups from the last 7 days.
- More details about backup: https://www.arangodb.com/docs/stable/programs-arangodump-examples.html

### Restore
- Restoring a database can be done using `arangorestore` as in the following example:
  ```
  arangorestore \
    --server.endpoint "http+tcp://<cluster-exporter-service-name>.<namespace>:8529" \
    --server.database <database_name> \
    --create-database true \
    --server.username <username> \
    --server.password <password> \
    --input-directory <backup_dir_path>
  ```
- More details about restore: https://www.arangodb.com/docs/stable/programs-arangorestore-examples.html


## Variables and parameters

| Variable                                 | Default                                                       | Description                                                                                                                                                                                                         |
|------------------------------------------|---------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| kustomize_version                        | 3.5.5                                                         | Kustomize version to download an use. Kustomize is used locally, even though there is one in the system.                                                                                                            |
| arangodb_operator_version                | 1.0.2                                                         | ArangoDB Operator version (do not put 'v' in front)                                                                                                                                                                 |
| arangodb_operator_namespace              | arangodb                                                      | Operator namespace                                                                                                                                                                                                  |
| arangodb_operator_manifests              | [arango-crd.yaml, arango-deployment.yaml, arango-backup.yaml] | List of Kubernetes manifests to download from the operator repo. This should not change normally.                                                                                                                   |
| arangodb_operator_manifest_local_storage | arango-storage.yaml                                           | Optional operator local storage manifest. Set to empty to not download it and not use local storage. If empty, `arangodb_local_storages` will be ignored                                                            |
| arangodb_operator_manifest_replication   | arango-deployment-replication.yaml                            | Optional operator for DC2DC replciation. Set to empty to not download and not use replication                                                                                                                       |
| |
| kustomizations.patchesStrategicMerge     | []                                                            | List of objects to locate and patch default kubernetes objects in the download manifests. You must specify `apiVersion`, `kind` and `metadata.name`. See: https://github.com/kubernetes-sigs/kustomize/blob/master/docs/plugins/builtins.md#patchesstrategicmerge |
| kustomizations.patches                   | []                                                            | List of PatchTransformer objects You must specify `kind` and `name`. See: https://github.com/kubernetes-sigs/kustomize/blob/master/docs/plugins/builtins.md#patchtransformer                                        |
| |
| arangodb_local_storages                  | []                                                            | Optional list of ArangoLocalStorage. See https://www.arangodb.com/docs/stable/deployment-kubernetes-storage-resource.html                                                                                           |
| arangodb_clusters                        | []                                                            | List of ArangoDeployment objects - actual ArangoDB cluster definition. See https://www.arangodb.com/docs/stable/deployment-kubernetes-deployment-resource.html                                                      |
| |
| kubeconfig_file_path                     |                                                               | Optional path to Kubernetes admin config file (on the master node is located at /etc/kubernetes/admin.conf). Setting this will not use SSH connection to master, instead try to access the Kubernetes API directly. |
| kubernetes_operation_retries             | 20                                                            | How many times to retry uploading/waiting for a kubernetes manifest  before giving up. Some resources, like Deployments may take longer to be ready.                                                                |
| kubernetes_operation_delay               | 10                                                            | Interval in seconds to retry                                                                                                                                                                                        |
| kubernetes_ignore_errors                 | yes                                                           | Set to 'no' to fail fast at the first encountered error when trying to create objects in kubernetes                                                                                                                 |


## License
MIT
