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





### chartmuseum
- Установил плагин helm-tiller, позволяющий запустить tiller локально
~~~
helm plugin install https://github.com/rimusz/helm-tiller
~~~
- Кастомизировал установку chartmuseum. Для этого необходимо отредактировать файл values.yaml(раскомментировать раздел ingress).  
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