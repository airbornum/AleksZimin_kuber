### [Вернуться в корень репо](/../../)

# Выполнено ДЗ №11

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
~~~

### Установил hashicorp vault
- Склонировал helm chart консула и установил его (для установки использовал helm2)
~~~
cd ~
git clone https://github.com/hashicorp/consul-helm.git
helm install --name=consul consul-helm
~~~
- Склонировал helm chart vault, отредактировал values.yaml и установил релиз
~~~
git clone https://github.com/hashicorp/vault-helm.git
cp vault-helm/values.yaml ~/REPO/AleksZimin_platform/kubernetes-vault/values-vault.yaml
helm install --name=vault vault-helm -f ~/REPO/AleksZimin_platform/kubernetes-vault/values-vault.yaml
~~~
- Проверим установку релиза
~~~
helm status vault
kubectl logs vault-0
~~~
- Вывод команды helm status vault
~~~
LAST DEPLOYED: Tue Dec 31 13:24:22 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME                              AGE
vault-agent-injector-clusterrole  4m31s

==> v1/ClusterRoleBinding
NAME                          AGE
vault-agent-injector-binding  4m31s

==> v1/ConfigMap
NAME          AGE
vault-config  4m31s

==> v1/Deployment
NAME                  AGE
vault-agent-injector  4m31s

==> v1/Pod(related)
NAME                                   AGE
vault-0                                4m31s
vault-1                                4m30s
vault-2                                4m30s
vault-agent-injector-6b5b89f68f-gpl7t  4m31s

==> v1/Service
NAME                      AGE
vault                     4m31s
vault-agent-injector-svc  4m31s
vault-ui                  4m31s

==> v1/ServiceAccount
NAME                  AGE
vault                 4m31s
vault-agent-injector  4m31s

==> v1/StatefulSet
NAME   AGE
vault  4m31s

==> v1beta1/ClusterRoleBinding
NAME                  AGE
vault-server-binding  4m31s

==> v1beta1/MutatingWebhookConfiguration
NAME                      AGE
vault-agent-injector-cfg  4m31s

==> v1beta1/PodDisruptionBudget
NAME   AGE
vault  4m31s


NOTES:

Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get vault
~~~

- Инициализировал vault
~~~
kubectl exec -it vault-0 -- vault operator init --key-shares=1 --key-threshold=1
~~~
- Вывод команды:
~~~
Unseal Key 1: PlwLr7ZzPh08pBOgW8hk3AqKfcdmj0D299zXt8ys5/g=

Initial Root Token: s.WsK0PCYydgqvKko0BiHKbhVV
~~~
- Проверил состояние vault, посмотрел переменные окружения в подах
~~~
for i in $(seq 0 2); do kubectl exec -it vault-$i -- vault operator unseal 'PlwLr7ZzPh08pBOgW8hk3AqKfcdmj0D299zXt8ys5/g='; done
kubectl exec -it vault-0 env | grep VAULT
kubectl exec -it vault-1 env | grep VAULT
kubectl exec -it vault-2 env | grep VAULT
~~~
- "Распечатал" все поды с vault
~~~
for i in $(seq 0 2); do kubectl exec -it vault-$i -- vault operator unseal 'PlwLr7ZzPh08pBOgW8hk3AqKfcdmj0D299zXt8ys5/g='; done
~~~
- Статус каждого пода, после распечатывания:
~~~
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.3.1
Cluster Name    vault-cluster-1879753f
Cluster ID      9e0191b3-0fbd-0ef9-4efb-90cb850d051e
HA Enabled      true
HA Cluster      https://10.52.2.7:8201
HA Mode         active
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.3.1
Cluster Name           vault-cluster-1879753f
Cluster ID             9e0191b3-0fbd-0ef9-4efb-90cb850d051e
HA Enabled             true
HA Cluster             https://10.52.2.7:8201
HA Mode                standby
Active Node Address    http://10.52.2.7:8200
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.3.1
Cluster Name           vault-cluster-1879753f
Cluster ID             9e0191b3-0fbd-0ef9-4efb-90cb850d051e
HA Enabled             true
HA Cluster             https://10.52.2.7:8201
HA Mode                standby
Active Node Address    http://10.52.2.7:8200
~~~

### Работа с hashicorp vault
- Посмотрел список доступных авторизаций, получил ошибку.
~~~
kubectl exec -it vault-0 -- vault auth list
~~~
- Подключился к vault, используя Initial Root Token, и снова запросил список авторизаций
~~~
kubectl exec -it vault-0 -- vault login 's.WsK0PCYydgqvKko0BiHKbhVV'
kubectl exec -it vault-0 -- vault auth list
~~~
- Вывод команд:
~~~
$ kubectl exec -it vault-0 -- vault login 's.WsK0PCYydgqvKko0BiHKbhVV'
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.WsK0PCYydgqvKko0BiHKbhVV
token_accessor       O05tFc91HwKqRSgjwI3TPymb
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]

$ kubectl exec -it vault-0 -- vault auth list
Path      Type     Accessor               Description
----      ----     --------               -----------
token/    token    auth_token_6586256b    token based credential
~~~
- Завел секреты
~~~
kubectl exec -it vault-0 -- vault secrets enable --path=otus kv
kubectl exec -it vault-0 -- vault secrets list --detailed
kubectl exec -it vault-0 -- vault kv put otus/otus-ro/config username='otus' password='asajkjkahs'
kubectl exec -it vault-0 -- vault kv put otus/otus-rw/config username='otus' password='asajkjkahs'
~~~
- Просмотрел секреты
~~~
$ kubectl exec -it vault-0 -- vault read otus/otus-ro/config
Key                 Value
---                 -----
refresh_interval    768h
password            asajkjkahs
username            otus

$ kubectl exec -it vault-0 -- vault kv get otus/otus-rw/config
====== Data ======
Key         Value
---         -----
password    asajkjkahs
username    otus
~~~
- Включил авторизацию через k8s
~~~
kubectl exec -it vault-0 -- vault auth enable kubernetes
kubectl exec -it vault-0 -- vault auth list
~~~
- Обновленный список авторизаций:
~~~
Path           Type          Accessor                    Description
----           ----          --------                    -----------
kubernetes/    kubernetes    auth_kubernetes_580c3810    n/a
token/         token         auth_token_6586256b         token based credentials
~~~
- Создал манифест для ClusterRoleBinding
~~~
tee kubernetes-vault/vault-auth-service-account.yml <<EOF
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault-auth
    namespace: default
EOF
~~~
- Создал сервис акк vault-auth и применил созданный выше манифест
~~~
kubectl create serviceaccount vault-auth
kubectl apply -f kubernetes-vault/vault-auth-service-account.yml
~~~
- Настрил переменные среды, которые далее будем использовать при создании конфига k8s-авторизации
~~~
export VAULT_SA_NAME=$(kubectl get sa vault-auth -o jsonpath="{.secrets[*]['name']}")
export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)
export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
export K8S_HOST=$(more ~/.kube/config | grep server | awk '/http/ {print $NF}')
~~~
- Другой способ задать переменную K8S_HOST. Регулярка в этой команде убирает коды цветового офрмления:
~~~
export K8S_HOST=$(kubectl cluster-info | grep 'Kubernetes master' | awk '/https/ {print $NF}' | sed 's/\x1b\[[0-9;]*m//g')
~~~
- Проверил значения переменных
~~~
echo $VAULT_SA_NAME
echo $SA_JWT_TOKEN
echo $SA_CA_CRT
echo $K8S_HOST
~~~
- Создал конфиг k8s-авторизации в vault
~~~
kubectl exec -it vault-0 -- vault write auth/kubernetes/config \
  token_reviewer_jwt="$SA_JWT_TOKEN" \
  kubernetes_host="$K8S_HOST" \
  kubernetes_ca_cert="$SA_CA_CRT"
~~~
- Создал файл политики
~~~
tee kubernetes-vault/otus-policy.hcl <<EOF
path "otus/otus-ro/*" {
capabilities = ["read", "list"]
}
path "otus/otus-rw/*" {
capabilities = ["read", "create", "list"]
}
EOF
~~~
- Создадим политику и роль в vault
~~~
kubectl cp kubernetes-vault/otus-policy.hcl vault-0:/tmp/
kubectl exec -it vault-0 -- vault policy write otus-policy /tmp/otus-policy.hcl

kubectl exec -it vault-0 -- vault write auth/kubernetes/role/otus \
  bound_service_account_names=vault-auth \
  bound_service_account_namespaces=default policies=otus-policy ttl=24h
~~~
### Проверка работы авторизации и работа с секретами
- Создал под с сервис аккаунтом и установил туда curl и jq
~~~
kubectl run --generator=run-pod/v1 tmp --rm -i --tty --serviceaccount=vault-auth --image alpine:3.7
apk add curl jq
~~~
- Залогинился и получил клиентский токен
~~~
VAULT_ADDR=http://vault:8200
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq

TOKEN=$(curl -k -s --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq '.auth.client_token' | awk -F\" '{print $2}')

/ # echo $TOKEN
s.iMmX32tkBGJNtugf1S0iXSr2
~~~
- Проверим чтение секретов
~~~
curl --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-ro/config
curl --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config    
~~~
- Проверим запись секретов
~~~
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-ro/config
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config1
~~~
- Удалось только создать секрет config1 в /otus/otus-rw/, т.к. в политиках для пути otus/otus-rw/ не задана возможность изменять секреты. Добавил возможность изменять секреты, применил политику и проверил запись
~~~
tee kubernetes-vault/otus-policy_fixed.hcl <<EOF
path "otus/otus-ro/*" {
capabilities = ["read", "list"]
}
path "otus/otus-rw/*" {
capabilities = ["read", "create", "list", "update"]
}
EOF

kubectl cp kubernetes-vault/otus-policy_fixed.hcl vault-0:/tmp/
kubectl exec -it vault-0 -- vault policy write otus-policy /tmp/otus-policy_fixed.hcl

curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config
curl --header "X-Vault-Token:$TOKEN" $VAULT_ADDR/v1/otus/otus-rw/config 
{"request_id":"6e534c89-5ad0-9247-506e-0b77e069f922","lease_id":"","renewable":false,"lease_duration":2764800,"data":{"bar":"baz"},"wrap_info":null,"warnings":null,"auth":null}
~~~

### Авторизация nginx через k8s в vault
- Склонировал репо с примерами
~~~
cd ~
git clone https://github.com/hashicorp/vault-guides.git
mkdir -p ~/REPO/AleksZimin_platform/kubernetes-vault/vault-agent-k8s-demo/configs-k8s
cp -R vault-guides/identity/vault-agent-k8s-demo/configs-k8s/* ~/REPO/AleksZimin_platform/kubernetes-vault/vault-agent-k8s-demo/configs-k8s
cp vault-guides/identity/vault-agent-k8s-demo/example-k8s-spec.yml ~/REPO/AleksZimin_platform/kubernetes-vault/vault-agent-k8s-demo/
yes | rm -R vault-guides/
~~~
- Скорректировал конфиги с учетом ранее созданных ролей и секретов  
Изменил роль vault-agent-config.hcl на otus  
Изменил путь к секрету в consul-template-config.hcl на otus/otus-ro/config  
Изменил хост в example-k8s-spec.yml на http://vault:8200  
- Применил измененные манифесты, проверил содержимое созданного configmap и задеплоил под с vault-agent
~~~
kubectl create configmap example-vault-agent-config --from-file=kubernetes-vault/vault-agent-k8s-demo/configs-k8s/
kubectl get configmap example-vault-agent-config -o yaml
kubectl apply -f kubernetes-vault/vault-agent-k8s-demo/example-k8s-spec.yml --record
~~~
- Скопировал файл index.html из контейнера nginx-container
~~~
kubectl cp vault-agent-example:/usr/share/nginx/html/index.html kubernetes-vault/index.html -c nginx-container
~~~
- Содержимое файла kubernetes-vault/index.html
~~~
  <html>
  <body>
  <p>Some secrets:</p>
  <ul>
  <li><pre>username: otus</pre></li>
  <li><pre>password: asajkjkahs</pre></li>
  </ul>
  
  </body>
  </html>

~~~

### CA на базе vault
- Включил pki секреты
~~~
kubectl exec -it vault-0 -- vault secrets enable pki
kubectl exec -it vault-0 -- vault secrets tune -max-lease-ttl=87600h pki
mkdir -p kubernetes-vault/serts
kubectl exec -it vault-0 -- vault write -field=certificate pki/root/generate/internal \
  common_name="exmaple.ru" ttl=87600h > kubernetes-vault/serts/CA_cert.crt
~~~
- Прописал URL и CA для отозванных сертификатов
~~~
kubectl exec -it vault-0 -- vault write pki/config/urls \
  issuing_certificates="http://vault:8200/v1/pki/ca" \
  crl_distribution_points="http://vault:8200/v1/pki/crl"
~~~
- Создал промежуточный сертификат
~~~
kubectl exec -it vault-0 -- vault secrets enable --path=pki_int pki
kubectl exec -it vault-0 -- vault secrets tune -max-lease-ttl=87600h pki_int
sudo apt  install jq
kubectl exec -it vault-0 -- vault write -format=json \
  pki_int/intermediate/generate/internal \
  common_name="example.ru Intermediate Authority" | jq -r '.data.csr' > kubernetes-vault/serts/pki_intermediate.csr
~~~
- Записал промежуточный сертификат в vault
~~~
kubectl cp kubernetes-vault/serts/pki_intermediate.csr vault-0:/tmp

kubectl exec -it vault-0 -- vault write -format=json pki/root/sign-intermediate \
  csr=@/tmp/pki_intermediate.csr \
  format=pem_bundle ttl="43800h" | jq -r '.data.certificate' > kubernetes-vault/serts/intermediate.cert.pem

kubectl cp kubernetes-vault/serts/intermediate.cert.pem vault-0:/tmp
kubectl exec -it vault-0 -- vault write pki_int/intermediate/set-signed certificate=@/tmp/intermediate.cert.pem
~~~
- Создал роль для выдачи сертификатов
~~~
kubectl exec -it vault-0 -- vault write pki_int/roles/example-dot-ru \
  allowed_domains="example.ru" allow_subdomains=true max_ttl="720h"
~~~
- Создал сертификат
~~~
kubectl exec -it vault-0 -- vault write pki_int/issue/example-dot-ru common_name="gitlab.example.ru" ttl="24h"
~~~
- Вывод команды при создании сертификата
~~~
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUNuEVEPdHshzEa6A+NF/4TyBY0pYwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0xOTEyMzExNjIwNTdaFw0yNDEy
MjkxNjIxMjdaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAM+x7J24GhI1
xFjjqXqIFxxAWej+nEp8tMSqWfiIPx8MkeaPrvTvqjk4pOD1zTuLsUjH+EMZcEmB
q7rrkM1rEcUsVS0ZfyPEweRriw1PV9o2sUSp2JLOAbJWV/Aj0zMnZuimgi5z2IhC
9+wO1mcyvLJv0Rzb9+KZSH/kQcLJory2gvvSKTzO5AOTykNs0MFjf3qsDC+OXgji
3BGt0/A3T/pvOT3cn2JXXeKG0ncQ5otjhj9SyZJVb28s6BjstlsGBVQyZ5z1ZIbK
IYM4m+11yFTziViuMUWjiF/70yzXhLQMYcAqMg0nDRJJGwpzl9k27GgqZC2G7HzX
dsmZvC+43lcCAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUdRBx++yW3MPGmoORDA/RTGci+88wHwYDVR0jBBgwFoAU
ql1f0YhznDTxux2a4DamD91cTVswNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
AUmYk6wDKT/cr7bvDrmmuh5YwJnXCblgJ4tvpckHJtJdagXGyI+i2Y1CucD7PsrU
Dx1B7ziy8xO1nWMRjuN2EvOS+AzqS/ruF+ybskdKVtjZqk0bUU5Ubf2bw6f++udq
WfD65ADLo0oba3HCVGlZSiKMDdANC0P/e6Qs7LO3bQBlCWscDL1yuww5Oa5BJ9ds
xpWE2DYYUFz0pLc/JU7hhqMzslWpxZ9K+c/lLt9/XzqWFJuI9VDWGcyyZV1mdb/F
edgIGfLD8epG6ybDnGen0h24HcQjRPFJGphL5n3qoVlycawo7XKzGLj0AH4rVEnT
xnEGgPfYT1zcSrs6akIiXg==
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
MIIDZzCCAk+gAwIBAgIUbjkqpol7hTPXgyqnWTN9AWpo9xwwDQYJKoZIhvcNAQEL
BQAwLDEqMCgGA1UEAxMhZXhhbXBsZS5ydSBJbnRlcm1lZGlhdGUgQXV0aG9yaXR5
MB4XDTE5MTIzMTE2MjY0NFoXDTIwMDEwMTE2MjcxNFowHDEaMBgGA1UEAxMRZ2l0
bGFiLmV4YW1wbGUucnUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC9
8zSnOs/d7ZGf7odPICCdF8QIFNMHJodiN1pje3jOn21WZhYHo+DaR3jY2knkxL3v
QkcC4kyutyTMwKawyF3SIU1229HGxtlnSeGfpvHz015jQ8+of/gY1DCIZiKLUyX3
WAIre6mlu6miADVJLGdG57inNky300VFMU9uxBTVbJD5BHrAaiaXJbG+ZfyUMY5p
kdHHAhKHy6HJmjRN6mg7xCvSNSo2ZBLzlf7ffTeFesEAZh/A4PXZLn0NJ0EdA+6Y
gM+0x17aFzMJDdoL5tSCqwiLINLPKFCsHTjl3umVzn1A97/fv2THde7UeUqTQFz1
PygQrDfOWIxxA+jlf/6tAgMBAAGjgZAwgY0wDgYDVR0PAQH/BAQDAgOoMB0GA1Ud
JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4EFgQUMl7/PPCrS7LWRKB9
wcNrWgh6mcowHwYDVR0jBBgwFoAUdRBx++yW3MPGmoORDA/RTGci+88wHAYDVR0R
BBUwE4IRZ2l0bGFiLmV4YW1wbGUucnUwDQYJKoZIhvcNAQELBQADggEBAAzAkbMe
UdAAA+vzrLH0T4+vi+0NHCjM38ugMT8FfO6cQyCly/21iBpi3tOzqgF/Rw/y4xrK
cukQBTpa3HRkBIJmdlpAHxrbny/4JXGLww4kOOmbTrs2neXRQGox2CaxSXp+BHm2
iGzO2FpCBbcf6+n/hHcDS/lOFupk9gdEhtRcCp57kIs3vZyfF8ZfVFQKWPKmxk7R
Jmkjkrf66o0LG1u86Kdn9epgF7OwqZTZqHPa2TR+dne/wUoJpsYrFdiUwPVzcEFb
CvN1OtHpFwTrGq5rFP+hS6508ThaFMIyp+TX6M9zqSIa9yK3hN0sowv3BCFveEON
wXWfKtO/kyl5+Yo=
-----END CERTIFICATE-----
expiration          1577896034
issuing_ca          -----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUNuEVEPdHshzEa6A+NF/4TyBY0pYwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0xOTEyMzExNjIwNTdaFw0yNDEy
MjkxNjIxMjdaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAM+x7J24GhI1
xFjjqXqIFxxAWej+nEp8tMSqWfiIPx8MkeaPrvTvqjk4pOD1zTuLsUjH+EMZcEmB
q7rrkM1rEcUsVS0ZfyPEweRriw1PV9o2sUSp2JLOAbJWV/Aj0zMnZuimgi5z2IhC
9+wO1mcyvLJv0Rzb9+KZSH/kQcLJory2gvvSKTzO5AOTykNs0MFjf3qsDC+OXgji
3BGt0/A3T/pvOT3cn2JXXeKG0ncQ5otjhj9SyZJVb28s6BjstlsGBVQyZ5z1ZIbK
IYM4m+11yFTziViuMUWjiF/70yzXhLQMYcAqMg0nDRJJGwpzl9k27GgqZC2G7HzX
dsmZvC+43lcCAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUdRBx++yW3MPGmoORDA/RTGci+88wHwYDVR0jBBgwFoAU
ql1f0YhznDTxux2a4DamD91cTVswNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
AUmYk6wDKT/cr7bvDrmmuh5YwJnXCblgJ4tvpckHJtJdagXGyI+i2Y1CucD7PsrU
Dx1B7ziy8xO1nWMRjuN2EvOS+AzqS/ruF+ybskdKVtjZqk0bUU5Ubf2bw6f++udq
WfD65ADLo0oba3HCVGlZSiKMDdANC0P/e6Qs7LO3bQBlCWscDL1yuww5Oa5BJ9ds
xpWE2DYYUFz0pLc/JU7hhqMzslWpxZ9K+c/lLt9/XzqWFJuI9VDWGcyyZV1mdb/F
edgIGfLD8epG6ybDnGen0h24HcQjRPFJGphL5n3qoVlycawo7XKzGLj0AH4rVEnT
xnEGgPfYT1zcSrs6akIiXg==
-----END CERTIFICATE-----
private_key         -----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEAvfM0pzrP3e2Rn+6HTyAgnRfECBTTByaHYjdaY3t4zp9tVmYW
B6Pg2kd42NpJ5MS970JHAuJMrrckzMCmsMhd0iFNdtvRxsbZZ0nhn6bx89NeY0PP
qH/4GNQwiGYii1Ml91gCK3uppbupogA1SSxnRue4pzZMt9NFRTFPbsQU1WyQ+QR6
wGomlyWxvmX8lDGOaZHRxwISh8uhyZo0TepoO8Qr0jUqNmQS85X+3303hXrBAGYf
wOD12S59DSdBHQPumIDPtMde2hczCQ3aC+bUgqsIiyDSzyhQrB045d7plc59QPe/
379kx3Xu1HlKk0Bc9T8oEKw3zliMcQPo5X/+rQIDAQABAoIBAQC7FiDlGSsFVelZ
kZEZ4PXeZDsDDqeu4kbz+LsBQuqA8Eu7jj7idYmQ1FZ1l8KyHQlJ74iLkaKfbulC
9fj4I9Eslvp6OBYM52vXrNAZ9E9YrPXJZU/RkYEly1Cl95rMiS/ax4cTlvBHuWdh
lTzmfmKWVsLrhrLXV7JhbSjkWyJ99YVhX8jPDhrUKVfnWgevLqhS0VbGnzgoR0ih
vkvFZ254JUa0HhTcs8zXNK4qfsvSvhtp0tozU3FzgGTKY40MjzeAAuXCd3hDCcDF
hhbp1Tnwb4o4UE4Rgsabbn9wsuQ6DEhrp6sTHnKXLjLKTQ5Tr/KX6PfIphjkS5FQ
5SG1e8eBAoGBAPC3vX0AQUZhueUw98Zz7xlCw1YVDfnCAo1YN92tLGgUNPLfFq1T
8y0WKaLE4y38aW8c/MxWFquyfuZBzXiR+e2iL65gWuoUD5lEAqLdGWNVLMdajKZ0
Rlu1mvSj5tcIRB47hRys0c2jcCeNqrYr1cKKNuobbPDe9XypwU6EzYQhAoGBAMoC
XURz6Pv+RwWyDHU0PikUCHCNBrrVGrM3gUJTxgxO1uuHOrzSKXjxeSQCJs70rUYB
q7ljw45p7Pw2leHQXAAJhceL5x/9pQHV/bGc580H+BpQ2Q+REwwcjNQbYPUBr6hP
EWOHqsbVRV12WAKiqsSFgLzUyT5RxvFBHPSgBykNAoGBAOR7ovKJUWv6yrZO+oB1
/pcdlceZiIButHlxKOXSv/myZGe7dQzkSEedZ7vF4lT95x2+h/10IWSrsmPgRaWR
+YajkVqUvva8P+occdwgvT5Z1H0M57//UeEuyXw4Lp4gjHedy0VijGoCHiyM/WKY
zPcwtdsUWR1wo9bGUmOzDlfBAoGAeHDEbuW0yVmnuquXZeHKFe/NwF003/viuWuk
c4lDEV+IIFE2IhIji+pc0a0+ujGDhbPFUPk8RRK+qvlYj5QM5jDHRFwTZy1xThDp
+xWT1tijgf0mDXPvqU70YBoayrlAo9bQhUkD9xx9COZgPuIBcr4uLWeovLFBLeIm
g2tOGZkCgYEA6txXJ5Ciews80M3vziUSHCJd1gx+kVCv5vsjFBscqnH6iH8aImzD
HRBrafnZHygrRPTJ0mWJqpH3hCQQrjIXkmRNHsyqjOnUtuzWjcgf9bxD+prwzpCT
D8g/S9w43J16YCmEPIoX6QGpb865RyumlbUdKPEa4oR946JlPR+0JVU=
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       6e:39:2a:a6:89:7b:85:33:d7:83:2a:a7:59:33:7d:01:6a:68:f7:1c
~~~

- Отозвал сертификат
~~~
kubectl exec -it vault-0 -- vault write pki_int/revoke serial_number="6e:39:2a:a6:89:7b:85:33:d7:83:2a:a7:59:33:7d:01:6a:68:f7:1c"
~~~

- Вывод команды при отзыве сертификата
~~~
Key                        Value
---                        -----
revocation_time            1577809759
revocation_time_rfc3339    2019-12-31T16:29:19.784001292Z
~~~

### [Вернуться в корень репо](/../../)