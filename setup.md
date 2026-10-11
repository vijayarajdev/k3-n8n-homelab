# 1. namespaces + secrets (one shell)
export KUBECONFIG=~/.kube/config-k3s
cd

kubectl create namespace pg-dev
kubectl create namespace mongodb-dev
kubectl create namespace n8n-dev

PGPASS=$(openssl rand -base64 24 | tr -d '/+=')
MGOPASS=$(openssl rand -base64 24 | tr -d '/+=')
N8N_KEY=$(openssl rand -base64 32)

kubectl -n pg-dev create secret generic n8n-pg-dev-app \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=n8n \
  --from-literal=password="$PGPASS"

kubectl -n mongodb-dev create secret generic mongo-user-pass \
  --from-literal=password="$MGOPASS"

kubectl -n n8n-dev create secret generic n8n-db-secret \
  --from-literal=password="$PGPASS"

kubectl -n n8n-dev create secret generic n8n-core-secrets \
  --from-literal=N8N_ENCRYPTION_KEY="$N8N_KEY" \
  --from-literal=N8N_HOST=n8n.home.lab \
  --from-literal=N8N_PORT=5678 \
  --from-literal=N8N_PROTOCOL=http

echo "$PGPASS" > pgpass-encryption-key.txt && chmod 600 pgpass-encryption-key.txt
echo "$MGOPASS" > mgo-encryption-key.txt && chmod 600 mgo-encryption-key.txt
echo "$N8N_KEY" > n8n-encryption-key.txt && chmod 600 n8n-encryption-key.txt


# 2. Postgres
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update
helm upgrade --install cnpg cnpg/cloudnative-pg --version 0.29.0 \
  -n pg-dev -f postgres/operator-values.yaml --wait --timeout 10m
helm upgrade --install postgres-dev cnpg/cluster --version 0.8.1 \
  -n pg-dev -f postgres/cluster-values.yaml
kubectl -n pg-dev get cluster,pods,svc,pvc

# 3. MongoDB
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo update
helm search repo mongodb/mongodb-kubernetes

helm upgrade --install mongodb-operator mongodb/mongodb-kubernetes \
  -n mongodb-dev -f mongodb/operator-values.yaml --wait --timeout 10m

kubectl get crd | grep mongodbcommunity
kubectl -n mongodb-dev get pods

helm lint mongodb/database
helm template mongo mongodb/database -n mongodb-dev | head -40

helm upgrade --install mongo ./mongodb/database -n mongodb-dev
kubectl -n mongodb-dev get mongodbcommunity,pods,pvc -w

# 4. n8n
helm install n8n oci://ghcr.io/n8n-io/n8n-helm-chart/n8n --version 1.0.0 \
  -n n8n-dev -f n8n/values.yaml
kubectl -n n8n-dev get pods,pvc,ingress