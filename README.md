# kube_stack
Modern Data Stack on Kubernetes

argo-cd
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

senha-atual:
AqVdxDeKTPNezOnk

apply minio operator
kubectl apply -f .\infra\src\app_manifests\deepstorage\minio-operator.yml

apply minio tenant
kubectl apply -f .\infra\src\app_manifests\deepstorage\minio-tenant.yml