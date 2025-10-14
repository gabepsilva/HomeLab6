

Set user permission
CREATE ROLE postgres WITH 
	SUPERUSER
	CREATEDB
	CREATEROLE
	INHERIT
	LOGIN
	REPLICATION
	BYPASSRLS
	CONNECTION LIMIT -1;


kubectl exec -n appflowy appflowy-pg-cluster-1 -- psql -U postgres -d appflowy -c "ALTER ROLE supabase SUPERUSER CREATEDB CREATEROLE INHERIT LOGIN REPLICATION BYPASSRLS CONNECTION LIMIT -1;"


# Restore Snapshot

1. Scale down
```bash
kubectl cnpg hibernate on appflowy-pg-cluster -n appflowy
```
Wait until the pods are terminated (may take a 2 or 3 minutes)

but if there indication that it is safe -  Delete the pod.
```bash
kubectl delete pod -n appflowy -l cnpg.io/cluster=appflowy-pg-cluster --grace-period=60
```


2. Set maintenance mode
In Longhorn UI:

Go to Volume → find your PVC
Click the volume name
Click Attach (top right)
Select a node and check Maintenance mode
Click OK


3. Revert the snapshot
In Longhorn UI:

Once attached in maintenance mode, go to the Snapshot tab
Find your snapshot
Click Revert (⟲ icon)
Confirm the revert

4. Detach the volume
Click Detach in the volume details

5. Scale up 
`kubectl cnpg hibernate off appflowy-pg-cluster -n appflowy`




# DATABASE MONITORING

## Enable pod monitor
``` yaml
spec:
  monitoring:
    enablePodMonitor: true
```


## Label it 
`kubectl patch podmonitor appflowy-pg-cluster -n appflowy --type merge -p '{"metadata":{"labels":{"release":"monitoring"}}}'`

check if it appears in prometheous
`kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090`

curl localhost:9090/metrics


kubectl patch podmonitor n8n-pg-cluster -n n8n --type merge -p '{"metadata":{"labels":{"release":"monitoring"}}}'