### [Вернуться в корень репо](/../../)

# Выполнено ДЗ №5

 - [x] Основное ДЗ
 - [ ] Задание со *
## В процессе сделано:

### Подготовил инфраструктуру
- Создал манифест ./kubernetes-storage/cluster/cluster.yaml с описанием настроек кластера (включена альфа фича по работе со снапшотами).
- Создал кластер с необходимыми настройками(k8s версии 1.15)
~~~
kind create cluster --config cluster/cluster.yaml --image=kindest/node:v1.15.6 
kubectl cluster-info --context kind-kind
~~~
- Склонировал репо с драйвером csi-driver-host-path и запустил скрипт деплоя драйвера
~~~
cd && git clone https://github.com/kubernetes-csi/csi-driver-host-path.git
cd csi-driver-host-path/deploy/kubernetes-1.15/
chmod +x deploy-hostpath.sh && ./deploy-hostpath.sh
~~~
- создал StorageClass для CSI Host Path Driver
~~~
touch hw/01-storage-class.yaml
kubectl apply -f hw/01-storage-class.yaml
~~~
- создал VolumeSnapshotClass (deploy-hostpath.sh его не создал)
~~~
touch hw/02-snapshot-class.yaml 
kubectl apply -f hw/02-snapshot-class.yaml 
~~~

### Протестировал функционал снапшотов
- Создал pvc
~~~
touch hw/03-pvc.yaml
kubectl apply -f hw/03-pvc.yaml
~~~
- Создал под, который монтирует том в /data и использует созданный выше pvc (том будет создан драйвером)
~~~
touch hw/04-storage-pod.yaml
kubectl apply -f hw/04-storage-pod.yaml
~~~
- Записал данные в файл /data/IMPORTANT_DATA.txt
~~~
kubectl exec -it storage-pod -- bash -c "echo My important data>/data/IMPORTANT_DATA.txt"
kubectl exec -it storage-pod cat /data/IMPORTANT_DATA.txt
~~~
- Создал снапшот и удостоверился в его наличии
~~~
touch hw/05-snapshot.yaml
kubectl apply -f hw/05-snapshot.yaml
kubectl get volumesnapshots.snapshot.storage.k8s.io
kubectl describe volumesnapshots.snapshot.storage.k8s.io
~~~
- Удалил данные в поде, удалил под и pvc
~~~
kubectl exec -it storage-pod rm /data/IMPORTANT_DATA.txt
kubectl exec -it storage-pod cat /data/IMPORTANT_DATA.txt
kubectl delete pod storage-pod
kubectl delete pvc storage-pvc
~~~
- Восстановил данные
~~~
touch hw/06-restore.yaml
kubectl apply -f hw/06-restore.yaml
kubectl apply -f hw/04-storage-pod.yaml
kubectl exec -it storage-pod cat /data/IMPORTANT_DATA.txt
~~~


### [Вернуться в корень репо](/../../)