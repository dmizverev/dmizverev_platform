- Изменен кластер GKE в связи с требованиями к заданию:
  - добавлен еще один пул
  - выключен Stackdriver
  - добавлен taint
    ```bash
    kubectl taint nodes gke-cluster-3-infra-pool-f3c93b44-6kxp node-role=infra:NoSchedule
    kubectl taint nodes gke-cluster-3-infra-pool-f3c93b44-972j node-role=infra:NoSchedule
    kubectl taint nodes gke-cluster-3-infra-pool-f3c93b44-hx4s node-role=infra:NoSchedule
    ```

- Установлен HipsterShop
  ```bash
  kubectl create ns microservices-demo
  kubectl apply -f https://raw.githubusercontent.com/express42/otus-platform-snippets/master/Module-02/Logging/microservices-demo-without-resources.yaml -n microservices-demo
  ```
  
- Установка EFK
  ```bash
  helm repo add elastic https://helm.elastic.co
  helm repo update
  kubectl create ns observability
  ```
  
  Создан файл, который указывает elasticsearch запуститься на нодах из infra-pool: `kubernetes-logging\elasticsearch.values.yaml`
  ```bash
  helm upgrade --install elasticsearch elastic/elasticsearch --namespace observability -f elasticsearch.values.yaml
  ```
  
  Установлен nginx-ingress c параметрами из файла `kubernetes-logging\nginx-ingress.values.yaml`
  
  ```bash
  kubectl create ns nginx-ingress
  helm upgrade --install nginx-ingress stable/nginx-ingress --namespace=nginx-ingress -f nginx-ingress.values.yaml
  ```
  
  Установлена Kibana c параметрами из файла `kubernetes-logging\kibana.values.yaml`
  
  ```bash
  helm upgrade --install kibana elastic/kibana --namespace observability -f kibana.values.yaml --version 7.3.0
  ```
  
  Адрес Kibana: `http://kibana.34.66.61.20.xip.io`
  
  Установлен Fluentbit
  
  ```bash
  helm upgrade --install fluent-bit stable/fluent-bit --namespace observability -f fluent-bit.values.yaml
  ```
  
- Установлен Prometheus Operator c параметрами из файла `prometheus-operator.values.yaml`
  ```bash
  helm upgrade --install prometheus-operator stable/prometheus-operator --namespace=observability --values=prometheus-operator.values.yaml
  ```
  
  Адрес Grafana: `http://grafana.34.66.61.20.xip.io`
  
- Установлен elasticsearch-exporter с параметрами из файла `kubernetes-logging\elasticsearch-exporter.values.yaml`
  ```bash
  helm upgrade --install elasticsearch-exporter stable/elasticsearch-exporter --wait --namespace=observability --values=elasticsearch-exporter.values.yaml
  ```
  
- Импорт Grafana Dashboard [Elasticsearch](https://grafana.com/grafana/dashboards/4358)

- Nginx-ingress перенастроен для более точного отображения логов по полям

- В Kibana создан dashboard отображающий сообщения с различными статусами: 
  `http://kibana.34.66.61.20.xip.io/app/kibana#/dashboard/1eaa0870-a730-11ea-87b6-0b3eac1b87dd`
  
  Он экспортирован в файл `kubernetes-logging\export.ndjson`
  
- Установка Loki
  
  В настройку prometheus-operator.values.yaml добавлен additionalDataSources Loki
  
  ```bash
  helm repo add loki https://grafana.github.io/loki/charts
  helm repo update
  helm upgrade --install loki loki/loki-stack --wait --namespace=observability --values=loki.values.yaml
  ```
  
  В настройку nginx-ingress добавлен serviceMonitor, параметр `metrics`
  
- Создан grafana dashboard c отображением логов и визуализацией запросов к nginx: `kubernetes-logging\nginx-ingress.json`

  `http://grafana.34.66.61.20.xip.io/d/y4dyGMrZz/nginx-ingress?orgId=1`
  

  