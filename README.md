# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию
### Создайте кластер `minikube`
```shell-session
minikube start --driver=virtualbox --no-vtx-check --extra-config=kubelet.housekeeping-interval=10s
```
Создайте образ проекта в кластере
```shell-session
minikube image build -t django_app .
```

### Запустите базу данных.

Зайдите в папку `backend_main_django` и в ней создайте файл `.env` с указанием переменных окружения для запуска `Postgre`

```shell-session
POSTGRES_DB='test_k8s'
POSTGRES_USER='test_k8s'
POSTGRES_PASSWORD='OwOtBep9Frut'
```
Затем запустите 

```shell-session
$ docker-compose up
```
В докере будет создана ваша база данных для запуска джанговского проекта.

### Запустите проект.

Зайдите в папку `backend_main_django` `kubernetes` и создайте кластер

```shell-session
minikube start --driver=virtualbox --no-vtx-check
```

Затем создайте образ проекта из `Dockerfile`

```shell-session
minikube image build -t django_app .
```

Зайдите в папку `kubernetes` и создайте файл `secrets` для хранения переменных окружения джанговского проекта
в данном файле укажите переменные окружения проекта в поле `data`

```shell-session
apiVersion: v1
kind: ConfigMap
metadata:
  name: max-config
  labels:
    app: my-django-app
data:
  SECRET_KEY: "replace_me"
  DEBUG: 'False'
  DATABASE_URL: "postgres://test_k8s:OwOtBep9Frut@192.168.0.170:15432/test_k8s"
  ALLOWED_HOSTS: "127.0.0.1,localhost,star-burger.test,additional"
```

запустите загрузку переменных окружения

```shell-session
kubectl apply -f secrets.yaml
```

запустите загрузку джанго проекта

```shell-session
kubectl apply -f kuber_deploy.yaml
```
Файл `kuber_ingress.yaml` запускает настройку связывающую имя хоста с подом джанго в кластере

Запуск настройки
```shell-session
kubectl apply -f kuber_ingress.yaml
```
После запуска настройки необходимо посмотреть ip, на котором запустилась `ingress`
```shell-session
kubectl get ingress
```

и прописать в `etc/hosts`
```shell-session
192.168.59.106 star-burger.test
```
По адресу
- http://star-burger.test - доступен джанго проект

### clearsessions 
В папке `kubernetes` есть файл `django_clearsession.yaml` - это настройка таймера, который 
раз в месяц запускает под для очистки отработанных сессий в джанго проекте
Запуск настройки
```shell-session
kubectl apply -f django_clearsession.yaml
```
Если хотим запустить очистку вне таймера, отдельно, то:
```shell-session
kubectl get cronjob
kubectl create job --from=cronjob/cron-max cron-max-1
```
### migrate
В папке `kubernetes` есть файл `migrate_job.yaml` - это манифест для быстрого 
запуска миграцйи после обновления проекта
Запуск настройки
```shell-session
kubectl apply -f migrate_job.yaml
```

### Запуск postgresql внутри кластера, для тренировки
В папке `kubernetes` есть файл `postgree-pv.yaml` и `postgree-pvс.yaml` это манифесты
для настройки postgresql и создания в кластере папки для постоянного хранения информации

Эти манифесты нужно запустить
```shell-session
kubectl apply -f postgree-pv.yaml
kubectl apply -f postgree-pvc.yaml
```
После этого запускаем создание пода postgree из образа
```shell-session
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install max-postgree bitnami/postgresql `
--set primary.persistence.existingClaim=postgresql-pv-claim `
--set volumePermissions.enabled=true `
--set global.postgresql.auth.postgresPassword= ..... `
--set global.postgresql.auth.username=test_k8s `
--set global.postgresql.auth.password=OwOtBep9Frut `
--set global.postgresql.auth.database=test_k8s `
--set global.postgresql.service.ports.postgresql=25432
```
Запускается создание БД (укажите свой пароль администратора postgresPassword)
с `username=test_k8s`, `database=test_k8s`, `password=OwOtBep9Frut`, порт `ports.postgresql=25432`

Чтобы получить доступ к созданной БД и посмотреть как она работает:
```shell-session
kubectl run max-postgree-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:16.0.0-debian-11-r3 --env="PGPASSWORD=OwOtBep9Frut" `
 --command -- psql --host max-postgree-postgresql -U test_k8s -d test_k8s -p 25432
```

Чтобы иметь к данной БД доступ в проекте джанго, необходимо в файле `my_secrets.yaml`
прописать:
```shell-session
DATABASE_URL: "postgres://test_k8s:OwOtBep9Frut@max-postgree-postgresql:25432/test_k8s"
```

## Как запустить prod-версию
### Создайте кластер на внешнем ресурсе и подключитись к нему с вашего хоста
Кладем файл с настройками в папку `C:\Users\user\.kube`
Записываем переменную окружения в PowerShell
```shell-session
$env:KUBECONFIG = 'C:\Users\user\.kube\kubernetes-cluster-proba_kubeconfig.yaml'
```
и вызываем для проверки
```shell-session
kubectl cluster-info
```

### Запустите базу данных (описано выше).
### Запустите проект.

Образ для проекта прописан в манифесте и находится на DockerHub `anikeevmaks/kuber-django:first`

Зайдите в папку `kubernetes` и создайте файл `secrets` для хранения переменных окружения джанговского проекта
в данном файле укажите переменные окружения проекта в поле `data`

```shell-session
apiVersion: v1
kind: ConfigMap
metadata:
  name: max-config
  labels:
    app: my-django-app
data:
  SECRET_KEY: "replace_me"
  DEBUG: 'False'
  DATABASE_URL: "postgres://test_k8s:OwOtBep9Frut@max-postgree-postgresql:15432/test_k8s"
  ALLOWED_HOSTS: "127.0.0.1,localhost,star-burger.test,additional"
```
запустите загрузку переменных окружения
```shell-session
kubectl apply -f secrets.yaml
```

запустите загрузку джанго проекта
```shell-session
kubectl apply -f kuber_deploy.yaml
```
запуситите сервисы
```shell-session
kubectl get svc
```
увидите адрес, на котором находится созданный проект
```shell-session
django-service  LoadBalancer   10.98.192.255   158.160.3.24    8888:31044/TCP
```
запишите `158.160.3.24` в ALLOWED_HOSTS в манифесте `secrets` и перезапустите деплоймент
```shell-session
kubectl rollout restart deployment kuber-deployment
```

### migrate
В папке `kubernetes` есть файл `migrate_job.yaml` - это манифест для быстрого 
запуска миграцйи после обновления проекта
Запуск настройки
```shell-session
kubectl apply -f migrate_job.yaml
```

### проект будет доступен по адресу `158.160.3.24:8888`

### Запустите проект на доменном имени
В папке `kubernetes` в файле `secrets` укажите доменное имя `edu-stoic-dubinsky.sirius-k8s.dvmn.org`
полученное в настройках ALB-роутера в Яндекс облаке

```shell-session
ALLOWED_HOSTS: "127.0.0.1,localhost,star-burger.test,edu-stoic-dubinsky.sirius-k8s.dvmn.org"
```
запустите загрузку переменных окружения
```shell-session
kubectl apply -f secrets.yaml
```
запустите деплоймент
```shell-session
kubectl apply -f kuber_domen_deploy.yaml
```
### проект будет доступен по адресу `edu-stoic-dubinsky.sirius-k8s.dvmn.org`