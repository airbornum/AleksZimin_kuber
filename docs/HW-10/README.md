### [Вернуться в корень репо](/../../)

# Выполнено ДЗ №10

 - [x] Основное ДЗ
 - [ ] Задание со *
## В процессе сделано:

### Подготовил инфраструктуру
- Создал кластер в GKE
~~~
CLUSTER_NAME=zav-cluster
gcloud beta container --project "k8s-zav" clusters create "$CLUSTER_NAME" --zone "europe-north1-b" --no-enable-basic-auth --cluster-version "1.15.4-gke.22" --machine-type "n1-standard-2" --image-type "UBUNTU" --disk-type "pd-standard" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "3" --enable-stackdriver-kubernetes --enable-ip-alias --network "projects/k8s-zav/global/networks/default" --subnetwork "projects/k8s-zav/regions/europe-north1/subnetworks/default" --default-max-pods-per-node "110" --enable-network-policy --addons HorizontalPodAutoscaling,HttpLoadBalancing --enable-autoupgrade --enable-autorepair
~~~
- Установил клиент helm2
~~~
curl -LO https://get.helm.sh/helm-v2.16.1-linux-amd64.tar.gz \
  && tar -zxvf helm-v2.16.1-linux-amd64.tar.gz && chmod +x linux-amd64/helm \
  && sudo mv linux-amd64/helm /usr/local/bin/helm \
  && rm helm-v2.16.1-linux-amd64.tar.gz \
  && rm -R linux-amd64
helm version --client
~~~
- Дал необходимые права для tiller в кластере (выполнять из корня репо)
~~~ 
kubectl apply -f kubernetes-templating/tiller/
~~~
- Установил tiller в кластер и проверил корректность его установки
~~~
helm init --service-account=tiller
helm version
~~~
### Установил nginx-ingress и cert-manager
- Установил релиз nginx-ingress, используя первый способ установки - Helm 2 и tiller с правами cluster-admin
~~~
helm upgrade --install nginx-ingress stable/nginx-ingress --wait \
  --namespace=nginx-ingress \
  --version=1.11.1
~~~
- Подготовил tiller для работы воторого способа - Helm 2 и tiller с ограниченными привилегиями 
~~~
kubectl apply -f kubernetes-templating/cert-manager/
helm init --tiller-namespace cert-manager --service-account tiller-cert-manager
helm repo add jetstack https://charts.jetstack.io
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/deploy/manifests/00-crds.yaml
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation="true"
~~~
- Проверил права tiller в других ns (прав нет)
~~~
helm upgrade --install cert-manager jetstack/cert-manager --wait \
--namespace=nginx-ingress \
--version=0.9.0 \
--tiller-namespace cert-manager
~~~

- Установил релиз cert-manager
~~~
helm upgrade --install cert-manager jetstack/cert-manager --wait \
--namespace=cert-manager \
--version=0.9.0 \
--tiller-namespace cert-manager
~~~
- Установка релиза провалилась, т.к. прошлый релиз делали в другой ns
- Удалил провалившийся релиз
~~~
helm delete --purge cert-manager \
  --tiller-namespace cert-manager
~~~
- Установим, используя ключ atomic
~~~
helm upgrade --install cert-manager jetstack/cert-manager --wait \
  --namespace=cert-manager \
  --version=0.9.0 \
  --tiller-namespace cert-manager \
  --atomic
~~~
### Самостоятельное задание
- Снова ошибка. Для данного релиза нет смысла использовать tiller с ограниченными правами, т.к. чарт создает clusterrole (необходимы права на создание объектов в других ns). Использовал ранее созданный аккаунт tiller, который обладаем правами cluster-admin
~~~
helm init --upgrade --service-account=tiller
helm upgrade --install cert-manager jetstack/cert-manager --wait \
  --namespace=cert-manager \
  --version=0.9.0 \
  --atomic
~~~
- Создал clusterissuer
~~~
kubectl apply -f kubernetes-templating/cert-manager/05-clusterissuer.yaml 
~~~





### Chartmuseum
- Установил плагин helm-tiller, позволяющий запустить tiller локально
~~~
helm plugin install https://github.com/rimusz/helm-tiller
~~~
- Кастомизировал установку chartmuseum. Для этого необходимо в файле values.yaml описать раздел ingress.  
- Включил создание ingress ресурса с корректным hosts.name. Hostname получим из внешнего ip сервиса nginx-ingress + .nip.io (таким образом полчим dns имя)
~~~
kubectl get svc -A | grep nginx-ingress
~~~
- Установил chartmuseum (запускать из корня репо)
~~~
helm tiller run \
  helm upgrade --install chartmuseum stable/chartmuseum --wait \
  --namespace=chartmuseum \
  --version=2.3.2 \
  -f kubernetes-templating/chartmuseum/values.yaml
~~~
- У tiller внутри кластера нет данных о релизе chartmuseum, т.к. helm-tiller использует секреты для хранения инфы о релизах, а tiller внутри кластера - в configmaps
~~~
helm list
helm tiller run helm list
~~~
- Переустановил chartmuseum, указав helm-tiller, что он должен сохранять информацию о релизах в configmaps
~~~
export HELM_TILLER_STORAGE=configmap
helm delete --purge chartmuseum 
helm tiller run \
  helm upgrade --install chartmuseum stable/chartmuseum --wait \
  --namespace=chartmuseum \
  --version=2.3.2 \
  -f kubernetes-templating/chartmuseum/values.yaml
~~~
- Убедился, что tiller внутри кластера увидел релиз
~~~
helm status chartmuseum
~~~
- Зашел на chartmuseum по URL. Серт выдан Fake LE Intermediate X1, т.к. я использовал тестовый сервер letsencrypt (https://acme-staging-v02.api.letsencrypt.org/directory)


### Harbor и helm 3.
- Установил helm3
~~~
curl -LO https://get.helm.sh/helm-v3.0.2-linux-amd64.tar.gz \
  && tar -zxvf helm-v3.0.2-linux-amd64.tar.gz && chmod +x linux-amd64/helm \
  && sudo mv linux-amd64/helm /usr/local/bin/helm3 \
  && rm helm-v3.0.2-linux-amd64.tar.gz \
  && rm -R linux-amd64
helm3 version --client
~~~
- Удалил tiler из кластера и проверил работу helm2 и helm3
~~~
kubectl delete deployment tiller-deploy -n kube-system
helm list
helm3 list
~~~

### Самостоятельное задание. Установка harbor с помощью helm3
- Добавил репозиторий harbor
~~~
helm3 repo add harbor https://helm.goharbor.io
~~~
- Описал параметры чарта, которые необходимо изменить, в файле kubernetes-templating/harbor/values.yaml
- Создал ns для harbor
~~~
kubectl apply -f kubernetes-templating/harbor/01-ns-harbor.yaml
~~~
- Установил harbor
~~~
helm3 upgrade --install harbor harbor/harbor --wait \
--namespace=harbor \
--version=1.1.2 \
-f kubernetes-templating/harbor/values.yaml
~~~
- Зашел на harbor по адресу https://harbor.35.228.208.40.nip.io работает и SSL сертификат валидный(Серт выдан Fake LE Intermediate X1, т.к. я использовал тестовый сервер letsencrypt (https://acme-staging-v02.api.letsencrypt.org/directory)
- Посмотрел инфо о релизе
~~~
kubectl get secrets -n harbor -l owner=helm
~~~

### Свой helm chart
- Инициализировал структуру директории
~~~
helm create kubernetes-templating/socks-shop
~~~
- Удалил файл kubernetes-templating/socks-shop/values.yaml и содержимое каталога kubernetes-templating/socks-shop/templates

- Скопировал файл all.yaml из репо
~~~
curl https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-04/05-Templating/manifests/all.yaml   -o kubernetes-templating/socks-shop/templates/all.yaml
~~~
- Установил чарт и проверил его работу (зашел на ноду, т.к. порты извне не открыты)
~~~
helm3 upgrade --install socks-shop kubernetes-templating/socks-shop
kubectl get svc | grep front-end  # - запомнил порт
gcloud compute ssh $(kubectl get pods -o wide | grep -m 1 front-end | awk '{print $7}') --zone "europe-north1-b"
curl 127.0.0.1:30001
~~~
- В таком виде чарт использовать неудобно. Рекомендуется разбить его на разные и использовать зависимости. Выполнил инструкции из презентации. 
~~~
helm create kubernetes-templating/frontend
rm kubernetes-templating/frontend/values.yaml
rm -R kubernetes-templating/frontend/templates/*
touch kubernetes-templating/frontend/templates/{deployment.yaml,service.yaml,ingress.yaml}
touch kubernetes-templating/frontend/values.yaml
touch kubernetes-templating/socks-shop/requirements.yaml
~~~
- Обновил зависимости

helm dep update kubernetes-templating/socks-shop

- Установил чарт с другим значением nodeport
~~~
helm3 uninstall socks-shop
kubectl create ns socks-shop
helm3 upgrade --install socks-shop kubernetes-templating/socks-shop --namespace socks-shop --set frontend.service.NodePort=31234
~~~
- Проверил новое значение порта у nodeport. Значение изменилось
~~~
kubectl get svc -n socks-shop
~~~
- Зашел на сайт http://shop.35.228.208.40.nip.io

### Проверка.
- Заархивировал чарты
~~~
helm3 package kubernetes-templating/socks-shop
helm3 package kubernetes-templating/frontend
~~~
- Загрузил их в harbor, используя web-интерфес
- Добавил свой harbor как репо в helm, используя скрипт kubernetes-templating/repo.sh 

### Kubecfg
- Вынес из конфига all.yaml Deployment и Service для catalogue и payment в директорию kubernetes-templating/kubecfg
~~~
mkdir -p kubernetes-templating/kubecfg
touch kubernetes-templating/kubecfg/{catalogue-deployment.yaml,catalogue-service.yaml,payment-deployment.yaml,payment-service.yaml}
~~~
- Установил kubecfg
~~~
curl -LO https://github.com/bitnami/kubecfg/releases/download/v0.14.0/kubecfg-linux-amd64 \
  && chmod +x ./kubecfg-linux-amd64 \
  && sudo mv ./kubecfg-linux-amd64 /usr/local/bin/kubecfg


~~~
- Вывод команды kubecfg version
~~~
kubecfg version: v0.14.0
jsonnet version: v0.14.0
client-go version: v0.0.0-master+$Format:%h$
~~~
- Написал свой файл services.jsonnet
~~~
touch kubernetes-templating/kubecfg/services.jsonnet
~~~
- Проверил генерацию манифестов и установил их
~~~
kubecfg show kubernetes-templating/kubecfg/services.jsonnet
kubecfg update services.jsonnet --namespace socks-shop
~~~

### Kustomize
- Кастомизировал сервис и деплоймент queue-master
~~~
mkdir -p kubernetes-templating/kustomize/base
mkdir -p kubernetes-templating/kustomize/overrides/{socks-shop,socks-shop-prod}
touch kubernetes-templating/kustomize/base/{deployment.yaml,kustomization.yaml,service.yaml}
touch kubernetes-templating/kustomize/overrides/{socks-shop,socks-shop-prod}/kustomization.yaml
~~~
- Установил кастомизированный релиз
~~~
kubectl apply -k kubernetes-templating/kustomize/overrides/socks-shop
~~~
- Кастомизированные деплоймент и сервис создались:
~~~
service/dev-queue-master created
deployment.apps/dev-queue-master created
~~~


### [Вернуться в корень репо](/../../)
