# GitOps и инструменты поставки

- Подготовлен Gitlab-репозиторий

    `https://gitlab.com/dmizverev/microservices-demo`


    ```bash
    git clone https://github.com/GoogleCloudPlatform/microservices-demo
    cd microservices-demo
    git remote add gitlab git@gitlab.com:dmizverev/microservices-demo.git
    git remote remove origin
    git push gitlab master
    ```

- Подготовлены Helm charts в директории `deploy/charts`

    ```bash
    $ tree -L 1 deploy/charts

    deploy/charts
    ├── adservice
    ├── cartservice
    ├── checkoutservice
    ├── currencyservice
    ├── emailservice
    ├── frontend
    ├── grafana-load-dashboards
    ├── loadgenerator
    ├── paymentservice
    ├── productcatalogservice
    ├── recommendationservice
    └── shippingservice
    ```

- Создан кластер GKE c Istio addon

- Собраны docker images микросервисов проекта microservices-demo и помещены в Docker Hub.

- GitOps

    - Установлен CRD HelmRelease

        ```bash
        kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml
        ```

    - Добавлен репозиторий Flux

        ```
        helm repo add fluxcd https://charts.fluxcd.io
        ```

    - Установка Flux c настройками из `flux.values.yaml`

        ```bash
        kubectl create namespace flux
        helm upgrade --install flux fluxcd/flux -f flux.values.yaml --namespace flux
        ```

    - Установка Helm Operator с настройками из файла `helm-operator.values.yaml`

        ```bash
        helm upgrade --install helm-operator fluxcd/helm-operator -f helm-operator.values.yaml --namespace flux
        ```

    - Установка fluxctl на локальную машину

    - Получение ключа:

        ```bash
        $ fluxctl identity --k8s-fwd-ns=flux

        ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDFKYGpmH5h0ZaQXHyiEqwbqSj3+ycOncAHVXGHH71CGSmEQjysk+NDckDrBIqGnqfbvpegQNtjG3ieDl2L5KQ/VC54AyjYXiN3ZzR5LIPTvSziHqfu8eDIjPvmf+u8nxBbQ5NOH4w0MI4Syx0Bt4JhK6PdHbeFS2Or5AiowVP5BiQQt5HJ2B+cksedkMogIi58qHHc78CPA437od2//sdIZyUE9AlWIXsHDgBMFLIlX6AxqfhkFTWRmX0moE++u+vLlDrMdRgO3YR8yRFTAPuXBsCxeSNEAdFKEEznhtcD0a/kMFxJhmL/pNfCpUCBNWSNnfnq8OeiOZxJgj1PFv/xNreifH5gOLg/mUobaY+kxlR1eDwSF8/3shntaFmqIGlguiC5AtPgk/jWETMYRzgEY5dykHtOPTkHVgUE/mz5M7FxolVlTyQSRUk9C/+Xu4a152zOmVfyYenW2FyksilYsfg83RIBMCLIRtR9awLqEx+g9fpUNx1KNNrYD6r/+f0= root@flux-69cd86ccb9-wdh8n
        ```

    - Ключ добавлен в Gitlab

    - Поместим manifest, описывающий namespace, в git repo microservices-demo в директорию `deploy/namespaces`

    - Новый namespace автоматически создался:

        ```bash
        $ kubectl get namespaces

        NAME                 STATUS   AGE
        default              Active   76m
        flux                 Active   22m
        istio-system         Active   76m
        kube-node-lease      Active   77m
        kube-public          Active   77m
        kube-system          Active   77m
        microservices-demo   Active   1s
        ```
    - Создана директория `deploy/release`, в которую помещен файл `frontend.yaml` с описанием конфигурации релиза.
    
        После git push в репозиторий microservices-demo появился HelmRelease

        ```bash

        NAME       RELEASE    PHASE       STATUS     MESSAGE                                                                       AGE
        frontend   frontend   Succeeded   deployed   Release was successful for Helm release 'frontend' in 'microservices-demo'.   29m
        ```

        ```bash
        helm list -n microservices-demo
        NAME            NAMESPACE               REVISION        UPDATED                                 STATUS          CHART           APP VERSION
        frontend        microservices-demo      1               2020-06-28 10:45:56.501893154 +0000 UTC deployed        frontend-0.21.0 1.16.0
        ```

    - Изменен тег в docker image

        Релиз автоматически обновился при этом в git добавился commit от Flux: [screenshot](/img/lesson-16-1.PNG)

    - Имя deployment изменено на frontend-heapster

        Логи Helm Operator

        ```bash
        kubectl logs helm-operator-745586d945-9gzzr | grep heapster
        ts=2020-06-28T10:50:00.907030422Z caller=helm.go:69 component=helm version=v3 info="Created a new Deployment called \"frontend-heapster\" in microservices-demo\n" targetNamespace=microservices-demo release=frontend
        ```
    - Добавлены манифесты HelmReleases для всех микросервисов HipsterShop

    - Микросервисы установлены в Kubernetes

        ```bash
        kubectl get helmrelease -n microservices-demo
        NAME                      RELEASE                   PHASE       STATUS     MESSAGE                                                                                      AGE
        adservice                 adservice                 Succeeded   deployed   Release was successful for Helm release 'adservice' in 'microservices-demo'.                 4m56s
        cartservice               cartservice               Succeeded   deployed   Release was successful for Helm release 'cartservice' in 'microservices-demo'.               4m56s
        checkoutservice           checkoutservice           Succeeded   deployed   Release was successful for Helm release 'checkoutservice' in 'microservices-demo'.           4m56s
        currencyservice           currencyservice           Succeeded   deployed   Release was successful for Helm release 'currencyservice' in 'microservices-demo'.           4m56s
        emailservice              emailservice              Succeeded   deployed   Release was successful for Helm release 'emailservice' in 'microservices-demo'.              4m56s
        frontend                  frontend                  Succeeded   deployed   Release was successful for Helm release 'frontend' in 'microservices-demo'.                  44m
        grafana-load-dashboards   grafana-load-dashboards   Succeeded   deployed   Release was successful for Helm release 'grafana-load-dashboards' in 'microservices-demo'.   4m56s
        loadgenerator             loadgenerator             Succeeded   deployed   Release was successful for Helm release 'loadgenerator' in 'microservices-demo'.             4m56s
        paymentservice            paymentservice            Succeeded   deployed   Release was successful for Helm release 'paymentservice' in 'microservices-demo'.            4m56s
        productcatalogservice     productcatalogservice     Succeeded   deployed   Release was successful for Helm release 'productcatalogservice' in 'microservices-demo'.     4m56s
        recommendationservice     recommendationservice     Succeeded   deployed   Release was successful for Helm release 'recommendationservice' in 'microservices-demo'.     4m56s
        shippingservice           shippingservice           Succeeded   deployed   Release was successful for Helm release 'shippingservice' in 'microservices-demo'.           4m56s
        ```

- Canary deployments с Flagger и Istio

    - Установка Istio.

        Кластер уже создан с Istio. Еще раз ставить его не надо.

    - Установка Flagger

        ```bash
        helm repo add flagger https://flagger.app
        kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml
        helm upgrade --install flagger flagger/flagger \
        --namespace=istio-system \
        --set crd.create=false \
        --set meshProvider=istio \
        --set metricsServer=http://prometheus:9090
        ```

    - Добавлен Istio sidecar injection  в namespace `microservices-demo.yaml`

        ```bash
        kubectl get ns microservices-demo --show-labels
        NAME                 STATUS   AGE   LABELS
        microservices-demo   Active   81m   istio-injection=enabled
        ```

    - Удаление Pod для добавление sidecar envoy

        ```bash
        kubectl delete pods --all -n microservices-demo
        ```

        Init container действительно добавился

        ```
        Init Containers:
        istio-init:
            Container ID:  docker://b64dcebd338ed3ced7a6de04366b24c7297a3357e81f22ba8c15a760492fafdf
            Image:         gke.gcr.io/istio/proxyv2:1.4.6-gke.0
            Image ID:      docker-pullable://gke.gcr.io/istio/proxyv2@sha256:618d0477e88004e0b599e455afd2080360507a7d37eca9c22904b3f6bb0fec36
        ```

    - Создана директория `deploy/istio`, где размещены `frontend-vs.yaml` и `frontend-gw.yaml`

        Созданный gateway

        ```bash
        $ kubectl get gateway -n microservices-demo
        NAME               AGE
        frontend-gateway   65m
        ```

    - External-ip сервиса

        ```bash
        $ kubectl get svc istio-ingressgateway -n istio-system
        NAME                   TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)                                                                                                                                      AGE
        istio-ingressgateway   LoadBalancer   10.0.10.246   34.83.98.133   15020:31614/TCP,80:32364/TCP,443:31853/TCP,31400:30077/TCP,15029:30770/TCP,15030:31477/TCP,15031:31934/TCP,15032:31524/TCP,15443:32566/TCP   3h11m
        ```

    - Файлы `frontend-vs.yaml` и `frontend-gw.yaml` перенесены в Helm Chart frontend
    
    - Добавим файл `canary.yaml` в Helm Chart frontend

    - Flagger успешно инициализировал canary ресурс frontend
    
        ```bash
        kubectl get canary -n microservices-demo
        NAME       STATUS         WEIGHT   LASTTRANSITIONTIME
        frontend   Initializing   0        2020-06-28T12:28:41Z
        ```

        Обновил pod, добавив ему к названию постфикс primary

        ```bash
        kubectl get pods -n microservices-demo -l app=frontend-primary
        NAME                               READY   STATUS    RESTARTS   AGE
        frontend-primary-b6cc9c4b8-tf5f6   2/2     Running   0          65s
        ```

    - В docker image frontend добавлен тег v0.0.3   

      После этого начала происходить выкатка canary релиза

      ```bash
      $ kubectl describe canary frontend -n microservices-demo

      Normal   Synced  3m41s                  flagger  New revision detected! Scaling up frontend.microservices-demo
      Normal   Synced  3m11s                  flagger  Starting canary analysis for frontend.microservices-demo
      Normal   Synced  3m11s                  flagger  Advance frontend.microservices-demo canary weight 5
      Warning  Synced  41s (x5 over 2m41s)    flagger  Prometheus query failed: running query failed: request failed: Get "http://prometheus:9090/api/v1/query?query=+sum%28+rate%28+istio_requests_total%7B+reporter%3D%22destination%22%2C+destination_workload_namespace%3D%22microservices-demo%22%2C+destination_workload%3D~%22frontend%22%2C+response_code%21~%225.%2A%22+%7D%5B30s%5D+%29+%29+%2F+sum%28+rate%28+istio_requests_total%7B+reporter%3D%22destination%22%2C+destination_workload_namespace%3D%22microservices-demo%22%2C+destination_workload%3D~%22frontend%22+%7D%5B30s%5D+%29+%29+%2A+100": dial tcp: lookup prometheus on 10.0.0.10:53: no such host
      Warning  Synced  11s                    flagger  Rolling back frontend.microservices-demo failed checks threshold reached 5
      Warning  Synced  11s                    flagger  Canary failed! Scaling down frontend.microservices-demo
      ```

      Произошла ошибка и откат helm chart.

      Надо поставить Istio с Prometheus. Затем добавить в Helm chart loadgenerator настройку ingress


      В итоге имеем

        ```bash
        $ kubectl describe canary frontend -n microservices-demo
        Normal   Synced  67s                   flagger  New revision detected! Scaling up frontend.microservices-demo
        Normal   Synced  37s                   flagger  New revision detected! Restarting analysis for frontend.microservices-demo
        Normal   Synced  7s                    flagger  Starting canary analysis for frontend.microservices-demo
        Normal   Synced  7s                    flagger  Advance frontend.microservices-demo canary weight 5
        Normal   Synced  9s                    flagger  Advance frontend.microservices-demo canary weight 10
        ...

        $ kubectl get canaries
        NAME       STATUS      WEIGHT   LASTTRANSITIONTIME
        frontend   Succeeded   0        2020-06-28T13:48:45Z

        ```

Кластер я выключил, потому что осталось очень мало денег.


