# Домашнее задание «Обновление приложений»

Выбор стратегии, обновление nginx и откат Deployment.

> Команды и вывод приведены из предоставленного ответа; повторный запуск в кластере при оформлении не выполнялся.

## Задание 1. Выбор стратегии обновления
**Выбранная стратегия — Recreate (пересоздание).**

### Обоснование

**Ограничение по ресурсам.** Запас по ресурсам составляет всего 20%. Такой запас ограничивает параллельный запуск старых и новых реплик. RollingUpdate можно настроить без дополнительных реплик, но это не устраняет несовместимость версий. Recreate сначала полностью удаляет старые поды, освобождая 100% выделенных ресурсов, и только потом запускает новые.

**Несовместимость версий.** Новые версии несовместимы со старыми. Стратегия RollingUpdate создала бы момент, когда трафик обрабатывают и старая, и новая версии одновременно (что может привести к ошибкам в БД или API). Recreate гарантирует, что в любой момент времени работает только одна версия приложения.

**Компромисс.** Мы сознательно идем на короткий простой (downtime) во время переключения, чтобы избежать одновременной работы несовместимых версий при ограниченных ресурсах.

## Задание 2. Обновление приложения  
### 2.1. Создание Deployment с nginx 1.19

Создадим Deployment с пятью репликами и контейнерами nginx и multitool.  
Для ускорения обновления заданы `maxSurge: 100%` и `maxUnavailable: 50%`. При пяти репликах допускается до пяти дополнительных реплик и недоступность не более двух: минимум три остаются доступными с точки зрения Deployment. Этот вариант требует дополнительных ресурсов; условия задания 1 рассматриваются отдельно.

<details>

  <summary><b>Манифест deployment-rollingupdate.yml</b></summary>
  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netology-deployment-back
  labels:
    app: netology-back
spec:
  replicas: 5
  revisionHistoryLimit: 10
  strategy:
    rollingUpdate:
      maxSurge: 100%
      maxUnavailable: 50%
  selector:
    matchLabels:
      app: netology-back
  template:
    metadata:
      labels:
        app: netology-back
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        env:
          - name: HTTP_PORT
            value: "8080"
        ports:
        - containerPort: 8080      
      - name: nginx119
        image: nginx:1.19
        ports:
        - containerPort: 80
```
</details>

  
Создадим и проверим
```bash
kubectl apply -f deployment-rollingupdate.yml
```
```bash
kubectl get deployments.apps
```
```
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
netology-deployment-back   5/5     5            5           4m50s
```
### 2.2. Обновление до nginx 1.20

Теперь изменим версию nginx 
```yaml
- name: nginx120
  image: nginx:1.20
```
В отдельном окне можно ввести команду для отображения состояния подов в реальном времени
```bash
kubectl get pod -w
```
Применим изменения, тем самым запустив обновление
```bash
kubectl apply -f deployment-rollingupdate.yml
```
Когда вывод завершится, посмотрим на наши поды
```bash
kubectl get pods
```
```
NAME                                       READY   STATUS    RESTARTS   AGE
netology-deployment-back-7574fd5b7-5ngwx   2/2     Running   0          84s
netology-deployment-back-7574fd5b7-82ncn   2/2     Running   0          84s
netology-deployment-back-7574fd5b7-n6xr4   2/2     Running   0          84s
netology-deployment-back-7574fd5b7-rtbsm   2/2     Running   0          84s
netology-deployment-back-7574fd5b7-ztw92   2/2     Running   0          84s
```
Подключимся к поду и проверим текущую версию nginx
```bash
kubectl exec -it netology-deployment-back-7574fd5b7-5ngwx -c nginx120 -- /bin/bash
```
```bash
nginx -version
```
```
nginx version: nginx/1.20.2
```
Приложение обновлено.  
Проверим историю обновлений командой
```bash
kubectl rollout history deployment/netology-deployment-back
```
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

### 2.3. Попытка обновления до nginx 1.28

Попытаемся обновиться на версию 1.28
```yaml
      - name: nginx128
        image: nginx:1.28
```
```bash
kubectl apply -f deployment-rollingupdate.yml
```
```bash
kubectl get pods
```
```
NAME                                        READY   STATUS             RESTARTS   AGE
netology-deployment-back-6ff467cd6c-6jmkj   1/2     ErrImagePull       0          76s
netology-deployment-back-6ff467cd6c-94p49   1/2     ImagePullBackOff   0          76s
netology-deployment-back-6ff467cd6c-bnzbr   1/2     ImagePullBackOff   0          76s
netology-deployment-back-6ff467cd6c-c7szx   1/2     ErrImagePull       0          76s
netology-deployment-back-6ff467cd6c-tnb26   1/2     ImagePullBackOff   0          76s
netology-deployment-back-7574fd5b7-82ncn    2/2     Running            0          25m
netology-deployment-back-7574fd5b7-n6xr4    2/2     Running            0          25m
netology-deployment-back-7574fd5b7-ztw92    2/2     Running            0          25m
```
100% нашего приложения это 5 реплик.
Новые реплики не запустились из-за ошибки загрузки образа. Статусы `ErrImagePull` и `ImagePullBackOff` сами по себе не доказывают отсутствие тега: точная причина определяется по Events в `kubectl describe pod <имя-pod>`.
Половина старого приложения (округлённая до бОльшего значения, то есть до 3 реплик) осталась функционировать.  
Посмотрим на количество наших ревизий
```bash
kubectl get rs
```
```
NAME                                  DESIRED   CURRENT   READY   AGE
netology-deployment-back-6ff467cd6c   5         5         0       7m42s
netology-deployment-back-7574fd5b7    3         3         3       31m
netology-deployment-back-7d6db7d77f   0         0         0       36m
```
```bash
kubectl rollout history deployment/netology-deployment-back
```
```
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
3         <none>
```
### 2.4. Откат обновления

Выполним откат
```bash
kubectl rollout undo deployment/netology-deployment-back
```
```bash
kubectl get pods
```
```
NAME                                       READY   STATUS    RESTARTS   AGE
netology-deployment-back-7574fd5b7-82ncn   2/2     Running   0          42m
netology-deployment-back-7574fd5b7-8c65g   2/2     Running   0          37s
netology-deployment-back-7574fd5b7-98h75   2/2     Running   0          37s
netology-deployment-back-7574fd5b7-n6xr4   2/2     Running   0          42m
netology-deployment-back-7574fd5b7-ztw92   2/2     Running   0          42m
```
Мы выполнили откат.  
Посмотрим на историю ревизий
```bash
kubectl rollout history deployment/netology-deployment-back
```
```
REVISION  CHANGE-CAUSE
1         <none>
3         <none>
4         <none>
```
Вывод говорит о том, что ревизия, которая шла под номером 2 теперь актуальна и ей был присвоен номер 4
