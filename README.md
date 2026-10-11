# k3s-lab

n8n on a k3s cluster running on Proxmox. Everything is installed with Helm and the values files in this repository.

| Component | Namespace | Chart | Version |
|---|---|---|---|
| CloudNativePG operator | `pg-dev` | `cnpg/cloudnative-pg` | 0.29.0 |
| Postgres cluster (pgvector) | `pg-dev` | `cnpg/cluster` | 0.8.1 |
| MongoDB operator | `mongodb-dev` | `mongodb/mongodb-kubernetes` | latest from repo |
| MongoDB replica set | `mongodb-dev` | `./mongodb/database` (local chart) | 0.1.0 |
| n8n | `n8n-dev` | `oci://ghcr.io/n8n-io/n8n-helm-chart/n8n` | 1.0.0 |

n8n uses Postgres as its own database. MongoDB is available to workflows through the MongoDB node.

## Layout

```
k3s-lab/
├── setup.md                      # all commands, in run order
├── postgres/
│   ├── operator-values.yaml      # CloudNativePG operator
│   └── cluster-values.yaml       # Postgres 18 cluster, pgvector enabled
├── mongodb/
│   ├── operator-values.yaml      # MongoDB operator
│   └── database/                 # local wrapper chart
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/mongodbcommunity.yaml
└── n8n/
    └── values.yaml               # standalone mode, external Postgres, Traefik ingress
```

## Prerequisites

- A k3s cluster with one server and two agents (built here on Proxmox VMs running Ubuntu).
- `kubectl` and `helm` on your workstation, with `KUBECONFIG` pointing at the cluster:
  ```sh
  export KUBECONFIG=~/.kube/config-k3s
  ```
- Worker nodes labelled `node-role.kubernetes.io/worker=worker`. The operator values use this as a `nodeSelector`.
- The default k3s `local-path` StorageClass and the bundled Traefik ingress.
- A DNS or `/etc/hosts` entry that points `n8n.home.lab` at a cluster node.
- `openssl` for generating secrets.

## Install

Follow `setup.md` in order. In short:

1. **Namespaces and secrets.** Create `pg-dev`, `mongodb-dev` and `n8n-dev`, then create these secrets:
   - `n8n-pg-dev-app` in `pg-dev` (Postgres app user and password)
   - `mongo-user-pass` in `mongodb-dev` (MongoDB user password)
   - `n8n-db-secret` and `n8n-core-secrets` in `n8n-dev` (the same Postgres password, and `N8N_ENCRYPTION_KEY` with the host settings)
2. **Postgres.** Install the operator, then the cluster.
   ```sh
   helm upgrade --install cnpg cnpg/cloudnative-pg --version 0.29.0 -n pg-dev -f postgres/operator-values.yaml --wait
   helm upgrade --install postgres-dev cnpg/cluster --version 0.8.1 -n pg-dev -f postgres/cluster-values.yaml
   ```
3. **MongoDB.** Install the operator, then the local chart that renders the `MongoDBCommunity` resource.
   ```sh
   helm upgrade --install mongodb-operator mongodb/mongodb-kubernetes -n mongodb-dev -f mongodb/operator-values.yaml --wait
   helm upgrade --install mongo ./mongodb/database -n mongodb-dev
   ```
4. **n8n.**
   ```sh
   helm install n8n oci://ghcr.io/n8n-io/n8n-helm-chart/n8n --version 1.0.0 -n n8n-dev -f n8n/values.yaml
   ```

Open http://n8n.home.lab and create the owner account.

## Connection details

| Target | Address |
|---|---|
| Postgres (read/write) | `postgres-dev-rw.pg-dev.svc.cluster.local:5432`, database `n8n`, user `n8n` |
| MongoDB | `mongo-svc.mongodb-dev.svc.cluster.local:27017`, user `n8n` on `admin`, readWrite on `n8ndata` |

MongoDB connection string for the n8n MongoDB credential:

```
mongodb://n8n:<password>@mongo-svc.mongodb-dev.svc.cluster.local:27017/n8ndata?authSource=admin&replicaSet=mongo
```

n8n has no setting that creates node credentials from the Helm values. Create the MongoDB credential in the n8n UI (Credentials, Create credential, MongoDB), or script it with `n8n import:credentials`.

## Verify

```sh
kubectl -n pg-dev get cluster,pods,svc,pvc
kubectl -n mongodb-dev get mongodbcommunity,pods,pvc
kubectl -n n8n-dev get pods,pvc,ingress
kubectl -n n8n-dev logs deploy/n8n-main | head -50     # look for completed Postgres migrations
```

The n8n deployment is named `n8n-main`, not `n8n`.

## Notes

- **Secrets stay out of Git.** The password and encryption-key files that `setup.md` writes (`pgpass-encryption-key.txt`, `mgo-encryption-key.txt`, `n8n-encryption-key.txt`) must not be committed. Add them to `.gitignore`. Back up the n8n encryption key, because losing it makes saved credentials unreadable.
- **Plain HTTP.** `N8N_SECURE_COOKIE` is `false` because the ingress has no TLS. Add a certificate and remove that setting before exposing n8n beyond the LAN.
- **Storage is node-local.** `local-path` volumes are tied to one node, cannot be expanded and are deleted with the claim. Postgres is sized at 30Gi and MongoDB at 20Gi data plus 2Gi logs. Check free disk on the node that hosts them.
- **Single instances.** Postgres runs one instance and MongoDB one member, so there is no failover.
- **n8n uses `strategy: Recreate`** so two pods never share the data volume during an upgrade. Confirm it with `kubectl -n n8n-dev get deploy n8n-main -o jsonpath='{.spec.strategy.type}'`.
- **Pinned versions.** `n8n/values.yaml` pins `image.tag` to `2.42.3`. Confirm that the n8n version you pin supports PostgreSQL 18 before upgrading.
- **Helm on macOS.** If `helm install` fails with `docker-credential-desktop` not found, run it with `DOCKER_CONFIG=$(mktemp -d)` or remove `credsStore` from `~/.docker/config.json`.
- **Empty templates.** Helm installs an empty template without complaint. If `mongo-svc` does not resolve, check that `mongodb/database/templates/mongodbcommunity.yaml` is not empty and run `helm get manifest mongo -n mongodb-dev`.

## Uninstall

```sh
helm uninstall n8n -n n8n-dev
helm uninstall mongo mongodb-operator -n mongodb-dev
helm uninstall postgres-dev cnpg -n pg-dev
```

Uninstalling keeps the CloudNativePG CRDs. Deleting those CRDs destroys every Postgres cluster. The `local-path` volumes are removed when their claims are deleted.
