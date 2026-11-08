# Домашнее задание 3.5 — Troubleshooting

> Черновик: исправление подготовлено по исходному манифесту. Прогон в учебном Kubernetes и скриншоты ещё не выполнены. Работа пока не готова к сдаче.

[Задание](https://github.com/netology-code/kuber-homeworks/blob/main/3.5/3.5.md) · [Исходный манифест](https://github.com/netology-code/kuber-homeworks/blob/main/3.5/files/task.yaml)

## Причина

Deployment `web-consumer` работает в namespace `web`, а Service `auth-db` и одноимённый Deployment — в `data`. Команда контейнера `while true; do curl auth-db; sleep 5; done` обращается к короткому DNS-имени. Оно ищется в namespace вызывающего pod, поэтому сервис из `data` таким именем не находится.

В исходном манифесте selector сервиса `app: auth-db` совпадает с меткой pod; `port` и `targetPort` равны 80. Исправление: обращаться к `auth-db.data`. Namespace в имени позволяет найти сервис через DNS Kubernetes. При стандартном домене полное имя — `auth-db.data.svc.cluster.local`.

Источник: [документация DNS Kubernetes](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/). Вывод по манифесту требует подтверждения в кластере.

## Воспроизведение и диагностика

Все команды выполняются только после выбора учебного context:

```bash
export LAB_CONTEXT='<учебный-context>'
kubectl --context="$LAB_CONTEXT" get nodes -o wide
kubectl --context="$LAB_CONTEXT" version
kubectl --context="$LAB_CONTEXT" apply -f files/namespaces.yaml
kubectl --context="$LAB_CONTEXT" apply -f https://raw.githubusercontent.com/netology-code/kuber-homeworks/main/3.5/files/task.yaml
kubectl --context="$LAB_CONTEXT" -n data rollout status deployment/auth-db --timeout=180s
kubectl --context="$LAB_CONTEXT" -n web rollout status deployment/web-consumer --timeout=180s
kubectl --context="$LAB_CONTEXT" -n web get pods -o wide
kubectl --context="$LAB_CONTEXT" -n data get pods,svc -o wide
kubectl --context="$LAB_CONTEXT" -n data get endpointslices -l kubernetes.io/service-name=auth-db
kubectl --context="$LAB_CONTEXT" -n web logs -l app=web-consumer --prefix --tail=10
kubectl --context="$LAB_CONTEXT" -n web exec deployment/web-consumer -- cat /etc/resolv.conf
kubectl --context="$LAB_CONTEXT" -n web exec deployment/web-consumer -- nslookup auth-db
kubectl --context="$LAB_CONTEXT" -n web exec deployment/web-consumer -- nslookup auth-db.data
```

Namespaces создаются отдельно, поскольку исходный манифест не содержит их определения. Ожидается ошибка разрешения `auth-db`, при этом `auth-db.data` должен разрешаться в ClusterIP сервиса. Это ожидаемые результаты, а не фактический вывод.

## Исправление

В [task-fixed.yaml](files/task-fixed.yaml) изменена только команда consumer:

```diff
-        - while true; do curl auth-db; sleep 5; done
+        - while true; do curl auth-db.data; sleep 5; done
```

```bash
kubectl --context="$LAB_CONTEXT" apply -f files/task-fixed.yaml
kubectl --context="$LAB_CONTEXT" -n web rollout status deployment/web-consumer --timeout=180s
```

## Подтверждение результата

```bash
kubectl --context="$LAB_CONTEXT" -n web get pods -l app=web-consumer
for pod in $(kubectl --context="$LAB_CONTEXT" -n web get pods -l app=web-consumer -o name); do
  echo "$pod"
  kubectl --context="$LAB_CONTEXT" -n web exec "$pod" -- sh -c \
    'code=$(curl -sS --connect-timeout 5 --max-time 10 -o /dev/null -w "%{http_code}" http://auth-db.data); rc=$?; printf "HTTP %s\n" "$code"; [ "$rc" -eq 0 ] && [ "$code" = 200 ]' || exit 1
done
kubectl --context="$LAB_CONTEXT" -n web logs -l app=web-consumer --prefix --tail=25
```

Критерий успеха: две готовые реплики, HTTP 200 из каждой и HTML nginx в логах после rollout. Статус Running сам по себе не доказывает доступность: бесконечный цикл продолжает работать даже при ошибках curl.

Фактические результаты и скриншоты будут добавлены после прогона в учебном кластере. Изображения учебного вывода не генерировались.

## Манифесты

- [Namespaces](files/namespaces.yaml).
- [Исправленное приложение](files/task-fixed.yaml).
- [Оригинал задания](https://github.com/netology-code/kuber-homeworks/blob/main/3.5/files/task.yaml).

Образы сохранены из задания. Возможные проблемы загрузки старых образов и совместимость с архитектурой узлов предстоит проверить при запуске.
