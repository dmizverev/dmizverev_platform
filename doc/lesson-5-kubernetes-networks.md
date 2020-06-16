- В Pod web-pod.yaml добавлены Liveness и Readiness probes
- Создан Deployment и Service с типом ClusterIP
- Создан Service с типом ClusterIP с типом LoadBalancer
- Создан headless Service web-svc
- Создан Ingress перенаправляющий запрос с `/web` на headless Service web-svc по порту 8000
  
  Использовать https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml