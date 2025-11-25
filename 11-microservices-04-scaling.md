
# Домашнее задание к занятию «Микросервисы: масштабирование» - Барышков Михаил

Вы работаете в крупной компании, которая строит систему на основе микросервисной архитектуры.
Вам как DevOps-специалисту необходимо выдвинуть предложение по организации инфраструктуры для разработки и эксплуатации.

## Задача 1: Кластеризация

Предложите решение для обеспечения развёртывания, запуска и управления приложениями.
Решение может состоять из одного или нескольких программных продуктов и должно описывать способы и принципы их взаимодействия.

Решение должно соответствовать следующим требованиям:
- поддержка контейнеров;
- обеспечивать обнаружение сервисов и маршрутизацию запросов;
- обеспечивать возможность горизонтального масштабирования;
- обеспечивать возможность автоматического масштабирования;
- обеспечивать явное разделение ресурсов, доступных извне и внутри системы;
- обеспечивать возможность конфигурировать приложения с помощью переменных среды, в том числе с возможностью безопасного хранения чувствительных данных таких как пароли, ключи доступа, ключи шифрования и т. п.

Обоснуйте свой выбор.

## Задача 1: Кластеризация

**Предлагаемое решение:** Kubernetes + Istio + Helm + HashiCorp Vault

### Компоненты решения:
- **Kubernetes** - оркестратор контейнеров
- **Istio** - service mesh для управления трафиком
- **Helm** - менеджер пакетов для Kubernetes
- **HashiCorp Vault** - управление секретами
- **Prometheus + Horizontal Pod Autoscaler** - автоматическое масштабирование
- **Cert-Manager** - управление TLS сертификатами
- **ExternalDNS** - автоматическое управление DNS

### Соответствие требованиям:

| Требование | Реализация в Kubernetes |
|------------|---------------------|
| Поддержка контейнеров | Нативная поддержка Docker, containerd через CRI |
| Обнаружение сервисов и маршрутизация | CoreDNS + Service Discovery + Istio Service Mesh |
| Горизонтальное масштабирование | Horizontal Pod Autoscaler (HPA) |
| Автоматическое масштабирование | HPA + Cluster Autoscaler + Custom Metrics |
| Разделение ресурсов | Namespaces, Network Policies, RBAC |
| Конфигурирование через переменные среды | ConfigMaps, Secrets, Environment Variables |
| Безопасное хранение чувствительных данных | HashiCorp Vault + Kubernetes Secrets с шифрованием |

### Детали реализации:

#### 1. Поддержка контейнеров
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user
  template:
    metadata:
      labels:
        app: user
    spec:
      containers:
      - name: user-container
        image: user-service:1.0.0
        ports:
        - containerPort: 8080
```

#### 2. Обнаружение сервисов и маршрутизация

```yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

#### 3. Горизонтальное масштабирование

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

#### 4. Автоматическое масштабирование

- HPA на основе CPU, памяти и кастомных метрик

- Cluster Autoscaler для автоматического добавления узлов

- Vertical Pod Autoscaler для оптимизации ресурсов

#### 5. Разделение ресурсов

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-isolation
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: database
```

#### 6. Конфигурирование приложений

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database.host: "postgresql-service"
  log.level: "INFO"

apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
```

#### 7. Безопасное хранение секретов с Vault

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-vault
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-app:latest
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: vault-secrets
              key: db-password
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: vault-secrets
              key: api-key
```

#### Архитектура решения:

```text
Внешние пользователи
        ↓
[ Load Balancer ]
        ↓
[ Ingress Controller ] ← [ Cert-Manager ] ← [ Let's Encrypt ]
        ↓
[ Istio Ingress Gateway ]
        ↓
[ Kubernetes Services ]
        ↓
[ Application Pods ] → [ Vault ] → [ Prometheus ]
        ↓
[ CoreDNS ] ← [ ExternalDNS ]
```

#### Преимущества выбора:

- Kubernetes - промышленный стандарт с богатой экосистемой

- Istio - продвинутые возможности маршрутизации и безопасности

- Helm - управление сложными приложениями через charts

- Vault - безопасное хранение и ротация секретов

- Автоматическое масштабирование - эффективное использование ресурсов

- Мульти-клауд - переносимость между облачными провайдерами

#### Мониторинг и логирование:

```yaml
# Дополнительные компоненты для observability
- Prometheus для сбора метрик
- Grafana для визуализации
- Jaeger для распределенной трассировки
- Fluentd/Fluent Bit для сбора логов
- Elasticsearch + Kibana для анализа логов
```

#### Безопасность:

- Pod Security Standards - политики безопасности подов

- Network Policies - изоляция сетевого трафика

- RBAC - управление доступом на основе ролей

- TLS - сквозное шифрование через Istio

Данное решение предоставляет полный стек для управления микросервисами в production-среде с соблюдением всех требований безопасности и масштабируемости.


## Задача 2: Распределённый кеш * (необязательная)

Разработчикам вашей компании понадобился распределённый кеш для организации хранения временной информации по сессиям пользователей.
Вам необходимо построить Redis Cluster, состоящий из трёх шард с тремя репликами.

### Схема:

![11-04-01](https://user-images.githubusercontent.com/1122523/114282923-9b16f900-9a4f-11eb-80aa-61ed09725760.png)

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
