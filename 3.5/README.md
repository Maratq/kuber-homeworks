# Домашнее задание 3.5 — Troubleshooting

> Оформлено по предоставленным командам и выводу. Повторный запуск в кластере при оформлении не выполнялся.

### Цель задания

Устранить неисправности при деплое приложения.

### Чеклист готовности к домашнему заданию

1. Кластер k8s

### Задание 1. При деплое приложение web-consumer не может подключиться к auth-db. Необходимо исправить

1. Установить приложение по команде:

```shell
kubectl apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml

```
2. Выявить проблему и описать
3. Исправить проблему, описать, что сделано
4. Продемонстрировать, что проблема решена

## Решение

### 1. Анализ исходного манифеста

Скачаем исходный манифест и посмотрим, какие ресурсы он описывает:

```bash
$ wget https://github.com/netology-code/kuber-homeworks/raw/main/3.5/files/task.yaml
--2023-04-07 00:38:47--  https://github.com/netology-code/kuber-homeworks/raw/main/3.5/files/task.yaml
Распознаётся github.com (github.com)… 140.82.121.3
Подключение к github.com (github.com)|140.82.121.3|:443... соединение установлено.
HTTP-запрос отправлен. Ожидание ответа… 302 Found
Адрес: https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml [переход]
--2023-04-07 00:38:48--  https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml
Распознаётся raw.githubusercontent.com (raw.githubusercontent.com)… 185.199.108.133, 185.199.109.133, 185.199.110.133, ...
Подключение к raw.githubusercontent.com (raw.githubusercontent.com)|185.199.108.133|:443... соединение установлено.
HTTP-запрос отправлен. Ожидание ответа… 200 OK
Длина: 937 [text/plain]
Сохранение в: ‘task.yaml’

task.yaml                                   100%[===========================================================================================>]     937  --.-KB/s    за 0s      

2023-04-07 00:38:48 (13,0 MB/s) - ‘task.yaml’ сохранён [937/937]

```

| Ресурс | Namespace | Назначение |
|---|---|---|
| Deployment `web-consumer` | `web` | Две реплики `radial/busyboxplus:curl`, выполняющие `curl auth-db` |
| Deployment `auth-db` | `data` | Одна реплика nginx, порт `80/TCP` |
| Service `auth-db` | `data` | Тип `ClusterIP`, selector `app: auth-db`, порт `80/TCP` |

### 2. Причина ошибки

`web-consumer` находится в namespace `web`, а Service `auth-db` — в `data`. Короткое имя `auth-db` ищется в пространстве имён вызывающего pod. Поэтому consumer не находит нужный сервис.

Исправить это можно двумя способами:

- Сохранить разные namespaces и обращаться к `auth-db.data` (при стандартном домене полное имя — `auth-db.data.svc.cluster.local`).
- Разместить consumer и сервис в одном namespace.

В этом решении выбран второй вариант: `web-consumer` переносится в `data`. Команда `curl auth-db` при этом остаётся прежней.

[Документация Kubernetes: DNS для сервисов и pod](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/).

### 3. Исправление манифеста

Поэтому исправим namespace в Deployment web-consumer на data   

```bash
$ sed 's/namespace: web/namespace: data/' task.yaml > new_task.yaml
$ diff task.yaml new_task.yaml 
5c5
<   namespace: web
---
>   namespace: data

```

### 4. Развёртывание

Создадим namespace `data` и применим два Deployment и Service из [new_task.yaml](files/new_task.yaml):    

```bash
$ kubectl create ns data
namespace/data created
$ kubectl get ns
NAME              STATUS   AGE
data              Active   9s
default           Active   4h13m
kube-node-lease   Active   4h13m
kube-public       Active   4h13m
kube-system       Active   4h13m
$ kubectl apply -f new_task.yaml 
deployment.apps/web-consumer created
deployment.apps/auth-db created
service/auth-db created

```

### 5. Проверка результата

Проверим pod, Service и endpoints:

```bash
$ kubectl -n data get pods
NAME                            READY   STATUS    RESTARTS   AGE
auth-db-795c96cddc-2hfv7        1/1     Running   0          6m53s
web-consumer-577d47b97d-lbm9r   1/1     Running   0          6m53s
web-consumer-577d47b97d-sgtwm   1/1     Running   0          6m53s
$ kubectl -n data get svc
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
auth-db   ClusterIP   10.233.26.139   <none>        80/TCP    6m58s
$ kubectl -n data get endpoints
NAME      ENDPOINTS         AGE
auth-db   10.233.71.13:80   7m7s

```

Посмотрим логи consumer. В предоставленном выводе видна HTML-страница nginx:  

```bash
$ kubectl -n data logs --tail=15 pods/web-consumer-577d47b97d-lbm9r
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

**Результат по предоставленному выводу:** consumer получает страницу `Welcome to nginx!`. После переноса в `data` короткое имя `auth-db` указывает на нужный Service.

## Манифесты

- [Исходный манифест](files/task.yaml).
- [Исправленный манифест: общий namespace data](files/new_task.yaml).

Скриншоты в присланном материале отсутствуют; приведён текстовый вывод команд.
