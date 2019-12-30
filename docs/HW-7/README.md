### [Вернуться в корень репо](/../../)

# Выполнено ДЗ №7

 - [x] Основное ДЗ
 - [] Задание со *
## В процессе сделано:



### Подготовил инфраструктуру
- Создал кластер в minikube и подготовил окружение (команды выполнял из корня репо, )
~~~
minikube start --cpus 8 --memory 10240 --kubernetes-version v1.15.7 --vm-driver kvm2


mkdir -p kubernetes-operators/deploy && cd kubernetes-operators
~~~
### Практика по созданию cr/crd (содержимое манифестов брал из сниппетов)
- Попытка создать CR
~~~
touch deploy/cr.yml
kubectl apply -f deploy/cr.yml
~~~
- Получил ошибку, т.к. в API нет описания объекта типа MySQL. Создал CRD
~~~
touch deploy/crd.yml
kubectl apply -f deploy/crd.yml
~~~
- Создал CR
~~~
kubectl apply -f deploy/cr.yml
~~~
- Проверил созданные объекты и удалил CR
~~~
kubectl get crd
kubectl get mysqls.otus.homework
kubectl describe mysqls.otus.homework mysql-instance
kubectl delete mysqls.otus.homework mysql-instance
~~~
- Добавил валидацию в spec CRD и попробовал снова создать CR
~~~
kubectl apply -f deploy/crd.yml
kubectl apply -f deploy/cr.yml
~~~
- Убрал из cr.yml ненужную строку. Применил манифест
~~~
kubectl apply -f deploy/cr.yml
~~~
- Добавил описание обязательных полей в манифест crd.yml

### Деплой оператора
- Скачал минифесты (выполнять в папке kubernetes-operators/deploy) и применил их
~~~
cd deploy
curl -LO https://gist.githubusercontent.com/Evgenikk/581fa5bba6d924a3438be1e3d31aa468/raw/99429270c474cc434748e1058919e27df01d9a48/service-account.yml
kubectl apply -f service-account.yml

curl -LO https://gist.githubusercontent.com/Evgenikk/581fa5bba6d924a3438be1e3d31aa468/raw/99429270c474cc434748e1058919e27df01d9a48/role.yml
kubectl apply -f role.yml

curl  https://gist.githubusercontent.com/Evgenikk/581fa5bba6d924a3438be1e3d31aa468/raw/99429270c474cc434748e1058919e27df01d9a48/ClusterRoleBinding.yml -o role-binding.yaml
kubectl apply -f role-binding.yml

curl -LO https://gist.githubusercontent.com/Evgenikk/581fa5bba6d924a3438be1e3d31aa468/raw/619023d01e49ca3702357d4fded4d054cd523a9a/deploy-operator.yml
kubectl apply -f deploy-operator.yml
~~~
- Создал CR и проверил создание pvc
~~~
kubectl apply -f deploy/cr.yml
kubectl get pvc
~~~
- Проверил БД
~~~
export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
kubectl exec -it $MYSQLPOD -- mysql -u root -potuspassword -e "CREATE TABLE test ( id smallint unsigned not null auto_increment, name varchar(20) not null, constraint pk_example primary key (id) );" otus-database
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name )
VALUES ( null, 'some data' );" otus-database
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES ( null, 'some data-2' );" otus-database
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
~~~
- Удалил инстанс с mysql и проверил pv и job
~~~
kubectl delete mysqls.otus.homework mysql-instance
kubectl get pv
kubectl get jobs.batch
~~~
- Создал заново инстанс с mysql и проверил данные. Данные восстановились (для быстроты удалил контейнер с restore-job, т.к. он начал работу сразу после деплоя, когда файла с бекапом еще не было)
~~~
kubectl apply -f deploy/cr.yml
export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
~~~
- Вывод команды kubectl get jobs:
~~~
backup-mysql-instance-job    1/1           2s         3m27s
restore-mysql-instance-job   1/1           5m48s      8m21s
~~~
- Содержимое БД:
~~~
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data-2 |
+----+-------------+
~~~

### [Вернуться в корень репо](/../../)