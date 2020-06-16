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