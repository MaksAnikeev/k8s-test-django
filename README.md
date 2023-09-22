# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

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
  ALLOWED_HOSTS: "127.0.0.1,localhost,192.168.59.105,kubermax"
```

запустите загрузку переменных окружения

```shell-session
kubectl apply -f secrets.yaml
```

запустите загрузку джанго проекта

```shell-session
kubectl apply -f my_deploy.yaml
```
для доступа к проекту по внешнему ip нужно запустить его формирование:

```shell-session
minikube service django-service --url
```
по полученному url будет доступен сайт

```shell-session
http://192.168.59.105:32288
```

### Дополнительно
В папке `kubernetes` есть файл `my_deploy2.yaml` он создан для тестирования разных инструментов кубера, при его запуске 
также помимо джанговского проекта развертывается еще один под с проектом `tomcat` в качестве образца.
Также запускается формирование 2х реплик проекта и запускается `autoscalling`. 
Реализован еще один способ передачи переменных окружения в проект, а также используется тип сервиса `LoadBalancer`