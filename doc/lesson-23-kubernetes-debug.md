# Отладка и тестирование в Kubernetes

## Kubectl-debug

- Установка kubectl-debug

    ```bash
    export PLUGIN_VERSION=0.1.1
    # linux x86_64
    $ curl -Lo kubectl-debug.tar.gz https://github.com/aylei/kubectl-debug/releases/download/v${PLUGIN_VERSION}/kubectl-debug_${PLUGIN_VERSION}_linux_amd64.tar.gz
    $ tar -zxvf kubectl-debug.tar.gz kubectl-debug
    $ sudo mv kubectl-debug /usr/local/bin/
    $ kubectl-debug --version
    
    debug version v0.0.0-master+$Format:%h$
    ```

- Установка kubectl-debug DaemonSet  

    Скачать манифест

    ```bash
    wget https://raw.githubusercontent.com/aylei/kubectl-debug/dd7e4965e4ae5c4f53e6cf9fd17acc964274ca5c/scripts/agent_daemonset.yml
    ```

    Исправить версию API

    Установить

    ```bash
    kubectl apply -f strace/agent_daemonset.yml
    ```

- Проверка strace

    Создать Pod с nginx: kubernetes-debug/strace/test-pod.yaml

    Установить его:

    ```bash
    kubectl apply -f strace/test-pod.yaml
    ```

    Запустить debug

    ```bash
    kubectl-debug testpod
    ```

    Запустить strace

    ```bash
    strace: attach: ptrace(PTRACE_SEIZE, 1): Operation not permitted
    ```

    Нужная capability не в версии `aylei/debug-agent:0.0.1`, а в версии `aylei/debug-agent:v0.1.1`.
    Необходимо изменить версию docker image и переустановить DaemonSet.
    Вообще говоря, его проще поставить из официальной документации.

    После этого запустить strace

    ```bash
    $ kubectl-debug testpod

    strace -c -p1
    strace: Process 1 attached
    ```

## iptables-tailer

Основной кейс - сообщить разработчикам сервисов о проблемах с NetworkPolicy.

- Установка netperf-operator

    Это Kubernetes-оператор, который позволяет запускать тесты пропускной
способности сети между нодами кластера

    ```bash
    kubectl apply -f kit/deploy/crd.yaml
    kubectl apply -f kit/deploy/rbac.yaml
    kubectl apply -f kit/deploy/operator.yaml
    kubectl apply -f kit/deploy/cr.yaml
    ```

    Проверка

    ```bash
    $ kubectl describe netperf.app.example.com/example

    Name:         example
    Namespace:    default
    Labels:       <none>
    Annotations:  API Version:  app.example.com/v1alpha1
    Kind:         Netperf
    Metadata:
    Creation Timestamp:  2020-07-29T08:27:56Z
    Generation:          4
    Resource Version:    24623
    Self Link:           /apis/app.example.com/v1alpha1/namespaces/default/netperfs/example
    UID:                 38c8a232-4425-4398-a910-1f3dbdd28842
    Spec:
    Client Node:  
    Server Node:  
    Status:
    Client Pod:          netperf-client-1f3dbdd28842
    Server Pod:          netperf-server-1f3dbdd28842
    Speed Bits Per Sec:  7785.21
    Status:              Done
    Events:                <none>
    ```

    Должно быть выведено `Status: Done` и `Speed Bits Per Sec:`.

- Добавление сетевой политики calico.

    ```bash
    kubectl apply -f kit/networkpolicy.yaml
    ```

- Надо перезапустить netperf-тест cr.yaml

    ```bash
    kubectl delete -f kit/deploy/cr.yaml 
    kubectl apply -f kit/deploy/cr.yaml
    ```
- Проверка теста еще раз

    ```bash
    $ kubectl describe netperf.app.example.com/example

    Name:         example
    Namespace:    default
    Labels:       <none>
    Annotations:  API Version:  app.example.com/v1alpha1
    Kind:         Netperf
    Metadata:
    Creation Timestamp:  2020-07-29T08:38:09Z
    Generation:          3
    Resource Version:    27272
    Self Link:           /apis/app.example.com/v1alpha1/namespaces/default/netperfs/example
    UID:                 e533cfdf-3a45-4f65-ad71-8f9a87097dec
    Spec:
    Client Node:  
    Server Node:  
    Status:
    Client Pod:          netperf-client-8f9a87097dec
    Server Pod:          netperf-server-8f9a87097dec
    Speed Bits Per Sec:  0
    Status:              Started test
    Events:                <none>
    ```

    Видим `Status: Started test`. В нашей сетевой политике есть ошибка.

- Проверка, что в логах ноды есть сообщения об отброшенных пакетах

    ```bash
    # Зайти по SSH на node
    gcloud compute ssh gke-cluster-1-default-pool-a68cb8d6-1kjh --zone us-east1-b

    # Выполнить команды
    sudo iptables --list -nv | grep DROP
    sudo iptables --list -nv | grep LOG
    journalctl -k | grep calico
    ```

- Установка iptables-tailer

    Документация по установке: https://github.com/box/kube-iptables-tailer

    Надо взять [DaemonSet](https://github.com/express42/otus-platform-snippets/blob/master/Module-03/Debugging/iptables-tailer.yaml). 
    В нем должно быть `value: "calico-packet:"`
    Переменная:

    ```yaml
    - name: "JOURNAL_DIRECTORY"
      value: "/var/log/journal"
    ``` 

    Добавить к нему serviceaccount, clusterrole, clusterrolebinfing


    ```bash
    kubectl apply -f kit/kit-serviceaccount.yaml
    kubectl apply -f kit/kit-clusterrole.yaml
    kubectl apply -f kit/kit-clusterrolebinding.yaml
    kubectl apply -f kit/iptables-tailer.yaml
    ```
    
    Перезапустить netperf-тест

    ```bash
    kubectl delete -f kit/deploy/cr.yaml 
    kubectl apply -f kit/deploy/cr.yaml
    ```

    Проверка теста еще раз

    ```bash
    $ kubectl describe netperf.app.example.com/example

    Name:         netperf-server-a5c3f74c693e
    Namespace:    default
    Priority:     0
    Node:         gke-cluster-1-default-pool-a68cb8d6-1kjh/10.142.0.3
    Start Time:   Wed, 29 Jul 2020 12:33:48 +0300
    Labels:       app=netperf-operator
                netperf-type=server
    Annotations:  cni.projectcalico.org/podIP: 10.8.1.11/32
                kubernetes.io/limit-ranger: LimitRanger plugin set: cpu request for container netperf-server-a5c3f74c693e
    Status:       Running
    IP:           10.8.1.11
    IPs:
    IP:           10.8.1.11
    Controlled By:  Netperf/example
    Containers:
    netperf-server-a5c3f74c693e:
        Container ID:   docker://353ff0dc1e8518ca4f2f05bdecada75cc1019f8bbb2b0c6a3df05d7af3976c7d
        Image:          tailoredcloud/netperf:v2.7
        Image ID:       docker-pullable://tailoredcloud/netperf@sha256:0361f1254cfea87ff17fc1bd8eda95f939f99429856f766db3340c8cdfed1cf1
        Port:           <none>
        Host Port:      <none>
        State:          Running
        Started:      Wed, 29 Jul 2020 12:33:49 +0300
        Ready:          True
        Restart Count:  0
        Requests:
        cpu:        100m
        Environment:  <none>
        Mounts:
        /var/run/secrets/kubernetes.io/serviceaccount from default-token-r9lbz (ro)
    Conditions:
    Type              Status
    Initialized       True 
    Ready             True 
    ContainersReady   True 
    PodScheduled      True 
    Volumes:
    default-token-r9lbz:
        Type:        Secret (a volume populated by a Secret)
        SecretName:  default-token-r9lbz
        Optional:    false
    QoS Class:       Burstable
    Node-Selectors:  <none>
    Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                    node.kubernetes.io/unreachable:NoExecute for 300s
    Events:
    Type     Reason      Age   From                                               Message
    ----     ------      ----  ----                                               -------
    Normal   Scheduled   16s   default-scheduler                                  Successfully assigned default/netperf-server-a5c3f74c693e to gke-cluster-1-default-pool-a68cb8d6-1kjh
    Normal   Pulled      15s   kubelet, gke-cluster-1-default-pool-a68cb8d6-1kjh  Container image "tailoredcloud/netperf:v2.7" already present on machine
    Normal   Created     15s   kubelet, gke-cluster-1-default-pool-a68cb8d6-1kjh  Created container netperf-server-a5c3f74c693e
    Normal   Started     15s   kubelet, gke-cluster-1-default-pool-a68cb8d6-1kjh  Started container netperf-server-a5c3f74c693e
    Warning  PacketDrop  13s   kube-iptables-tailer                               Packet dropped when receiving traffic from 10.8.2.8
    ```

    Есть запись c `Reason = PacketDrop`

- Для того, чтобы в логе отображались имена Pod надо в `kubernetes-debug/kit/iptables-tailer.yaml` добавить переменную

    ```yaml
    - name: "POD_IDENTIFIER"
      value: "name"
    ```

    Тогда получим вот такой вывод:

    ```bash
    $ kubectl describe netperf.app.example.com/example

    Events:
    Type     Reason      Age    From                                               Message
    ----     ------      ----   ----                                               -------
    Warning  PacketDrop  40s    kube-iptables-tailer                               Packet dropped when receiving traffic from netperf-client-a5c3f74c693e (10.8.2.8)
    ```