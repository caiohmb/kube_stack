# Melhores Práticas para Data Stack no Kubernetes

## 1. Estrutura do Projeto

### Organização de Namespaces
```
kube_stack/
├── infra/
│   ├── src/
│   │   ├── app_manifests/          # Manifestos ArgoCD
│   │   │   ├── deepstorage/        # MinIO
│   │   │   ├── metastore/          # Hive Metastore
│   │   │   ├── warehouse/          # Trino
│   │   │   ├── orchestration/      # Airflow
│   │   │   └── monitoring/         # Prometheus, Grafana
│   │   ├── helm_charts/            # Charts Helm customizados
│   │   └── git_ops/                # Documentação GitOps
│   └── kustomize/                  # Overlays para ambientes
└── apps/                           # Aplicações de dados
    ├── dbt/
    └── data_models/
```

## 2. Configuração de Segurança

### Secrets Management
```yaml
# Usar External Secrets Operator ou Sealed Secrets
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: minio-credentials
spec:
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: minio-credentials
  data:
    - secretKey: access-key
      remoteRef:
        key: minio/access-key
    - secretKey: secret-key
      remoteRef:
        key: minio/secret-key
```

### Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: trino-network-policy
spec:
  podSelector:
    matchLabels:
      app: trino
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: metastore
      ports:
        - protocol: TCP
          port: 8080
```

## 3. Configuração de Recursos

### Resource Limits e Requests
```yaml
resources:
  requests:
    memory: "2Gi"
    cpu: "1000m"
  limits:
    memory: "4Gi"
    cpu: "2000m"
```

### HPA (Horizontal Pod Autoscaler)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: trino-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: trino-worker
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

## 4. Monitoramento e Observabilidade

### ServiceMonitor para Prometheus
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: trino-monitor
spec:
  selector:
    matchLabels:
      app: trino
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

### Grafana Dashboards
- Configurar dashboards para:
  - Performance do Trino
  - Uso de recursos do cluster
  - Latência de queries
  - Status do Hive Metastore

## 5. Backup e Disaster Recovery

### Backup do PostgreSQL (Hive Metastore)
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hive-metastore-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:13
              command:
                - /bin/sh
                - -c
                - |
                  pg_dump -h hive-metastore-postgresql -U hive metastore > /backup/metastore-$(date +%Y%m%d).sql
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: hive-postgresql
                      key: password
```

### Backup do MinIO
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: minio-backup
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: minio/mc:latest
              command:
                - /bin/sh
                - -c
                - |
                  mc mirror minio/data-lake /backup/data-lake
```

## 6. CI/CD com GitOps

### ArgoCD ApplicationSet
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: data-stack-apps
spec:
  generators:
    - list:
        elements:
          - name: trino
            namespace: warehouse
            path: infra/src/helm_charts/trino
          - name: hive-metastore
            namespace: metastore
            path: infra/src/helm_charts/hive-metastore
  template:
    metadata:
      name: '{{name}}'
    spec:
      project: default
      source:
        repoURL: 'https://github.com/seu-repo/kube_stack.git'
        path: '{{path}}'
        targetRevision: HEAD
      destination:
        name: 'in-cluster'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## 7. Configuração de Storage

### Storage Classes
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: k8s.io/minikube-hostpath
parameters:
  type: pd-ssd
reclaimPolicy: Retain
allowVolumeExpansion: true
```

### Persistent Volume Claims
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: trino-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
```

## 8. Configuração de Networking

### Ingress Configuration
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: trino-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - trino.seu-dominio.com
      secretName: trino-tls
  rules:
    - host: trino.seu-dominio.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: trino
                port:
                  number: 8080
```

## 9. Configuração de Logs

### Centralized Logging
```yaml
# Configurar Fluentd ou Fluent Bit para coletar logs
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>
```

## 10. Configuração de Airflow

### Airflow com DBT
```yaml
# Configurar Airflow para executar DBT
apiVersion: v1
kind: ConfigMap
metadata:
  name: airflow-dbt-config
data:
  dbt_project.yml: |
    name: 'data_stack'
    version: '1.0.0'
    config-version: 2
    
    profile: 'default'
    
    models:
      data_stack:
        materialized: table
        staging:
          materialized: view
```

## 11. Checklist de Produção

### Antes de ir para produção:
- [ ] Configurar Resource Limits e Requests
- [ ] Implementar Network Policies
- [ ] Configurar Backup automático
- [ ] Implementar Monitoramento
- [ ] Configurar Alertas
- [ ] Testar Disaster Recovery
- [ ] Configurar SSL/TLS
- [ ] Implementar RBAC
- [ ] Configurar Logs centralizados
- [ ] Testar Auto-scaling

## 12. Troubleshooting Comum

### Problemas de Conexão Hive-Trino
```bash
# Verificar se o Hive Metastore está rodando
kubectl get pods -n metastore

# Verificar logs do Hive Metastore
kubectl logs -n metastore deployment/hive-metastore

# Testar conectividade
kubectl exec -n warehouse deployment/trino-coordinator -- nc -zv hive-metastore.metastore.svc.cluster.local 9083
```

### Problemas de S3/MinIO
```bash
# Verificar configuração do S3
kubectl exec -n warehouse deployment/trino-coordinator -- env | grep S3

# Testar acesso ao MinIO
kubectl exec -n warehouse deployment/trino-coordinator -- curl -I http://minio.deepstorage.svc.cluster.local:9000
``` 