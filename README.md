# kube_stack
Modern Data Stack on Kubernetes

argo-cd
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get secret argocd-initial-admin-secret -n argocd `
  -o jsonpath="{.data.password}" | ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }

senha-atual:
Ghx89vp0EIb3nhEM

apply minio operator
kubectl apply -f .\infra\src\app_manifests\deepstorage\minio-operator.yml

apply minio tenant
kubectl apply -f .\infra\src\app_manifests\deepstorage\minio-tenant.yml

hive-metastore
kubectl apply -f .\infra\src\app_manifests\metastore\hive-metastore.yml