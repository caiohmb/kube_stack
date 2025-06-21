# kube_stack
Modern Data Stack on Kubernetes

Uma stack de dados moderna construída com Kubernetes, incluindo MinIO para storage, Trino para query engine, Hive Metastore para metadata, ArgoCD para GitOps, e suporte a tabelas Iceberg.

## 🏗️ Arquitetura

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   ArgoCD        │    │   MinIO         │    │   Hive          │
│   (GitOps)      │    │   (S3 Storage)  │    │   Metastore     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   Trino         │
                    │   (Query Engine)│
                    └─────────────────┘
```

## 🚀 Componentes

- **MinIO**: Object storage compatível com S3
- **Trino**: Query engine distribuído
- **Hive Metastore**: Gerenciamento de metadata
- **ArgoCD**: GitOps para deployment
- **PostgreSQL**: Database para Hive Metastore
- **Iceberg**: Formato de tabela para data lake

## 📋 Pré-requisitos

- Kubernetes cluster (Minikube, Kind, ou similar)
- kubectl configurado
- Helm 3.x
- ArgoCD CLI (opcional)

## 🛠️ Instalação

### 1. Instalar ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2. Obter senha do ArgoCD

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d
```

### 3. Aplicar MinIO Operator

```bash
kubectl apply -f ./infra/src/app_manifests/deepstorage/minio-operator.yml
```

### 4. Aplicar MinIO Tenant

```bash
kubectl apply -f ./infra/src/app_manifests/deepstorage/minio-tenant.yml
```

### 5. Aplicar Hive Metastore

```bash
kubectl apply -f ./infra/src/app_manifests/metastore/hive-metastore.yml
```

### 6. Aplicar Trino

```bash
kubectl apply -f ./infra/src/app_manifests/warehouse/trino.yml
```

## 🔗 Port Forward

### MinIO Console
```bash
kubectl port-forward svc/minio-tenant-console -n deepstorage 9443:9443
```
Acesse: https://localhost:9443
- Usuário: minio
- Senha: minio123

### ArgoCD
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Acesse: https://localhost:8080
- Usuário: admin
- Senha: (obtida no passo 2)

### Trino
```bash
kubectl port-forward svc/trino -n warehouse 8081:8080
```
Acesse: http://localhost:8081

## 📊 Catalogs Disponíveis

### Hive Catalog
- **Nome**: `hive`
- **Tipo**: Hive tables via Hive Metastore
- **Storage**: MinIO S3

### Iceberg Catalog
- **Nome**: `iceberg`
- **Tipo**: Iceberg tables via Hive Metastore
- **Storage**: MinIO S3

### TPCDS Catalog
- **Nome**: `tpcds`
- **Tipo**: Dados de benchmark para testes

## 🔧 Configurações

### Hive Metastore
- **Database**: PostgreSQL
- **Warehouse**: `s3a://data-lake/warehouse`
- **Metastore URI**: `thrift://hive-metastore.metastore.svc.cluster.local:9083`

### Trino
- **Workers**: 2 (configurável)
- **Memory**: 4GB por query
- **Catalogs**: hive, iceberg, tpcds

### MinIO
- **Endpoint**: `http://minio.deepstorage.svc.cluster.local:9000`
- **Bucket**: `data-lake`
- **Access Key**: `minio`
- **Secret Key**: `minio123`

## 🧪 Testes

### Testar Conexão Hive-Trino
```bash
# Verificar se os pods estão rodando
kubectl get pods -n metastore
kubectl get pods -n warehouse

# Testar conectividade
kubectl exec -n warehouse deployment/trino-coordinator -- nc -zv hive-metastore.metastore.svc.cluster.local 9083
```

### Query de Teste
```sql
-- Conectar ao Trino e executar:
SELECT * FROM tpcds.tiny.nation LIMIT 5;
```

## 📈 Monitoramento

### Métricas Disponíveis
- **Trino**: http://localhost:8081/v1/jmx
- **Hive Metastore**: Logs via kubectl
- **MinIO**: Console web

### Logs
```bash
# Logs do Trino
kubectl logs -n warehouse deployment/trino-coordinator

# Logs do Hive Metastore
kubectl logs -n metastore deployment/hive-metastore

# Logs do MinIO
kubectl logs -n deepstorage deployment/minio-tenant
```

## 🔒 Segurança

### Secrets Management
- Credenciais do MinIO em secrets
- Senha do PostgreSQL em secrets
- Recomendado usar External Secrets Operator para produção

### Network Policies
- Comunicação restrita entre namespaces
- Acesso ao MinIO apenas dos serviços necessários

## 🚨 Troubleshooting

### Problemas Comuns

1. **Hive Metastore não conecta ao Trino**
   ```bash
   # Verificar se o serviço está rodando
   kubectl get svc -n metastore
   
   # Verificar logs
   kubectl logs -n metastore deployment/hive-metastore
   ```

2. **Trino não acessa MinIO**
   ```bash
   # Verificar configuração S3
   kubectl exec -n warehouse deployment/trino-coordinator -- env | grep S3
   
   # Testar conectividade
   kubectl exec -n warehouse deployment/trino-coordinator -- curl -I http://minio.deepstorage.svc.cluster.local:9000
   ```

3. **ArgoCD não sincroniza**
   ```bash
   # Verificar status da aplicação
   kubectl get applications -n argocd
   
   # Verificar logs do ArgoCD
   kubectl logs -n argocd deployment/argocd-server
   ```

## 📚 Próximos Passos

- [ ] Adicionar Airflow para orquestração
- [ ] Integrar DBT para transformações
- [ ] Configurar monitoramento com Prometheus/Grafana
- [ ] Implementar backup automático
- [ ] Adicionar autoscaling
- [ ] Configurar SSL/TLS
- [ ] Implementar RBAC

## 📖 Documentação Adicional

- [Melhores Práticas](BEST_PRACTICES.md)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Trino Documentation](https://trino.io/docs/)
- [MinIO Documentation](https://docs.min.io/)
- [Apache Iceberg](https://iceberg.apache.org/)

## 🤝 Contribuição

1. Fork o projeto
2. Crie uma branch para sua feature
3. Commit suas mudanças
4. Push para a branch
5. Abra um Pull Request

## 📄 Licença

Este projeto está sob a licença MIT. Veja o arquivo [LICENSE](LICENSE) para mais detalhes.