# 深入理解StatefulSet

## 拓扑状态

在上一篇文章中，我在结尾处讨论到了 Deployment 实际上并不足以覆盖所有的应用编排问题。

造成这个问题的根本原因，在于 Deployment 对应用做了一个简单化假设。

它认为，一个应用的所有 Pod，是完全一样的。所以，它们互相之间没有顺序，也无所谓运行在哪台宿主机上。需要的时候，Deployment 就可以通过 Pod 模板创建新的 Pod；不需要的时候，Deployment 就可以“杀掉”任意一个 Pod。

但是，在实际的场景中，并不是所有的应用都可以满足这样的要求。

尤其是分布式应用，它的多个实例之间，往往有依赖关系，比如：主从关系、主备关系。

还有就是数据存储类应用，它的多个实例，往往都会在本地磁盘上保存一份数据。而这些实例一旦被杀掉，即便重建出来，实例与数据之间的对应关系也已经丢失，从而导致应用失败。

所以，这种实例之间有不对等关系，以及实例对外部数据有依赖关系的应用，就被称为“有状态应用”（Stateful Application）。

容器技术诞生后，大家很快发现，它用来封装“无状态应用”（Stateless Application），尤其是 Web 服务，非常好用。但是，一旦你想要用容器运行“有状态应用”，其困难程度就会直线上升。而且，这个问题解决起来，单纯依靠容器技术本身已经无能为力，这也就导致了很长一段时间内，“有状态应用”几乎成了容器技术圈子的“忌讳”，大家一听到这个词，就纷纷摇头。

不过，Kubernetes 项目还是成为了“第一个吃螃蟹的人”。

得益于“控制器模式”的设计思想，Kubernetes 项目很早就在 Deployment 的基础上，扩展出了对“有状态应用”的初步支持。这个编排功能，就是：StatefulSet。

StatefulSet 的设计其实非常容易理解。它把真实世界里的应用状态，抽象为了两种情况：

1. **拓扑状态**。这种情况意味着，应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动，比如应用的主节点 A 要先于从节点 B 启动。而如果你把 A 和 B 两个 Pod 删除掉，它们再次被创建出来时也必须严格按照这个顺序才行。并且，新创建出来的 Pod，必须和原来 Pod 的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新 Pod。
2. **存储状态**。这种情况意味着，应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。

所以，**StatefulSet 的核心功能，就是通过某种方式记录这些状态，然后在 Pod 被重新创建时，能够为新 Pod 恢复这些状态。**

在开始讲述 StatefulSet 的工作原理之前，我就必须先为你讲解一个 Kubernetes 项目中非常实用的概念：Headless Service。

我在和你一起讨论 Kubernetes 架构的时候就曾介绍过，Service 是 Kubernetes 项目中用来将一组 Pod 暴露给外界访问的一种机制。比如，一个 Deployment 有 3 个 Pod，那么我就可以定义一个 Service。然后，用户只要能访问到这个 Service，它就能访问到某个具体的 Pod。

那么，这个 Service 又是如何被访问的呢？

**第一种方式，是以 Service 的 VIP（Virtual IP，即：虚拟 IP）方式**。比如：当我访问 10.0.23.1 这个 Service 的 IP 地址时，10.0.23.1 其实就是一个 VIP，它会把请求转发到该 Service 所代理的某一个 Pod 上。这里的具体原理，我会在后续的 Service 章节中进行详细介绍。

**第二种方式，就是以 Service 的 DNS 方式**。比如：这时候，只要我访问“my-svc.my-namespace.svc.cluster.local”这条 DNS 记录，就可以访问到名叫 my-svc 的 Service 所代理的某一个 Pod。

而在第二种 Service DNS 的方式下，具体还可以分为两种处理方法：

第一种处理方法，是 Normal Service。这种情况下，你访问“my-svc.my-namespace.svc.cluster.local”解析到的，正是 my-svc 这个 Service 的 VIP，后面的流程就跟 VIP 方式一致了。

而第二种处理方法，正是 Headless Service。这种情况下，你访问“my-svc.my-namespace.svc.cluster.local”解析到的，直接就是 my-svc 代理的某一个 Pod 的 IP 地址。**可以看到，这里的区别在于，Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址。**

那么，这样的设计又有什么作用呢？

想要回答这个问题，我们需要从 Headless Service 的定义方式看起。

下面是一个标准的 Headless Service 对应的 YAML 文件：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```

可以看到，所谓的 Headless Service，其实仍是一个标准 Service 的 YAML 文件。只不过，它的 clusterIP 字段的值是：None，即：这个 Service，没有一个 VIP 作为“头”。这也就是 Headless 的含义。所以，这个 Service 被创建后并不会被分配一个 VIP，而是会以 DNS 记录的方式暴露出它所代理的 Pod。

而它所代理的 Pod，依然是采用我在前面第 12 篇文章[《牛刀小试：我的第一个容器化应用》](https://time.geekbang.org/column/article/40008)中提到的 Label Selector 机制选择出来的，即：所有携带了 app=nginx 标签的 Pod，都会被这个 Service 代理起来。

然后关键来了。

当你按照这样的方式创建了一个 Headless Service 之后，它所代理的所有 Pod 的 IP 地址，都会被绑定一个这样格式的 DNS 记录，如下所示：

```
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```

这个 DNS 记录，正是 Kubernetes 项目为 Pod 分配的唯一的“可解析身份”（Resolvable Identity）。

有了这个“可解析身份”，只要你知道了一个 Pod 的名字，以及它对应的 Service 的名字，你就可以非常确定地通过这条 DNS 记录访问到 Pod 的 IP 地址。

那么，StatefulSet 又是如何使用这个 DNS 记录来维持 Pod 的拓扑状态的呢？

为了回答这个问题，现在我们就来编写一个 StatefulSet 的 YAML 文件，如下所示：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
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
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
```

这个 YAML 文件，和我们在前面文章中用到的 nginx-deployment 的唯一区别，就是多了一个 serviceName=nginx 字段。

这个字段的作用，就是告诉 StatefulSet 控制器，在执行控制循环（Control Loop）的时候，请使用 nginx 这个 Headless Service 来保证 Pod 的“可解析身份”。

所以，当你通过 kubectl create 创建了上面这个 Service 和 StatefulSet 之后，就会看到如下两个对象：

```
$ kubectl create -f svc.yaml
$ kubectl get service nginx
NAME      TYPE         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx     ClusterIP    None         <none>        80/TCP    10s
 
$ kubectl create -f statefulset.yaml
$ kubectl get statefulset web
NAME      DESIRED   CURRENT   AGE
web       2         1         19s
```

这时候，如果你手比较快的话，还可以通过 kubectl 的 -w 参数，即：Watch 功能，实时查看 StatefulSet 创建两个有状态实例的过程：

> 备注：如果手不够快的话，Pod 很快就创建完了。不过，你依然可以通过这个 StatefulSet 的 Events 看到这些信息。

```
$ kubectl get pods -w -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     0/1       Pending   0          0s
web-0     0/1       Pending   0         0s
web-0     0/1       ContainerCreating   0         0s
web-0     1/1       Running   0         19s
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         20s
```

通过上面这个 Pod 的创建过程，我们不难看到，StatefulSet 给它所管理的所有 Pod 的名字，进行了编号，编号规则是：-。

而且这些编号都是从 0 开始累加，与 StatefulSet 的每个 Pod 实例一一对应，绝不重复。

更重要的是，这些 Pod 的创建，也是严格按照编号顺序进行的。比如，在 web-0 进入到 Running 状态、并且细分状态（Conditions）成为 Ready 之前，web-1 会一直处于 Pending 状态。

> 备注：Ready 状态再一次提醒了我们，为 Pod 设置 livenessProbe 和 readinessProbe 的重要性。

当这两个 Pod 都进入了 Running 状态之后，你就可以查看到它们各自唯一的“网络身份”了。

我们使用 kubectl exec 命令进入到容器中查看它们的 hostname：

```
$ kubectl exec web-0 -- sh -c 'hostname'
web-0
$ kubectl exec web-1 -- sh -c 'hostname'
web-1
```

可以看到，这两个 Pod 的 hostname 与 Pod 名字是一致的，都被分配了对应的编号。接下来，我们再试着以 DNS 的方式，访问一下这个 Headless Service：

```
$ kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh 
```

通过这条命令，我们启动了一个一次性的 Pod，因为–rm 意味着 Pod 退出后就会被删除掉。然后，在这个 Pod 的容器里面，我们尝试用 nslookup 命令，解析一下 Pod 对应的 Headless Service：

```
$ kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh
$ nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local
 
Name:      web-0.nginx
Address 1: 10.244.1.7
 
$ nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local
 
Name:      web-1.nginx
Address 1: 10.244.2.7
```

从 nslookup 命令的输出结果中，我们可以看到，在访问 web-0.nginx 的时候，最后解析到的，正是 web-0 这个 Pod 的 IP 地址；而当访问 web-1.nginx 的时候，解析到的则是 web-1 的 IP 地址。

这时候，如果你在另外一个 Terminal 里把这两个“有状态应用”的 Pod 删掉：

```
$ kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted
```

然后，再在当前 Terminal 里 Watch 一下这两个 Pod 的状态变化，就会发现一个有趣的现象：

```
$ kubectl get pod -w -l app=nginx
NAME      READY     STATUS              RESTARTS   AGE
web-0     0/1       ContainerCreating   0          0s
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          2s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         32s
```

可以看到，当我们把这两个 Pod 删除之后，Kubernetes 会按照原先编号的顺序，创建出了两个新的 Pod。并且，Kubernetes 依然为它们分配了与原来相同的“网络身份”：web-0.nginx 和 web-1.nginx。

通过这种严格的对应规则，**StatefulSet 就保证了 Pod 网络标识的稳定性**。

比如，如果 web-0 是一个需要先启动的主节点，web-1 是一个后启动的从节点，那么只要这个 StatefulSet 不被删除，你访问 web-0.nginx 时始终都会落在主节点上，访问 web-1.nginx 时，则始终都会落在从节点上，这个关系绝对不会发生任何变化。

所以，如果我们再用 nslookup 命令，查看一下这个新 Pod 对应的 Headless Service 的话：

```
$ kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh 
$ nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local
 
Name:      web-0.nginx
Address 1: 10.244.1.8
 
$ nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local
 
Name:      web-1.nginx
Address 1: 10.244.2.8
```

我们可以看到，在这个 StatefulSet 中，这两个新 Pod 的“网络标识”（比如：web-0.nginx 和 web-1.nginx），再次解析到了正确的 IP 地址（比如：web-0 Pod 的 IP 地址 10.244.1.8）。

通过这种方法，**Kubernetes 就成功地将 Pod 的拓扑状态（比如：哪个节点先启动，哪个节点后启动），按照 Pod 的“名字 + 编号”的方式固定了下来**。此外，Kubernetes 还为每一个 Pod 提供了一个固定并且唯一的访问入口，即：这个 Pod 对应的 DNS 记录。

这些状态，在 StatefulSet 的整个生命周期里都会保持不变，绝不会因为对应 Pod 的删除或者重新创建而失效。

不过，相信你也已经注意到了，尽管 web-0.nginx 这条记录本身不会变，但它解析到的 Pod 的 IP 地址，并不是固定的。这就意味着，对于“有状态应用”实例的访问，你必须使用 DNS 记录或者 hostname 的方式，而绝不应该直接访问这些 Pod 的 IP 地址。

### 总结

在今天这篇文章中，我首先和你分享了 StatefulSet 的基本概念，解释了什么是应用的“状态”。

紧接着 ，我为你分析了 StatefulSet 如何保证应用实例之间“拓扑状态”的稳定性。

如果用一句话来总结的话，你可以这么理解这个过程：

> StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作。而当 StatefulSet 的“控制循环”发现 Pod 的“实际状态”与“期望状态”不一致，需要新建或者删除 Pod 进行“调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作。

所以，StatefulSet 其实可以认为是对 Deployment 的改良。

与此同时，通过 Headless Service 的方式，StatefulSet 为每个 Pod 创建了一个固定并且稳定的 DNS 记录，来作为它的访问入口。

实际上，在部署“有状态应用”的时候，应用的每个实例拥有唯一并且稳定的“网络标识”，是一个非常重要的假设。

在下一篇文章中，我将会继续为你剖析 StatefulSet 如何处理存储状态。

## 存储状态

在上一篇文章中，我和你分享了 StatefulSet 如何保证应用实例的拓扑状态，在 Pod 删除和再创建的过程中保持稳定。

而在今天这篇文章中，我将继续为你解读 StatefulSet 对存储状态的管理机制。这个机制，主要使用的是一个叫作 Persistent Volume Claim 的功能。

在前面介绍 Pod 的时候，我曾提到过，要在一个 Pod 里声明 Volume，只要在 Pod 里加上 spec.volumes 字段即可。然后，你就可以在这个字段里定义一个具体类型的 Volume 了，比如：hostPath。

可是，你有没有想过这样一个场景：**如果你并不知道有哪些 Volume 类型可以用，要怎么办呢**？

更具体地说，作为一个应用开发者，我可能对持久化存储项目（比如 Ceph、GlusterFS 等）一窍不通，也不知道公司的 Kubernetes 集群里到底是怎么搭建出来的，我也自然不会编写它们对应的 Volume 定义文件。

所谓“术业有专攻”，这些关于 Volume 的管理和远程持久化存储的知识，不仅超越了开发者的知识储备，还会有暴露公司基础设施秘密的风险。

比如，下面这个例子，就是一个声明了 Ceph RBD 类型 Volume 的 Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rbd
spec:
  containers:
    - image: kubernetes/pause
      name: rbd-rw
      volumeMounts:
      - name: rbdpd
        mountPath: /mnt/rbd
  volumes:
    - name: rbdpd
      rbd:
        monitors:
        - '10.16.154.78:6789'
        - '10.16.154.82:6789'
        - '10.16.154.83:6789'
        pool: kube
        image: foo
        fsType: ext4
        readOnly: true
        user: admin
        keyring: /etc/ceph/keyring
        imageformat: "2"
        imagefeatures: "layering"
```

其一，如果不懂得 Ceph RBD 的使用方法，那么这个 Pod 里 Volumes 字段，你十有八九也完全看不懂。其二，这个 Ceph RBD 对应的存储服务器的地址、用户名、授权文件的位置，也都被轻易地暴露给了全公司的所有开发人员，这是一个典型的信息被“过度暴露”的例子。

这也是为什么，在后来的演化中，**Kubernetes 项目引入了一组叫作 Persistent Volume Claim（PVC）和 Persistent Volume（PV）的 API 对象，大大降低了用户声明和使用持久化 Volume 的门槛。**

举个例子，有了 PVC 之后，一个开发人员想要使用一个 Volume，只需要简单的两步即可。

**第一步：定义一个 PVC，声明想要的 Volume 的属性：**

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

可以看到，在这个 PVC 对象里，不需要任何关于 Volume 细节的字段，只有描述性的属性和定义。比如，storage: 1Gi，表示我想要的 Volume 大小至少是 1 GiB；accessModes: ReadWriteOnce，表示这个 Volume 的挂载方式是可读写，并且只能被挂载在一个节点上而非被多个节点共享。

> 备注：关于哪种类型的 Volume 支持哪种类型的 AccessMode，你可以查看 Kubernetes 项目官方文档中的[详细列表](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)。

**第二步：在应用的 Pod 中，声明使用这个 PVC：**

```
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
    - name: pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-storage
  volumes:
    - name: pv-storage
      persistentVolumeClaim:
        claimName: pv-claim
```

可以看到，在这个 Pod 的 Volumes 定义中，我们只需要声明它的类型是 persistentVolumeClaim，然后指定 PVC 的名字，而完全不必关心 Volume 本身的定义。

这时候，只要我们创建这个 PVC 对象，Kubernetes 就会自动为它绑定一个符合条件的 Volume。可是，这些符合条件的 Volume 又是从哪里来的呢？

答案是，它们来自于由运维人员维护的 PV（Persistent Volume）对象。接下来，我们一起看一个常见的 PV 对象的 YAML 文件：

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-volume
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  rbd:
    monitors:
    - '10.16.154.78:6789'
    - '10.16.154.82:6789'
    - '10.16.154.83:6789'
    pool: kube
    image: foo
    fsType: ext4
    readOnly: true
    user: admin
    keyring: /etc/ceph/keyring
    imageformat: "2"
    imagefeatures: "layering"
```

可以看到，这个 PV 对象的 spec.rbd 字段，正是我们前面介绍过的 Ceph RBD Volume 的详细定义。而且，它还声明了这个 PV 的容量是 10 GiB。这样，Kubernetes 就会为我们刚刚创建的 PVC 对象绑定这个 PV。

所以，Kubernetes 中 PVC 和 PV 的设计，**实际上类似于“接口”和“实现”的思想**。开发者只要知道并会使用“接口”，即：PVC；而运维人员则负责给“接口”绑定具体的实现，即：PV。

这种解耦，就避免了因为向开发者暴露过多的存储系统细节而带来的隐患。此外，这种职责的分离，往往也意味着出现事故时可以更容易定位问题和明确责任，从而避免“扯皮”现象的出现。

而 PVC、PV 的设计，也使得 StatefulSet 对存储状态的管理成为了可能。我们还是以上一篇文章中用到的 StatefulSet 为例：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
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
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```

这次，我们为这个 StatefulSet 额外添加了一个 volumeClaimTemplates 字段。从名字就可以看出来，它跟 Deployment 里 Pod 模板（PodTemplate）的作用类似。也就是说，凡是被这个 StatefulSet 管理的 Pod，都会声明一个对应的 PVC；而这个 PVC 的定义，就来自于 volumeClaimTemplates 这个模板字段。更重要的是，这个 PVC 的名字，会被分配一个与这个 Pod 完全一致的编号。

这个自动创建的 PVC，与 PV 绑定成功后，就会进入 Bound 状态，这就意味着这个 Pod 可以挂载并使用这个 PV 了。

如果你还是不太理解 PVC 的话，可以先记住这样一个结论：**PVC 其实就是一种特殊的 Volume**。只不过一个 PVC 具体是什么类型的 Volume，要在跟某个 PV 绑定之后才知道。关于 PV、PVC 更详细的知识，我会在容器存储部分做进一步解读。

当然，PVC 与 PV 的绑定得以实现的前提是，运维人员已经在系统里创建好了符合条件的 PV（比如，我们在前面用到的 pv-volume）；或者，你的 Kubernetes 集群运行在公有云上，这样 Kubernetes 就会通过 Dynamic Provisioning 的方式，自动为你创建与 PVC 匹配的 PV。

所以，我们在使用 kubectl create 创建了 StatefulSet 之后，就会看到 Kubernetes 集群里出现了两个 PVC：

```
$ kubectl create -f statefulset.yaml
$ kubectl get pvc -l app=nginx
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   AGE
www-web-0   Bound     pvc-15c268c7-b507-11e6-932f-42010a800002   1Gi        RWO           48s
www-web-1   Bound     pvc-15c79307-b507-11e6-932f-42010a800002   1Gi        RWO           48s
```

可以看到，这些 PVC，都以“<PVC 名字 >-<StatefulSet 名字 >-< 编号 >”的方式命名，并且处于 Bound 状态。

我们前面已经讲到过，这个 StatefulSet 创建出来的所有 Pod，都会声明使用编号的 PVC。比如，在名叫 web-0 的 Pod 的 volumes 字段，它会声明使用名叫 www-web-0 的 PVC，从而挂载到这个 PVC 所绑定的 PV。

所以，我们就可以使用如下所示的指令，在 Pod 的 Volume 目录里写入一个文件，来验证一下上述 Volume 的分配情况：

```shell
$ for i in 0 1; do kubectl exec web-$i -- sh -c 'echo hello $(hostname) > /usr/share/nginx/html/index.html'; done
```

如上所示，通过 kubectl exec 指令，我们在每个 Pod 的 Volume 目录里，写入了一个 index.html 文件。这个文件的内容，正是 Pod 的 hostname。比如，我们在 web-0 的 index.html 里写入的内容就是 "hello web-0"。

此时，如果你在这个 Pod 容器里访问`“http://localhost”`，你实际访问到的就是 Pod 里 Nginx 服务器进程，而它会为你返回 /usr/share/nginx/html/index.html 里的内容。这个操作的执行方法如下所示：

```
$ for i in 0 1; do kubectl exec -it web-$i -- curl localhost; done
hello web-0
hello web-1
```

现在，关键来了。

如果你使用 kubectl delete 命令删除这两个 Pod，这些 Volume 里的文件会不会丢失呢？

```
$ kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted
```

可以看到，正如我们前面介绍过的，在被删除之后，这两个 Pod 会被按照编号的顺序被重新创建出来。而这时候，如果你在新创建的容器里通过访问`“http://localhost”`的方式去访问 web-0 里的 Nginx 服务：

```
# 在被重新创建出来的 Pod 容器里访问 http://localhost
$ kubectl exec -it web-0 -- curl localhost
hello web-0
```

就会发现，这个请求依然会返回：hello web-0。也就是说，原先与名叫 web-0 的 Pod 绑定的 PV，在这个 Pod 被重新创建之后，依然同新的名叫 web-0 的 Pod 绑定在了一起。对于 Pod web-1 来说，也是完全一样的情况。

**这是怎么做到的呢？**

其实，我和你分析一下 StatefulSet 控制器恢复这个 Pod 的过程，你就可以很容易理解了。

首先，当你把一个 Pod，比如 web-0，删除之后，这个 Pod 对应的 PVC 和 PV，并不会被删除，而这个 Volume 里已经写入的数据，也依然会保存在远程存储服务里（比如，我们在这个例子里用到的 Ceph 服务器）。

此时，StatefulSet 控制器发现，一个名叫 web-0 的 Pod 消失了。所以，控制器就会重新创建一个新的、名字还是叫作 web-0 的 Pod 来，“纠正”这个不一致的情况。

需要注意的是，在这个新的 Pod 对象的定义里，它声明使用的 PVC 的名字，还是叫作：www-web-0。这个 PVC 的定义，还是来自于 PVC 模板（volumeClaimTemplates），这是 StatefulSet 创建 Pod 的标准流程。

所以，在这个新的 web-0 Pod 被创建出来之后，Kubernetes 为它查找名叫 www-web-0 的 PVC 时，就会直接找到旧 Pod 遗留下来的同名的 PVC，进而找到跟这个 PVC 绑定在一起的 PV。

这样，新的 Pod 就可以挂载到旧 Pod 对应的那个 Volume，并且获取到保存在 Volume 里的数据。

**通过这种方式，Kubernetes 的 StatefulSet 就实现了对应用存储状态的管理。**

看到这里，你是不是已经大致理解了 StatefulSet 的工作原理呢？现在，我再为你详细梳理一下吧。

**首先，StatefulSet 的控制器直接管理的是 Pod**。这是因为，StatefulSet 里的不同 Pod 实例，不再像 ReplicaSet 中那样都是完全一样的，而是有了细微区别的。比如，每个 Pod 的 hostname、名字等都是不同的、携带了编号的。而 StatefulSet 区分这些实例的方式，就是通过在 Pod 的名字里加上事先约定好的编号。

**其次，Kubernetes 通过 Headless Service，为这些有编号的 Pod，在 DNS 服务器中生成带有同样编号的 DNS 记录**。只要 StatefulSet 能够保证这些 Pod 名字里的编号不变，那么 Service 里类似于 web-0.nginx.default.svc.cluster.local 这样的 DNS 记录也就不会变，而这条记录解析出来的 Pod 的 IP 地址，则会随着后端 Pod 的删除和再创建而自动更新。这当然是 Service 机制本身的能力，不需要 StatefulSet 操心。

**最后，StatefulSet 还为每一个 Pod 分配并创建一个同样编号的 PVC**。这样，Kubernetes 就可以通过 Persistent Volume 机制为这个 PVC 绑定上对应的 PV，从而保证了每一个 Pod 都拥有一个独立的 Volume。

在这种情况下，即使 Pod 被删除，它所对应的 PVC 和 PV 依然会保留下来。所以当这个 Pod 被重新创建出来之后，Kubernetes 会为它找到同样编号的 PVC，挂载这个 PVC 对应的 Volume，从而获取到以前保存在 Volume 里的数据。

这么一看，原本非常复杂的 StatefulSet，是不是也很容易理解了呢？

### 总结

在今天这篇文章中，我为你详细介绍了 StatefulSet 处理存储状态的方法。然后，以此为基础，我为你梳理了 StatefulSet 控制器的工作原理。

从这些讲述中，我们不难看出 StatefulSet 的设计思想：StatefulSet 其实就是一种特殊的 Deployment，而其独特之处在于，它的每个 Pod 都被编号了。而且，这个编号会体现在 Pod 的名字和 hostname 等标识信息上，这不仅代表了 Pod 的创建顺序，也是 Pod 的重要网络标识（即：在整个集群里唯一的、可被的访问身份）。

有了这个编号后，StatefulSet 就使用 Kubernetes 里的两个标准功能：Headless Service 和 PV/PVC，实现了对 Pod 的拓扑状态和存储状态的维护。

实际上，在下一篇文章的“有状态应用”实践环节，以及后续的讲解中，你就会逐渐意识到，StatefulSet 可以说是 Kubernetes 中作业编排的“集大成者”。

因为，几乎每一种 Kubernetes 的编排功能，都可以在编写 StatefulSet 的 YAML 文件时被用到。

## 有状态应用实践

在前面的两篇文章中，我详细讲解了 StatefulSet 的工作原理，以及处理拓扑状态和存储状态的方法。而在今天这篇文章中，我将通过一个实际的例子，再次为你深入解读一下部署一个 StatefulSet 的完整流程。

今天我选择的实例是部署一个 MySQL 集群，这也是 Kubernetes 官方文档里的一个经典案例。但是，很多工程师都曾向我吐槽说这个例子“完全看不懂”。

其实，这样的吐槽也可以理解：相比于 Etcd、Cassandra 等“原生”就考虑了分布式需求的项目，MySQL 以及很多其他的数据库项目，在分布式集群的搭建上并不友好，甚至有点“原始”。

所以，这次我就直接选择了这个具有挑战性的例子，和你分享如何使用 StatefulSet 将它的集群搭建过程“容器化”。

> 备注：在开始实践之前，请确保我们之前一起部署的那个 Kubernetes 集群还是可用的，并且网络插件和存储插件都能正常运行。

首先，用自然语言来描述一下我们想要部署的“有状态应用”。

1. 是一个“主从复制”（Maser-Slave Replication）的 MySQL 集群；
2. 有 1 个主节点（Master）；
3. 有多个从节点（Slave）；
4. 从节点需要能水平扩展；
5. 所有的写操作，只能在主节点上执行；
6. 读操作可以在所有节点上执行。

这就是一个非常典型的主从模式的 MySQL 集群了。我们可以把上面描述的“有状态应用”的需求，通过一张图来表示。

![img](data:image/png;base64,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)
在常规环境里，部署这样一个主从模式的 MySQL 集群的主要难点在于：如何让从节点能够拥有主节点的数据，即：如何配置主（Master）从（Slave）节点的复制与同步。

所以，在安装好 MySQL 的 Master 节点之后，你需要做的第一步工作，就是**通过 XtraBackup 将 Master 节点的数据备份到指定目录。**

> 备注：XtraBackup 是业界主要使用的开源 MySQL 备份和恢复工具。

这一步会自动在目标目录里生成一个备份信息文件，名叫：xtrabackup_binlog_info。这个文件一般会包含如下两个信息：

```
$ cat xtrabackup_binlog_info
TheMaster-bin.000001     481
```

这两个信息会在接下来配置 Slave 节点的时候用到。

**第二步：配置 Slave 节点**。Slave 节点在第一次启动前，需要先把 Master 节点的备份数据，连同备份信息文件，一起拷贝到自己的数据目录（/var/lib/mysql）下。然后，我们执行这样一句 SQL：

```
TheSlave|mysql> CHANGE MASTER TO
                MASTER_HOST='$masterip',
                MASTER_USER='xxx',
                MASTER_PASSWORD='xxx',
                MASTER_LOG_FILE='TheMaster-bin.000001',
                MASTER_LOG_POS=481;
```

其中，MASTER_LOG_FILE 和 MASTER_LOG_POS，就是该备份对应的二进制日志（Binary Log）文件的名称和开始的位置（偏移量），也正是 xtrabackup_binlog_info 文件里的那两部分内容（即：TheMaster-bin.000001 和 481）。

**第三步，启动 Slave 节点**。在这一步，我们需要执行这样一句 SQL：

```
TheSlave|mysql> START SLAVE;
```

这样，Slave 节点就启动了。它会使用备份信息文件中的二进制日志文件和偏移量，与主节点进行数据同步。

**第四步，在这个集群中添加更多的 Slave 节点**。

需要注意的是，新添加的 Slave 节点的备份数据，来自于已经存在的 Slave 节点。

所以，在这一步，我们需要将 Slave 节点的数据备份在指定目录。而这个备份操作会自动生成另一种备份信息文件，名叫：xtrabackup_slave_info。同样地，这个文件也包含了 MASTER_LOG_FILE 和 MASTER_LOG_POS 两个字段。

然后，我们就可以执行跟前面一样的“CHANGE MASTER TO”和“START SLAVE” 指令，来初始化并启动这个新的 Slave 节点了。

通过上面的叙述，我们不难看到，**将部署 MySQL 集群的流程迁移到 Kubernetes 项目上，需要能够“容器化”地解决下面的“三座大山”：**

1. Master 节点和 Slave 节点需要有不同的配置文件（即：不同的 my.cnf）；
2. Master 节点和 Salve 节点需要能够传输备份信息文件；
3. 在 Slave 节点第一次启动之前，需要执行一些初始化 SQL 操作；

而由于 MySQL 本身同时拥有拓扑状态（主从节点的区别）和存储状态（MySQL 保存在本地的数据），我们自然要通过 StatefulSet 来解决这“三座大山”的问题。

**其中，“第一座大山：Master 节点和 Slave 节点需要有不同的配置文件”，很容易处理**：我们只需要给主从节点分别准备两份不同的 MySQL 配置文件，然后根据 Pod 的序号（Index）挂载进去即可。

正如我在前面文章中介绍过的，这样的配置文件信息，应该保存在 ConfigMap 里供 Pod 使用。它的定义如下所示：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    # 主节点 MySQL 的配置文件
    [mysqld]
    log-bin
  slave.cnf: |
    # 从节点 MySQL 的配置文件
    [mysqld]
    super-read-only
```

在这里，我们定义了 master.cnf 和 slave.cnf 两个 MySQL 的配置文件。

- master.cnf 开启了 log-bin，即：使用二进制日志文件的方式进行主从复制，这是一个标准的设置。
- slave.cnf 的开启了 super-read-only，代表的是从节点会拒绝除了主节点的数据同步操作之外的所有写操作，即：它对用户是只读的。

而上述 ConfigMap 定义里的 data 部分，是 Key-Value 格式的。比如，master.cnf 就是这份配置数据的 Key，而“|”后面的内容，就是这份配置数据的 Value。这份数据将来挂载进 Master 节点对应的 Pod 后，就会在 Volume 目录里生成一个叫作 master.cnf 的文件。

> ConfigMap 跟 Secret，无论是使用方法还是实现原理，几乎都是一样的。

接下来，我们需要创建两个 Service 来供 StatefulSet 以及用户使用。这两个 Service 的定义如下所示：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```

可以看到，这两个 Service 都代理了所有携带 app=mysql 标签的 Pod，也就是所有的 MySQL Pod。端口映射都是用 Service 的 3306 端口对应 Pod 的 3306 端口。

不同的是，第一个名叫“mysql”的 Service 是一个 Headless Service（即：clusterIP= None）。所以它的作用，是通过为 Pod 分配 DNS 记录来固定它的拓扑状态，比如“mysql-0.mysql”和“mysql-1.mysql”这样的 DNS 名字。其中，编号为 0 的节点就是我们的主节点。

而第二个名叫“mysql-read”的 Service，则是一个常规的 Service。

并且我们规定，所有用户的读请求，都必须访问第二个 Service 被自动分配的 DNS 记录，即：“mysql-read”（当然，也可以访问这个 Service 的 VIP）。这样，读请求就可以被转发到任意一个 MySQL 的主节点或者从节点上。

> 备注：Kubernetes 中的所有 Service、Pod 对象，都会被自动分配同名的 DNS 记录。具体细节，我会在后面 Service 部分做重点讲解。

而所有用户的写请求，则必须直接以 DNS 记录的方式访问到 MySQL 的主节点，也就是：“mysql-0.mysql“这条 DNS 记录。

接下来，我们再一起解决“第二座大山：Master 节点和 Salve 节点需要能够传输备份文件”的问题。

**翻越这座大山的思路，我比较推荐的做法是：先搭建框架，再完善细节。其中，Pod 部分如何定义，是完善细节时的重点。**

**所以首先，我们先为 StatefulSet 对象规划一个大致的框架，如下图所示：**

![img](data:image/png;base64,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)

在这一步，我们可以先为 StatefulSet 定义一些通用的字段。

比如：selector 表示，这个 StatefulSet 要管理的 Pod 必须携带 app=mysql 标签；它声明要使用的 Headless Service 的名字是：mysql。

这个 StatefulSet 的 replicas 值是 3，表示它定义的 MySQL 集群有三个节点：一个 Master 节点，两个 Slave 节点。

可以看到，StatefulSet 管理的“有状态应用”的多个实例，也都是通过同一份 Pod 模板创建出来的，使用的是同一个 Docker 镜像。这也就意味着：如果你的应用要求不同节点的镜像不一样，那就不能再使用 StatefulSet 了。对于这种情况，应该考虑我后面会讲解到的 Operator。

除了这些基本的字段外，作为一个有存储状态的 MySQL 集群，StatefulSet 还需要管理存储状态。所以，我们需要通过 volumeClaimTemplate（PVC 模板）来为每个 Pod 定义 PVC。比如，这个 PVC 模板的 resources.requests.strorage 指定了存储的大小为 10 GiB；ReadWriteOnce 指定了该存储的属性为可读写，并且一个 PV 只允许挂载在一个宿主机上。将来，这个 PV 对应的的 Volume 就会充当 MySQL Pod 的存储数据目录。

**然后，我们来重点设计一下这个 StatefulSet 的 Pod 模板，也就是 template 字段。**

由于 StatefulSet 管理的 Pod 都来自于同一个镜像，这就要求我们在编写 Pod 时，一定要保持清醒，用“人格分裂”的方式进行思考：

1. 如果这个 Pod 是 Master 节点，我们要怎么做；
2. 如果这个 Pod 是 Slave 节点，我们又要怎么做。

想清楚这两个问题，我们就可以按照 Pod 的启动过程来一步步定义它们了。

**第一步：从 ConfigMap 中，获取 MySQL 的 Pod 对应的配置文件。**

为此，我们需要进行一个初始化操作，根据节点的角色是 Master 还是 Slave 节点，为 Pod 分配对应的配置文件。此外，MySQL 还要求集群里的每个节点都有一个唯一的 ID 文件，名叫 server-id.cnf。

而根据我们已经掌握的 Pod 知识，这些初始化操作显然适合通过 InitContainer 来完成。所以，我们首先定义了一个 InitContainer，如下所示：

```yaml
      ...
      # template.spec
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 从 Pod 的序号，生成 server-id
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # 由于 server-id=0 有特殊含义，我们给 ID 加一个 100 来避开它
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # 如果 Pod 序号是 0，说明它是 Master 节点，从 ConfigMap 里把 Master 的配置文件拷贝到 /mnt/conf.d/ 目录；
          # 否则，拷贝 Slave 的配置文件
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
```

在这个名叫 init-mysql 的 InitContainer 的配置中，它从 Pod 的 hostname 里，读取到了 Pod 的序号，以此作为 MySQL 节点的 server-id。

然后，init-mysql 通过这个序号，判断当前 Pod 到底是 Master 节点（即：序号为 0）还是 Slave 节点（即：序号不为 0），从而把对应的配置文件从 /mnt/config-map 目录拷贝到 /mnt/conf.d/ 目录下。

其中，文件拷贝的源目录 /mnt/config-map，正是 ConfigMap 在这个 Pod 的 Volume，如下所示：

```yaml
      ...
      # template.spec
      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql
```

通过这个定义，init-mysql 在声明了挂载 config-map 这个 Volume 之后，ConfigMap 里保存的内容，就会以文件的方式出现在它的 /mnt/config-map 目录当中。

而文件拷贝的目标目录，即容器里的 /mnt/conf.d/ 目录，对应的则是一个名叫 conf 的、emptyDir 类型的 Volume。基于 Pod Volume 共享的原理，当 InitContainer 复制完配置文件退出后，后面启动的 MySQL 容器只需要直接声明挂载这个名叫 conf 的 Volume，它所需要的.cnf 配置文件已经出现在里面了。这跟我们之前介绍的 Tomcat 和 WAR 包的处理方法是完全一样的。

**第二步：在 Slave Pod 启动前，从 Master 或者其他 Slave Pod 里拷贝数据库数据到自己的目录下。**

为了实现这个操作，我们就需要再定义第二个 InitContainer，如下所示：

```yaml
      ...
      # template.spec.initContainers
      - name: clone-mysql
        image: gcr.io/google-samples/xtrabackup:1.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 拷贝操作只需要在第一次启动时进行，所以如果数据已经存在，跳过
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Master 节点 (序号为 0) 不需要做这个操作
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # 使用 ncat 指令，远程地从前一个节点拷贝数据到本地
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # 执行 --prepare，这样拷贝来的数据就可以用作恢复了
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
```

在这个名叫 clone-mysql 的 InitContainer 里，我们使用的是 xtrabackup 镜像（它里面安装了 xtrabackup 工具）。

而在它的启动命令里，我们首先做了一个判断。即：当初始化所需的数据（/var/lib/mysql/mysql 目录）已经存在，或者当前 Pod 是 Master 节点的时候，不需要做拷贝操作。

接下来，clone-mysql 会使用 Linux 自带的 ncat 指令，向 DNS 记录为“mysql-< 当前序号减一 >.mysql”的 Pod，也就是当前 Pod 的前一个 Pod，发起数据传输请求，并且直接用 xbstream 指令将收到的备份数据保存在 /var/lib/mysql 目录下。

> 备注：3307 是一个特殊端口，运行着一个专门负责备份 MySQL 数据的辅助进程。我们后面马上会讲到它。

当然，这一步你可以随意选择用自己喜欢的方法来传输数据。比如，用 scp 或者 rsync，都没问题。

你可能已经注意到，这个容器里的 /var/lib/mysql 目录，**实际上正是一个名为 data 的 PVC**，即：我们在前面声明的持久化存储。

这就可以保证，哪怕宿主机宕机了，我们数据库的数据也不会丢失。更重要的是，由于 Pod Volume 是被 Pod 里的容器共享的，所以后面启动的 MySQL 容器，就可以把这个 Volume 挂载到自己的 /var/lib/mysql 目录下，直接使用里面的备份数据进行恢复操作。

不过，clone-mysql 容器还要对 /var/lib/mysql 目录，执行一句 xtrabackup --prepare 操作，目的是让拷贝来的数据进入一致性状态，这样，这些数据才能被用作数据恢复。

至此，我们就通过 InitContainer 完成了对“主、从节点间备份文件传输”操作的处理过程，也就是翻越了“第二座大山”。

接下来，我们可以开始定义 MySQL 容器, 启动 MySQL 服务了。由于 StatefulSet 里的所有 Pod 都来自用同一个 Pod 模板，所以我们还要“人格分裂”地去思考：这个 MySQL 容器的启动命令，在 Master 和 Slave 两种情况下有什么不同。

有了 Docker 镜像，在 Pod 里声明一个 Master 角色的 MySQL 容器并不是什么困难的事情：直接执行 MySQL 启动命令即可。

但是，如果这个 Pod 是一个第一次启动的 Slave 节点，在执行 MySQL 启动命令之前，它就需要使用前面 InitContainer 拷贝来的备份数据进行初始化。

可是，别忘了，**容器是一个单进程模型。**

所以，一个 Slave 角色的 MySQL 容器启动之前，谁能负责给它执行初始化的 SQL 语句呢？

这就是我们需要解决的“第三座大山”的问题，即：如何在 Slave 节点的 MySQL 容器第一次启动之前，执行初始化 SQL。

你可能已经想到了，我们可以为这个 MySQL 容器额外定义一个 sidecar 容器，来完成这个操作，它的定义如下所示：

```yaml
      ...
      # template.spec.containers
      - name: xtrabackup
        image: gcr.io/google-samples/xtrabackup:1.0
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql
          
          # 从备份信息文件里读取 MASTER_LOG_FILEM 和 MASTER_LOG_POS 这两个字段的值，用来拼装集群初始化 SQL
          if [[ -f xtrabackup_slave_info ]]; then
            # 如果 xtrabackup_slave_info 文件存在，说明这个备份数据来自于另一个 Slave 节点。这种情况下，XtraBackup 工具在备份的时候，就已经在这个文件里自动生成了 "CHANGE MASTER TO" SQL 语句。所以，我们只需要把这个文件重命名为 change_master_to.sql.in，后面直接使用即可
            mv xtrabackup_slave_info change_master_to.sql.in
            # 所以，也就用不着 xtrabackup_binlog_info 了
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # 如果只存在 xtrabackup_binlog_inf 文件，那说明备份来自于 Master 节点，我们就需要解析这个备份信息文件，读取所需的两个字段的值
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            # 把两个字段的值拼装成 SQL，写入 change_master_to.sql.in 文件
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi
          
          # 如果 change_master_to.sql.in，就意味着需要做集群初始化工作
          if [[ -f change_master_to.sql.in ]]; then
            # 但一定要先等 MySQL 容器启动之后才能进行下一步连接 MySQL 的操作
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done
            
            echo "Initializing replication from clone position"
            # 将文件 change_master_to.sql.in 改个名字，防止这个 Container 重启的时候，因为又找到了 change_master_to.sql.in，从而重复执行一遍这个初始化流程
            mv change_master_to.sql.in change_master_to.sql.orig
            # 使用 change_master_to.sql.orig 的内容，也是就是前面拼装的 SQL，组成一个完整的初始化和启动 Slave 的 SQL 语句
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi
          
          # 使用 ncat 监听 3307 端口。它的作用是，在收到传输请求的时候，直接执行 "xtrabackup --backup" 命令，备份 MySQL 的数据并发送给请求者
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
```

可以看到，**在这个名叫 xtrabackup 的 sidecar 容器的启动命令里，其实实现了两部分工作。**

**第一部分工作，当然是 MySQL 节点的初始化工作**。这个初始化需要使用的 SQL，是 sidecar 容器拼装出来、保存在一个名为 change_master_to.sql.in 的文件里的，具体过程如下所示：

sidecar 容器首先会判断当前 Pod 的 /var/lib/mysql 目录下，是否有 xtrabackup_slave_info 这个备份信息文件。

- 如果有，则说明这个目录下的备份数据是由一个 Slave 节点生成的。这种情况下，XtraBackup 工具在备份的时候，就已经在这个文件里自动生成了 "CHANGE MASTER TO" SQL 语句。所以，我们只需要把这个文件重命名为 change_master_to.sql.in，后面直接使用即可。
- 如果没有 xtrabackup_slave_info 文件、但是存在 xtrabackup_binlog_info 文件，那就说明备份数据来自于 Master 节点。这种情况下，sidecar 容器就需要解析这个备份信息文件，读取 MASTER_LOG_FILE 和 MASTER_LOG_POS 这两个字段的值，用它们拼装出初始化 SQL 语句，然后把这句 SQL 写入到 change_master_to.sql.in 文件中。

接下来，sidecar 容器就可以执行初始化了。从上面的叙述中可以看到，只要这个 change_master_to.sql.in 文件存在，那就说明接下来需要进行集群初始化操作。

所以，这时候，sidecar 容器只需要读取并执行 change_master_to.sql.in 里面的“CHANGE MASTER TO”指令，再执行一句 START SLAVE 命令，一个 Slave 节点就被成功启动了。

> 需要注意的是：Pod 里的容器并没有先后顺序，所以在执行初始化 SQL 之前，必须先执行一句 SQL（select 1）来检查一下 MySQL 服务是否已经可用。

当然，上述这些初始化操作完成后，我们还要删除掉前面用到的这些备份信息文件。否则，下次这个容器重启时，就会发现这些文件存在，所以又会重新执行一次数据恢复和集群初始化的操作，这是不对的。

同理，change_master_to.sql.in 在使用后也要被重命名，以免容器重启时因为发现这个文件存在又执行一遍初始化。

**在完成 MySQL 节点的初始化后，这个 sidecar 容器的第二个工作，则是启动一个数据传输服务。**

具体做法是：sidecar 容器会使用 ncat 命令启动一个工作在 3307 端口上的网络发送服务。一旦收到数据传输请求时，sidecar 容器就会调用 xtrabackup --backup 指令备份当前 MySQL 的数据，然后把这些备份数据返回给请求者。这就是为什么我们在 InitContainer 里定义数据拷贝的时候，访问的是“上一个 MySQL 节点”的 3307 端口。

值得一提的是，由于 sidecar 容器和 MySQL 容器同处于一个 Pod 里，所以它是直接通过 Localhost 来访问和备份 MySQL 容器里的数据的，非常方便。

同样地，我在这里举例用的只是一种备份方法而已，你完全可以选择其他自己喜欢的方案。比如，你可以使用 innobackupex 命令做数据备份和准备，它的使用方法几乎与本文的备份方法一样。

至此，我们也就翻越了“第三座大山”，完成了 Slave 节点第一次启动前的初始化工作。

扳倒了这“三座大山”后，我们终于可以定义 Pod 里的主角，MySQL 容器了。有了前面这些定义和初始化工作，MySQL 容器本身的定义就非常简单了，如下所示：

```yaml
      ...
      # template.spec
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # 通过 TCP 连接的方式进行健康检查
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
```

在这个容器的定义里，我们使用了一个标准的 MySQL 5.7 的官方镜像。它的数据目录是 /var/lib/mysql，配置文件目录是 /etc/mysql/conf.d。

这时候，你应该能够明白，如果 MySQL 容器是 Slave 节点的话，它的数据目录里的数据，就来自于 InitContainer 从其他节点里拷贝而来的备份。它的配置文件目录 /etc/mysql/conf.d 里的内容，则来自于 ConfigMap 对应的 Volume。而它的初始化工作，则是由同一个 Pod 里的 sidecar 容器完成的。这些操作，正是我刚刚为你讲述的大部分内容。

另外，我们为它定义了一个 livenessProbe，通过 mysqladmin ping 命令来检查它是否健康；还定义了一个 readinessProbe，通过查询 SQL（select 1）来检查 MySQL 服务是否可用。当然，凡是 readinessProbe 检查失败的 MySQL Pod，都会从 Service 里被摘除掉。

至此，一个完整的主从复制模式的 MySQL 集群就定义完了。

现在，我们就可以使用 kubectl 命令，尝试运行一下这个 StatefulSet 了。

**首先，我们需要在 Kubernetes 集群里创建满足条件的 PV**。你可以按照如下方式使用存储插件 Rook：

```yaml
$ kubectl create -f rook-storage.yaml
$ cat rook-storage.yaml
apiVersion: ceph.rook.io/v1beta1
kind: Pool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  clusterNamespace: rook-ceph
```

在这里，我用到了 StorageClass 来完成这个操作。它的作用，是自动地为集群里存在的每一个 PVC，调用存储插件（Rook）创建对应的 PV，从而省去了我们手动创建 PV 的机械劳动。我在后续讲解容器存储的时候，会再详细介绍这个机制。

> 备注：在使用 Rook 的情况下，mysql-statefulset.yaml 里的 volumeClaimTemplates 字段需要加上声明 storageClassName=rook-ceph-block，才能使用到这个 Rook 提供的持久化存储。

**然后，我们就可以创建这个 StatefulSet 了**，如下所示：

```
$ kubectl create -f mysql-statefulset.yaml
$ kubectl get pod -l app=mysql
NAME      READY     STATUS    RESTARTS   AGE
mysql-0   2/2       Running   0          2m
mysql-1   2/2       Running   0          1m
mysql-2   2/2       Running   0          1m
```

可以看到，StatefulSet 启动成功后，会有三个 Pod 运行。

**接下来，我们可以尝试向这个 MySQL 集群发起请求，执行一些 SQL 操作来验证它是否正常：**

```
$ kubectl run mysql-client --image=mysql:5.7 -i --rm --restart=Never --\
  mysql -h mysql-0.mysql <<EOF
CREATE DATABASE test;
CREATE TABLE test.messages (message VARCHAR(250));
INSERT INTO test.messages VALUES ('hello');
EOF
```

如上所示，我们通过启动一个容器，使用 MySQL client 执行了创建数据库和表、以及插入数据的操作。需要注意的是，我们连接的 MySQL 的地址必须是 mysql-0.mysql（即：Master 节点的 DNS 记录）。因为，只有 Master 节点才能处理写操作。

而通过连接 mysql-read 这个 Service，我们就可以用 SQL 进行读操作，如下所示：

```
$ kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
 mysql -h mysql-read -e "SELECT * FROM test.messages"
Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod "mysql-client" deleted
```

在有了 StatefulSet 以后，你就可以像 Deployment 那样，非常方便地扩展这个 MySQL 集群，比如：

```
$ kubectl scale statefulset mysql  --replicas=5
```

这时候，你就会发现新的 Slave Pod mysql-3 和 mysql-4 被自动创建了出来。

而如果你像如下所示的这样，直接连接 mysql-3.mysql，即 mysql-3 这个 Pod 的 DNS 名字来进行查询操作：

```
$ kubectl run mysql-client --image=mysql:5.7 -i -t --rm --restart=Never --\
  mysql -h mysql-3.mysql -e "SELECT * FROM test.messages"
Waiting for pod default/mysql-client to be running, status is Pending, pod ready: false
+---------+
| message |
+---------+
| hello   |
+---------+
pod "mysql-client" deleted
```

就会看到，从 StatefulSet 为我们新创建的 mysql-3 上，同样可以读取到之前插入的记录。也就是说，我们的数据备份和恢复，都是有效的。

### 总结

在今天这篇文章中，我以 MySQL 集群为例，和你详细分享了一个实际的 StatefulSet 的编写过程。这个 YAML 文件的[链接在这里](https://github.com/oracle/kubernetes-website/blob/master/docs/tasks/run-application/mysql-statefulset.yaml)，希望你能多花一些时间认真消化。

在这个过程中，有以下几个关键点（坑）特别值得你注意和体会。

1. “人格分裂”：在解决需求的过程中，一定要记得思考，该 Pod 在扮演不同角色时的不同操作。
2. “阅后即焚”：很多“有状态应用”的节点，只是在第一次启动的时候才需要做额外处理。所以，在编写 YAML 文件时，你一定要考虑“容器重启”的情况，不要让这一次的操作干扰到下一次的容器启动。
3. “容器之间平等无序”：除非是 InitContainer，否则一个 Pod 里的多个容器之间，是完全平等的。所以，你精心设计的 sidecar，绝不能对容器的顺序做出假设，否则就需要进行前置检查。

最后，相信你也已经能够理解，StatefulSet 其实是一种特殊的 Deployment，只不过这个“Deployment”的每个 Pod 实例的名字里，都携带了一个唯一并且固定的编号。这个编号的顺序，固定了 Pod 的拓扑关系；这个编号对应的 DNS 记录，固定了 Pod 的访问方式；这个编号对应的 PV，绑定了 Pod 与持久化存储的关系。所以，当 Pod 被删除重建时，这些“状态”都会保持不变。

而一旦你的应用没办法通过上述方式进行状态的管理，那就代表了 StatefulSet 已经不能解决它的部署问题了。这时候，我后面讲到的 Operator，可能才是一个更好的选择。