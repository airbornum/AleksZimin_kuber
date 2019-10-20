### [Вернуться в корень репо](/../../)

# Выполнено ДЗ №1

 - [x] Основное ДЗ

## В процессе сделано:
 - Установил и настроил docker на локальной ВМ (Vbox, в котором крутится локальная ВМ не поддерживает вложенную виртуализацию, поэтому при запуске minikube использовал флаг --vm-driver=none (все компоненты k8s запускаются в docker контейнерах))
 - Установил последнюю версию kubectl
 - Настроил автозаполнение для bash
 - Установил Minikube и запустил кластер:
   ~~~~
   minikube start --vm-driver=none
   ~~~~
 - Установил Dashboard UI:
   ~~~~
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
   ~~~~
 
 - Создал service account с админскими правами для доступа к dashboard.
   ~~~~
   touch dashboard-adminuser+ClusterRoleBinding.yaml (файл находится в kubernetes-intro/dashboard-adminuser+ClusterRoleBinding.yaml)
   kubectl apply -f dashboard-adminuser+ClusterRoleBinding.yaml
   kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
   ~~~~
   
 - Получил токен для доступа к dashboard:
   ~~~~
   kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
   ~~~~
   
 - Для открытия доступа к dashboard UI использовал kubectl proxy
   ~~~~
   kubectl proxy
   ~~~~
 - dashboard будет доступен по адресу:
   ~~~~
   http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
   ~~~~
 - Установил консольную утилиту k9s (скачивал бинарник из github. Установленный из snap k9s не заработал). 
 - Проверил отказоустойчивость кластера. Причины, по которым восстанавливаются поды опишу ниже.
 - Создал Dockerfile с веб-сервером nginx. Конфиг для nginx использовал свой. Внутри контейнера nginx запускается от пользователя с uid 1001 
 - Собрал образ и запушил его в dockerhub
 - Написал манифест web-pod.yaml. Для генерации стартовой страницы используется init контейнер. Страница в под с nginx передается через volume типа emptyDir
 - Установил на ноде socat
 - Для доступа к поду использовал kubectl port-forward:
   ~~~~
   kubectl port-forward --address 0.0.0.0 pod/web-nginx 8000:8000
   ~~~~
 
 
## В k8s существует несколько способов контроля за жизненным циклом подов. Они зависят от типа пода.
### 1. Static pods - статические поды, описания которых лежат в каталоге, заданном в параметре --pod-manifest-path в строке запуска kubelet. Данными подами управляет непосредственно kubelet. 
 - kubelet периодически проверяет содержимое каталога с манифестами и создает/удаляет поды при необходимости (если файлы описаний были добавлены или удалены). Так же kubelet самостоятельно следит за состоянием запущенных подов. И перезапускает их при необходимости. В api сервере kubelet создает "двойников" данных подов (это позволяет api серверу видеть статические поды, но управлять оригиналами он не может).
 - Путь к файлам манифестов можно посмотреть так(запускать внутри minikube):
  ~~~~
  sudo cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
  ~~~~
#### В моем случае статические поды такие (описание лежит в /etc/kubernetes/manifests):
  ~~~~
  kube-addon-manager  etcd  kube-apiserver  kube-controller-manager  kube-scheduler
  ~~~~

### 2. За жизненным циклом стандартных подов следят контроллеры. 
 - Узнать чем контролируется под можно, посмотрев описание пода:
  ~~~~
  kubectl describe pods -n kube-system POD_NAME (Строка Controlled By)
  ~~~~
 - То же верно и для replicaset:
  ~~~~
  kubectl describe replicaset -n kube-system REPLICASET_NAME
  ~~~~
 
#### core-dns управляется ReplicaSet, а replicaset управляется deployment контроллером:
  ~~~~
  kubectl describe pods -n kube-system $(kubectl get pods -n kube-system | grep -m 1 coredns | awk '{print $1}') | grep 'Controlled By'
  kubectl describe replicaset -n kube-system  $(kubectl get replicasets -n kube-system | grep coredns | awk '{print $1}') | grep 'Controlled By'
  kubectl describe deployment -n kube-system  $(kubectl get deployment -n kube-system | grep coredns | awk '{print $1}')
  ~~~~
#### Для kube-proxy это DaemonSet:
  ~~~~
  kubectl describe pods -n kube-system $(kubectl get pods -n kube-system | grep -m 1 kube-proxy | awk '{print $1}') | grep 'Controlled By'
  kubectl describe daemonset -n kube-system  $(kubectl get replicasets -n kube-system | grep kube-proxy | awk '{print $1}') 
  ~~~~
	
#### В случае с этими подами (core-dns и kube-proxy) перезапуск осуществляется всегда, если с подом что-то произошло и он стал недоступен (вне зависимости от кода завершения контейнера). Такое поведение задается в спецификации пода в политике перезапуска (restartPolicy: Always)
  ~~~~
  kubectl get pod -n kube-system $(kubectl get pods -n kube-system | grep -m 1 coredns | awk '{print $1}') -o yaml | grep restartPolicy
  kubectl get pod -n kube-system $(kubectl get pods -n kube-system | grep -m 1 kube-proxy | awk '{print $1}') -o yaml | grep restartPolicy
  ~~~~

### 3. Так же существуют поды-аддоны, жизненный цикл которых управляется менеджером аддонов (addon-manager) 
#### storage-provisioner -  аддон. Выяснил я это так:

 - 1 - в описании пода нет строки Controlled By (команда ниже ничего не выводит, но это не является обязательным условием(?)):
	~~~~
	kubectl describe pods -n kube-system $(kubectl get pods -n kube-system | grep -m 1 storage-provisioner | awk '{print $1}') | grep 'Controlled By'
	~~~~
 - 2 - в метках пода есть такая запись:  addonmanager.kubernetes.io/mode=Reconcile
	~~~~
	kubectl describe pods -n kube-system $(kubectl get pods -n kube-system | grep -m 1 storage-provisioner | awk '{print $1}') | grep -C 3 'Labels:'
	~~~~

 - 3 - манифест пода лежит в /etc/kubernetes/addons (путь по умолчанию, если переменная $ADDON_PATH не задана(?))

 - В случае с подом storage-provisioner жизненный цикл задается меткой addonmanager.kubernetes.io/mode=Reconcile. Поды с этой меткой:
	~~~~
	Addon will be re-created if it is deleted.
	Addon will be reconfigured to the state given by the supplied fields in the template file periodically.
	Addon will be deleted when its manifest file is deleted from the $ADDON_PATH.
	~~~~
### [Вернуться в корень репо](/../../)
