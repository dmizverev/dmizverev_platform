# CSI. Обзор подсистем хранения данных в Kubernetes

## Обычное домашнее задание

- Установка CSI HostPath Driver

    [Репозиторий](https://github.com/kubernetes-csi/csi-driver-host-path)

    [Документация по установке](https://github.com/kubernetes-csi/csi-driver-host-path/blob/master/docs/deploy-1.17-and-later.md)

    - Установка CRD:

        ```bash
        #!/usr/bin/env bash

        # Change to the latest supported snapshotter version
        SNAPSHOTTER_VERSION=v2.0.1

        # Apply VolumeSnapshot CRDs
        kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/${SNAPSHOTTER_VERSION}/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
        kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/${SNAPSHOTTER_VERSION}/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
        kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/${SNAPSHOTTER_VERSION}/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

        # Create snapshot controller
        kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/${SNAPSHOTTER_VERSION}/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
        kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/${SNAPSHOTTER_VERSION}/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml 
        ```

        Проверка

        ```bash
        $ kubectl get volumesnapshotclasses.snapshot.storage.k8s.io 
        $ kubectl get volumesnapshots.snapshot.storage.k8s.io 
        $ kubectl get volumesnapshotcontents.snapshot.storage.k8s.io
        ```

        Искомые ресурсы не найдены, но нет ошибок `error: the server doesn't have a resource type "volumesnapshotclasses"`

        ```bash
        kubectl get pods --all-namespaces -o=jsonpath='{range .items[*]}{"\n"}{range .spec.containers[*]}{.image}{", "}{end}{end}' | grep snapshot-controller
        ```

        Ответ: `quay.io/k8scsi/snapshot-controller:v2.0.0-rc4,`

    - Установка драйвера

        Взять из [репозитория](https://github.com/kubernetes-csi/csi-driver-host-path.git) последнюю релизную версию.
        Выполнить скрипт  `deploy/kubernetes-1.17/deploy.sh`

        Смотрим, что он установил:
        ```bash
        kubectl get pods
        NAME                         READY   STATUS    RESTARTS   AGE
        csi-hostpath-attacher-0      1/1     Running   0          7m44s
        csi-hostpath-provisioner-0   1/1     Running   0          6m57s
        csi-hostpath-resizer-0       1/1     Running   0          6m44s
        csi-hostpath-snapshotter-0   1/1     Running   0          6m32s
        csi-hostpath-socat-0         1/1     Running   0          6m18s
        csi-hostpathplugin-0         3/3     Running   0          7m11s
        snapshot-controller-0        1/1     Running   0          22m

        ```

        Выполнение примера из документации

        ```bash
        $ for i in ./examples/csi-storageclass.yaml ./examples/csi-pvc.yaml ./examples/csi-app.yaml; do kubectl apply -f $i; done

        storageclass.storage.k8s.io/csi-hostpath-sc created
        persistentvolumeclaim/csi-pvc created
        pod/my-csi-app created
        ```

        ```bash
        $ kubectl get pv

        NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS      REASON   AGE
        pvc-4f02c603-4819-4cc9-9199-c711f4d3a2ed   1Gi        RWO            Delete           Bound    default/csi-pvc   csi-hostpath-sc            32m
        ```

        ```bash
        $ kubectl get pvc

        NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
        csi-pvc   Bound    pvc-4f02c603-4819-4cc9-9199-c711f4d3a2ed   1Gi        RWO            csi-hostpath-sc   33m
        ```

        ```bash
        $ kubectl describe pods/my-csi-app

        Name:         my-csi-app
        Namespace:    default
        Priority:     0
        Node:         gke-cluster-1-default-pool-f5aa82ff-80m8/10.132.0.9
        Start Time:   Fri, 24 Jul 2020 10:51:29 +0300
        Labels:       <none>
        Annotations:  Status:  Running
        IP:           10.60.1.9
        IPs:
        IP:  10.60.1.9
        Containers:
        my-frontend:
            Container ID:  docker://1f2b13c6b053eb87c56ceb3ea45b799f2fb1cd04ddc44847a024075ba5e65cdc
            Image:         busybox
            Image ID:      docker-pullable://busybox@sha256:2131f09e4044327fd101ca1fd4043e6f3ad921ae7ee901e9142e6e36b354a907
            Port:          <none>
            Host Port:     <none>
            Command:
            sleep
            1000000
            State:          Running
            Started:      Fri, 24 Jul 2020 10:51:39 +0300
            Ready:          True
            Restart Count:  0
            Environment:    <none>
            Mounts:
            /data from my-csi-volume (rw)
            /var/run/secrets/kubernetes.io/serviceaccount from default-token-b6drx (ro)
        Conditions:
        Type              Status
        Initialized       True 
        Ready             True 
        ContainersReady   True 
        PodScheduled      True 
        Volumes:
        my-csi-volume:
            Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
            ClaimName:  csi-pvc
            ReadOnly:   false
        default-token-b6drx:
            Type:        Secret (a volume populated by a Secret)
            SecretName:  default-token-b6drx
            Optional:    false
        QoS Class:       BestEffort
        Node-Selectors:  <none>
        Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                        node.kubernetes.io/unreachable:NoExecute for 300s
        Events:
        Type    Reason                  Age   From                                               Message
        ----    ------                  ----  ----                                               -------
        Normal  Scheduled               33m   default-scheduler                                  Successfully assigned default/my-csi-app to gke-cluster-1-default-pool-f5aa82ff-80m8
        Normal  SuccessfulAttachVolume  33m   attachdetach-controller                            AttachVolume.Attach succeeded for volume "pvc-4f02c603-4819-4cc9-9199-c711f4d3a2ed"
        Normal  Pulling                 33m   kubelet, gke-cluster-1-default-pool-f5aa82ff-80m8  Pulling image "busybox"
        Normal  Pulled                  33m   kubelet, gke-cluster-1-default-pool-f5aa82ff-80m8  Successfully pulled image "busybox"
        Normal  Created                 33m   kubelet, gke-cluster-1-default-pool-f5aa82ff-80m8  Created container my-frontend
        Normal  Started                 33m   kubelet, gke-cluster-1-default-pool-f5aa82ff-80m8  Started container my-frontend
        ```

        Persistent volume замонтирован в `/data`

    - Проверка работы драйвера

        Создание файла внутри `/data`

        ```bash
        kubectl exec -it my-csi-app -- /bin/sh
        / # touch /data/hello-world
        / # exit
        ```

        Проверка наличия созданного файла в контейнере HostPath

        ```bash
        $ kubectl exec -it $(kubectl get pods --selector app=csi-hostpathplugin -o jsonpath='{.items[*].metadata.name}') -c hostpath -- /bin/sh

        / # find / -name hello-world
        /var/lib/kubelet/pods/575082df-7db5-4c48-969a-f177679c68e5/volumes/kubernetes.io~csi/pvc-4f02c603-4819-4cc9-9199-c711f4d3a2ed/mount/hello-world
        /csi-data-dir/7b7f4f3d-cd82-11ea-a56a-8af4b4b9b5b4/hello-world
        ```

        ```bash
        $ kubectl describe volumeattachment

        Name:         csi-df70ec31d49641fb76371853191401c0cd20709e9bcb95e31a626f8106270ac5
        Namespace:    
        Labels:       <none>
        Annotations:  <none>
        API Version:  storage.k8s.io/v1
        Kind:         VolumeAttachment
        Metadata:
        Creation Timestamp:  2020-07-24T07:51:29Z
        Resource Version:    12542
        Self Link:           /apis/storage.k8s.io/v1/volumeattachments/csi-df70ec31d49641fb76371853191401c0cd20709e9bcb95e31a626f8106270ac5
        UID:                 88199a89-83f3-4ba4-844e-9100a12e91d4
        Spec:
        Attacher:   hostpath.csi.k8s.io
        Node Name:  gke-cluster-1-default-pool-f5aa82ff-80m8
        Source:
            Persistent Volume Name:  pvc-4f02c603-4819-4cc9-9199-c711f4d3a2ed
        Status:
        Attached:  true
        Events:      <none>
        ```

- Выполнение домашнего задания

    Созданы StorageClass, PVC и PV.
    Создан Pod, который монтирует volume `/data`

    ```bash
    kubectl apply -f hw/storageClass.yaml 
    kubectl apply -f hw/pvc.yaml 
    kubectl apply -f hw/pod.yaml
    ```

    Проверка созданного

    ```bash
    $ kubectl get pv
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS      REASON   AGE
    pvc-4f02c603-4819-4cc9-9199-c711f4d3a2ed   1Gi        RWO            Delete           Bound    default/csi-pvc   csi-hostpath-sc            71m
    
    $ kubectl get pvc
    NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
    csi-pvc       Bound    pvc-4f02c603-4819-4cc9-9199-c711f4d3a2ed   1Gi        RWO            csi-hostpath-sc   77m
    storage-pvc   Bound    pvc-2f76ecbc-1290-49e6-97a6-d81aac9dc3b6   1Gi        RWO            storage-class     4m35s                                                                       storageclass      63s
    
    $ kubectl describe pods/storage-pod
    Name:         storage-pod
    Namespace:    default
    Priority:     0
    Node:         gke-cluster-1-default-pool-f5aa82ff-80m8/10.132.0.9
    Start Time:   Fri, 24 Jul 2020 12:08:48 +0300
    Labels:       <none>
    Annotations:  Status:  Running
    IP:           10.60.1.12
    IPs:
    IP:  10.60.1.12
    Containers:
    app:
        Container ID:  docker://db50097c44657f312ab99295509f0fe73a4babe48790b4676e11a82b5366c206
        Image:         bash
        Image ID:      docker-pullable://bash@sha256:6676e4f947008db9a6424ae5f2ac569c14d10a1ab5318c0998fd43c80ac3e296
        Port:          <none>
        Host Port:     <none>
        Command:
        sleep
        1000000
        State:          Running
        Started:      Fri, 24 Jul 2020 12:08:49 +0300
        Ready:          True
        Restart Count:  0
        Environment:    <none>
        Mounts:
        /data from data-volume (rw)
        /var/run/secrets/kubernetes.io/serviceaccount from default-token-b6drx (ro)
    Conditions:
    Type              Status
    Initialized       True 
    Ready             True 
    ContainersReady   True 
    PodScheduled      True 
    Volumes:
    data-volume:
        Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
        ClaimName:  storage-pvc
        ReadOnly:   false
    default-token-b6drx:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-b6drx
        Optional:    false
    QoS Class:       BestEffort
    Node-Selectors:  <none>
    Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                    node.kubernetes.io/unreachable:NoExecute for 300s
    Events:
    Type    Reason     Age   From                                               Message
    ----    ------     ----  ----                                               -------
    Normal  Scheduled  108s  default-scheduler                                  Successfully assigned default/storage-pod to gke-cluster-1-default-pool-f5aa82ff-80m8
    Normal  Pulling    107s  kubelet, gke-cluster-1-default-pool-f5aa82ff-80m8  Pulling image "bash"
    Normal  Pulled     107s  kubelet, gke-cluster-1-default-pool-f5aa82ff-80m8  Successfully pulled image "bash"
    Normal  Created    107s  kubelet, gke-cluster-1-default-pool-f5aa82ff-80m8  Created container app
    Normal  Started    107s  kubelet, gke-cluster-1-default-pool-f5aa82ff-80m8  Started container app
    ```
- Проверка snapshot

    Создан `VolumeSnapshotClass`

    ```bash
    kubectl apply -f hw/volumeSnapshotClass.yaml
    ```

    Создание файлов

    ```bash
    kubectl exec -it storage-pod bash

    bash-5.0# touch /data/tst1
    bash-5.0# touch /data/tst2
    bash-5.0# touch /data/tst3
    bash-5.0# ls -la /data/
    total 8
    drwxr-xr-x    2 root     root          4096 Jul 24 09:17 .
    drwxr-xr-x    1 root     root          4096 Jul 24 09:08 ..
    -rw-r--r--    1 root     root             0 Jul 24 09:17 tst1
    -rw-r--r--    1 root     root             0 Jul 24 09:17 tst2
    -rw-r--r--    1 root     root             0 Jul 24 09:17 tst3
    ```

    Создание snapshot

    ```bash
    kubectl apply -f hw/snapshotVolume.yaml
    ```

    ```bash
    kubectl describe volumesnapshot

    Name:         volumesnapshot
    Namespace:    default
    Labels:       <none>
    Annotations:  API Version:  snapshot.storage.k8s.io/v1beta1
    Kind:         VolumeSnapshot
    Metadata:
    Creation Timestamp:  2020-07-24T09:20:05Z
    Finalizers:
        snapshot.storage.kubernetes.io/volumesnapshot-as-source-protection
        snapshot.storage.kubernetes.io/volumesnapshot-bound-protection
    Generation:        1
    Resource Version:  44109
    Self Link:         /apis/snapshot.storage.k8s.io/v1beta1/namespaces/default/volumesnapshots/volumesnapshot
    UID:               d4f95f45-3394-4442-9c77-7b2cb51f1a24
    Spec:
    Source:
        Persistent Volume Claim Name:  storage-pvc
    Volume Snapshot Class Name:      snapshotclass
    Status:
    Bound Volume Snapshot Content Name:  snapcontent-d4f95f45-3394-4442-9c77-7b2cb51f1a24
    Creation Time:                       2020-07-24T09:20:05Z
    Ready To Use:                        true
    Restore Size:                        1Gi
    Events:                                <none>
    ```

    Добавим еще файлов, чтобы потом проверить, что их не будет после восстановления snapshot

    ```bash
    kubectl exec -it storage-pod bash

    bash-5.0# touch /data/aftersnapshot1
    bash-5.0# touch /data/aftersnapshot2
    
    bash-5.0# ls -la /data
    total 8
    drwxr-xr-x    2 root     root          4096 Jul 24 09:24 .
    drwxr-xr-x    1 root     root          4096 Jul 24 09:08 ..
    -rw-r--r--    1 root     root             0 Jul 24 09:24 aftersnapshot1
    -rw-r--r--    1 root     root             0 Jul 24 09:24 aftersnapshot2
    -rw-r--r--    1 root     root             0 Jul 24 09:17 tst1
    -rw-r--r--    1 root     root             0 Jul 24 09:17 tst2
    -rw-r--r--    1 root     root             0 Jul 24 09:17 tst3
    ```

    Создание PVC из snapshot

    ```bash
    kubectl apply -f hw/pvcFromSnapshot.yaml
    ```

    Проверка

    ```bash
    $ kubectl get pvc
    NAME          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
    csi-pvc       Bound    pvc-4f02c603-4819-4cc9-9199-c711f4d3a2ed   1Gi        RWO            csi-hostpath-sc   98m
    pvc-resored   Bound    pvc-81d1c652-6ba6-4901-8d50-29fc430c5b4c   1Gi        RWO            storage-class     33s
    storage-pvc   Bound    pvc-2f76ecbc-1290-49e6-97a6-d81aac9dc3b6   1Gi        RWO            storage-class     25m


    kubectl get pv
    NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                 STORAGECLASS      REASON   AGE
    pvc-2f76ecbc-1290-49e6-97a6-d81aac9dc3b6   1Gi        RWO            Delete           Terminating   default/storage-pvc   storage-class              26m
    pvc-4f02c603-4819-4cc9-9199-c711f4d3a2ed   1Gi        RWO            Delete           Terminating   default/csi-pvc       csi-hostpath-sc            99m
    pvc-81d1c652-6ba6-4901-8d50-29fc430c5b4c   1Gi        RWO            Delete           Bound         default/pvc-resored   storage-class              68s
    ```

    Артефакты *-restored есть.

    Восстановление volume из snapshot.

    ```bash
    kubectl apply -f hw/podRestore.yaml
    ```

    Проверим, что из snapshot восстановились те данные, что мы ранее записали и не восстановились добавочные.

    ```bash
    $ kubectl exec -it storage-pod-restored -- bash

    bash-5.0# ls -la /data
    total 8
    drwxr-xr-x    2 root     root          4096 Jul 24 09:29 .
    drwxr-xr-x    1 root     root          4096 Jul 24 09:35 ..
    -rw-r--r--    1 root     root             0 Jul 24 09:17 tst1
    -rw-r--r--    1 root     root             0 Jul 24 09:17 tst2
    -rw-r--r--    1 root     root             0 Jul 24 09:17 tst3
    ```

