### [Вернуться в корень репо](/../../)

# Выполнено ДЗ №4

 - [x] Основное ДЗ
 - [x] Задание со *
## В процессе сделано:

### Подготовка к самостоятельной работе:
#### Установка kind
- Создадим каталог для бинарников go и добавим в Path
~~~
mkdir -p ~/go/bin
echo export PATH="$PATH:~/go/bin" >> ~/.bashrc
exec bash
~~~
- Скомпилируем kind из исходников (компиляция будет происходить в docker контейнере).
~~~
mkdir -p ~/kind_repo && cd ~/kind_repo
git clone https://github.com/kubernetes-sigs/kind.git . &&\
  make build &&\
  chmod +x bin/kind &&\
  cp bin/kind ~/go/bin/
~~~

#### Работа с кластером
- Создадим кластер в kind и запишем путь к его конфигурации в переменную KUBECONFIG (не забыть удалить строку из .bashrc после окончания работы с кластером kind)
~~~
kind create cluster
echo export KUBECONFIG="$(kind get kubeconfig-path --name="kind")" >> ~/.bashrc
exec bash
~~~
- Развернем StatefulSet с MiniO (для кластера версии 1.16 необходимо скачать манифест и заменить apiVersion c apps/v1beta1 на apps/v1)
~~~
kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Kuberenetes-volumes/minio-statefulset.yaml
~~~
- Создадим headless сервис
~~~
kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Kuberenetes-volumes/minio-headless-service.yaml
~~~

### Самостоятельная работа. Задание со *.
- Создадим секрет, в котором будем хранить MINIO_SECRET_KEY с помощью команды kubectl create secret
~~~
kubectl create secret generic minio-secret --from-literal=MINIO_ACCESS_KEY='minio_key' --from-literal=MINIO_SECRET_KEY='minio_passw0rd'
~~~
- Так же секрет можно создать из манифеста.
- Создадим манифест minio-secret.yaml. Содержимое файла:
~~~
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
stringData:
  MINIO_ACCESS_KEY: minio_key
  MINIO_SECRET_KEY: minio_passw0rd
~~~
- Применим манифест
~~~
kubectl apply -f minio-secret.yaml
~~~
- Создадим новый манифест с описанием StatefulSet minio. Для этого можно скопировать предыдущий манифест
~~~
wget -O minio-statefulset-with-secret.yaml https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Kuberenetes-volumes/minio-statefulset.yaml
~~~
- Заменим в нем описание переменной MINIO_ACCESS_KEY и MINIO_SECRET_KEY 
~~~

        env:
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: MINIO_ACCESS_KEY
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-secret
              key: MINIO_SECRET_KEY
~~~
- Применить новый манифест
~~~
kubectl apply -f minio-statefulset-with-secret.yaml
~~~
- Проверим, что секрет изменился. Для этого зайдем в под и посмотрим, что хранится в переменной окружения $MINIO_SECRET_KEY
~~~
kubectl exec -it minio-0 /bin/sh
echo $MINIO_ACCESS_KEY
echo $MINIO_SECRET_KEY
~~~

### [Вернуться в корень репо](/../../)