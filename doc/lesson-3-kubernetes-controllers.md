- Создан ReplicaSet frontend с replicas=3.

> почему обновление ReplicaSet не повлекло обновление
запущенных pod?

- ReplicaSet не проверяет соответствие запущенных Pod'ов шаблону. Ему важен только факт наличия Pod.

- Создан ReplicaSet pymentservice с replicaset=3.

- Создан Deployment paymentservice.

- Созданы Deployment с реализацией стратегии обновления Blue-Green и Reverse Rolling Update.

- Создан Deployment frontend с readinessProbe.

- Создан DaemonSet node-exporter, в котором выставлен toleration к запуску Pod на Kubernetes master.