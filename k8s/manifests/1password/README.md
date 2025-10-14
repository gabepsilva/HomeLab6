
# OP CONNECT SERVER + OPERATOR - IN K8S
https://developer.1password.com/docs/connect/get-started/?deploy=kubernetes#step-2-deploy-1password-connect-server


helm repo add 1password https://1password.github.io/connect-helm-charts/

helm install/upgrade connect 1password/connect \
  --namespace op-connect-operator-allenvs \
  --create-namespace \
  --set-file connect.credentials=1password-credentials.json \
  --set operator.create=true \
  --set operator.token.value=<OP connect token> \
  --set operator.autoRestart=true \
  --set operator.pollingInterval=60

kubectl apply -f 01-ingress.yaml


helm uninstall connect --namespace op-connect-operator-allenvs
  
## Using Connect  
  
### CURL


curl -i https://opconnect.p.i.psilva.org/v1/vaults/<vaultid>/items/<secretid> -H "Authorization: Bearer <token>"



### CLI

export OP_CONNECT_HOST=https://opconnect.i.psilva.org
export OP_CONNECT_TOKEN=<token>


# USING OPERATOR

```
apiVersion: onepassword.com/v1
kind: OnePasswordItem
metadata:
  name: g-secret-name
spec:
  itemPath: "vaults/<vault name>/items/<item name>"
```
