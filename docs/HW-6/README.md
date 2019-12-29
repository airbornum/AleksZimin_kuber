### [Вернуться в корень репо](/../../)

# Выполнено ДЗ №6

 - [x] Основное ДЗ
 - [x] Задание со *
## В процессе сделано:

### Подготовил инфраструктуру
- Создал кластер в GKE
~~~
CLUSTER_NAME=debug-cluster
gcloud beta container --project "k8s-zav" clusters create "$CLUSTER_NAME" --zone "europe-north1-b" --no-enable-basic-auth --cluster-version "1.15.4-gke.22" --machine-type "n1-standard-2" --image-type "UBUNTU" --disk-type "pd-standard" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "3" --enable-stackdriver-kubernetes --enable-ip-alias --network "projects/k8s-zav/global/networks/default" --subnetwork "projects/k8s-zav/regions/europe-north1/subnetworks/default" --default-max-pods-per-node "110" --enable-network-policy --addons HorizontalPodAutoscaling,HttpLoadBalancing --enable-autoupgrade --enable-autorepair
~~~
- Установил kubectl debug на свое рабочее место
~~~
export PLUGIN_VERSION=0.1.1
curl -Lo kubectl-debug.tar.gz https://github.com/aylei/kubectl-debug/releases/download/v${PLUGIN_VERSION}/kubectl-debug_${PLUGIN_VERSION}_linux_amd64.tar.gz
tar -zxvf kubectl-debug.tar.gz kubectl-debug
sudo mv kubectl-debug /usr/local/bin/
~~~
- Задеплоил в кластер приложение из ДЗ-3 (выполнять из корня репо)
~~~
kubectl apply -f kubernetes-networks/web-deploy.yaml
~~~

### Проверил работу kubectl debug
- Подключился к поду, используя дебаг контейнер
~~~
kubectl debug  $(kubectl get pods | grep -m 1 web | awk '{print $1}')
~~~
- Ничего не получилось, т.к. плагин пытается подключиться напрямую к ноде, но у нас нет прямого доступа к ней, поэтому необходим параметр --port-forward. Так же мы не создавали daemon set с дебаг агентом, поэтому при каждом запуске необходимо указывать параметр --agentless:
~~~
kubectl debug  --agentless --port-forward $(kubectl get pods | grep -m 1 web | awk '{print $1}')
~~~
- Проверил работу приложения
~~~
curl http://127.0.0.1:8000/index.html
~~~
- Проверил работу команды strace
~~~
strace -c -p1
~~~
- Команда успешно выполнилась, т.к. разработчики добавили в контейнер capability SYS_PTRACE. В этом можно убедиться, если зайти на ноду, где запущен под и проверить контейнер
- Подключился через gcloud на ноду нашего кластера, где запущен дебаг контейнер 
~~~
gcloud compute ssh $(kubectl get pods -o wide | grep -m 1 web | awk '{print $7}') --zone "europe-north1-b"
~~~
- Проверил дебаг контейнер
~~~
docker inspect $(docker ps | grep nicolaka | awk '{print $1}') | grep -A 3 CapAdd
~~~

### Попрактиковался с  iptables-tailer
- Установил netperf-operator
~~~
cd && git clone https://github.com/piontec/netperf-operator.git
kubectl apply -f ./netperf-operator/deploy/crd.yaml
kubectl apply -f ./netperf-operator/deploy/rbac.yaml
kubectl apply -f ./netperf-operator/deploy/operator.yaml
~~~
- Выполнил замер скорости
~~~
kubectl apply -f ./netperf-operator/deploy/cr.yaml
kubectl describe netperfs.app.example.com
~~~
- Добавил сетевую политику для Calico (выполнять из корня репо)
~~~
mkdir -p kubernetes-debug/kit
curl https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-03/Debugging/netperf-calico-policy.yaml -o kubernetes-debug/kit/netperf-calico-policy.yaml
kubectl apply -f kubernetes-debug/kit/netperf-calico-policy.yaml
~~~
- Повторно запустил тест скорости
~~~
kubectl delete -f ./netperf-operator/deploy/cr.yaml
kubectl apply -f ./netperf-operator/deploy/cr.yaml
kubectl describe netperfs.app.example.com
~~~
- Тест висит в статусе Started. Проверим логи на ноде, где запущен клиент netperf
~~~
gcloud compute ssh $(kubectl get pods -o wide | grep -m 1 netperf-client | awk '{print $7}') --zone "europe-north1-b"
sudo iptables --list -nv | grep DROP
sudo iptables --list -nv | grep LOG
journalctl -k | grep calico
~~~
- Логи так смотреть неудобно. Установил iptables-tailer из репо проекта
~~~
curl https://raw.githubusercontent.com/box/kube-iptables-tailer/master/demo/daemonset.yaml -o kubernetes-debug/kit/iptables-tailer-daemonset.yaml
kubectl apply -f kubernetes-debug/kit/iptables-tailer-daemonset.yaml
~~~
- iptables-tailer не работает, т.к. нет прав. Создал сервис аккаунт, дал ему права и запустил из под него iptables-tailer (использовал манифест из репо со сниппетами, в котором исправлены баги)
~~~
curl https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-03/Debugging/kit-serviceaccount.yaml -o kubernetes-debug/kit/kit-serviceaccount.yaml
kubectl apply -f kubernetes-debug/kit/kit-serviceaccount.yaml

curl https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-03/Debugging/kit-clusterrole.yaml -o kubernetes-debug/kit/kit-clusterrole.yaml
kubectl apply -f kubernetes-debug/kit/kit-clusterrole.yaml

curl https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-03/Debugging/kit-clusterrolebinding.yaml -o kubernetes-debug/kit/kit-clusterrolebinding.yaml
kubectl apply -f kubernetes-debug/kit/kit-clusterrolebinding.yaml

curl https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-03/Debugging/iptables-tailer.yaml -o kubernetes-debug/kit/iptables-tailer-daemonset.yaml
kubectl apply -f kubernetes-debug/kit/iptables-tailer-daemonset.yaml
~~~
- Снова запустил тест и посмотрел события
~~~
kubectl delete -f ./netperf-operator/deploy/cr.yaml
kubectl apply -f ./netperf-operator/deploy/cr.yaml
kubectl get events -A
kubectl describe pod --selector=app=netperf-operator
~~~
- Теперь в событиях видны сетевые проблемы

## Задание со *
- Изменил манифест DaemonSet, чтобы в логах указывались имена подов, а не их лейблы (выставил значение переменной окружения POD_IDENTIFIER в name). Применил манифест и проверил логи
~~~
kubectl apply -f kubernetes-debug/extra/iptables-tailer-daemonset.yaml
kubectl delete -f ./netperf-operator/deploy/cr.yaml
kubectl apply -f ./netperf-operator/deploy/cr.yaml
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl describe pod --selector=app=netperf-operator
~~~
- В событиях вместо метки отображаются имена подов
- Исправил селекторы в политике calico, применил их и запустил тест снова
~~~
kubectl apply -f kubernetes-debug/extra/netperf-calico-policy.yaml
kubectl delete -f ./netperf-operator/deploy/cr.yaml
kubectl apply -f ./netperf-operator/deploy/cr.yaml
kubectl describe netperfs.app.example.com
~~~
- Тест выполнился

### [Вернуться в корень репо](/../../)