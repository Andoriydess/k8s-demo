# Практична робота 10: Kubernetes (60 балів)

- Fork репозиторію: https://github.com/Andoriydess/k8s-demo
- Сервіс: Node.js + Express API (`k8s-demo`), порт 8080.

---

## 1. Локальний запуск

Команди:

```bash
git clone https://github.com/Andoriydess/k8s-demo
cd k8s-demo
npm install
npm run dev
```

Сервіс стартував із повідомленням `App started on port 8080 in mode dev`. Також було попередження `WARN: SECRET_TOKEN is not set`: локально секрет не передавався, на роботу сервісу це не впливає.

### Опис ендпоінтів

| Ендпоінт | Що повертає |
|---|---|
| `GET /api/status` | JSON зі статусом `ok`, режимом `appMode` (з `APP_MODE`, за замовчуванням `dev`), версією `1.0.0` і поточним `timestamp`. |
| `GET /api/items?filter=a` | Масив об'єктів `{id, name}` зі статичного списку (Item A, B, C), відфільтрований за підрядком `a` у назві (без урахування регістру). Без `filter` повертає всі три. |
| `GET /api/slow` | Відповідь `{status: "ok", delayMs: ...}` із затримкою `SLOW_MS` (за замовчуванням 2000 мс) або значенням параметра `?ms=`. |
| `GET /api/error` | Залежить від `ERROR_MODE`: при `off` (за замовчуванням) повертає 200 `{status: "ok", message: "No error"}`, при `always` завжди 500 `Simulated error`, при `probabilistic` у ~50% випадків 500. |
| `GET /healthz` | Текст `ok` (200). Використовується як liveness-перевірка. |
| `GET /ready` | Поки не минув `READINESS_DELAY_MS` (5 с), повертає 503 `{status: "not-ready"}`, після цього 200 `{status: "ready"}`. |

### Вивід curl

```text
PS> curl.exe http://localhost:8080/api/status
{"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-11-25T05:53:03.727Z"}

PS> curl.exe "http://localhost:8080/api/items?filter=a"
[{"id":1,"name":"Item A"}]

PS> curl.exe http://localhost:8080/api/slow
{"status":"ok","delayMs":2000}

PS> curl.exe http://localhost:8080/api/error
{"status":"ok","message":"No error"}

PS> curl.exe http://localhost:8080/healthz
ok

PS> curl.exe http://localhost:8080/ready
{"status":"ready"}
```

`/api/items?filter=a` повернув лише `Item A`, бо в назвах Item B і Item C немає літери `a`.

---

## 2. Контейнеризація

Команда збірки образу:

```bash
docker build -t k8s-demo-app:1.0.0 .
```

Команда запуску контейнера:

```bash
docker run --rm -p 8080:8080 -e SECRET_TOKEN=local k8s-demo-app:1.0.0
```

Перевірка:

```text
PS> curl.exe http://localhost:8080/api/status
{"status":"ok","appMode":"dev","version":"1.0.0","timestamp":"2026-11-25T05:55:27.244Z"}

PS> curl.exe http://localhost:8080/healthz
ok
```

Контейнер відповідає `appMode: dev`, бо ConfigMap у Docker не підключається, діє значення за замовчуванням із коду. У кластері значення `prod` береться з ConfigMap (див. розділ 5).

---

## 3. Minikube та перевірка кластера

Команда запуску:

```bash
minikube start --memory=4096 --cpus=2 --driver=docker
```

```text
PS> minikube profile list
┌──────────┬────────┬────────────┬──────────────┬─────────┬────────┬───────┬────────────────┬────────────────────┐
│ PROFILE  │ DRIVER │  RUNTIME   │      IP      │ VERSION │ STATUS │ NODES │ ACTIVE PROFILE │ ACTIVE KUBECONTEXT │
│ minikube │ docker │ containerd │ 192.168.49.2 │ v1.37.0 │ OK     │ 1     │ *              │ *                  │
└──────────┴────────┴────────────┴──────────────┴─────────┴────────┴───────┴────────────────┴────────────────────┘

PS> kubectl config get-contexts
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         minikube   minikube   minikube   default

PS> kubectl config use-context minikube
Switched to context "minikube".

PS> kubectl config current-context
minikube

PS> kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   92s   v1.37.0

PS> kubectl get pods -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-559f6c778d-xsgc9           1/1     Running   0          88s
kube-system   etcd-minikube                      1/1     Running   0          95s
kube-system   kindnet-qmlfs                      1/1     Running   0          88s
kube-system   kube-apiserver-minikube            1/1     Running   0          95s
kube-system   kube-controller-manager-minikube   1/1     Running   0          95s
kube-system   kube-proxy-z7x5k                   1/1     Running   0          88s
kube-system   kube-scheduler-minikube            1/1     Running   0          95s
kube-system   storage-provisioner                1/1     Running   0          93s
```

- Використовуваний контекст: **`minikube`**
- Статус ноди: **`Ready`** (одна нода, роль `control-plane`)

---

## 4. Розгортання Namespace, ConfigMap, Secret, Deployment, Service

Образ у Deployment (`maxnavrotsky/k8s-demo-app:1.0.0`) збирається всередині Minikube, щоб кластер його бачив без зовнішнього реєстру:

```bash
minikube image build -t maxnavrotsky/k8s-demo-app:1.0.0 .
```

Застосування маніфестів:

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

Перевірка:

```text
PS> kubectl get ns k8s-demo
NAME       STATUS   AGE
k8s-demo   Active   33s

PS> kubectl get configmap -n k8s-demo
NAME               DATA   AGE
k8s-demo-config    4      33s
kube-root-ca.crt   1      40s

PS> kubectl get secret -n k8s-demo
NAME              TYPE     DATA   AGE
k8s-demo-secret   Opaque   1      31s

PS> kubectl get deployment,rs,pods,svc -n k8s-demo
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/k8s-demo-api   2/2     2            2           31s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/k8s-demo-api-5c45559886   2         2         2       31s

NAME                                READY   STATUS    RESTARTS   AGE
pod/k8s-demo-api-5c45559886-7j59q   1/1     Running   0          31s
pod/k8s-demo-api-5c45559886-gl6nc   1/1     Running   0          31s

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/k8s-demo-api   ClusterIP   10.106.126.5   <none>        80/TCP    25s
```

Обидва pod-и мають статус `Running` і `READY 1/1`, Deployment показує `2/2`.

---

## 5. Доступ через Service та port-forward

```bash
kubectl port-forward svc/k8s-demo-api -n k8s-demo 8080:80
```

В іншому терміналі:

```text
PS> curl.exe http://localhost:8080/api/status
{"status":"ok","appMode":"prod","version":"1.0.0","timestamp":"2026-11-25T06:11:46.684Z"}

PS> curl.exe http://localhost:8080/api/items
[{"id":1,"name":"Item A"},{"id":2,"name":"Item B"},{"id":3,"name":"Item C"}]
```

`"appMode":"prod"` підтверджує, що відповідь надходить від сервісу в кластері: значення `APP_MODE: "prod"` береться з ConfigMap `k8s-demo-config` (локально і в Docker було `dev`).

### Через які сутності проходить запит

`curl` на `localhost:8080` потрапляє на `kubectl port-forward`, який відкриває тунель до Service `k8s-demo-api` (порт 80). Service за селектором `app: k8s-demo-api` обирає один зі здорових Pod-ів і перенаправляє запит на його `targetPort 8080`, де Express обробляє його. Шлях запиту: **`curl → port-forward → Service → Pod`**.

### Примітка щодо діагностики

Під час першої спроби `curl` повернув `"appMode":"dev"`, тобто відповідь прийшла не з кластера. Причина: контейнер `k8s-demo-app:1.0.0` із розділу 2 лишався запущеним і займав порт 8080 (це видно з `docker ps`: `0.0.0.0:8080->8080/tcp`). Після `docker stop` цього контейнера і перезапуску port-forward `curl` повернув `"appMode":"prod"`. Висновок: перед `port-forward` потрібно переконатися, що локальний порт вільний.

Q