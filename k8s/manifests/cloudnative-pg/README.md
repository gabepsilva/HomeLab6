# Installation

kubectl apply --server-side -f \
  https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.27/releases/cnpg-1.27.0.yaml


# Install kubectl cnpg plugin 

curl -sSfL \
  https://github.com/cloudnative-pg/cloudnative-pg/raw/main/hack/install-cnpg-plugin.sh | \
  sudo sh -s -- -b /usr/local/bin


# Getting started
https://cloudnative-pg.io/documentation/1.27/quickstart/


# Connect as postgres superuser
kubectl exec -it n8n-cluster-1 -n n8n -- psql -U postgres -d app