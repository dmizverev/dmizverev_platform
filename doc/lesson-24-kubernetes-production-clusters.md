# Подходы к развертыванию и обновлению production-grade кластера

В этом ДЗ через kubeadm мы поднимем кластер версии 1.17 и обновим его

## Подготовка нод

- Создание нод в Google cloud

    ```shell 
    gcloud compute instances create master --image-family ubuntu-minimal-1804-lts --image-project ubuntu-os-cloud --machine-type=n1-standard-2 

    gcloud compute instances create node-1 --image-family ubuntu-minimal-1804-lts --image-project ubuntu-os-cloud --machine-type=n1-standard-1 

    gcloud compute instances create node-2 --image-family ubuntu-minimal-1804-lts --image-project ubuntu-os-cloud --machine-type=n1-standard-1 

    gcloud compute instances create node-3 --image-family ubuntu-minimal-1804-lts --image-project ubuntu-os-cloud --machine-type=n1-standard-1
    ```

    ```text
    Created [https://www.googleapis.com/compute/v1/projects/otus1-278509/zones/europe-west2-a/instances/master].
    NAME    ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
    master  europe-west2-a  n1-standard-2               10.154.0.2   35.246.31.109  RUNNING

    Created [https://www.googleapis.com/compute/v1/projects/otus1-278509/zones/europe-west2-a/instances/node-1].
    NAME    ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
    node-1  europe-west2-a  n1-standard-1               10.154.0.3   35.246.29.111  RUNNING

    Created [https://www.googleapis.com/compute/v1/projects/otus1-278509/zones/europe-west2-a/instances/node-2].
    NAME    ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP   STATUS
    node-2  europe-west2-a  n1-standard-1               10.154.0.4   34.89.62.194  RUNNING

    Created [https://www.googleapis.com/compute/v1/projects/otus1-278509/zones/europe-west2-a/instances/node-3].
    NAME    ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
    node-3  europe-west2-a  n1-standard-1               10.154.0.5   34.105.198.145  RUNNING
    ```

- Отключение на машинах swap

    ```shell
    $ gcloud beta compute ssh --zone "europe-west2-a" "master" --project "otus1-278509"
    $ swapoff -a

    $ gcloud beta compute ssh --zone "europe-west2-a" "node-1" --project "otus1-278509"
    $ swapoff -a

    $ gcloud beta compute ssh --zone "europe-west2-a" "node-2" --project "otus1-278509"
    $ swapoff -a

    $ gcloud beta compute ssh --zone "europe-west2-a" "node-3" --project "otus1-278509"
    $ swapoff -a
    ```

- Включение маршрутизации на всех нодах

    Выполнить на всех нодах от имени root

    ```shell
    cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    EOF

    sysctl --system
    ```

- Установка docker на всех нодах

    ```shell
    apt-get update && apt-get install -y \
    apt-transport-https ca-certificates curl software-properties-common gnupg2 && curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add - && add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" && apt-get update && apt-get install -y containerd.io=1.2.13-1 docker-ce=5:19.03.8~3-0~ubuntu-$(lsb_release -cs) docker-ce-cli=5:19.03.8~3-0~ubuntu-$(lsb_release -cs)

     cat > /etc/docker/daemon.json <<EOF
    {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
            "max-size": "100m"
        },
        "storage-driver": "overlay2"
    }
    EOF

    mkdir -p /etc/systemd/system/docker.service.d

    systemctl daemon-reload && systemctl restart docker
    ```

- Установка kubeadm, kubectl, kubelet на всех нодах

    Выполнить на всех нодах:

    ```shell
    apt-get update && apt-get install -y apt-transport-https curl && curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - 

    cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF

    apt-get update && apt-get install -y kubelet=1.17.4-00 kubeadm=1.17.4-00 kubectl=1.17.4-00
    ```

## Создание кластера

- Создание и настройка master

    ```shell
    $ kubeadm init --pod-network-cidr=192.168.0.0/24

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 10.154.0.2:6443 --token xar7d6.t0wvhsrp24t4elbx \
        --discovery-token-ca-cert-hash sha256:14360742ccaa7fa79de8f95db7bc0bce816df884278cf775fa35911748ef989f 
    ```

- Копирование конфига kubectl

    ```shell
    $ mkdir -p $HOME/.kube
    $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    $ kubectl get nodes 
    ```

- Установка сетевого плагина calico

    ```shell
    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
    ```

- Подключение worker нод

    ```shell
    $ kubeadm token list

    TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
    xar7d6.t0wvhsrp24t4elbx   23h         2020-08-05T08:31:56Z   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
    ```

    ```shell
    $ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

    14360742ccaa7fa79de8f95db7bc0bce816df884278cf775fa35911748ef989f
    ```

    На worker нодах выполнить:

    ```shell
    $ kubeadm join 10.154.0.2:6443 --token xar7d6.t0wvhsrp24t4elbx --discovery-token-ca-cert-hash sha256:14360742ccaa7fa79de8f95db7bc0bce816df884278cf775fa35911748ef989f
    ```

    На master:
    ```shell
    $ kubectl get nodes

    NAME     STATUS   ROLES    AGE     VERSION
    master   Ready    master   9m56s   v1.17.4
    node-1   Ready    <none>   65s     v1.17.4
    node-2   Ready    <none>   54s     v1.17.4
    node-3   Ready    <none>   43s     v1.17.4
    ```

## Запуск нагрузки

- Создание deployment

    На master:
    ```shell
    cat <<EOF > deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: nginx-deployment
    spec:
    selector:
        matchLabels:
        app: nginx
    replicas: 4
    template:
        metadata:
        labels:
            app: nginx
        spec:
        containers:
        - name: nginx
            image: nginx:1.17.2
            ports:
            - containerPort: 80
    EOF
    ```

    ```shell
    $ kubectl apply -f deployment.yaml
    
    deployment.apps/nginx-deployment created
    ```

    ```shell
    $ kubectl get pods

    NAME                               READY   STATUS    RESTARTS   AGE
    nginx-deployment-c8fd555cc-8jtpn   1/1     Running   0          24s
    nginx-deployment-c8fd555cc-tkfz7   1/1     Running   0          24s
    nginx-deployment-c8fd555cc-wmfvh   1/1     Running   0          24s
    nginx-deployment-c8fd555cc-zc76p   1/1     Running   0          24s
    ```

## Обновление кластера

Так как кластер мы разворачивали с помощью kubeadm, то и производить обновление будем с помощью него. Обновлять ноды будем по очереди.

- Обновление master

    Обновление пакетов
    ```shell
    sudo su - 
    apt-get update && apt-get install -y kubeadm=1.18.0-00 kubelet=1.18.0-00 kubectl=1.18.0-00
    ```
    Проверка

    ```shell
    $ kubectl get nodes

    NAME     STATUS   ROLES    AGE     VERSION
    master   Ready    master   18m     v1.18.0
    node-1   Ready    <none>   10m     v1.17.4
    node-2   Ready    <none>   9m50s   v1.17.4
    node-3   Ready    <none>   9m39s   v1.17.4
    ```

    ```shell
    $ kubeadm version

    kubeadm version: &version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:56:30Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}
    ```

    ```shell
    $ kubelet --version

    Kubernetes v1.18.0
    ```

    ```shell
    $ kubectl version

    Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:58:59Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}
    Server Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.9", GitCommit:"4fb7ed12476d57b8437ada90b4f93b17ffaeed99", GitTreeState:"clean", BuildDate:"2020-07-15T16:10:45Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
    ```

    ```shell
    $ kubectl describe pod kube-apiserver-master -n kube-system

    Name:                 kube-apiserver-master
    Namespace:            kube-system
    Priority:             2000000000
    Priority Class Name:  system-cluster-critical
    Node:                 master/10.154.0.2
    Start Time:           Tue, 04 Aug 2020 08:31:58 +0000
    Labels:               component=kube-apiserver
                        tier=control-plane
    Annotations:          kubernetes.io/config.hash: 81cdabfc720501d9e81b425b15921b4d
                        kubernetes.io/config.mirror: 81cdabfc720501d9e81b425b15921b4d
                        kubernetes.io/config.seen: 2020-08-04T08:49:21.008357855Z
                        kubernetes.io/config.source: file
    Status:               Running
    IP:                   10.154.0.2
    IPs:
    IP:           10.154.0.2
    Controlled By:  Node/master
    Containers:
    kube-apiserver:
        Container ID:  docker://aad4dbb2afdcb755bc1f917ebbad122ec0aeaa182b27521633dea1ab9330fb24
        Image:         k8s.gcr.io/kube-apiserver:v1.17.9
        Image ID:      docker-pullable://k8s.gcr.io/kube-apiserver@sha256:9d9f9b92e7c88b74618ee6e063886ad6aedd9c144c41702f4eded0f01d221536
        Port:          <none>
        Host Port:     <none>
        Command:
        kube-apiserver
        --advertise-address=10.154.0.2
        --allow-privileged=true
        --authorization-mode=Node,RBAC
        --client-ca-file=/etc/kubernetes/pki/ca.crt
        --enable-admission-plugins=NodeRestriction
        --enable-bootstrap-token-auth=true
        --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
        --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
        --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
        --etcd-servers=https://127.0.0.1:2379
        --insecure-port=0
        --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
        --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
        --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
        --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
        --requestheader-allowed-names=front-proxy-client
        --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
        --requestheader-extra-headers-prefix=X-Remote-Extra-
        --requestheader-group-headers=X-Remote-Group
        --requestheader-username-headers=X-Remote-User
        --secure-port=6443
        --service-account-key-file=/etc/kubernetes/pki/sa.pub
        --service-cluster-ip-range=10.96.0.0/12
        --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
        --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
        State:          Running
        Started:      Tue, 04 Aug 2020 08:49:25 +0000
        Ready:          True
        Restart Count:  0
        Requests:
        cpu:        250m
        Liveness:     http-get https://10.154.0.2:6443/healthz delay=15s timeout=15s period=10s #success=1 #failure=8
        Environment:  <none>
        Mounts:
        /etc/ca-certificates from etc-ca-certificates (ro)
        /etc/kubernetes/pki from k8s-certs (ro)
        /etc/ssl/certs from ca-certs (ro)
        /usr/local/share/ca-certificates from usr-local-share-ca-certificates (ro)
        /usr/share/ca-certificates from usr-share-ca-certificates (ro)
    Conditions:
    Type              Status
    Initialized       True 
    Ready             True 
    ContainersReady   True 
    PodScheduled      True 
    Volumes:
    ca-certs:
        Type:          HostPath (bare host directory volume)
        Path:          /etc/ssl/certs
        HostPathType:  DirectoryOrCreate
    etc-ca-certificates:
        Type:          HostPath (bare host directory volume)
        Path:          /etc/ca-certificates
        HostPathType:  DirectoryOrCreate
    k8s-certs:
        Type:          HostPath (bare host directory volume)
        Path:          /etc/kubernetes/pki
        HostPathType:  DirectoryOrCreate
    usr-local-share-ca-certificates:
        Type:          HostPath (bare host directory volume)
        Path:          /usr/local/share/ca-certificates
        HostPathType:  DirectoryOrCreate
    usr-share-ca-certificates:
        Type:          HostPath (bare host directory volume)
        Path:          /usr/share/ca-certificates
        HostPathType:  DirectoryOrCreate
    QoS Class:         Burstable
    Node-Selectors:    <none>
    Tolerations:       :NoExecute
    Events:
    Type    Reason   Age    From             Message
    ----    ------   ----   ----             -------
    Normal  Pulled   4m11s  kubelet, master  Container image "k8s.gcr.io/kube-apiserver:v1.17.9" already present on machine
    Normal  Created  4m11s  kubelet, master  Created container kube-a
    ```

- Обновление остальных компонентов кластера

    Просмотр изменений, которые собирает сделать kubeadm
    ```shell
    $ sudo su -
    # kubeadm upgrade plan

    [upgrade/config] Making sure the configuration is correct:
    [upgrade/config] Reading configuration from the cluster...
    [upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
    [preflight] Running pre-flight checks.
    [upgrade] Running cluster health checks
    [upgrade] Fetching available versions to upgrade to
    [upgrade/versions] Cluster version: v1.17.9
    [upgrade/versions] kubeadm version: v1.18.0
    [upgrade/versions] Latest stable version: v1.18.6
    [upgrade/versions] Latest stable version: v1.18.6
    [upgrade/versions] Latest version in the v1.17 series: v1.17.9
    [upgrade/versions] Latest version in the v1.17 series: v1.17.9

    Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
    COMPONENT   CURRENT       AVAILABLE
    Kubelet     3 x v1.17.4   v1.18.6
                1 x v1.18.0   v1.18.6

    Upgrade to the latest stable version:

    COMPONENT            CURRENT   AVAILABLE
    API Server           v1.17.9   v1.18.6
    Controller Manager   v1.17.9   v1.18.6
    Scheduler            v1.17.9   v1.18.6
    Kube Proxy           v1.17.9   v1.18.6
    CoreDNS              1.6.5     1.6.7
    Etcd                 3.4.3     3.4.3-0

    You can now apply the upgrade by executing the following command:

            kubeadm upgrade apply v1.18.6

    Note: Before you can perform this upgrade, you have to update kubeadm to v1.18.6.

    _____________________________________________________________________
    ```

    Обновиться сразу на 1.18.6 не получится. Сначало надо обновиться на 1.18.0.

    Обновление:
    ```shell
    # kubeadm upgrade apply v1.18.0

    [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.18.0". Enjoy!
    ```

    Проверка
    ```shell
    $ kubectl get nodes && kubeadm version && kubelet --version && kubectl version && kubectl describe pod kube-apiserver-master -n kube-system

    NAME     STATUS   ROLES    AGE   VERSION
    master   Ready    master   32m   v1.18.0
    node-1   Ready    <none>   23m   v1.17.4
    node-2   Ready    <none>   23m   v1.17.4
    node-3   Ready    <none>   23m   v1.17.4

    kubeadm version: &version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:56:30Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}
    Kubernetes v1.18.0
    Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:58:59Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}
    Server Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:50:46Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}

    Name:                 kube-apiserver-master
    Namespace:            kube-system
    Priority:             2000000000
    Priority Class Name:  system-cluster-critical
    Node:                 master/10.154.0.2
    Start Time:           Tue, 04 Aug 2020 08:31:58 +0000
    Labels:               component=kube-apiserver
                        tier=control-plane
    Annotations:          kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 10.154.0.2:6443
                        kubernetes.io/config.hash: 2a815550b4b9f327ebff5e11b834ca19
                        kubernetes.io/config.mirror: 2a815550b4b9f327ebff5e11b834ca19
                        kubernetes.io/config.seen: 2020-08-04T09:02:49.614527748Z
                        kubernetes.io/config.source: file
    Status:               Running
    IP:                   10.154.0.2
    IPs:
    IP:           10.154.0.2
    Controlled By:  Node/master
    Containers:
    kube-apiserver:
        Container ID:  docker://ae09d2559d6a640c03b1c248c4f4f7e61ab4d57a90ea33d41146cfe9cdf96cf6
        Image:         k8s.gcr.io/kube-apiserver:v1.18.0
        Image ID:      docker-pullable://k8s.gcr.io/kube-apiserver@sha256:fc4efb55c2a7d4e7b9a858c67e24f00e739df4ef5082500c2b60ea0903f18248
        Port:          <none>
        Host Port:     <none>
        Command:
        kube-apiserver
        --advertise-address=10.154.0.2
        --allow-privileged=true
        --authorization-mode=Node,RBAC
        --client-ca-file=/etc/kubernetes/pki/ca.crt
        --enable-admission-plugins=NodeRestriction
        --enable-bootstrap-token-auth=true
        --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
        --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
        --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
        --etcd-servers=https://127.0.0.1:2379
        --insecure-port=0
        --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
        --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
        --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
        --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
        --requestheader-allowed-names=front-proxy-client
        --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
        --requestheader-extra-headers-prefix=X-Remote-Extra-
        --requestheader-group-headers=X-Remote-Group
        --requestheader-username-headers=X-Remote-User
        --secure-port=6443
        --service-account-key-file=/etc/kubernetes/pki/sa.pub
        --service-cluster-ip-range=10.96.0.0/12
        --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
        --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
        State:          Running
        Started:      Tue, 04 Aug 2020 09:02:51 +0000
        Ready:          True
        Restart Count:  0
        Requests:
        cpu:        250m
        Liveness:     http-get https://10.154.0.2:6443/healthz delay=15s timeout=15s period=10s #success=1 #failure=8
        Environment:  <none>
        Mounts:
        /etc/ca-certificates from etc-ca-certificates (ro)
        /etc/kubernetes/pki from k8s-certs (ro)
        /etc/ssl/certs from ca-certs (ro)
        /usr/local/share/ca-certificates from usr-local-share-ca-certificates (ro)
        /usr/share/ca-certificates from usr-share-ca-certificates (ro)
    Conditions:
    Type              Status
    Initialized       True 
    Ready             True 
    ContainersReady   True 
    PodScheduled      True 
    Volumes:
    ca-certs:
        Type:          HostPath (bare host directory volume)
        Path:          /etc/ssl/certs
        HostPathType:  DirectoryOrCreate
    etc-ca-certificates:
        Type:          HostPath (bare host directory volume)
        Path:          /etc/ca-certificates
        HostPathType:  DirectoryOrCreate
    k8s-certs:
        Type:          HostPath (bare host directory volume)
        Path:          /etc/kubernetes/pki
        HostPathType:  DirectoryOrCreate
    usr-local-share-ca-certificates:
        Type:          HostPath (bare host directory volume)
        Path:          /usr/local/share/ca-certificates
        HostPathType:  DirectoryOrCreate
    usr-share-ca-certificates:
        Type:          HostPath (bare host directory volume)
        Path:          /usr/share/ca-certificates
        HostPathType:  DirectoryOrCreate
    QoS Class:         Burstable
    Node-Selectors:    <none>
    Tolerations:       :NoExecute
    Events:
    Type    Reason   Age   From             Message
    ----    ------   ----  ----             -------
    Normal  Pulled   106s  kubelet, master  Container image "k8s.gcr.io/kube-apiserver:v1.18.0" already present on machine
    Normal  Created  106s  kubelet, master  Created container kube-apiserver
    Normal  Started  106s  kubelet, master  Started container kube-apiserver
    ```

    Вывод worker нод из планирования

    ```shell
    $ kubectl drain node-1 --ignore-daemonsets 

    node/node-1 cordoned
    WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-j6l68, kube-system/kube-proxy-dxdwl
    evicting pod default/nginx-deployment-c8fd555cc-8jtpn
    evicting pod kube-system/coredns-66bff467f8-mnh2s
    pod/nginx-deployment-c8fd555cc-8jtpn evicted
    pod/coredns-66bff467f8-mnh2s evicted
    node/node-1 evicted
    ```

    ```shell
    $ kubectl get nodes -o wide

    NAME     STATUS                     ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
    master   Ready                      master   42m   v1.18.0   10.154.0.2    <none>        Ubuntu 18.04.4 LTS   5.3.0-1032-gcp   docker://19.3.8
    node-1   Ready,SchedulingDisabled   <none>   33m   v1.17.4   10.154.0.3    <none>        Ubuntu 18.04.4 LTS   5.3.0-1032-gcp   docker://19.3.8
    node-2   Ready                      <none>   33m   v1.17.4   10.154.0.4    <none>        Ubuntu 18.04.4 LTS   5.3.0-1032-gcp   docker://19.3.8
    node-3   Ready                      <none>   32m   v1.17.4   10.154.0.5    <none>        Ubuntu 18.04.4 LTS   5.3.0-1032-gcp   docker://19.3.8
    ```

    Выполнить на worker ноде 1 от root

    ```shell
    apt-get install -y kubelet=1.18.0-00 kubeadm=1.18.0-00 && systemctl restart kubelet
    ```

    Выполнить на master:

    ```shell
    kubectl get nodes

    NAME     STATUS                     ROLES    AGE   VERSION
    master   Ready                      master   45m   v1.18.0
    node-1   Ready,SchedulingDisabled   <none>   37m   v1.18.0
    node-2   Ready                      <none>   36m   v1.17.4
    node-3   Ready                      <none>   36m   v1.17.4
    ```

    Версия ноды стала 1.18.0

    Возвращение ноды в планирование. На master:

    ```shell
    $ kubectl uncordon node-1

    node/node-1 uncordoned
    ```

    Нода перешла в статус Ready

    Обновление оставшихся двух нод

    Master:
    ```shell
    $ kubectl drain node-2 --ignore-daemonsets 
    
    $ kubectl drain node-3 --ignore-daemonsets
    ```

    Node 2:
    ```shell
    # apt-get install -y kubelet=1.18.0-00 kubeadm=1.18.0-00 && systemctl restart kubelet
    ```

    Node 3
    ```shell
    # apt-get install -y kubelet=1.18.0-00 kubeadm=1.18.0-00 && systemctl restart kubelet
    ```

    Master:
    ```shell
    $ kubectl uncordon node-2 && kubectl uncordon node-3
    
    $ kubectl get nodes

    NAME     STATUS   ROLES    AGE   VERSION
    master   Ready    master   52m   v1.18.0
    node-1   Ready    <none>   43m   v1.18.0
    node-2   Ready    <none>   43m   v1.18.0
    node-3   Ready    <none>   43m   v1.18.0
    ```

- Удаление нод

    ```shell
    gcloud compute instances delete master && \
    gcloud compute instances delete node-1 && \
    gcloud compute instances delete node-2 && \
    gcloud compute instances delete node-3
    ```

## Автоматическое развертывание кластеров при помощи Kubespray

- Создание нод в Google cloud

    ```shell
    gcloud compute instances create master --image-family ubuntu-minimal-1804-lts --image-project ubuntu-os-cloud --machine-type=n1-standard-2 

    gcloud compute instances create node-1 --image-family ubuntu-minimal-1804-lts --image-project ubuntu-os-cloud --machine-type=n1-standard-1 

    gcloud compute instances create node-2 --image-family ubuntu-minimal-1804-lts --image-project ubuntu-os-cloud --machine-type=n1-standard-1 

    gcloud compute instances create node-3 --image-family ubuntu-minimal-1804-lts --image-project ubuntu-os-cloud --machine-type=n1-standard-1
    ```

    ```text
    Created [https://www.googleapis.com/compute/v1/projects/otus1-278509/zones/europe-west2-a/instances/master].
    NAME    ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
    master  europe-west2-a  n1-standard-2               10.154.0.6   35.246.31.109  RUNNING

    Created [https://www.googleapis.com/compute/v1/projects/otus1-278509/zones/europe-west2-a/instances/node-1].
    NAME    ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
    node-1  europe-west2-a  n1-standard-1               10.154.0.7   34.105.198.145  RUNNING

    Created [https://www.googleapis.com/compute/v1/projects/otus1-278509/zones/europe-west2-a/instances/node-2].
    NAME    ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
    node-2  europe-west2-a  n1-standard-1               10.154.0.8   35.246.29.111  RUNNING

    Created [https://www.googleapis.com/compute/v1/projects/otus1-278509/zones/europe-west2-a/instances/node-3].
    NAME    ZONE            MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP   STATUS
    node-3  europe-west2-a  n1-standard-1               10.154.0.9   34.89.62.194  RUNNING
    ```

- Установка Kubespray

    ```shell
    $ git clone https://github.com/kubernetes-sigs/kubespray.git && cd kubespray 
    $ sudo pip install -r requirements.txt 
    $ cp -rfp inventory/sample inventory/mycluster
    ```

- Генерация ключей

    ```shell
    $ ssh-keygen -t rsa -f ~/.ssh/google_compute_engine -C dmizverev
    $ ssh-copy-id -i ~/.ssh/google_compute_engine dmizverev@35.246.31.109
    $ ssh-copy-id -i ~/.ssh/google_compute_engine dmizverev@34.105.198.145
    $ ssh-copy-id -i ~/.ssh/google_compute_engine dmizverev@35.246.29.111
    $ ssh-copy-id -i ~/.ssh/google_compute_engine dmizverev@34.89.62.194
    ```

- Настройка Ansible inventory

    ```ini
    [all]
    node1 ansible_host=35.246.31.109 etcd_member_name=etcd1
    node2 ansible_host=34.105.198.145
    node3 ansible_host=35.246.29.111  
    node4 ansible_host=34.89.62.194

    # ## configure a bastion host if your nodes are not directly reachable
    # bastion ansible_host=x.x.x.x ansible_user=some_user

    [kube-master]
    node1

    [etcd]
    node1


    [kube-node]
    node2
    node3
    node4

    [k8s-cluster:children]
    kube-master
    kube-node
    ```

- Запуск установки

    ```shell
    $ ansible-playbook -i inventory/mycluster/inventory.ini --become --become-user=root --user=dmizverev --key-file="~/.ssh/google_compute_engine" cluster.yml

    PLAY RECAP ********
    localhost                  : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
    node1                      : ok=585  changed=130  unreachable=0    failed=0    skipped=1111 rescued=0    ignored=0   
    node2                      : ok=360  changed=85   unreachable=0    failed=0    skipped=578  rescued=0    ignored=0   
    node3                      : ok=360  changed=85   unreachable=0    failed=0    skipped=577  rescued=0    ignored=0   
    node4                      : ok=360  changed=85   unreachable=0    failed=0    skipped=577  rescued=0    ignored=0 
    ```

   Проверка

    ```shell
    $ kubectl get nodes 
    NAME    STATUS   ROLES    AGE   VERSION
    node1   Ready    master   17m   v1.18.6
    node2   Ready    <none>   14m   v1.18.6
    node3   Ready    <none>   14m   v1.18.6
    node4   Ready    <none>   14m   v1.18.6
    ```

- Удаление нод

    ```shell
    gcloud compute instances delete master && \
    gcloud compute instances delete node-1 && \
    gcloud compute instances delete node-2 && \
    gcloud compute instances delete node-3
    ```