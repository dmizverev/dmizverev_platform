- Установка Vault
  
  Установка Consul
  ```bash
  git clone https://github.com/hashicorp/consul-helm.git
  helm install consul ./consul-helm
  ```
  
  Установка Vault
  ```bash
  git clone https://github.com/hashicorp/vault-helm.git
  helm upgrade --install vault ./vault-helm --values "C:\Users\Dmitriy.Zverev\Work\git\dmizverev_platform\kubernetes-vault\vault.yaml"
  ```
  
  Проверка установки Vault
  ```bash
  $ helm status vault
  
  NAME: vault
  LAST DEPLOYED: Fri Jun 12 11:04:58 2020
  NAMESPACE: default
  STATUS: deployed
  REVISION: 1
  TEST SUITE: None
  NOTES:
  Thank you for installing HashiCorp Vault!
  
  Now that you have deployed Vault, you should look over the docs on using
  Vault with Kubernetes available here:
  
  https://www.vaultproject.io/docs/
  
  
  Your release is named vault. To learn more about the release, try:
  
    $ helm status vault
    $ helm get vault
  
  ```
  
  ```bash
   $ kubectl logs vault-0
   Show log entriesMatch expression
   Filter
    ==> Vault server configuration:
   
                Api Address: http://10.44.6.35:8200
                        Cgo: disabled
            Cluster Address: https://vault-0.vault-internal:8201
                 Listener 1: tcp (addr: "[::]:8200", cluster address: "[::]:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
                  Log Level: info
                      Mlock: supported: true, enabled: false
              Recovery Mode: false
                    Storage: consul (HA available)
                    Version: Vault v1.4.2
   
   2020-06-12T09:17:33.443Z [INFO]  proxy environment: http_proxy= https_proxy= no_proxy=
   2020-06-12T09:17:33.443Z [WARN]  storage.consul: appending trailing forward slash to path
   
  ```  
  
  Pods находятся в состоянии Not Ready. Это нормально, работаем дальше.

- Инициализация кластера 
  ```bash
  $ kubectl exec -ti vault-0 -- vault operator init --key-shares=1 --key-threshold=1 --address=http://127.0.0.1:8200
  
  Unseal Key 1: zPcIdN+9XamSMS9ng/up3DOrNKYhqwvCsPJVYziZR+w=
  Initial Root Token: s.WcoZ1AMKgqo2bt4U9PW9AuTw
  ```

- Unseal кластера

  ```bash
  $ kubectl exec -it vault-0 -- vault operator unseal
  'zPcIdN+9XamSMS9ng/up3DOrNKYhqwvCsPJVYziZR+w='
  $ kubectl exec -it vault-1 -- vault operator unseal
  'zPcIdN+9XamSMS9ng/up3DOrNKYhqwvCsPJVYziZR+w='
  $ kubectl exec -it vault-2 -- vault operator unseal
  'zPcIdN+9XamSMS9ng/up3DOrNKYhqwvCsPJVYziZR+w='
  ```   
  
  Теперь все Pod в состоянии Ready.
  
- Статус Vault
  ```bash
  $ kubectl exec -it vault-0 -- vault status  
  
  Key             Value
  ---             -----
  Seal Type       shamir
  Initialized     true
  Sealed          false
  Total Shares    1
  Threshold       1
  Version         1.4.2
  Cluster Name    vault-cluster-9809aaab
  Cluster ID      63e6019d-be1e-ab7c-829f-a6126e184c16
  HA Enabled      true
  HA Cluster      https://vault-0.vault-internal:8201
  HA Mode         active
  
  ```
  
- Залогиниться в Vault с полученным ранее root-токеном `s.WcoZ1AMKgqo2bt4U9PW9AuTw`

  ```bash
  $ kubectl exec -it vault-0 -- vault login
  
  Token (will be hidden):
  Success! You are now authenticated. The token information displayed below
  is already stored in the token helper. You do NOT need to run "vault login"
  again. Future Vault requests will automatically use this token.
  
  Key                  Value
  ---                  -----
  token                s.WcoZ1AMKgqo2bt4U9PW9AuTw
  token_accessor       GmLWKc1iLAlUv85rM3MXuN88
  token_duration       ∞
  token_renewable      false
  token_policies       ["root"]
  identity_policies    []
  policies             ["root"]
  
  ```
  
- Запросить список авторизации

  ```bash
  $ kubectl exec -it vault-0 -- vault auth list
  
  Path      Type     Accessor               Description
  ----      ----     --------               -----------
  token/    token    auth_token_c6e6df89    token based credentials
  
  ```
  
- Создание секретов

  ```bash
  kubectl exec -it vault-0 -- vault secrets enable --path=otus kv
  kubectl exec -it vault-0 -- vault secrets list --detailed
  kubectl exec -it vault-0 -- vault kv put otus/otus-ro/config username='otus' password='asajkjkahs'
  kubectl exec -it vault-0 -- vault kv put otus/otus-rw/config username='otus' password='asajkjkahs'
  
  ```
  
  ```bash
  $ kubectl exec -it vault-0 -- vault read otus/otus-ro/config
  
  Key                 Value
  ---                 -----
  refresh_interval    768h
  password            asajkjkahs
  username            otus
  
  ```
  
  ```bash
  $ kubectl exec -it vault-0 -- vault kv get otus/otus-rw/config
  
  ====== Data ======
  Key         Value
  ---         -----
  password    asajkjkahs
  username    otus
  
  ```
  
- Включение авторизации через Kubernetes

  ```bash
  kubectl exec -ti vault-0 -- vault auth enable kubernetes
  ```
  
  Обновленный список авторизации
  
  ```bash
  $ kubectl exec -ti vault-0 -- vault auth list
  
  Path           Type          Accessor                    Description
  ----           ----          --------                    -----------
  kubernetes/    kubernetes    auth_kubernetes_061359f5    n/a
  token/         token         auth_token_c6e6df89         token based credentials
  ```
  
  Создан ClusterRoleBinding manifest в файле `kubernetes-vault\vault-auth-service-account.yml`
  
  ```bash
  kubectl create serviceaccount vault-auth
  kubectl apply -f vault-auth-service-account.yml
  ```
  Запись конфигов в Vault
  
  ```bash
  export VAULT_SA_NAME=$(kubectl get sa vault-auth -o jsonpath="{.secrets[*]['name']}")
  export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)
  export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
  # Команда ниже получает адрес API Kubernetes кластера, вырезая символы кодов цвета:
  # `export K8S_HOST=$(kubectl cluster-info | grep ‘Kubernetes master’ | awk ‘/https/ {print$NF}’ | sed ’s/\x1b\[[0-9;]*m//g’ )`
  # Я не доверяю этому шаманству, поэтому лучше либо посмотреть адрес самому,
  # либо сделать через `kubectl config view -o jsonpath=...`
  export K8S_HOST='https://35.238.100.229'

  ```
  
  ```bash
  kubectl exec -it vault-0 -- vault write auth/kubernetes/config \
  token_reviewer_jwt="$SA_JWT_TOKEN" \
  kubernetes_host="$K8S_HOST" \
  kubernetes_ca_cert="$SA_CA_CRT"
  ```
  
  Создан файл политики `kubernetes-vault\otus-policy.hcl`
  
  ```bash
  # Копирование происходит в /tmp
  # При попытке записать в ./ получаю ошибку "Permission denied"
  kubectl cp otus-policy.hcl vault-0:./tmp
  
  kubectl exec -it vault-0 -- vault policy write otus-policy /otus-policy.hcl
  
  kubectl exec -it vault-0 -- vault write auth/kubernetes/role/otus \
  bound_service_account_names=vault-auth \
  bound_service_account_namespaces=default policies=otus-policy ttl=24h
  ```
  
- Проверка работы авторизации

  Создание Pod с привязанным сервис аккоунтом и установка туда curl и jq
  
  ```bash
  kubectl run --generator=run-pod/v1 tmp  -i  --serviceaccount=vault-auth --image alpine:3.7 sleep 10000
  apk add curl jq
  ```
  
  Получение клиентского token
  
  ```bash
  kubectl exec -ti tmp -- sh
  
  $ VAULT_ADDR=http://vault:8200
  $ KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  $ curl --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq
  # Почему в описании домашней работы роль была otus, а в конце стала test я не понял 
  $ TOKEN=$(curl -k -s --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq '.auth.client_token' | awk -F\" '{print $2}')
  
  ```
  
  Клиентский token: `s.EbAlhXym1LMC15TXzocDYTgg`
  
- Чтение и обновление секретов

  Проверка чтения
  
  ```bash
  curl --header "X-Vault-Token:s.EbAlhXym1LMC15TXzocDYTgg" $VAULT_ADDR/v1/otus/otus-ro/config
  curl --header "X-Vault-Token:s.EbAlhXym1LMC15TXzocDYTgg" $VAULT_ADDR/v1/otus/otus-rw/config
  ```
  
  Проверка записи
  
  ```bash
  curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:s.EbAlhXym1LMC15TXzocDYTgg" $VAULT_ADDR/v1/otus/otus-ro/config
  curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:s.EbAlhXym1LMC15TXzocDYTgg" $VAULT_ADDR/v1/otus/otus-rw/config
  curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:s.EbAlhXym1LMC15TXzocDYTgg" $VAULT_ADDR/v1/otus/otus-rw/config1
  ```
  
  Для того, чтобы менять `/otus/otus-rw/config` надо в `kubernetes-vault\otus-policy.hcl` добавить в capabilities "update"
  
- Use case использования авторизации через кубер
  
  Получение секрета для Nginx.
  
  Созданы файлы kubernetes-vault\configs-k8s\example-k8s-spec.yml и kubernetes-vault\configs-k8s\example-vault-agent-config.yaml
  Выставлен корректный VAULT_ADDR и role
  
  ```bash
  kubectl apply -f configs-k8s\example-vault-agent-config.yaml
  kubectl get configmap example-vault-agent-config -o yaml
  kubectl apply -f configs-k8s\example-k8s-spec.yml --record
  kubectl exec -ti vault-agent-example -c nginx-container  -- cat /etc/secrets/index.html
  ```
  
  В итоге получим такой html:
  
  ```
  <html>
  <body>
  <p>Some secrets:</p>
  <ul>
  <li><pre>username: otus</pre></li>
  <li><pre>password: asajkjkahs</pre></li>
  </ul>

  </body>
  </html>
  
  ```
  
- CA на базе Vault

  Включение PKI
  
  ```bash
  kubectl exec -it vault-0 -- vault secrets enable pki
  kubectl exec -it vault-0 -- vault secrets tune -max-lease-ttl=87600h pki
  kubectl exec -it vault-0 -- vault write -field=certificate pki/root/generate/internal common_name="example.ru"  ttl=87600h > CA_cert.crt
  ```
  
  Прописывание URL
  
  ```bash
  kubectl exec -it vault-0 -- vault write pki/config/urls issuing_certificates="http://vault:8200/v1/pki/ca" crl_distribution_points="http://vault:8200/v1/pki/crl"
  ```
  
  Создание промежуточного сертификата
  
  ```bash
  kubectl exec -it vault-0 -- vault secrets enable --path=pki_int pki
  kubectl exec -it vault-0 -- vault secrets tune -max-lease-ttl=87600h pki_int
  kubectl exec -it vault-0 -- vault write -format=json pki_int/intermediate/generate/internal common_name="example.ru Intermediate Authority"  | jq -r '.data.csr' > pki_intermediate.csr
  ```
  
  Прописывание промежуточного сертификата в Vault
  
  ```bash
  kubectl cp pki_intermediate.csr vault-0:./tmp
  kubectl exec -it vault-0 -- vault write -format=json pki/root/sign-intermediate csr=@/tmp/pki_intermediate.csr format=pem_bundle ttl="43800h" |  jq -r '.data.certificate' > intermediate.cert.pem
  kubectl cp intermediate.cert.pem vault-0:./tmp
  kubectl exec -it vault-0 -- vault write pki_int/intermediate/set-signed certificate=@/tmp/intermediate.cert.pem
  ```
  
- Создание роли
  
  ```bash
  kubectl exec -it vault-0 -- vault write pki_int/roles/example-dot-ru allowed_domains="example.ru" allow_subdomains=true   max_ttl="720h"
  ```
  
- Создание сертификатов
  
  ```bash
  $ kubectl exec -it vault-0 -- vault write pki_int/issue/example-dot-ru common_name="gitlab.example.ru" ttl="24h"
  
    Key                 Value
  ---                 -----
  ca_chain            [-----BEGIN CERTIFICATE-----
  MIIDnDCCAoSgAwIBAgIUVtFMbHmoD5yOMDn2mRJ4AemIA3wwDQYJKoZIhvcNAQEL
  BQAwFTETMBEGA1UEAxMKZXhhbXBsZS5ydTAeFw0yMDA2MTIxMzAxNTJaFw0yNTA2
  MTExMzAyMjJaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
  dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAJxaDzOnwofP
  7x6NoCjL5ehzdCKpkvz9QyyZDyKIPoOQAd4//mRQRIhMAA4DBUVWg75EQv0KRpIH
  CXbAV5kYXfNZzvLUe0m8yN9jNG4zo19HqJHpJ2PgshY6f+RpgpnYTmgQX26UvjNj
  59J/wP6xTEy5EX9Dt22YikkMwMk3soDQMQbEy1M1je3DKufiaEh5IZeLr5mFpvH/
  MmPtoXxTqu1gLj+EtghBiCy4lO29cK6zBukPACiOk7KafO8NyGb14Ndoxt4z3DoU
  zvXqMDvRUF6PDa/PBC8EPCCyQiCb83oExBkrOFjXbzbg7FeM1JnwZpvW1gjgm8qk
  CzOelGyR5K0CAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
  AwEB/zAdBgNVHQ4EFgQU6ERQ/wawr/YpG3aqezz28Q8givMwHwYDVR0jBBgwFoAU
  pyEgsTMegkjY3IS2RLUSVNRiO+cwNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
  hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
  aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
  FI71hUUoomDGCMG2a2fepQh+yu+oraVZVvoswQTQYchKM5JsoUBSmAVqtlBgWRpv
  DD8c3qtjJ/A9eRRyeVwhFHJSlNKb/MgR4ZfeWNbKo6DZu00MrOY3gQsm21F6eORH
  n3dMFxZ74A3XFzJvV/nyFNQ8WN31K1c9sZ+e6uhWuMl3uXHWAKhdNvqHPUs7pd1f
  bDZOrbEv38cx4ZD7qtXL/cC6taM6oyMvhNmEGH4QNUHyzWs3V8FmhzUmGeqF1x8g
  Y/nQdIuKYCBjO1DWeRoml7z/18kyZ4HmTuXxP1i4DMirjRNgzeTRnHr8G5ONnSdk
  HleCZzwAd2nU4raCN6Brwg==
  -----END CERTIFICATE-----]
  certificate         -----BEGIN CERTIFICATE-----
  MIIDZzCCAk+gAwIBAgIUHKadruZKM/44wS+9akGMZwEtk4UwDQYJKoZIhvcNAQEL
  BQAwLDEqMCgGA1UEAxMhZXhhbXBsZS5ydSBJbnRlcm1lZGlhdGUgQXV0aG9yaXR5
  MB4XDTIwMDYxMjEzMDQxNVoXDTIwMDYxMzEzMDQ0NVowHDEaMBgGA1UEAxMRZ2l0
  bGFiLmV4YW1wbGUucnUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCz
  GzJfneYV1ZMc2GTR23XF6+nRdZf0CZEnPiDWsI68aI1o2DJ5rYvRzK0PWnzwBThQ
  PK5OKn4IRbT+0fQ1NwTjJMz+LUkS6qZ0qJ0G9sRoKrfcL7+TBvJMWqlOAWcN7UB5
  oTd8swzLdPHJYGWu/4Gc9Gd7tbAwpV4ztmF+28n7TVfVi8TUJG2dJen5xZKcya4+
  KYtSpyEb0LT9m+hi1qGrjBGRtVInPLzGl0HDCPOqAlW9QvKvfzrwY5KrFfP0sCfK
  zd98lDSzxZjlNNzTx3EwGfNd3y/TmIUk4MXbaeYZpmUTnNi9S/t0vcPegvl7m5qm
  jx6C8HKYCyhazI7K7XltAgMBAAGjgZAwgY0wDgYDVR0PAQH/BAQDAgOoMB0GA1Ud
  JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4EFgQU4wwvUNg3VCD68Mk9
  2zEc+zX1bRQwHwYDVR0jBBgwFoAU6ERQ/wawr/YpG3aqezz28Q8givMwHAYDVR0R
  BBUwE4IRZ2l0bGFiLmV4YW1wbGUucnUwDQYJKoZIhvcNAQELBQADggEBAAuMnuiK
  ERsWL+ckwxOXb0hsq1HUWAHuxVAig8SJCMm6NU3KAji7AdywyKtRT8/aMZCGS9mC
  OQS9v9hwVX/wO1Gmo9TDhvB8mFW6unZpPv5bz8SW18cUQnFeEAjY87q1sJr7ettZ
  yukZ92qKydVCx692dugikrmVG1NxfElRwmvVwf2hmnfPjI3zFjX/XnTAMPaw/wUb
  r+2u12Sh3ra71+dIfkiUhkbUge6j0v790rgEA/o7DbUizVXbFomog1UgUsBkvqBG
  hLINLJARKJFn9D88bteWe5DIdeaoegEPK1zoo+g8ov/8PShV/758tU2pE+yRDyJM
  V4PgzDq5WfFzL4s=
  -----END CERTIFICATE-----
  expiration          1592053485
  issuing_ca          -----BEGIN CERTIFICATE-----
  MIIDnDCCAoSgAwIBAgIUVtFMbHmoD5yOMDn2mRJ4AemIA3wwDQYJKoZIhvcNAQEL
  BQAwFTETMBEGA1UEAxMKZXhhbXBsZS5ydTAeFw0yMDA2MTIxMzAxNTJaFw0yNTA2
  MTExMzAyMjJaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
  dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAJxaDzOnwofP
  7x6NoCjL5ehzdCKpkvz9QyyZDyKIPoOQAd4//mRQRIhMAA4DBUVWg75EQv0KRpIH
  CXbAV5kYXfNZzvLUe0m8yN9jNG4zo19HqJHpJ2PgshY6f+RpgpnYTmgQX26UvjNj
  59J/wP6xTEy5EX9Dt22YikkMwMk3soDQMQbEy1M1je3DKufiaEh5IZeLr5mFpvH/
  MmPtoXxTqu1gLj+EtghBiCy4lO29cK6zBukPACiOk7KafO8NyGb14Ndoxt4z3DoU
  zvXqMDvRUF6PDa/PBC8EPCCyQiCb83oExBkrOFjXbzbg7FeM1JnwZpvW1gjgm8qk
  CzOelGyR5K0CAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
  AwEB/zAdBgNVHQ4EFgQU6ERQ/wawr/YpG3aqezz28Q8givMwHwYDVR0jBBgwFoAU
  pyEgsTMegkjY3IS2RLUSVNRiO+cwNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
  hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
  aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
  FI71hUUoomDGCMG2a2fepQh+yu+oraVZVvoswQTQYchKM5JsoUBSmAVqtlBgWRpv
  DD8c3qtjJ/A9eRRyeVwhFHJSlNKb/MgR4ZfeWNbKo6DZu00MrOY3gQsm21F6eORH
  n3dMFxZ74A3XFzJvV/nyFNQ8WN31K1c9sZ+e6uhWuMl3uXHWAKhdNvqHPUs7pd1f
  bDZOrbEv38cx4ZD7qtXL/cC6taM6oyMvhNmEGH4QNUHyzWs3V8FmhzUmGeqF1x8g
  Y/nQdIuKYCBjO1DWeRoml7z/18kyZ4HmTuXxP1i4DMirjRNgzeTRnHr8G5ONnSdk
  HleCZzwAd2nU4raCN6Brwg==
  -----END CERTIFICATE-----
  private_key         -----BEGIN RSA PRIVATE KEY-----
  MIIEowIBAAKCAQEAsxsyX53mFdWTHNhk0dt1xevp0XWX9AmRJz4g1rCOvGiNaNgy
  ea2L0cytD1p88AU4UDyuTip+CEW0/tH0NTcE4yTM/i1JEuqmdKidBvbEaCq33C+/
  kwbyTFqpTgFnDe1AeaE3fLMMy3TxyWBlrv+BnPRne7WwMKVeM7ZhftvJ+01X1YvE
  1CRtnSXp+cWSnMmuPimLUqchG9C0/ZvoYtahq4wRkbVSJzy8xpdBwwjzqgJVvULy
  r3868GOSqxXz9LAnys3ffJQ0s8WY5TTc08dxMBnzXd8v05iFJODF22nmGaZlE5zY
  vUv7dL3D3oL5e5uapo8egvBymAsoWsyOyu15bQIDAQABAoIBAGTPshLPtWokxKE/
  y7+zXx8AIqObJORfXixQc/tjdXPnBXE1/3Mtk72LDv3NWPVgesnu3c1xbW8KjU3A
  r0wko8OWOyv2IWNcYETZg0kgLHzVTpfI6HPBPTBs907Iy1Czcc8ER08RGOqL8GwA
  rjtJ5ZKKnpSrN3iqG9PPnCDjZVTkynutZykcYel34fH1TA3yZG4Q4DJXGIbnoB5+
  GKU1Am6NfRJXuvyqkkVEDUsHwS8/mzKytzQzTw0DqK4QeiXUOmTztYkDJe95Akgx
  hpYn2QtDkoNweV+GvKTgAMBeSM/PFEwl5l9p/nsg7W2rk12jWbvyPipi9gCgbQdo
  dsih6qECgYEA36qwAWc9Ssw1t0NMp++Kx/U9SH1YSBz7/JBco3r8ywREWauhCM0N
  Ra9YMh2bv/Y3P5AiBCZo0QyyDoxejJonZihNNh9wnw8sngL2R4tTICK4tz9Du3je
  30GuF0nmAhv4zpAa/eH1a84vrbGiayU8Rfgem5VMzltlpSqS/9UNz+MCgYEAzP9x
  bIw/KzbqO9xXggAJMh8GQZboukBl90mr7Te0QR1tDsms7YxWHTS9IMCmzUai/BDw
  czwCWTs6jdLspTbKfWSqyJanAZwqe/VC3dle8nsqSTqfsjP4zNZJBGYbD88ZGb07
  cFVsOnLCcAPz0GWmLYuLs4f5saXPqv839f8yMm8CgYAyyw1rVCmsKdHtC2CGJrUK
  kdvX8Xcx8TsccSBIk+6CoDZxcrOATyi7cYWC5AxxvJVxXucKsDpPdyWcfi4emgdm
  gLKAHwWxaX3FaIDLYI2BF8GBA+H62gkrBDxn14VfZ0DKkBlBHKZiVBGpzVRIJs2Y
  Si+RP4eQuVrM9m0pohWf5wKBgBdBp5GD+6qgaURvQ/I4pNJt2JzaTP7MTYUXc4zO
  9AErIHM8CAVPFXnswMQVdxb0u4rTNSQtm6qZ4JO0aSp5I9HD+OgWx02UdPFpKrPW
  dEIYHPz/zJw/7yr16IS6PLm3agaUhEjDOCsNV+ezWxa6YXbrTOcKNxajVAL3P1cG
  I6C7AoGBAMK4dZHmHe4gzv5Ys3OXqK4AS+d/2mHHhlStbKrZOq9CJp55T2uzTPf/
  GPnkeaemLa0XEi2xfefVa+NpN1xexfXqwIgNoEvkaRw+gKSh4FEN4AqCP5a24kzk
  BvHKqJ3WQzs+e4OCT7TVQrkyFWhKx7cmRZDMbibHPm6uRv+vx5s1
  -----END RSA PRIVATE KEY-----
  private_key_type    rsa
  serial_number       1c:a6:9d:ae:e6:4a:33:fe:38:c1:2f:bd:6a:41:8c:67:01:2d:93:85
  
  ```
  
  ```bash
  $ kubectl exec -it vault-0 -- vault write pki_int/revoke serial_number=1c:a6:9d:ae:e6:4a:33:fe:38:c1:2f:bd:6a:41:8c:67:01:2d:93:85
  
  Key                        Value
  ---                        -----
  revocation_time            1591967186
  revocation_time_rfc3339    2020-06-12T13:06:26.722630735Z
  ```
  
- Включение TLS
  
  - Создать CSR
    ```bash
    openssl genrsa -out "vault_gke.key" 4096
    ```
    
    ```
    vi vault_gke_csr.cnf
    
    [req]
    default_bits = 2048
    default_md = sha256
    req_extensions = v3_ext
    distinguished_name = dn
    [req_distinguished_name]
    [ dn ]
    commonName = localhost
    stateOrProvinceName = Moscow
    countryName = RU
    emailAddress = lucky@perflabs.org
    organizationName = Perflabs
    organizationalUnitName = Development
    [ v3_ext ]
    basicConstraints = CA:FALSE
    keyUsage = keyEncipherment,dataEncipherment
    extendedKeyUsage = serverAuth
    subjectAltName = @alt_names
    [ alt_names ]
    DNS.0 = localhost
    DNS.1 = vault
    ```
    
    ```bash
    openssl req -config vault_gke_csr.cnf -new -key vault_gke.key -nodes -out vault.csr
    ```
  
    ```bash
    cat <<EOF >"csr.yaml"
    
    apiVersion: certificates.k8s.io/v1beta1
    kind: CertificateSigningRequest
    metadata:
      name: vault-csr
    spec:
      groups:
      - system:authenticated
      request: $(base64 < "vault.csr" | tr -d '\n')
      usages:
      - digital signature
      - key encipherment
      - server auth
    EOF
    
    kubectl create -f "csr.yaml"
    kubectl certificate approve "vault-csr"
    ```
    
  - Загрузить сертификаты в Kubernetes
  
    ```bash
    kubectl get csr vault-csr -o jsonpath='{.status.certificate}'  | base64 --decode > vault.crt
    kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 -d > "vault.ca"
    kubectl create secret tls vault-certs --cert=vault.crt --key=vault_gke.key
    ```
    
  - Переустановить Vault с включенным TLS
  
  - Получить секреты
  
    ```bash
    $ curl --cacert vault.ca -H "X-Vault-Token: s.EbAlhXym1LMC15TXzocDYTgg" -X GET https://localhost:8200/v1/otus/otus-ro/config
    
    {"request_id":"d981c792-a88c-abb5-516f-af2c00803c9a","lease_id":"","renewable":false,"lease_duration":2764800,"data":{"password":"asajkjkahs","username":"otus"},"wrap_info":null,"warnings":null,"auth":null}
  ```
  
- Автообновление сертификатов

  - Взять docker image nginx с включенным ssl: `dmizverev/nginx-ssl:1.1.0`
  
  - Политика по обновлению сертификатов: `kubernetes-vault\pki-policy.hcl`
    ```bash 
    kubectl cp pki-policy.hcl vault-0:/tmp
    kubectl exec -it vault-0 -- vault policy write pki-policy /tmp/pki-policy.hcl
    ```

  - Роль renew-pki
    ```bash
    kubectl exec -it vault-0 -- vault write auth/kubernetes/role/renew-pki bound_service_account_names=vault-auth bound_service_account_namespaces=default policies=pki-policy  ttl=24h
    ```
  
  - Pod для nginx
    ```bash
    kubectl apply -f nginx.yaml
    ```
  
  - Просмотр сертификатов
    
    - Сертификат до обновления:
    
      ![cert1](img/lesson-12-1.PNG)
      
    - Сертификат после обновления:
    
      ![cert2](img/lesson-12-2.PNG)
  
  