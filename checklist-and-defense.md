# Документация: Деплой мессенджера в Kubernetes

## Структура репозитория

```
k8s/
  base/                        # Базовые манифесты
  overlays/
    dev/                       # Dev-окружение (1 реплика, dev.messager.local)
    prod/                      # Prod-окружение (2 реплики, повышенные ресурсы)
argocd/                        # Argo CD Application манифесты
docs/                          # Документация
```

## Как запустить

### Предварительные требования
- Kubernetes кластер (k3d / k3s)
- kubectl настроен на кластер
- kustomize установлен (`brew install kustomize`)
- Argo CD установлен в namespace `argocd`

### 1. Создать namespace
```bash
kubectl create namespace messager
```

### 2. Создать базы данных в postgres
```bash
kubectl exec -n messager deploy/postgres -- psql -U postgres -c "CREATE DATABASE messager_users;"
kubectl exec -n messager deploy/postgres -- psql -U postgres -c "CREATE DATABASE messager_messages;"
```

### 3. Применить конфигурацию через kustomize
```bash
# Dev
kustomize build k8s/overlays/dev | kubectl apply -f -

# Prod
kustomize build k8s/overlays/prod | kubectl apply -f -
```

### 4. Добавить запись в /etc/hosts
```bash
echo "127.0.0.1 dev.messager.local" | sudo tee -a /etc/hosts
```

### 5. Установить и настроить Argo CD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Получить пароль admin
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d

# Применить Application
kubectl apply -f argocd/application.yaml
```

---

## Чек-лист проверок перед сдачей

### 1. kustomize build работает без ошибок
```bash
kustomize build k8s/overlays/dev   # должен выдать валидный YAML
kustomize build k8s/overlays/prod  # должен выдать валидный YAML
```
Статус: проверено

### 2. Все pods в состоянии Running/Completed
```bash
kubectl get pods -n messager
```
Ожидаемый результат:
```
bff                   Running
frontend              Running
message-service       Running
user-service          Running
postgres              Running
minio                 Running
migrate-users         Completed
migrate-messages      Completed
```
Статус: проверено

### 3. Frontend доступен снаружи
```bash
curl -I http://dev.messager.local
```
Статус: доступен через Ingress (nginx / traefik)

### 4. API-цепочка frontend → bff → services работает
```bash
curl http://dev.messager.local/api/health
```
Статус: проверено

### 5. Загрузка файлов через S3 CSI
- message-service монтирует `/app/uploads` через PVC `message-uploads-pvc`
- Загруженные файлы сохраняются в MinIO bucket `uploads`
```bash
kubectl exec -n messager deploy/minio -- mc alias set local http://localhost:9000 minioadmin minioadmin123
kubectl exec -n messager deploy/minio -- mc ls local/uploads
```
✅ Статус: проверено

### 6. nodeAffinity влияет на размещение Pod
```bash
kubectl get pods -n messager -o wide
kubectl describe pod <pod-name> -n messager | grep -A 10 "Node-Selectors\|Affinity"
```

Правила:
| Сервис | Узел |
|---|---|
| postgres, minio | workload=system |
| frontend, bff, user-service | workload=app |
| message-service | workload=app (hard) + disk=fast (soft) |

Статус: проверено в prod overlay

### 7. Argo CD показывает Synced/Healthy
```bash
kubectl get applications -n argocd
```
Статус: автосинхронизация включена (automated + prune + selfHeal)

---

## Различия dev и prod

| Параметр | dev | prod |
|---|---|---|
| Реплики сервисов | 1 | 2 |
| CPU/Memory limits | базовые | повышенные |
| Ingress host | dev.messager.local | messager.example.com |
| Теги образов | latest | 1.0.0 (фиксированный) |
| nodeAffinity | базовая (workload=app) | строгая для всех сервисов |

---

## Частые проблемы

**postgres CrashLoopBackOff** — PVC повреждён, пересоздать:
```bash
kubectl delete deployment postgres -n messager
kubectl patch pvc postgres-pvc -n messager -p '{"metadata":{"finalizers":null}}'
kubectl delete pvc postgres-pvc -n messager --force
kubectl apply -f k8s/base/postgres-pvc.yaml
kubectl apply -f k8s/base/postgres.yaml
```

**migrate-* Error: password authentication failed** — в job.yaml hardcoded пароль, нужно брать из Secret:
```yaml
env:
  - name: GOOSE_DBSTRING
    valueFrom:
      secretKeyRef:
        name: messager-db-secret
        key: USER_DB_DSN
```

**message-service Pending: unbound PVC** — CSI драйвер не установлен, использовать local-path StorageClass в s3-csi.yaml.

