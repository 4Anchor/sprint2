## В этом спринте мы соберем и задеплоим приложение из нашего GitLab репозитория в созданный кластер Kubernetes

Предварительно в Kubernetes требуется выполнить команды :

Данная команда создаст нам namespace в который будет деплоиться наш проект и сопутствующие к нему компоненты.

```
kubectl create namespace django-app
```
Так же забегая в будущее для Спринта 3 заранее создадим namespace monitoring
```
kubernetes create pod busybox.yaml -n monitoring
```

## Для начала первым этапом до настроим наш gitlab-runner (srv-0) для взаимодействия с Gitlab CI/CD и Kybernetes кластером.

### 1. Про инициализируем наш gitlab-runner:
- Перейдем в Settings - CI/CD Settings - Runners и так как gitlab-runner уже был развернут в 1м спринте, то остается только инициализация командой :

```
sudo gitlab-runner register --url https://anchor.gitlab.yandexcloud.net/ --registration-token $REGISTRATION_TOKEN
```
REGISTRATION_TOKEN - Находится в Runners - Project runners

При регистрации все шаги можно оставить по умолчанию, требуется выбрать только: executor shell


В доступных раннерах (Assigned project runners) должен появиться и быть активен ранер.

### 2. Так как gitlab-runner в пайплайне будет использовать shell команды и билдлить докер образ, то ему необходимо дать права на использование docker service, на хосте srv-o.
Выполните команду:
```
sudo usermod -aG docker gitlab-runner
```

### 3. Теперь требуется настроить соединение gitlab-runner к кластеру Kybernetes.

Для этого нам потребуется создать каталог /home/gitlab-runner/.kube и поместить в него config - конфигурационный файл для доступа к кластеру Kubernetes.
Выполняем команду:
```
 sudo chmod -R 600 /home/gitlab-runner/.kube/config
```
После чего от имени пользователя gitlab-runner выполняем команду:
```
kubectl get nodes
```
Должны отобразиться все ноды Kybernetes кластера.

На этом наш этам по подготовке gitlab-runner для взаимодействия с Kybernetes кластером считается завершенным.

## Следующим этом приступим к описанию пайплайна Gitlab CI/CD согласно требованиям изложенным в Спринте 2.

### 1. На основании предоставленного проекта (https://github.com/vinhlee95/django-pg-docker-tutorial.git) создадим Helm cahrt для автоматизации процесса Deploy django-app.

После того как Helm cahrt будет создан и добавлен в репозиторий, требуется создать файл .gitlab-ci.yml:

```
stages:
  - build
  - deploy
variables:
  REGISTRY_USER: $CI_REGISTRY_USER
  REGISTRY_PASSWORD: $CI_REGISTRY_PASSWORD
  DOCKER_REGISTRY_IMAGE: 4anchor/testapp
  DOCKER_REGISTRY_TAG: $DOCKER_REGISTRY_IMAGE:$CI_COMMIT_TAG
  CHART_PATH: ./
build:
  stage: build
  script:
    - docker build -t $DOCKER_REGISTRY_TAG ./app
    - docker login -u "$REGISTRY_USER" -p "$REGISTRY_PASSWORD" docker.io
    - docker push $DOCKER_REGISTRY_TAG
  only:
    - tags
deploy:
  stage: deploy
  script:
    - helm upgrade --install myapp $CHART_PATH -n django-app --set app.image.repository=$DOCKER_REGISTRY_IMAGE,image.tag=$CI_COMMIT_TAG
  only:
    - tags
```
 - На шаге build будет происходить сборка Docker image из каталога ./app с последующей его отправкой в Docker registry (docker.io)
   - В Variables надо добавить 2ве переменные с данными авторизации в Docker registry:
   ```
   CI_REGISTRY_PASSWORD
   CI_REGISTRY_USER
   ```
   - DOCKER_REGISTRY_TAG будет автоматически браться из вашего GitLab проекта и передаваться как в Registry так и в Helm chart.

 - На шаге deploy будет происходить helm --install если такой проект не разу не инсталлировался еще в Kybernetes кластер или helm upgrade  если проект уже есть, и вы пытаетесь изменить один из манифестов или параметров.
```
helm upgrade --install myapp $CHART_PATH -n django-app --set app.image.repository=$DOCKER_REGISTRY_IMAGE,image.tag=$CI_COMMIT_TAG
```
 
На данном этапе у нас уже есть сервис который развернут в Kybernetes.

Сам Helm chart является почти базовым, деплоится через kind: Deployment, имеет в себе values.yaml и секреты вынесены в отдельный kind: Secret.
Ingress манифест добавлен но для работы сервиса не используется.

### Этап до настройки для работы сервиса Django 

```
kubectl get pods -n django-app
```
Если выполним команду то увидите что STATUS не соответствует значению Running, так как сервису для работы требуется наличие БД PostgreSQL, давайте задеплоим ее:
Страница проекта: (https://artifacthub.io/packages/helm/bitnami/postgresql)
На srv-0 выполните команду:
 
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```
```
helm update
```
```
helm install my-release -f values.yaml oci://registry-1.docker.io/bitnamicharts/postgresql -n django-app
```
В файле values.yaml вам потребуется изменить параметры:

```
username: "admin"
password: PASSWORD
database: "tutorial-dev"

storageClass: local-storage
```
В соответствии с вашими параметрами Helm chart описанными в django-app

#### Так же для корректно деплоя и запуска PostgreSQL требуется предварительно создать storageClass с название local-storage. Все дополнительный компоненты  и манифесты содержаться в репозитории и доступны по адресу: https://github.com/4Anchor/components.git

Требуется скачать репозиторий, перейти в него и выполнить команды.

```
kubectl apply -f pv.yaml --namespace django-app
kubectl apply -f pstorageclass.yaml --namespace django-app
```
Так как PersistentVolume является локальным и размещен на worker-0, требуется обязательное наличие данной директории: /mnt/storage

Выполните команду:

```
kubectl get pods -n django-app
```
Оба пода должны находиться в состоянии Running

NAME                      READY   STATUS    RESTARTS   AGE
my-release-postgresql-0   1/1     Running   0          24h
myapp-6c464b69fd-x5vcq    1/1     Running   0          15h

## На этом этап Спринта 2  является завершенным. 
