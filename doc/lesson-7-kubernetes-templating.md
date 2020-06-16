- Создан аккаунт в Google Cloud.

- Создан Kubernetes кластер. Его конфигурация подключена к `kubectl`.
    ```bash
    $ kubectl cluster-info
    Kubernetes master is running at https://34.76.167.115
    GLBCDefaultBackend is running at https://34.76.167.115/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
    KubeDNS is running at https://34.76.167.115/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    Metrics-server is running at https://34.76.167.115/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
    ```
- Установлен Helm 3.

- Добавлен Helm-репозиторий `https://kubernetes-charts.storage.googleapis.com`.

- Установлен Helm release stable/nginx-ingress версии 1.39.0.
    
    Версия 1.11.1 устанавливается с ошибкой.
    ```bash
    helm repo add stable https://kubernetes-charts.storage.googleapis.com
    helm repo list
    kubectl apply -f kubernetes-templating/nginx-ingress/namespace.yaml
    helm upgrade --install nginx-ingress stable/nginx-ingress --wait --namespace=nginx-ingress --version=1.39.0  
    ```

- Установлен cert-manager версии 0.15.0
  
  Создан ClusterIssuer
  ```bash
  helm repo add jetstack https://charts.jetstack.io
  kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/deploy/manifests/00-crds.yaml
  kubectl create namespace cert-manager
  kubectl label namespace cert-manager certmanager.k8s.io/disable-validation="true"
  helm upgrade --install cert-manager jetstack/cert-manager --wait --namespace=cert-manager --version=0.15.0
  kubectl apply -f kubernetes-templating/cert-manager/cluster-issuer.yaml
  ```

  Cert-manager протестирован при помощи файла `kubernetes-templating\cert-manager\test-resources.yaml`

- Установлен chartmuseum версии 2.13.0
  ```bash
  kubectl create namespace chartmuseum
  helm upgrade --install chartmuseum stable/chartmuseum --namespace=chartmuseum --version=2.13.0 -f kubernetes-templating/chartmuseum/values.yaml
  ``` 
  
  Chartmuseum доступен по адресу: https://chartmuseum.34.78.254.203.nip.io

- Сборка helm chart 
    
    Команды для сборки Helm chart и итправки их в helm repository 
    ```bash
    #!/usr/bin/env bash
   
    set -e
    
    export MYPRODUCT_VERSION=8.1.0
  
    helm repo add chartmuseum https://chartmuseum.34.78.254.203.nip.io
    helm plugin install https://github.com/chartmuseum/helm-push.git
    
    # Lint Helm chart
    helm lint myproduct
    
    # Package Helm chart
    echo "==> Package myproduct"
    helm package myproduct \
    --app-version $MYPRODUCT_VERSION \
    --version $MYPRODUCT_VERSION \
    --dependency-update \
    --destination .  
  
    echo "==> Push myproduct"
    helm push myproduct-$MYPRODUCT_VERSION.tgz chartmuseum
  ```

- Установлен harbor версии 1.3.2
  ```bash 
  helm repo add harbor https://helm.goharbor.io
  kubectl create namespace harbor
  helm upgrade --install harbor harbor/harbor --namespace=harbor --version=1.3.2  -f kubernetes-templating/harbor/values.yaml 
  ```
  Harbor доступен по адресу: `https://harbor.34.78.254.203.nip.io`.  
  
  Реквизиты по умолчанию - admin/Harbor12345
    
- Созданы helm chart frontend и heapster-shop
  
  Frontend является зависимостью к heapster-shop

- Установлен heapster-shop
  ```bash
  kubectl create ns hipster-shop
  helm dep update kubernetes-templating/hipster-shop 
  helm upgrade --install hipster-shop --atomic kubernetes-templating/hipster-shop --namespace hipster-shop
  ```
  
  Heapster-shop доступен по адресу: `http://shop.34.78.254.203.nip.io/` 

- Создан пакет hipster-shop
    ```bash
    helm package kubernetes-templating/hipster-shop
  ```

- Kubecfg
  
  Установлен kubecfg
  
  Создан шаблон `kubernetes-templating\kubecfg\services.jsonnet`, описывающий сервисы paymentservice и shippingservice
  
  Применение шаблона
  ```bash
  kubecfg update kubernetes-templating/kubecfg/services.jsonnet --namespace hipster-shop
  ```
  
- Kustomize
  
  Шаблонизирован компонент cartservice в каталоге `kubernetes-templating\kustomize`
  
  
  Установка на production
  ```bash
  kubectl apply -k kubernetes-templating/kustomize/overrides/prod/
  ```