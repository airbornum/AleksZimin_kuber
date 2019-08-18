### [Вернуться в корень репо](/../../)

# Выполнено ДЗ №2

 - [x] Основное ДЗ

## В процессе сделано:
### task01 
  - Создал манифесты:
    01-sa-admin-bob.yaml - создание sa bob и ClusterRoleBinding пользователя bob к кластерной роли admin
    02-sa-dave.yaml - создание sa dave
  - Применил манифесты:
    ~~~~
    kubectl apply -f 01-sa-admin-bob.yaml
    kubectl apply -f 02-sa-dave.yaml
    ~~~~
  - Проверил, что clusterrolesbinding создался верно:
    ~~~~
    kubectl get clusterrolebindings bob-to-admin -o yaml
    ~~~~
  - Проверил права bob:
    ~~~~
    kubectl auth can-i create pod --as=system:serviceaccount:default:bob -n default
    ~~~~
    
### task02    
 - Создал манифесты:
   01-ns-prometheus.yaml - создание ns  prometheus 
   02-sa-carol-ns-prometheus.yaml - создание sa carol в ns prometheus
   03-clusterrole-pod-reader.yaml - создание кластерной роли pod-reader
   04-clusterrolesbinding.yaml - создание clusterrolesbinding для всех sa в ns prometheus к кластерной роли pod-reader (используем в качестве субъекта группу system:serviceaccounts:prometheus)
  - Применил манифесты:
    ~~~~
    kubectl apply -f 01-ns-prometheus.yaml
    kubectl apply -f 02-sa-carol-ns-prometheus.yaml
    kubectl apply -f 03-clusterrole-pod-reader.yaml
    kubectl apply -f 04-clusterrolesbinding.yaml 
    ~~~~
  - Проверил, что кластерная роль создалась верно:
    ~~~~
    kubectl describe clusterrole pod-reader
    kubectl get clusterrole pod-reader -o yaml
    ~~~~
  - Проверил права sa carol из ns prometheus:
    ~~~~
    kubectl auth can-i create pod --as=system:serviceaccount:prometheus:carol -n default
    kubectl auth can-i list pod --as=system:serviceaccount:prometheus:carol -n prometheus
    kubectl auth can-i list pod --as=system:serviceaccount:prometheus:carol -n default
    ~~~~
  - Проверил права sa default из ns prometheus:
    ~~~~
    kubectl auth can-i create pod --as=system:serviceaccount:prometheus:default -n default
    kubectl auth can-i list pod --as=system:serviceaccount:prometheus:default -n prometheus
    kubectl auth can-i list pod --as=system:serviceaccount:prometheus:default -n default
    ~~~~

### task03    
 - Создал манифесты:
   01-ns-dev.yaml - создание ns dev 
   02-sa-jane-ns-dev-admin.yaml - создание sa jane в ns dev и rolebinding данного пользователя к кластерной роли admin для ns dev
   03-sa-ken-ns-dev-view.yaml- создание sa ken в ns dev и rolebinding данного пользователя к кластерной роли view для ns dev
  - Применил манифесты:
    ~~~~
    kubectl apply -f 01-ns-dev.yaml
    kubectl apply -f 02-sa-jane-ns-dev-admin.yaml
    kubectl apply -f 03-sa-ken-ns-dev-view.yaml
    ~~~~
  - Проверил права sa jane из ns dev:
    ~~~~
    kubectl auth can-i create pod --as=system:serviceaccount:dev:jane -n default
    kubectl auth can-i create pod --as=system:serviceaccount:dev:jane -n dev
    ~~~~
  - Проверил права sa ken из ns dev:
    ~~~~
    kubectl auth can-i list pod --as=system:serviceaccount:dev:ken -n default
    kubectl auth can-i list pod --as=system:serviceaccount:dev:ken -n dev
    kubectl auth can-i get pod --as=system:serviceaccount:dev:ken -n dev
    ~~~~
  
 
### [Вернуться в корень репо](/../../)
