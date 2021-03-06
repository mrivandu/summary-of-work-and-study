# 深入剖析 Kubernetes

## 一 Pod

Pod 是一个逻辑概念，Kubernetes 真正处理的，还是宿主机操作系统上 Linux 容器的 Namespace 和 Cgroups，而并不存在一个所谓的 Pod 的边界或者隔离环境。 Pod 是一组共享了某些资源的容器，Pod里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume 。在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫做Infra容器（Infra容器（k8s.gcr.io/pause）占用极少的资源，它的镜像时用汇编语言编写的，永远处于“暂停”状态的容器）。 在Pod中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 join Network Namespace的方式，与Infra容器关联在一起，对于同一个Pod里面的所有用户容器，它们的进出流量都是通过Infra容器完成的。同一个 Pod 里面的所有用户容器来说，它们的进出流量，也可以认为都是通过 Infra 容器完成的。凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的。

```bash
kubectl create namespace rook-ceph
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/operator.yaml

```

### 2.1 Pod 中几个重要字段的含义和用法

**NodeSelector**：是一个供用户将 Pod 与 Node 进行绑定的字段，用法如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gysl-nodeselect
spec:
  nodeSelector:
    kubernetes.io/hostname: 172.31.2.12
  containers:
  - name: gysl-nginx
    image: nginx
```

这就意味着这个 Pod 只能在携带 kubernetes.io/hostname 标签的 Node 上运行了，否则，调度失败。

**NodeName**：一旦 Pod 的这个字段被赋值，Kubernetes 项目就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字。所以，这个字段一般由调度器负责设置，但用户也可以设置它来“骗过”调度器，当然这个做法一般是在测试或者调试的时候才会用到。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gysl-nodename
spec:
  nodeName: 172.31.2.12
  containers:
  - name: gysl-nginx
    image: nginx
```

**HostAliases**：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gysl-hostaliases
spec:
  hostAliases:
  - ip: "10.0.0.20"
    hostnames:
    - "test.gysl"
    - "app.gysl"
  containers:
  - name: gysl-nginx
    image: nginx
```

最下面两行记录，就是我通过 HostAliases 字段为 Pod 设置的。需要指出的是，在 Kubernetes 项目中，如果要设置 hosts 文件里的内容，一定要通过这种方法。否则，如果直接修改了 hosts 文件的话，在 Pod 被删除重建之后，kubelet 会自动覆盖掉被修改的内容。

凡是跟容器的 Linux Namespace 相关的属性，也一定是 Pod 级别的。这个原因也很容易理解：Pod 的设计，就是要让它里面的容器尽可能多地共享 Linux Namespace，仅保留必要的隔离和限制能力。

继续看以下例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gysl-shareprocessnamespace
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: busybox
    image: busybox
    tty: true
    stdin: true
```

使用以下命令进入指定的 container ：

```bash
kubectl attach -it gysl-shareprocessnamespace -c busybox
```

进入之后查看一下进程共享情况：

```bash
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    6 root      0:00 nginx: master process nginx -g daemon off;
   11 101       0:00 nginx: worker process
   12 root      0:00 sh
   32 root      0:00 ps aux
```

在这个容器里，我们不仅可以看到它本身的 ps aux 指令，还可以看到 nginx 容器的进程，以及 Infra 容器的 /pause 进程。这就意味着，整个 Pod 里的每个容器的进程，对于所有容器来说都是可见的：它们共享了同一个 PID Namespace。凡是 Pod 中的容器要共享宿主机的 Namespace，也一定是 Pod 级别的定义。

再看一个例子：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gysl-share-namespace
spec:
  hostPID: true
  hostIPC: true
  hostNetwork: true
  nodeName: 172.31.2.11
  shareProcessNamespace: true
  containers:
  - name: nginx-gysl
    image: nginx
    imagePullPolicy: IfNotPresent
  - name: busybox-gysl
    image: busybox
    stdin: true
    tty: true
    imagePullPolicy: Always
    lifecycle:
      postStart:
        exec:
          command: ['/bin/sh','-c','echo "This is a test of gysl. ">/gysl.txt']
      preStop:
        exec:
          command: ['/bin/sh','-c','echo "This is a demo of gysl."']
```

上面的例子中，定义了共享宿主机的 Network、IPC 和 PID Namespace。这就意味着，这个 Pod 里的所有容器，会直接使用宿主机的网络、直接与宿主机进行 IPC 通信、看到宿主机里正在运行的所有进程。

除此之外，ImagePullPolicy 和 Lifecycle 也是值得我们关注的两个字段。

**ImagePullPolicy** 字段定义了镜像拉取的策略。而它之所以是一个 Container 级别的属性，是因为容器镜像本来就是 Container 定义中的一部分。ImagePullPolicy 的值默认是 Always，即每次创建 Pod 都重新拉取一次镜像。如果它的值被定义为 Never 或者 IfNotPresent，则意味着 Pod 永远不会主动拉取这个镜像，或者只在宿主机上不存在这个镜像时才拉取。

**Lifecycle** 字段。它定义的是 Container Lifecycle Hooks。顾名思义，Container Lifecycle Hooks 的作用，是在容器状态发生变化时触发一系列“钩子”。在这个字段中，我们看到了 postStart 和 preStop 两个参数。postStart 参数在容器启动后，立刻执行一个指定的操作。需要明确的是，postStart 定义的操作，虽然是在 Docker 容器 ENTRYPOINT 执行之后，但它并不严格保证顺序。也就是说，在 postStart 启动时，ENTRYPOINT 有可能还没有结束。如果 postStart 执行超时或者错误，Kubernetes 会在该 Pod 的 Events 中报出该容器启动失败的错误信息，导致 Pod 也处于失败的状态。preStop 发生的时机，则是容器被杀死之前（比如，收到了 SIGKILL 信号）。而需要明确的是，preStop 操作的执行，是同步的。所以，它会阻塞当前的容器杀死流程，直到这个 Hook 定义操作完成之后，才允许容器被杀死，这跟 postStart 不一样。

### 2.2 Projected Volume

作为 Kubernetes 比较核心的编排对象，Pod 携带的信息极其丰富。在 Kubernetes 中，有几种特殊的 Volume，它们存在的意义不是为了存放容器里的数据，也不是用来进行容器和宿主机之间的数据交换。这些特殊 Volume 的作用，是为容器提供预先定义好的数据。从容器的角度来看，这些 Volume 里的信息就是仿佛是被 Kubernetes“投射”。Kubernetes 支持的 Projected Volume 有如下四种：

- Secret

- ConfigMap

- DownWarAPI

- ServiceAccountToken

#### 2.2.1 Secret

Kubernetes 把 Pod 想要访问的东西存放在 etcd 中，然后通过在 Pod 的容器里挂载 volume 的方式来进行访问。存放数据库的凭证信息就是 Secret 最典型的应用场景之一。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-gysl
spec:
  containers:
  - name: secret-gysl
    image: busybox
    args:
    - sleep
    - "3600"
    volumeMounts:
    - name: secret-mysql-gysl
      mountPath: "/projected-volume-secret"
      readOnly: true
  volumes:
  - name: secret-mysql-gysl
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: passwd
```

在这个例子中，声明挂载的 Volume 的类型是 projected 类型。这个 Volume 的数据来源（sources）是名为 user 和 passwd 的 Secret 对象，分别对应的是数据库的用户名和密码。

在编写 yaml 文件的时候需要最后几行，secret 下面依然是需要进一步缩进的。

在 apply 以上 yaml 文件之后，我们会发现 Pod 的状态一直是 ContainerCreating ，原因是我们还没有创建相关的 secret 。使用以下命令进行创建：

```bash
kubectl create secret generic user --from-file=username.txt
kubectl create secret generic passwd --from-file=passwd.txt
```

username.txt 和 passwd.txt 两个文件的内容分别如下：

```txt
cat username.txt
gysl
cat passwd.txt
E#w23%dj2JK@
```

进入该 Pod:

```bash
kubectl exec -it secret-gysl sh
```

正常情况下，可以看到如下内容：

```bash
/ # ls -al /projected-volume-secret/
total 0
drwxrwxrwt    3 root     root           120 Aug  1 07:25 .
drwxr-xr-x    1 root     root            60 Aug  1 07:25 ..
drwxr-xr-x    2 root     root            80 Aug  1 07:25 ..2019_08_01_07_25_08.650769588
lrwxrwxrwx    1 root     root            31 Aug  1 07:25 ..data -> ..2019_08_01_07_25_08.650769588
lrwxrwxrwx    1 root     root            17 Aug  1 07:25 passwd.txt -> ..data/passwd.txt
lrwxrwxrwx    1 root     root            19 Aug  1 07:25 username.txt -> ..data/username.txt
/ # cat /projected-volume-secret/username.txt
gysl
/ # cat /projected-volume-secret/passwd.txt
E#w23%dj2JK@
```

也可以通过 yaml 文件的方式来进行创建，以上命令转化为 yaml 如下：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: user
type: Opaque
data:
  username.txt: Z3lzbAo=
---
apiVersion: v1
kind: Secret
metadata:
  name: passwd
type: Opaque
data:
  passwd.txt: RSN3MjMlZGoySktACg==
```

该yaml 文件中的 data 部分的字段都是经过 base64 转码的：

```bash
cat username.txt |base64
Z3lzbAo=
cat passwd.txt |base64
RSN3MjMlZGoySktACg==
```

使用上面的命令进入该 Pod 我们就可以看到跟之前操作一样的内容。上面我们使用了2个 secret 对象来进行本次实验。我们能否使用一个 secret 来达到一样的目标呢？

请看以下 yaml ：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-gysl
spec:
  containers:
  - name: secret-gysl
    image: busybox
    args:
    - sleep
    - "3600"
    volumeMounts:
    - name: secret-mysql-gysl
      mountPath: "/projected-volume-secret"
      readOnly: true
  volumes:
  - name: secret-mysql-gysl
    projected:
      sources:
#      - secret:
#          name: passwd
      - secret:
          name: secret-gysl
---
apiVersion: v1
kind: Secret
metadata:
  name: secret-gysl
type: Opaque
data:
  user: Z3lzbAo=
  passwd: RSN3MjMlZGoySktACg==
```

进入 Pod 观察：

```bash
/projected-volume-secret # ls -al
total 0
drwxrwxrwt    3 root     root           120 Aug  1 06:55 .
drwxr-xr-x    1 root     root            60 Aug  1 06:55 ..
drwxr-xr-x    2 root     root            80 Aug  1 06:55 ..2019_08_01_06_55_22.687706782
lrwxrwxrwx    1 root     root            31 Aug  1 06:55 ..data -> ..2019_08_01_06_55_22.687706782
lrwxrwxrwx    1 root     root            13 Aug  1 06:55 passwd -> ..data/passwd
lrwxrwxrwx    1 root     root            11 Aug  1 06:55 user -> ..data/user
```

从目录结构和内容来看，差异并不大，多层软连接，一些隐藏文件。使用这样的方法创建的 secret 仅仅进行了转码，并未进行加密，生产环境中使用一般情况下需要使用加密插件。

#### 2.2.2 ConfigMap

ConfigMap 与 Secret 的区别在于，ConfigMap 保存的是不需要加密的、应用所需的配置信息，用法几乎与 Secret 完全相同：可以使用 kubectl create configmap 从文件或者目录创建 ConfigMap，也可以直接编写 ConfigMap 对象的 YAML 文件。我们以 kube-controller-manager.conf 文件来演示一下。

创建 configMap:

```bash
 kubectl create configmap kube-scheduler --from-file=/etc/kubernetes/conf.d/kube-controller-manager.conf
```

同样，我们也可以使用 yaml 来创建：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-controller-manager-gysl
data:
  kube-controller-manager.conf: |
    KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=true --v=4 --master=127.0.0.1:8080 --leader-elect=true --address=127.0.0.1 --service-cluster-ip-range=10.0.0.0/24 --cluster-name=kubernetes --cluster-signing-cert-file=/etc/kubernetes/ca.d/ca.pem --cluster-signing-key-file=/etc/kubernetes/ca.d/ca-key.pem  --root-ca-file=/etc/kubernetes/ca.d/ca.pem --service-account-private-key-file=/etc/kubernetes/ca.d/ca-key.pem"
```

就这样，kube-controller-manager.conf 配置文件的内容就被保存到了 kube-controller-manager-gysl 这个ConfigMap 中。

#### 2.2.3 Downward API

Downward API 能让 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-gysl
  labels:
    zone: beijing
    cluster: gysl-cluster1
    rack: rack-gysl
spec:
  containers:
    - name: client-container
      image: busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: false
  volumes:
    - name: podinfo
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
```

Pod 的 Labels 字段的值，就会被 Kubernetes 自动挂载成为容器里的 /etc/podinfo/labels 文件。执行命令：

```bash
kubectl logs downward-api-gysl
```

看到的结果：

```text

cluster="gysl-cluster1"
rack="rack-gysl"
zone="beijing"

cluster="gysl-cluster1"
rack="rack-gysl"
zone="beijing"

cluster="gysl-cluster1"
rack="rack-gysl"
zone="beijing"
```

Downward API 支持的字段已经非常丰富，比如：

```text
1. 使用 fieldRef 可以声明使用:
spec.nodeName - 宿主机名字
status.hostIP - 宿主机 IP
metadata.name - Pod 的名字
metadata.namespace - Pod 的 Namespace
status.podIP - Pod 的 IP
spec.serviceAccountName - Pod 的 Service Account 的名字
metadata.uid - Pod 的 UID
metadata.labels['<KEY>'] - 指定 <KEY> 的 Label 值
metadata.annotations['<KEY>'] - 指定 <KEY> 的 Annotation 值
metadata.labels - Pod 的所有 Label
metadata.annotations - Pod 的所有 Annotation

2. 使用 resourceFieldRef 可以声明使用:
容器的 CPU limit
容器的 CPU request
容器的 memory limit
容器的 memory request
```

Downward API 能够获取到的信息，一定是 Pod 里的容器进程启动之前就能够确定下来的信息。而如果你想要获取 Pod 容器运行后才会出现的信息。比如，容器进程的 PID，那就肯定不能使用 Downward API 了，而应该考虑在 Pod 里定义一个 sidecar 容器。

#### 2.2.4 Service Account  

Service Account 对象的作用，就是 Kubernetes 系统内置的一种“服务账户”，它是 Kubernetes 进行权限分配的对象。比如，Service Account A，可以只被允许对 Kubernetes API 进行 GET 操作，而 Service Account B，则可以有 Kubernetes API 的所有操作的权限。

像这样的 Service Account 的授权信息和文件，实际上保存在它所绑定的一个特殊的 Secret 对象里的。这个特殊的 Secret 对象，就叫作ServiceAccountToken。任何运行在 Kubernetes 集群上的应用，都必须使用这个 ServiceAccountToken 里保存的授权信息，也就是 Token，才可以合法地访问 API Server。

因此，Kubernetes 项目的 Projected Volume 其实只有三种，因为第四种 ServiceAccountToken，只是一种特殊的 Secret 而已。

为了方便使用，Kubernetes 已经为你提供了一个的默认“服务账户”（default Service Account）。并且，任何一个运行在 Kubernetes 里的 Pod，都可以直接使用这个默认的 Service Account，而无需显示地声明挂载它。Kubernetes 在每个 Pod 创建的时候，自动在它的 spec.volumes 部分添加上了默认 ServiceAccountToken 的定义，然后自动给每个容器加上了对应的 volumeMounts 字段。这个过程对于用户来说是完全透明的。一旦 Pod 创建完成，容器里的应用就可以直接从这个默认 ServiceAccountToken 的挂载目录里访问到授权信息和文件。这个容器内的路径在 Kubernetes 里是固定的，即：/var/run/secrets/kubernetes.io/serviceaccount。

```bash
$ kubectl exec -it downward-api-gysl sh
/ # ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt     namespace  token
```

这种把 Kubernetes 客户端以容器的方式运行在集群里，然后使用 default Service Account 自动授权的方式，被称作“InClusterConfig”，也是我最推荐的进行 Kubernetes API 编程的授权方式。

考虑到自动挂载默认 ServiceAccountToken 的潜在风险，Kubernetes 允许你设置默认不为 Pod 里的容器自动挂载这个 Volume。

## 二 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-gysl
spec:
  replicas: 2
  selector:
    matchLabels:
      app-1: nginx
      app-2: busybox
  template:
    metadata:
      labels:
        app-1: nginx
        app-2: busybox
    spec:
      containers:
        - name: app-1
          image: nginx:1.16.0
          imagePullPolicy: Always
          ports:
            - containerPort: 80
            - containerPort: 8080
        - name: app-2
          image: busybox
          imagePullPolicy: Never
          command: ['/bin/sh', '-c']
          args:
            - while :;do sleep 20;done
```

这是一个编排得非常简单的 Deployment，确保携带 app-1=nginx 和 app-2=busybox 标签的 Pod 的个数等于 spec.replicas 指定的总数 2 个。也就是说在这个 Deployment 的 Pod 数量大等于2时，就会有 Pod 被删除，反之则会有 Pod 被创建。

这个 Deployment 由2个部分构成，例子中的  yaml 第1-10行定义了 Deployment 控制器，第10行以后的内容则定义了被控制的 Pod ,template 后面这一部分我们会发现跟之前的 Pod 定义大同小异。

此处顺便提一条命令(更新 Deployment 的镜像)：

```bash
kubectl set image deployment/deployment-gysl app-1=nginx:latest
```

这个命令还可以更新以下对象：

```text
  env            Update environment variables on a pod template
  image          更新一个 pod template 的镜像
  resources      在对象的 pod templates 上更新资源的 requests/limits
  selector       设置 resource 的 selector
  serviceaccount Update ServiceAccount of a resource
  subject        Update User, Group or ServiceAccount in a RoleBinding/ClusterRoleBinding
```

## 三 ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replica-set-gysl
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
```

一个 ReplicaSet 对象就是由副本数目的定义和一个 Pod 模板组成的, 它的定义就是 Deployment 的一个子集。Deployment 控制器实际操纵的是 ReplicaSet 对象，而不是 Pod 对象。

ReplicaSet 负责通过“控制器模式”，保证系统中 Pod 的个数永远等于指定的个数（比如， 2个）。这也正是 Deployment 只允许容器的 restartPolicy=Always 的主要原因：只有在容器能保证自己始终是 Running 状态的前提下，ReplicaSet 调整 Pod 的个数才有意义。

Deployment 通过“控制器模式”，来操作 ReplicaSet 的个数和属性，进而实现“水平扩展 / 收缩”和“滚动更新”这两个编排动作。

## 四 StatefulSet

### 4.1 背景知识及相关概念

StatefulSet 的设计其实非常容易理解。它把真实世界里的应用状态，抽象为了两种情况：

拓扑状态。这种情况意味着，应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动，比如应用的主节点 A 要先于从节点 B 启动。而如果你把 A 和 B 两个 Pod 删除掉，它们再次被创建出来时也必须严格按照这个顺序才行。并且，新创建出来的 Pod，必须和原来 Pod 的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新 Pod。

存储状态。这种情况意味着，应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。

StatefulSet 的核心功能，就是通过某种方式记录这些状态，然后在 Pod 被重新创建时，能够为新 Pod 恢复这些状态。

这个 Service 又是如何被访问的呢？

第一种方式，是以 Service 的 VIP（Virtual IP，即：虚拟 IP）方式。比如：当我访问 172.20.25.3 这个 Service 的 IP 地址时，172.20.25.3 其实就是一个 VIP，它会把请求转发到该 Service 所代理的某一个 Pod 上。

第二种方式，就是以 Service 的 DNS 方式。比如：这时候，只要我访问“my-svc.my-namespace.svc.cluster.local”这条 DNS 记录，就可以访问到名叫 my-svc 的 Service 所代理的某一个 Pod。

### 4.2 拓扑结构

让我们来看一下以下例子：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      name: web
  clusterIP: None
```

---

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-server-gysl
  labels:
    app: nginx
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      restartPolicy: Always
      containers:
        - name: web-server
          image: nginx:1.16.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              name: web-port
```

这些 Pod 的创建，也是严格按照编号顺序进行的。比如，在 web-server-gysl-0 进入到 Running 状态、并且细分状态（Conditions）成为 Ready 之前，web-server-gysl-1 会一直处于 Pending 状态。

使用以下命令测试：

```bash
kubectl run -i --tty  --image toolkit:v1.0.0821 dns-test --restart=Never --rm /bin/bash
```

```bash
[root@dns-test /]# nslookup web-server-gysl-0.nginx
Server:         10.0.0.2
Address:        10.0.0.2#53

Name:   web-server-gysl-0.nginx.default.svc.cluster.local
Address: 172.20.25.3

[root@dns-test /]# nslookup web-server-gysl-1.nginx
Server:         10.0.0.2
Address:        10.0.0.2#53

Name:   web-server-gysl-1.nginx.default.svc.cluster.local
Address: 172.20.72.7

[root@dns-test /]# nslookup nginx
Server:         10.0.0.2
Address:        10.0.0.2#53

Name:   nginx.default.svc.cluster.local
Address: 172.20.72.7
Name:   nginx.default.svc.cluster.local
Address: 172.20.25.3
```

由于最近版本的 busybox 有坑，我自己制作了一个 DNS 测试工具，Dockerfile 如下：

```Dockerfile
FROM centos:7.6.1810
RUN  yum -y install bind-utils
CMD  ["/bin/bash","-c","while true;do sleep 60000;done"]
```

回到 Master 节点看一下：

```bash
$ kubectl get pod -o wide
NAME                READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
web-server-gysl-0   1/1     Running   0          43m   172.20.25.3   172.31.2.12   <none>           <none>
web-server-gysl-1   1/1     Running   0          42m   172.20.72.7   172.31.2.11   <none>           <none>
$ kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
nginx        ClusterIP   None         <none>        80/TCP    43m   app=nginx
```

当我们在集群内部分别 ping 域名 web-server-gysl-0.nginx.default.svc.cluster.local 和 web-server-gysl-1.nginx.default.svc.cluster.local 时，正常返回了对应的 Pod IP， 在 ping 域名 nginx.default.svc.cluster.local 时，则随机返回2个 Pod IP 中的一个。完全印证了上文所述内容。

在上述操作过程中，我随机删除了这些 Pod 中的某一个或几个，稍后再次来查看的时候，新创建的 Pod 依然按照之前的编号进行了编排。

此外，我将 StatefulSet 的一个 Pod 所在的集群内节点下线，再次查看 Pod 的情况，系统在其他节点上以原 Pod 的名称迅速创建了新的 Pod。编号都是从 0 开始累加，与 StatefulSet 的每个 Pod 实例一一对应，绝不重复。

### 4.3 存储结构

由于测试环境资源有限，原计划使用 rook-ceph 来进行实验的，无奈使用 NFS 来进行实验。 Ceph 创建 PV 的相关 yaml 如下：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-gysl
  labels:
    type: local
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  rbd:
    monitors:
      - '172.31.2.11:6789'
      - '172.31.2.12:6789'
    pool: data
    image: data
    fsType: xfs
    readOnly: true
    user: admin
    keyring: /etc/ceph/keyrin
```

NFS 实验相关 yaml:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-gysl-0
  labels:
    environment: test
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /data-0
    server: 172.31.2.10
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-gysl-1
  labels:
    environment: test
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /data-1
    server: 172.31.2.10
---
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: nfs
provisioner: fuseim.pri/ifs
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset-pvc-gysl
spec:
  replicas: 2
  serviceName: "gysl-web"
  selector:
    matchLabels:
      app: pod-gysl
  template:
    metadata:
      name: web-pod
      labels:
        app: pod-gysl
    spec:
      containers:
        - name: nginx
          image: nginx
          imagePullPolicy: IfNotPresent
          ports:
            - name: web-port
              containerPort: 80
          volumeMounts:
            - name: www-vct
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www-vct
        annotations:
          volume.beta.kubernetes.io/storage-class: "nfs"
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
        storageClassName: nfs
---
apiVersion: v1
kind: Service
metadata:
  name: gysl-web
spec:
  type: NodePort
  selector:
    app: pod-gysl
  ports:
    - name: web-svc
      protocol: TCP
      nodePort: 31688
      port: 8080
      targetPort: 80
```

通过以下命令向相关 Pod 写入验证内容：

```bash
for node in 0 1;do kubectl exec statefulset-pvc-gysl-$node -- sh -c "echo \<h1\>Node: ${node}\</h1\>>/usr/share/nginx/html/index.html";done
```

观察实验结果：

```bash
$ kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP            NODE          NOMINATED NODE   READINESS GATES
statefulset-pvc-gysl-0   1/1     Running   0          51m   172.20.85.2   172.31.2.11   <none>           <none>
statefulset-pvc-gysl-1   1/1     Running   0          32m   172.20.65.4   172.31.2.12   <none>           <none>
$ kubectl get pvc
NAME                             STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-vct-statefulset-pvc-gysl-0   Bound    pv-nfs-gysl-0   1Gi        RWO            nfs            51m
www-vct-statefulset-pvc-gysl-1   Bound    pv-nfs-gysl-1   1Gi        RWO            nfs            49m
$ kubectl get pv
NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                    STORAGECLASS   REASON   AGE
pv-nfs-gysl-0   1Gi        RWO            Recycle          Bound    default/www-vct-statefulset-pvc-gysl-0   nfs                     51m
pv-nfs-gysl-1   1Gi        RWO            Recycle          Bound    default/www-vct-statefulset-pvc-gysl-1   nfs                     51m
$ cat /data-0/index.html
<h1>Node: 0</h1>
$ cat /data-1/index.html
<h1>Node: 1</h1>
$ curl 172.31.2.11:31688
<h1>Node: 0</h1>
$ curl 172.31.2.11:31688
<h1>Node: 1</h1>
$ curl 172.31.2.12:31688
<h1>Node: 1</h1>
$ curl 172.31.2.12:31688
<h1>Node: 0</h1>
```

---

```bash
kubectl run -i --tty  --image toolkit:v1.0.0821 test --restart=Never --rm /bin/bash
```

```text
[root@test /]# curl statefulset-pvc-gysl-0.gysl-web
<h1>Node: 0</h1>
[root@test /]# curl statefulset-pvc-gysl-0.gysl-web
<h1>Node: 0</h1>
[root@test /]# curl statefulset-pvc-gysl-1.gysl-web
<h1>Node: 1</h1>
[root@test /]# curl statefulset-pvc-gysl-1.gysl-web
<h1>Node: 1</h1>
[root@test /]# curl gysl-web:8080
<h1>Node: 1</h1>
[root@test /]# curl gysl-web:8080
<h1>Node: 1</h1>
[root@test /]# curl gysl-web:8080
<h1>Node: 0</h1>
[root@test /]# curl gysl-web:8080
<h1>Node: 0</h1>
```

从实验结果中我们可以看出 Pod 与 PV、PVC 的对应关系，结合上文中的 yaml 我们不难发现：

1. Pod 与对应的 PV 存储是一一对应的，在创 Pod 的同时， StatefulSet根据对应的规创建了相应的 PVC，PVC 选择符合条件的 PV 进绑定。当 Pod 被删除之后，数据依然保存在 PV 中，当被删除的 Pod 再次被创建时， 该 Pod 依然会立即与原来的 Pod 进行绑定，保持原有的对应关系。

2. 在集群内部，可以通过 pod 名加对应的服务名访问指定的 Pod 及其绑定的 PV。 如果通过服务名来访问 StatefulSet ，那么服务名的功能类似于 VIP 的功能。

### 4.4 StatefulSet 实践

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    [mysqld]
    log-bin
  slave.cnf: |
    [mysql]
    super-read-only
```

---

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
      clusterIP: None
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
```

---

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  replicas: 3
  selector:
  matchLabels:
    labels:
      app: mysql
  serviceName: mysql
  template:
    metadata:
      name: mysql
      labels:
        app: mysql
    spec:
      initContainers:
        - name: init-mysql
        - name: clone-mysql
      containers:
        - name: mysql
        - name: xtrabackup
      volumes:
        - name: conf
          emptyDir: {}
        - name: config-map
          configMap:
            name: mysql
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```
