# kube_stack
Modern Data Stack on Kubernetes

port foward
minio
kubectl port-forward svc/minio-tenant-console -n deepstorage 9443:9443
argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443
trino
kubectl port-forward svc/trino -n warehouse 8081:8080


argo-cd
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get secret argocd-initial-admin-secret -n argocd `
  -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

senha-atual:
eAA-Us9ddse2gQjP

apply minio operator
kubectl apply -f .\infra\src\app_manifests\deepstorage\minio-operator.yml

apply minio tenant
kubectl apply -f .\infra\src\app_manifests\deepstorage\minio-tenant.yml

hive-metastore
kubectl apply -f .\infra\src\app_manifests\metastore\hive-metastore.yml

trino
kubectl apply -f .\infra\src\app_manifests\warehouse\trino.yml