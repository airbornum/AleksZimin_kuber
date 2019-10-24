### [Вернуться в корень репо](/../../)

# Выполнено ДЗ №3

 - [x] Основное ДЗ
 - [x] Задание со *
## В процессе сделано:

### Подготовка к самостоятельной работе:
  - Изменил манифест web-pod.yaml. Добавил readinessProbe и livenessProbe. livenessProbe не начинает работать, т.к. контейнер с web сервером не проходит readinessProbe
  - Применил манифест
    ~~~~
    kubectl apply -f web-pod.yaml --force
    ~~~~
  - Создал манифест для деплоя web-deploy.yaml. Исправил ошибку в readinessProbe. Увеличил число реплик до 3х
  - Удалил старый под:
    ~~~~
    kubectl delete pod/web --grace-period=0 --force
    ~~~~
  - Применил манифест для деплоя и проверил его описание:
    ~~~~
    kubectl apply -f web-deploy.yaml
    kubectl describe deployment web | less
    ~~~~  
#### Ответ на вопрос из ДЗ:
  В нашем случае нет смысла проверять наличие процесса web сервера, т.к. это основной процесс docker контейнера. Если процесс умрет, то контейнер завершит работу
  Смысл в проверке наличия процесса web сервера может быть в нескольких случаях:
  - если процесс web сервера порождает дочерние процессы (в таком случае необходимо проводить проверку на наличие именно дочерних процессов).
  - если основным процессом контейнера будет являться иной процесс (не web сервер).

### Самостоятельная работа. Основное ДЗ:
#### Изучение настроек стратегии деплоя RollingUpdate
  Поробуем разные варианты деплоя с крайними значениями maxSurge и maxUnavailable (оба 0, оба 100%, 0 и 100%). За процессом будем наблюдать с помощью kubespy и kubectl.
  Kubespy скомпилировать из исходников (https://github.com/pulumi/kubespy.git). Для кубера v1.16 необходимо поправить в файле cmd/trace.go версию api для deployment и replicaset с extensions/v1beta1 на apps/v1 (актуально на 2019-09-21)
  
  - **оба 0.** Ошибка. maxUnavailable не может быть 0 при maxSurge=0
  - **оба 100%.** 3 новых пода сразу начинаются создаваться. 3 старых пода сразу начинают удаляться.
  - **maxUnavailable: 0; maxSurge: 100%.** Начинают создаваться 3 новых пода. Старые не удаляются, пока не становится готовым хотя бы один новый под. В итоге всегда поддерживается 100% кол-во готовых подов
  - **maxUnavailable: 100%; maxSurge: 0.** 3 старых пода сразу начинают удаляться. Новые не создаются, пока не остановится хотя бы один под.

  Для наблюдения за деплоем использовал следующие команды:
  ~~~~
  kubespy trace deployment web
  kubespy status apps/v1 deployment web
  kubespy changes apps/v1 deployment web
  kubectl get events --watch
  kubectl rollout status deployment web -w
  ~~~~

#### Сервисы
  ##### ClusterIP
  - Представляет собой набор правил iptables/ipvs на нодах
  - Сервису назначается ip адрес из специального диапазона. В случае iptables данный ip является виртуальным и нигде за пределами ноды не встречается. Когда под внутри кластера пытается подключиться к виртуальному IP-адресу сервиса, то нода, где запущен под меняет адрес получателя в сетевых пакетах на настоящий адрес пода, который выбирается с помощью различных алгоритмов из **endpoints** сервиса.
  - Создадим манифест сервиса ClusterIP web-svc-cip.yaml и применим его
    ~~~~
    kubectl apply -f web-svc-cip.yaml
    kubectl get services
    ~~~~
  - Зайдем на ВМ minikube
    ~~~~
    minikube ssh
    sudo -i
    export CLUSTER_IP=<CLUSTER_IP)>
    curl http://$CLUSTER_IP/index.html
    ping $CLUSTER_IP                        # - ping не работает, т.к. сервис работает с протоколом TCP/UDP
    ~~~~
  - CLUSTER_IP нет в таблице arp и он не назначен сетевым интерфейсам машины
    ~~~~
    arp -an
    ip addr show
    ~~~~
  - Данный ip можно найти в iptables
    ~~~~
    iptables --list -nv -t nat
    ~~~~

  ##### IPVS.
  - Включим IPVS, отредактировав конфиг kube-proxy. Изменить строчку mode: "" на mode: "ipvs"
    ~~~~
    kubectl --namespace kube-system edit configmap/kube-proxy
    ~~~~
  - Удалим Pod с kube-proxy, чтобы применить новую конфигурацию (он входит в DaemonSet и будет запущен автоматически)
    ~~~~
    kubectl --namespace kube-system delete pod --selector='k8s-app=kube-proxy'
    ~~~~

  - Теперь нужно очистить iptables, чтобы удалить мусор (kube-proxy периодически проверяет правила и восстанавливает их при необходимости)
  - Создадим в ВМ с Minikube файл /tmp/iptables.cleanup
    ~~~~
    minikube ssh
    sudo -i
    vi /tmp/iptables.cleanup
  
    ######## Содержимое файла /tmp/iptables.cleanup ########
    *nat
    -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
    COMMIT
    *filter
    COMMIT
    *mangle
    COMMIT
    ########################################################
    ~~~~

  - Применим конфигурацию
    ~~~~
    iptables-restore /tmp/iptables.cleanup
    ~~~~
  - Проверим конфигурацию
    ~~~~
    iptables --list -nv -t nat
    ~~~~

  - В виртуалке minikube нет утилиты ipvsadm. Чтобы посмотреть конфигурацию ipvs можно запустить и войти в контейнер toolbox (сделан на основе Fedora)
    ~~~~
    toolbox
    ~~~~

  - Установим ipvsadm и ipset в контейнере
    ~~~~
    dnf install -y ipvsadm ipset && dnf clean all
    ~~~~
  - Посмотрим конфигурацию ipvs
    ~~~~
    ipvsadm --list -n
    ~~~~
  - Правила в iptables теперь построены по-другому. Вместо цепочки правил для каждого сервиса, теперь используются хэш-таблицы
    ~~~~
    ipset list
    ~~~~

  - Выйдем из контейнера toolbox и сделаем ping кластерного ip.
    ~~~~
    exit
    ping -c1 10.104.183.102
    ~~~~
  - Кластерный ip появился на интерфейсе kube-ipvs0:
    ~~~~
    ip addr show kube-ipvs0
    ~~~~
#### LoadBalancer & Ingress
  ##### MetalLB
  - Скачаем и установим манифест:
    ~~~~
    kubectl apply -f \
      https://raw.githubusercontent.com/google/metallb/v0.8.0/manifests/metallb.yaml
    ~~~~
  - Проверим создание нужных объектов:
    ~~~~
    kubectl --namespace metallb-system get all
    ~~~~
  - Настроим MetalLB с помощью ConfigMap. Создаем манифест с конфигом  metallb-config.yaml в папке kubernetes-networks. В конфиге указываем режим L2 и задаем пул адресов для сервисов LoadBalancer
  - Применяем манифест (контроллер MetalLB подхватит изменения автоматически)
    ~~~~
    kubectl apply -f metallb-config.yaml
    ~~~~
  - Создадим манифест для сервиса типа LoadBalancer на основе предыдущего сервиса ClusterIP. Изменим имя сервиса и его тип на LoadBalancer. Применим манифест:
    ~~~~
    kubectl apply -f web-svc-lb.yaml
    ~~~~
  - Проверим создание сервиса:
    ~~~~
    kubectl get services
    ~~~~
  - Посмотреть логи пода-контроллера MetalLB:
    ~~~~
    kubectl -n metallb-system logs pod/$(kubectl -n metallb-system get pods | grep controller | awk '{print $1}')
    ~~~~
  - Найти назначенный нашему сервису ip адрес:
    ~~~~
    kubectl -n metallb-system logs pod/$(kubectl -n metallb-system get pods | grep controller | awk '{print $1}') | grep ipAllocated
    ~~~~
  - или так:
    ~~~~
    kubectl describe svc web-svc-lb
    ~~~~
  - Запустим поды с web - сервисом (если они не были запущены)
    ~~~~
    kubectl apply -f web-deploy.yaml
    ~~~~
  - Проверим доступ к web сервису с хостовой ОС:
    ~~~~
    curl http://172.17.255.1/index.html 
    ~~~~
  - Доступа нет, т.к. не прописаны маршруты. Найдем ip адрес, назначенный ВМ с Minikube
    ~~~~
    minikube ssh
    ip addr show
    ~~~~
  - Либо так (выполнять в хостовой ОС):
    ~~~~
    minikube ip
    ~~~~
  - В моем случае ip 192.168.99.100. Добавляем временный статический маршрут в хостовой ОС:
    ~~~~
    sudo ip route add 172.17.255.0/24 via 192.168.99.100
    ~~~~
  - Проверим доступ с хостовой ОС:
    ~~~~
    curl http://172.17.255.1/index.html 
    ~~~~
  - С помощью port-forward настроим доступ из интернета:
    ~~~~
    kubectl port-forward --address 0.0.0.0 service/web-svc-lb 8000:80
    ~~~~
  - Проверим доступ с домашнего компа:
    ~~~~
    http://35.228.3.120:8000/index.html
    ~~~~

  ##### Ingress
  - Установим ingress-nginx от проекта Kubernetes:
    ~~~~
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
    ~~~~
  - Далее необходим сервис, который завернет трафик на наш Ingress прокси. В инструкции по установке ingress-nginx рекомендуют создать NodePort сервис, но мы будем использовать MetalLB. Создадим файл nginx-lb.yaml c конфигурацией LoadBalancer-сервиса в папке kubernetes-networks.
  - Применим манифест
    ~~~~
    kubectl apply -f nginx-lb.yaml
    ~~~~
  - Посмотреть ip созданного сервиса:
    ~~~~
    kubectl get service -A | grep nginx
    ~~~~
  - Либо так:
    ~~~~
    kubectl get service ingress-nginx -n ingress-nginx -o jsonpath='{..ingress[0].ip}{"\n"}'
    ~~~~
  - Проверим работу созданного LB
    ~~~~
    ping $(kubectl get service ingress-nginx -n ingress-nginx -o jsonpath='{..ingress[0].ip}')
    ~~~~
  - Попробуем получить html:
    ~~~~
    curl http://$(kubectl get service ingress-nginx -n ingress-nginx -o jsonpath='{..ingress[0].ip}')
    ~~~~
  - Получили ответ от openresty о том, что страница не найдена. Значит ingress работает
  - Установленный Ingress-контроллер для балансировки не требует ClusterIP. Список узлов для балансировки заполняется из ресурса Endpoints нужного сервиса. Поэтому мы можем использовать headless-сервис для нашего веб-приложения.
  - Создадим манифест web-svc-headless.yaml для headless-сервиса на основе предыдущего сервиса ClusterIP. Изменим имя сервиса и добавим параметр clusterIP: None. Применим манифест:
    ~~~~
    kubectl apply -f web-svc-headless.yaml
    ~~~~
  - Проверим, что сервис создался без кластерного ip и является headless(запрос к dns должен возвращать ip адреса всех под):
    ~~~~
    kubectl get service -A
    ~~~~
  - Если манифест dns-svc-lb.yaml из первого задания со * уже применен, то можно запросить ip адреса так:
    ~~~~
    nslookup web-svc.default.svc.cluster.local $(kubectl get service dns-svc-lb-tcp -n kube-system -o jsonpath='{..ingress[0].ip}{"\n"}')
    ~~~~
  - Теперь создадим манифест web-ingress.yaml с описанием Ingress. Применим манифест
    ~~~~
    kubectl apply -f web-ingress.yaml
    ~~~~
  - Проверим, что корректно заполнены Address и Backends
    ~~~~
    kubectl describe ingress/web | less
    ~~~~
  - Проверим работу Ingress:
    ~~~~
    curl http://$(kubectl get service ingress-nginx -n ingress-nginx -o jsonpath='{..ingress[0].ip}')/web/index.html -k
    wget http://$(kubectl get service ingress-nginx -n ingress-nginx -o jsonpath='{..ingress[0].ip}')/web/index.html --no-check-certificate
    ~~~~
  - Если второе задание со * уже выполнялось, то проверяем так, предварительно добавив запись zav.k8s.local в hosts:
    ~~~~
    curl http://zav.k8s.local/web/index.html
    ~~~~

### Самостоятельная работа. Задание со *.
#### MetalLB. Открыть доступ к DNS снаружи кластера по TCP и UDP протоколу.
k8s на данный момент не поддерживает несколько протоколов в одном LoadBalancer. Решить данную проблему можно с помощью MetalLB, используя IP Address Sharing.
  - Создадим манифест dns-svc-lb.yaml с описанием двух новых сервисов типа LoadBalancer. Один из них будет для TCP, другой для UDP. В описании каждого из них необходимо указать одинаковые значения для ключа metallb.universe.tf/allow-shared-ip в аннотации, одинаковые ip для loadBalancerIP и одинаковые selector
  - Применим манифест:
    ~~~~
    kubectl apply -f dns-svc-lb.yaml
    ~~~~
  - Добавляем временный статический маршрут в хостовой ОС:
    ~~~~
    sudo ip route add 172.17.255.0/24 via 192.168.99.100
    ~~~~
  - Делаем dns запрос с хоста
    ~~~~
    nslookup web-svc.default.svc.cluster.local 172.17.255.254
    ~~~~
  - Смотрим, занят ли 53 порт.
    ~~~~
    sudo netstat -tupln | grep 53
    ~~~~
  - Если занят, то останавливаем прогу, которая это делает. В моем случае это был сервис systemd-resolved.service
    ~~~~
    sudo systemctl stop systemd-resolved.service
    ~~~~
  - Делаем порт форвард:
    ~~~~
    sudo kubectl port-forward --address 0.0.0.0 service/dns-svc-lb-tcp -n kube-system 53:53 
    ~~~~
  - Делаем dns запрос с нашего компа по TCP (отправить запрос по TCP можно, используя ключ "-set vc")
    ~~~~
    nslookup "-set vc" web-svc-lb.default.svc.cluster.local 35.228.3.120
    ~~~~

#### Установить Dashboard UI и открыть к нему доступ снаружи кластера.
  - Установить Dashboard UI:
    ~~~~
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml
    ~~~~
  - Создать service account с админскими правами для доступа к dashboard (использовать dashboard-adminuser+ClusterRoleBinding.yaml из каталога kubernetes-intro):
    ~~~~
    kubectl apply -f /kubernetes-intro/dashboard-adminuser+ClusterRoleBinding.yaml
    ~~~~
  - Для практики решил пробросить SSL непосредственно до пода с Dashboard UI. Для этого необходимо включить поддержку ssl-passthrough у ingress-nginx. Редактируем deployment nginx-ingress-controller Добавим аргумент --enable-ssl-passthrough в поле args
    ~~~~
    kubectl edit deploy -n ingress-nginx nginx-ingress-controller
    ~~~~
  - Создадим манифест dashboard-ingress.yaml на основе web-ingress.yaml. Изменим правило для rewrite-target, а так же добавим новые аннотации для проброса SSL. Применим созданный манифест:
    ~~~~
    kubectl apply -f dashboard-svc.yaml
    ~~~~
  - Получить токен для доступа к dashboard
    ~~~~
    kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')     
    ~~~~        
  - Включаем порт форвард:
    ~~~~
    sudo kubectl port-forward --address 0.0.0.0 service/ingress-nginx -n ingress-nginx 80:80 443:443
    ~~~~
  - Заходим на dashboard
    ~~~~
    https://35.228.3.120/dashboard/#/login
    ~~~~

#### Canary release.
Обязательное условие для работы canary - оригинальный deployment (ingress) и canary должны быть в разных namespace. Так же в обоих ingress должен быть указан host
  - Зайти в папку kubernetes-networks. Создать внутри папку canary и перейти в нее
  - Создать манифесты web-canary-deploy.yaml на основе web-deploy.yaml и web-canary-ingress.yaml на основе web-ingress.yaml
  - Создать манифест для нового namsespace web-canary
  - Добавить namespace web-canary в манифестах web-canary-deploy.yaml и web-canary-ingress.yaml
  - Заменить команду на  command: ['sh', '-c', 'echo "NEW DEPLOYMENT">/app/index.html'] в манифесте web-deply-canary.yaml
  - Добавить host для ../web-ingress.yaml и web-canary-ingress.yaml  - это обязательно действие!
  - Добавить аннотации для ингресса  web-canary-ingress.yaml:
    ~~~
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "Canary"
    nginx.ingress.kubernetes.io/canary-weight:
    ~~~
  - Создать ns, deploy, svc, ing:
    ~~~
    kubectl apply -f web-canary-ns.yaml
    kubectl apply -f .
    ~~~
  - Добавляем в файл hosts наше имя хоста
    ~~~
    sudo sh -c "echo $(kubectl get service ingress-nginx -n ingress-nginx -o jsonpath='{..ingress[0].ip}') zav.k8s.local>> /etc/hosts"
    ~~~
  - Проверяем canary
    ~~~
    curl http://zav.k8s.local/web/index.html -k
    curl http://zav.k8s.local/web/index.html -k -H "Canary:always"
    ~~~

### [Вернуться в корень репо](/../../)