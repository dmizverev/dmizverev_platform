# dmizverev_platform
Dmitriy Zverev Platform repository
dmizverev@yandex.ru

# Урок №2

> Разберитесь почему все pod в namespace kube-system
восстановились после удаления. Укажите причину в описании PR

Предположу, что: 
- kube-apiserver - это static Pod. За его восстановлением следит непосредственно kubelet.
См. https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/
- За восстановление cor-edns, kube-dns отвечает ReplicaSetController.

> Для выполнения домашней работы необходимо создать
> Dockerfile, в котором будет описан образ, запускающий web-сервер на порту 8000

Создан docker image с python web-server `http.server`. 
Docker image размещен в registry `docker.io/dmizverev/otus-frontend:1.0.0`.

> Напишем манифест web-pod.yaml для создания pod web c
> меткой app со значением web, содержащего один контейнер с
> названием web.

Выполнено.

> Добавим в наш pod init-container, генерирующий страницу
> index.html.

Выполнено.

> Для того, чтобы файлы, созданные в init контейнере, были
> доступны основному контейнеру в pod нам понадобится
> использовать volume типа emptyDir

Выполнено.

> Склонируйте и соберите собственный образ для
frontend (используйте готовый Dockerfile)
Поместите собранный образ на Docker Hub

Выполнено

> генерация манифестов средствами kubectl

Выполнено. Файл: frontend-pod.yaml

> Выясните причину, по которой pod frontend находится в статусе
Error

Не заданы переменные окружения

> Создайте новый манифест frontend-pod-healthy.yaml. При его
применении ошибка должна исчезнуть

Выполнено.

# Урок №3

Создан ReplicaSet frontend с replicas=3.

> почему обновление ReplicaSet не повлекло обновление
запущенных pod?

ReplicaSet не проверяет соответствие запущенных Pod'ов шаблону. Ему важен только факт наличия Pod.

Создан ReplicaSet pymentservice с replicaset=3.

Создан Deployment paymentservice.

Созданы Deployment с реализацией стратегии обновления Blue-Green и Reverse Rolling Update.

Создан Deployment frontend с readinessProbe.

Создан DaemonSet node-exporter, в котором выставлен toleration к запуску Pod на Kubernetes master.

# Урок №4

Создан ServiceAccount bob. Создан ClusterRoleBinding adminbob, где для bob выдана роль admin.

Создан ServiceAccount dave.

Создан Namespace prometheus. В нем создан ServiceAccount carol.
Создана clusterrole с возможностью для Pod выполнять get list watch.
Эта роль выдана на namespace prometheus.

Создан Namespace dev. В нем создан ServiceAccount jave с ролью admin и ServiceAccount ken
с ролью view.

# Урок №6 Volumes, Storages, StatefulSet

Создан StatefulSet с Pod minio. Environment variables передаются в container из
secret minio-secret. Для minio создан Headless Service. 
Statefulset хранит данные в каталоге `data`.
Проверка того, что данные там есть:
```bash
# in container
> ls -la /data
total 24
drwxr-xr-x    4 root     root          4096 May 22 08:35 .
drwxr-xr-x    1 root     root            64 May 22 08:35 ..
drwxr-xr-x    5 root     root          4096 May 22 08:34 .minio.sys
drwx------    2 root     root         16384 May 22 08:34 lost+found
```
Проверка создаваемых ресурсов Kubernetes
```bash
> kubectl get statefulset
NAME                                                   READY   AGE
alertmanager-prometheus-operator-158141-alertmanager   1/1     100d
minio                                                  1/1     6m9s
prometheus-prometheus-operator-158141-prometheus       1/1     100d

> kubectl get pods minio-0
NAME      READY   STATUS    RESTARTS   AGE
minio-0   1/1     Running   0          7m37s

> kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
csi-pvc-cinder   Bound    pvc-eb8f17da-6b0b-4a3e-b846-e869c24a564a   1Gi        RWO            csi-cinder     98d
data-minio-0     Bound    pvc-a9769bff-64c9-41f4-ac2d-ce783bf9c9a7   1Gi        RWO            csi-cinder     10m

> kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                    STORAGECLASS   REASON   AGE
pvc-a9769bff-64c9-41f4-ac2d-ce783bf9c9a7   1Gi        RWO            Delete           Bound    default/data-minio-0     csi-cinder              10m
pvc-eb8f17da-6b0b-4a3e-b846-e869c24a564a   1Gi        RWO            Delete           Bound    default/csi-pvc-cinder   csi-cinder              98d

> kubectl describe pvc data-minio-0
Name:          data-minio-0
Namespace:     default
StorageClass:  csi-cinder
Status:        Bound
Volume:        pvc-a9769bff-64c9-41f4-ac2d-ce783bf9c9a7
Labels:        app=minio
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
               volume.beta.kubernetes.io/storage-provisioner: cinder.csi.openstack.org
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      1Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Mounted By:    minio-0
Events:
  Type    Reason                 Age                From                                                                                         Message
  ----    ------                 ----               ----                                                                                         -------
  Normal  ExternalProvisioning   15m (x2 over 15m)  persistentvolume-controller                                                                  waiting for a volume to be created, either
 by external provisioner "cinder.csi.openstack.org" or manually created by system administrator
  Normal  Provisioning           8m43s              cinder.csi.openstack.org_csi-cinder-controllerplugin-0_b9cf93ad-4928-4011-9173-39c6bcc6ecbc  External provisioner is provisioning volum
e for claim "default/data-minio-0"
  Normal  ProvisioningSucceeded  8m42s              cinder.csi.openstack.org_csi-cinder-controllerplugin-0_b9cf93ad-4928-4011-9173-39c6bcc6ecbc  Successfully provisioned volume pvc-a9769b
ff-64c9-41f4-ac2d-ce783bf9c9a7

```