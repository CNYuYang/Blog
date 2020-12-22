# Kubernetes 进阶：部署高可用的业务

## 无状态应用：剖析 Kubernetes 业务副本及水平扩展底层原理

每一个 Pod 都是应用的一个实例，但是通常来说你不会直接在 Kubernetes 中创建和运行单个 Pod。因为 Pod 的生命周期是短暂的，即“用后即焚”。理解这一点很重要，这也是“不可变基础设施”这一理念在 Kubernetes 中的最佳实践。同样，对于你后续进行业务改造和容器化上云也具有指导意义。

当一个 Pod 被创建出来，不管是由你直接创建，还是由其他工作负载控制器（Workload Controller）自动创建，经过调度器调度以后，就永久地“长”在某个节点上了，直到该 Pod 被删除，或者因为资源不够被驱逐，抑或由于对应的节点故障导致宕机等。因此单独地用一个 Pod 来承载业务，是没办法保证高可用、可伸缩、负载均衡等要求，而且 Pod 也无法“自愈”。

这时我们就需要在 Pod 之上做一层抽象，通过多个副本（Replica）来保证可用 Pod 的数量，避免业务不可用。在介绍 Kubernetes 中对这种抽象的实现之前，我们先来看看应用的业务类型。

### 有状态服务 VS 无状态服务

一般来说，业务的服务类型可分为无状态服务和有状态服务。举个简单的例子，像打网络游戏这类的服务，就是有状态服务，而正常浏览网页这类服务一般都是无状态服务。其实判断两种请求的关键在于，两个来自相同发起者的请求在服务器端是否具备上下文关系。

- 如果是有状态服务，其请求是状态化的，服务器端需要保存请求的相关信息，这样每个请求都可以默认地使用之前的请求上下文。

- 而无状态服务就不需要这样，每次请求都包含了需要的所有信息，每次请求都和之前的没有任何关系。

有状态服务和无状态服务分别有各自擅长的业务类型和技术优势，在 Kubernetes 中，分别有不同的工作负载控制器来负责承载这两类服务。

本节课，我们先介绍 Kubernetes 中的无状态工作负载。

### Kubernetes 中的无状态工作负载

Kubernetes 中各个对象的 metadata 字段都有 [label](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/)（标签）和 [annotation](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/annotations/)（注解） 两个对象，可以用来标识一些元数据信息。不论是 label 还是 annotation，都是一组键值对，键值不允许相同。关于各个对象的 API 基本构成，可以参照此前中 API 的定义说明，Kubernetes 的各个对象的 API 都符合这个规范。

annotation 主要用来记录一些非识别的信息，并不用于标识和选择对象。

label 主要用来标识一些有意义且和对象密切相关的信息，用来支持labelSelector（标签选择器）以及一些查询操作，还有选择对象。

为了让这种抽象的对象可以跟 Pod 关联起来，Kubernetes 使用了labelSelector来跟 label 进行选择匹配，从而达到这种松耦合的关联效果。

```shell
$ kubectl get pod -l label1=value1,label2=value2 -n my-namespace
```

比如，我们就可以通过上述命令，查询出 my-namespace 这个命名空间下面，带有标签label1=value1和label2=value2的 pod。label 中的键值对在匹配的时候是“且”的关系。

### ReplicationController

Kubernetes 中有一系列的工作负载可以用来部署无状态服务。在最初，Kubernetes 中使用了ReplicationController来做 Pod 的副本控制，即确保该服务的 Pod 数量维持在特定的数量。为了简洁并便于使用，ReplicationController通常缩写为“rc”，并作为 kubectl 命令的快捷方式，例如：

```shell
$ kubectl get rc -n my-namespace
```

如果副本数少于预定的值，则创建新的 Pod。如果副本数大于预定的值，就删除多余的副本。因此即使你的业务应用只需要一个 Pod，你也可以使用 rc 来自动帮你维护和创建 Pod。

### ReplicaSet

随后社区开发了下一代的 Pod 控制器 ReplicaSet（可简写为 rs） 用来替代 ReplicaController。虽然 ReplicaController 目前依然可以使用，但是社区已经不推荐继续使用了。这两者的功能和目的完全相同，但是 ReplicaSet 具备更强大的[基于集合的标签选择器](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels/#基于集合-的需求)，这样你可以通过一组值来进行标签匹配选择。目前支持三种操作符：`in`、`notin`和`exists`。

例如，你可以用`environment in (production, qa)`来匹配 label 中带有`environment=production`或`environment=qa`的 Pod。

同样你也可以使用`tier notin (frontend,backend)`来匹配 label 中不带有`tier=frontend`或`tier=backend`的 Pod。

或者你可以用 partition来匹配 label 中带有 partition 这个 key 的 Pod。

了解了标签选择器，我们就可以通过如下的 kubectl 命令查找 Pod：

```shell
kubectl get pods -l environment=production,tier=frontend
```

或者使用：

```shell
kubectl get pods -l 'environment in (production),tier in (frontend)'
```

虽然 Replicaset 可以独立使用，但是为了能够更好地协调 Pod 的创建、删除以及更新等操作，我们都是直接**使用更高级的 Deployment**来管理 Replicaset，社区也是一直这么定位和推荐的。比如一些业务升级的场景，使用单一的 ReplicaController 或者 Replicaset 是无法实现滚动升级的诉求，至少需要定义两个该对象才能实现，而且这两个对象使用的标签选择器中的 label 至少要有一个不相同。通过不断地对这两个对象的副本进行增减，也可以称为**调和**（Reconcile），才可以完成滚动升级。这样使用起来不方便，也增加了用户的使用门槛，极大地降低了业务发布的效率。

### Deployment

通过 Deployment，我们就不需要再关心和操作 ReplicaSet 了。

![](./images/02-01.png)

Deployment、ReplicaSet 和 Pod 这三者之间的关系见上图。通过 Deployment，我们可以管理多个 label 各不相同的 ReplicaSet，每个 ReplicaSet 负责保证对应数目的 Pod 在运行。

我们来看一个定义 Deployment 的例子：

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: nginx-deployment-demo

  namespace: demo

  labels:

    app: nginx

spec:

  replicas: 3

  selector:

    matchLabels:

      app: nginx

  template:

    metadata:

      labels:

        app: nginx

        version: v1

    spec:

      containers:

      - name: nginx

        image: nginx:1.14.2

        ports:

        - containerPort: 80
```

我们这里定义了副本数`spec.replicas`为 3，同时在`spec.selector.matchLabels`中设置了`app=nginx`的 label，用于匹配`spec.template.metadata.labels`的 label。我们还增加了`version=v1`的 label 用于后续滚动升级做比较。

我们将上述内容保存到`deploy-demo.yaml`文件中。

注意，`spec.selector.matchLabels`中写的 label 一定要能匹配得了`spec.template.metadata.labels`中的 label。否则该 Deployment 无法关联创建出来的 ReplicaSet。

然后我们通过下面这些命令，便可以在命名空间 demo 中创建名为 nginx-deployment-demo 的 Deployment。

```shell
$ kubectl create ns demo
$ kubectl create -f deploy-demo.yaml
deployment.apps/nginx-deployment-demo created
```

创建完成后，我们可以查看自动创建出来的 rs。

```shell
$ kubectl get rs -n demo
NAME                               DESIRED   CURRENT   READY   AGE
nginx-deployment-demo-5d65f98bd9   3         3         0       5s
```

接下来，我们可以通过如下的几个命令不断地查询系统，看看对应的 Pod 是否运行成功。

```shell
$ kubectl get pod -n demo -l app=nginx,version=v1
NAME                                     READY   STATUS              RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   0/1     ContainerCreating   0          30s
nginx-deployment-demo-5d65f98bd9-d78fx   0/1     ContainerCreating   0          30s
nginx-deployment-demo-5d65f98bd9-ssstk   0/1     ContainerCreating   0          30s
$ kubectl get pod -n demo -l app=nginx,version=v1
NAME                                     READY   STATUS    RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   1/1     Running   0          63s
nginx-deployment-demo-5d65f98bd9-d78fx   1/1     Running   0          63s
nginx-deployment-demo-5d65f98bd9-ssstk   1/1     Running   0          63s
```

我们可以看到，从 Deployment 到这里，最后的 Pod 已经创建成功了。

现在我们试着做下镜像更新看看。更改`spec.template.metadata.labels`中的`version=v1`为`version=v2`，同时更新镜像`nginx:1.14.2`为`nginx:1.19.2`。

你可以直接通过下述命令来直接更改：

```shell
kubectl edit deploy nginx-deployment-demo -n demo 
```

也可以更改`deploy-demo.yaml`这个文件：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment-demo
  namespace: demo
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        version: v2
    spec:
      containers:
      - name: nginx
        image: nginx:1.19.2
        ports:
        - containerPort: 80
```

然后运行这些命令：

```shell
$ kubectl apply -f deploy-demo.yaml
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
deployment.apps/nginx-deployment-demo configured
```

这个时候，我们来看看 ReplicaSet 有什么变化：

```shell
$ kubectl get rs -n demo
NAME                               DESIRED   CURRENT   READY   AGE
nginx-deployment-demo-5d65f98bd9   3         3         3       4m10s
nginx-deployment-demo-7594578db7   1         1         0       3s
```

可以看到，这个时候自动创建了一个新的 rs 出来：`nginx-deployment-demo-7594578db7`。一般 Deployment 的默认更新策略是 RollingUpdate，即先创建一个新的 Pod，并待其成功运行以后，再删除旧的。

还有一个更新策略是Recreate，即先删除现有的 Pod，再创建新的。关于 Deployment，我们还可以设置最大不可用（`maxUnavailable`）和最大增量（`maxSurge`），来更精确地控制滚动更新操作，具体配置可参照[这个文档](https://kubernetes.io/zh/docs/concepts/workloads/controllers/deployment/#滚动更新-deployment)。

我建议你使用默认的策略来保证可用性。后面 Deployment 控制器会不断对这两个 rs 进行调和，直到新的 rs 副本数为 3，老的 rs 副本数为 0。

我们可以通过如下命令，能够观察到 Deployment 的调和过程。

```shell
$ kubectl get pod -n demo -l app=nginx,version=v1
NAME                                     READY   STATUS    RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   1/1     Running   0          4m15s
nginx-deployment-demo-5d65f98bd9-d78fx   1/1     Running   0          4m15s
nginx-deployment-demo-5d65f98bd9-ssstk   1/1     Running   0          4m15s
$ kubectl get pod -n demo -l app=nginx
NAME                                     READY   STATUS              RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   1/1     Running             0          4m22s
nginx-deployment-demo-5d65f98bd9-d78fx   1/1     Running             0          4m22s
nginx-deployment-demo-5d65f98bd9-ssstk   1/1     Running             0          4m22s
nginx-deployment-demo-7594578db7-zk8jq   0/1     ContainerCreating   0          15s
$ kubectl get pod -n demo -l app=nginx,version=v2
NAME                                     READY   STATUS              RESTARTS   AGE
nginx-deployment-demo-7594578db7-zk8jq   0/1     ContainerCreating   0          19s
$ kubectl get pod -n demo -l app=nginx,version=v2
NAME                                     READY   STATUS              RESTARTS   AGE
nginx-deployment-demo-7594578db7-4g4fk   0/1     ContainerCreating   0          1s
nginx-deployment-demo-7594578db7-zk8jq   1/1     Running             0          31s
$ kubectl get pod -n demo -l app=nginx,version=v1
NAME                                     READY   STATUS        RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   1/1     Running       0          4m40s
nginx-deployment-demo-5d65f98bd9-d78fx   1/1     Running       0          4m40s
nginx-deployment-demo-5d65f98bd9-ssstk   1/1     Terminating   0          4m40s
$ kubectl get pod -n demo -l app=nginx,version=v2
NAME                                     READY   STATUS              RESTARTS   AGE
nginx-deployment-demo-7594578db7-4g4fk   1/1     Running             0          5s
nginx-deployment-demo-7594578db7-ftzmf   0/1     ContainerCreating   0          2s
nginx-deployment-demo-7594578db7-zk8jq   1/1     Running             0          35s
$ kubectl get pod -n demo -l app=nginx,version=v1
NAME                                     READY   STATUS        RESTARTS   AGE
nginx-deployment-demo-5d65f98bd9-7w5gp   0/1     Terminating   0          4m52s
nginx-deployment-demo-5d65f98bd9-ssstk   0/1     Terminating   0          4m52s
$ kubectl get pod -n demo -l app=nginx,version=v2
NAME                                     READY   STATUS    RESTARTS   AGE
nginx-deployment-demo-7594578db7-4g4fk   1/1     Running   0          17s
nginx-deployment-demo-7594578db7-ftzmf   1/1     Running   0          14s
nginx-deployment-demo-7594578db7-zk8jq   1/1     Running   0          47s
$ kubectl get rs -n demo
NAME                               DESIRED   CURRENT   READY   AGE
nginx-deployment-demo-5d65f98bd9   0         0         0       5m5s
nginx-deployment-demo-7594578db7   3         3         3       58s
$ kubectl get pod -n demo -l app=nginx,version=v1
No resources found in demo namespace.
```

或者可以通过 kubectl 的 watch 功能：

```shell
$ kubectl get pod -n demo -l app=nginx -w
```

至此，我们完成了 Deployment 的升级过程。

## 有状态应用：Kubernetes 如何通过 StatefulSet 支持有状态应用？

我们来一起看看Kubernetes 中的另外一种工作负载 StatefulSet。从名字就可以看出，这个工作负载主要用于有状态的服务发布。关于有状态服务和无状态服务，你可以参考上一节课的内容。

这节课，我们从一个具体的例子来逐渐了解、认识 StatefulSet。在 kubectl 命令行中，我们一般将 StatefulSet 简写为 sts。在部署一个 StatefulSet 的时候，有个前置依赖对象，即 Service（服务）。这个对象在 StatefulSet 中的作用，我们在下文中会一一道来。另外，关于这个对象的详细介绍和其他作用，我们会在后面的课程中单独讲解。在此，你可以先暂时略过对 Service 的感知。我们先看如下一个 Service：

```yaml
$ cat nginx-svc.yaml

apiVersion: v1

kind: Service

metadata:

  name: nginx-demo

  namespace: demo

  labels:

    app: nginx

spec:

  clusterIP: None

  ports:

  - port: 80

    name: web

  selector:

    app: nginx

```

上面这段 yaml 的意思是，在 demo 这个命名空间中，创建一个名为 nginx-demo 的服务，这个服务暴露了 80 端口，可以访问带有`app=nginx`这个 label 的 Pod。

我们现在利用上面这段 yaml 在集群中创建出一个 Service：

```shell
$ kubectl create ns demo
$ kubectl create -f nginx-svc.yaml
service/nginx-demo created
$ kubectl get svc -n demo
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx-demo   ClusterIP   None             <none>        80/TCP    5s
```

创建好了这个前置依赖的 Service，下面我们就可以开始创建真正的 StatefulSet 对象，可参照如下的 yaml 文件：

```shell
$ cat web-sts.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-demo
  namespace: demo
spec:
  serviceName: "nginx-demo"
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
        image: nginx:1.19.2-alpine
        ports:
        - containerPort: 80
          name: web
$ kubectl create -f web-sts.yaml
$ kubectl get sts -n demo
NAME       READY   AGE
web-demo   0/2     9s
```

可以看到，到这里我已经将名为web-demo的StatefulSet部署完成了。

下面我们来一点点探索 StatefulSet 的秘密，看看它有哪些特性，为何可以保障服务有状态的运行。

### StatefulSet 的特性

通过 kubectl 的**watch**功能（命令行加参数`-w`），我们可以观察到 Pod 状态的一步步变化。

```shell
$ kubectl get pod -n demo -w
NAME         READY   STATUS              RESTARTS   AGE
web-demo-0   0/1     ContainerCreating   0          18s
web-demo-0   1/1     Running             0          20s
web-demo-1   0/1     Pending             0          0s
web-demo-1   0/1     Pending             0          0s
web-demo-1   0/1     ContainerCreating   0          0s
web-demo-1   1/1     Running             0          2s
```

**通过 StatefulSet 创建出来的 Pod 名字有一定的规律**，即`$(statefulset名称)-$(序号)`，比如这个例子中的web-demo-0、web-demo-1。

这里面还有个有意思的点，web-demo-0 这个 Pod 比 web-demo-1 优先创建，而且在 web-demo-0 变为 Running 状态以后，才被创建出来。为了证实这个猜想，我们在一个终端窗口观察 StatefulSet 的 Pod：

```shell
$ kubectl get pod -n demo -w -l app=nginx
```

我们再开一个终端端口来 watch 这个 namespace 中的 event：

```shell
$ kubectl get event -n demo -w
```

现在我们试着改变这个 St]atefulSet 的副本数，将它改成 5：

```shell
$ kubectl scale sts web-demo -n demo --replicas=5
statefulset.apps/web-demo scaled
```

此时我们观察到另外两个终端端口的输出：

```shell
$ kubectl get pod -n demo -w
NAME         READY   STATUS    RESTARTS   AGE
web-demo-0   1/1     Running   0          20m
web-demo-1   1/1     Running   0          20m
web-demo-2   0/1     Pending   0          0s
web-demo-2   0/1     Pending   0          0s
web-demo-2   0/1     ContainerCreating   0          0s
web-demo-2   1/1     Running             0          2s
web-demo-3   0/1     Pending             0          0s
web-demo-3   0/1     Pending             0          0s
web-demo-3   0/1     ContainerCreating   0          0s
web-demo-3   1/1     Running             0          3s
web-demo-4   0/1     Pending             0          0s
web-demo-4   0/1     Pending             0          0s
web-demo-4   0/1     ContainerCreating   0          0s
web-demo-4   1/1     Running             0          3s
```

我们再一次看到了 StatefulSet 管理的 Pod 按照 2、3、4 的顺序依次创建，名称有规律，跟上一节通过 Deployment 创建的随机 Pod 名有很大的区别。

通过观察对应的 event 信息，也可以再次证实我们的猜想。

```shell
$ kubectl get event -n demo -w
LAST SEEN   TYPE     REASON             OBJECT                 MESSAGE
20m         Normal   Scheduled          pod/web-demo-0         Successfully assigned demo/web-demo-0 to kraken
20m         Normal   Pulling            pod/web-demo-0         Pulling image "nginx:1.19.2-alpine"
20m         Normal   Pulled             pod/web-demo-0         Successfully pulled image "nginx:1.19.2-alpine"
20m         Normal   Created            pod/web-demo-0         Created container nginx
20m         Normal   Started            pod/web-demo-0         Started container nginx
20m         Normal   Scheduled          pod/web-demo-1         Successfully assigned demo/web-demo-1 to kraken
20m         Normal   Pulled             pod/web-demo-1         Container image "nginx:1.19.2-alpine" already present on machine
20m         Normal   Created            pod/web-demo-1         Created container nginx
20m         Normal   Started            pod/web-demo-1         Started container nginx
20m         Normal   SuccessfulCreate   statefulset/web-demo   create Pod web-demo-0 in StatefulSet web-demo successful
20m         Normal   SuccessfulCreate   statefulset/web-demo   create Pod web-demo-1 in StatefulSet web-demo successful
0s          Normal   SuccessfulCreate   statefulset/web-demo   create Pod web-demo-2 in StatefulSet web-demo successful
0s          Normal   Scheduled          pod/web-demo-2         Successfully assigned demo/web-demo-2 to kraken
0s          Normal   Pulled             pod/web-demo-2         Container image "nginx:1.19.2-alpine" already present on machine
0s          Normal   Created            pod/web-demo-2         Created container nginx
0s          Normal   Started            pod/web-demo-2         Started container nginx
0s          Normal   SuccessfulCreate   statefulset/web-demo   create Pod web-demo-3 in StatefulSet web-demo successful
0s          Normal   Scheduled          pod/web-demo-3         Successfully assigned demo/web-demo-3 to kraken
0s          Normal   Pulled             pod/web-demo-3         Container image "nginx:1.19.2-alpine" already present on machine
0s          Normal   Created            pod/web-demo-3         Created container nginx
0s          Normal   Started            pod/web-demo-3         Started container nginx
0s          Normal   SuccessfulCreate   statefulset/web-demo   create Pod web-demo-4 in StatefulSet web-demo successful
0s          Normal   Scheduled          pod/web-demo-4         Successfully assigned demo/web-demo-4 to kraken
0s          Normal   Pulled             pod/web-demo-4         Container image "nginx:1.19.2-alpine" already present on machine
0s          Normal   Created            pod/web-demo-4         Created container nginx
0s          Normal   Started            pod/web-demo-4         Started container nginx
```

现在我们试着进行一次缩容：

```shell
$ kubectl scale sts web-demo -n demo --replicas=2
statefulset.apps/web-demo scaled
```

此时观察另外两个终端窗口，分别如下：

```shell
web-demo-4   1/1     Terminating   0          11m
web-demo-4   0/1     Terminating   0          11m
web-demo-4   0/1     Terminating   0          11m
web-demo-4   0/1     Terminating   0          11m
web-demo-3   1/1     Terminating   0          12m
web-demo-3   0/1     Terminating   0          12m
web-demo-3   0/1     Terminating   0          12m
web-demo-3   0/1     Terminating   0          12m
web-demo-2   1/1     Terminating   0          12m
web-demo-2   0/1     Terminating   0          12m
web-demo-2   0/1     Terminating   0          12m
web-demo-2   0/1     Terminating   0          12m
```

```shell
0s          Normal   SuccessfulDelete   statefulset/web-demo   delete Pod web-demo-4 in StatefulSet web-demo successful
0s          Normal   Killing            pod/web-demo-4         Stopping container nginx
0s          Normal   Killing            pod/web-demo-3         Stopping container nginx
0s          Normal   SuccessfulDelete   statefulset/web-demo   delete Pod web-demo-3 in StatefulSet web-demo successful
0s          Normal   SuccessfulDelete   statefulset/web-demo   delete Pod web-demo-2 in StatefulSet web-demo successful
0s          Normal   Killing            pod/web-demo-2         Stopping container nginx
```

可以看到，在缩容的时候，StatefulSet 关联的 Pod 按着 4、3、2 的顺序依次删除。

可见，**对于一个拥有 N 个副本的 StatefulSet 来说**，**Pod 在部署时按照 {0 …… N-1} 的序号顺序创建的**，**而删除的时候按照逆序逐个删除**，这便是我想说的第一个特性。

接着我们来看，**StatefulSet 创建出来的 Pod 都具有固定的、且确切的主机名**，比如：

```shell
$ for i in 0 1; do kubectl exec web-demo-$i -n demo -- sh -c 'hostname'; done
web-demo-0
web-demo-1
```

我们再看看上面 StatefulSet 的 API 对象定义，有没有发现跟我们上一节中 Deployment 的定义极其相似，主要的差异在于`spec.serviceName`这个字段。它很重要，StatefulSet 根据这个字段，为每个 Pod 创建一个 DNS 域名，这个**域名的格式**为`$(podname).(headless service name)`，下面我们通过例子来看一下。

当前 Pod 和 IP 之间的对应关系如下：

```shell
$ kubectl get pod -n demo -l app=nginx -o wide
NAME         READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
web-demo-0   1/1     Running   0          3h17m   10.244.0.39   kraken   <none>           <none>
web-demo-1   1/1     Running   0          3h17m   10.244.0.40   kraken   <none>           <none>
```

Podweb-demo-0 的IP 地址是 10.244.0.39，web-demo-1的 IP 地址是 10.244.0.40。这里我们通过`kubectl run`在同一个命名空间`demo`中创建一个名为 dns-test 的 Pod，同时 attach 到容器中，类似于`docker run -it --rm`这个命令。
我么在容器中运行 nslookup 来查询它们在集群内部的 DNS 地址，如下所示：

```shell
$ kubectl run -it --rm --image busybox:1.28 dns-test -n demo
If you don't see a command prompt, try pressing enter.
/ # nslookup web-demo-0.nginx-demo
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      web-demo-0.nginx-demo
Address 1: 10.244.0.39 web-demo-0.nginx-demo.demo.svc.cluster.local
/ # nslookup web-demo-1.nginx-demo
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
Name:      web-demo-1.nginx-demo
Address 1: 10.244.0.40 web-demo-1.nginx-demo.demo.svc.cluster.local
```

可以看到，每个 Pod 都有一个对应的 [A 记录](https://baike.baidu.com/item/A记录/1188077?fr=aladdin)。
我们现在删除一下这些 Pod，看看会有什么变化：

```shell
$ kubectl delete pod -l app=nginx -n demo

pod "web-demo-0" deleted

pod "web-demo-1" deleted

$ kubectl get pod -l app=nginx -n demo -o wide

NAME         READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES

web-demo-0   1/1     Running   0          15s   10.244.0.50   kraken   <none>           <none>

web-demo-1   1/1     Running   0          13s   10.244.0.51   kraken   <none>           <none>
```

删除成功后，可以发现 StatefulSet 立即生成了新的 Pod，但是 Pod 名称维持不变。唯一变化的就是 IP 发生了改变。

我们再来看看 DNS 记录：

```shell
$ kubectl run -it --rm --image busybox:1.28 dns-test -n demo

If you don't see a command prompt, try pressing enter.

/ # nslookup web-demo-0.nginx-demo

Server:    10.96.0.10

Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-demo-0.nginx-demo

Address 1: 10.244.0.50 web-demo-0.nginx-demo.demo.svc.cluster.local

/ # nslookup web-demo-1.nginx-demo

Server:    10.96.0.10

Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-demo-1.nginx-demo

Address 1: 10.244.0.51 web-demo-1.nginx-demo.demo.svc.cluster.local
```

可以看出，**DNS 记录中 Pod 的域名没有发生变化**，**仅仅 IP 地址发生了更换**。因此当 Pod 所在的节点发生故障导致 Pod 飘移到其他节点上，或者 Pod 因故障被删除重建，Pod 的 IP 都会发生变化，但是 Pod 的域名不会有任何变化，这也就意味着**服务间可以通过不变的 Pod 域名来保障通信稳定**，**而不必依赖 Pod IP**。

有了`spec.serviceName`这个字段，保证了 StatefulSet 关联的 Pod 可以有稳定的网络身份标识，即 Pod 的序号、主机名、DNS 记录名称等。

最后一个我想说的是，对于有状态的服务来说，每个副本都会用到持久化存储，且各自使用的数据是不一样的。

StatefulSet 通过 PersistentVolumeClaim（PVC）可以保证 Pod 的存储卷之间一一对应的绑定关系。同时，删除 StatefulSet 关联的 Pod 时，不会删除其关联的 PVC。

我们会在后续网络存储的章节中来专门介绍，再次先略过。

### 如何更新升级 StatefulSet

那么，如果想对一个 StatefulSet 进行升级，该怎么办呢？

在 StatefulSet 中，支持两种更新升级策略，即 RollingUpdate 和 OnDelete。

RollingUpdate策略是**默认的更新策略**。可以实现 Pod 的滚动升级，跟我们上一节课中 Deployment 介绍的

RollingUpdate策略一样。比如我们这个时候做了镜像更新操作，那么整个的升级过程大致如下，先逆序删除所有的 Pod，然后依次用新镜像创建新的 Pod 出来。这里你可以通过`kubectl get pod -n demo -w -l app=nginx`来动手观察下。

同时使用 RollingUpdate 更新策略还支持通过 partition 参数来分段更新一个 StatefulSet。所有序号大于或者等于 partition 的Pod 都将被更新。你这里也可以手动更新 StatefulSet 的配置来实验下。

当你把更新策略设置为 OnDelete 时，我们就必须手动先删除 Pod，才能触发新的 Pod 更新。

现在我们就总结下 StatefulSet 的特点：

- 具备固定的网络标记，比如主机名，域名等；
- 支持持久化存储，而且最好能够跟实例一一绑定；
- 可以按照顺序来部署和扩展；
- 可以按照顺序进行终止和删除操作；
- 在进行滚动升级的时候，也会按照一定顺序。

## 配置管理：Kubernetes 管理业务配置方式有哪些？

在使用过程中，我们常常需要对 Pod 进行一些配置管理，比如参数配置文件怎么使用，敏感数据怎么保存传递，等等。有些人可能会觉得，为什么不把这些配置（不限于参数、配置文件、密钥等）打包到镜像中去啊？乍一听，好像有点可行，但是这种做法“硬伤”太多。

- 有些不变的配置是可以打包到镜像中的，那可变的配置呢？
- 信息泄漏，很容易引发安全风险，尤其是一些敏感信息，比如密码、密钥等。
- 每次配置更新后，都要重新打包一次，升级应用。镜像版本过多，也给镜像管理和镜像中心存储带来很大的负担。
- 定制化太严重，可扩展能力差，且不容易复用。

所以这里的一个最佳实践就是将配置信息和容器镜像进行解耦，以“不变应万变”。在 Kubernetes 中，一般有 ConfigMap 和 Secret 两种对象，可以用来做配置管理。

### ConfigMap

首先我们来讲一下 ConfigMap 这个对象，它主要用来保存一些非敏感数据，可以用作环境变量、命令行参数或者挂载到存储卷中。

![](./images/02-02.gif)

ConfigMap 通过键值对来存储信息，是个 namespace 级别的资源。在 kubectl 使用时，我们可以简写成 cm。

我们来看一下两个 ConfigMap 的 API 定义：

```yaml
$ cat cm-demo-mix.yaml

apiVersion: v1

kind: ConfigMap

metadata:

  name: cm-demo-mix # 对象名字

  namespace: demo # 所在的命名空间

data: # 这是跟其他对象不太一样的地方，其他对象这里都是spec

  # 每一个键都映射到一个简单的值

  player_initial_lives: "3" # 注意这里的值如果数字的话，必须用字符串来表示

  ui_properties_file_name: "user-interface.properties"

  # 也可以来保存多行的文本

  game.properties: |

    enemy.types=aliens,monsters

    player.maximum-lives=5

  user-interface.properties: |

    color.good=purple

    color.bad=yellow

    allow.textmode=true

$ cat cm-demo-all-env.yaml

apiVersion: v1

kind: ConfigMap

metadata:

  name: cm-demo-all-env

  namespace: demo

data:

  SPECIAL_LEVEL: very

  SPECIAL_TYPE: charm
```

可见，我们通过 ConfigMap 既可以存储简单的键值对，也能存储多行的文本。

现在我们来创建这两个 ConfigMap：

```shell
$ kubectl create -f cm-demo-mix.yaml

configmap/cm-demo-mix created

$ kubectl create -f cm-demo-all-env.yaml

configmap/cm-demo-all-env created
```

创建 ConfigMap，你也可以通过`kubectl create cm`基于[目录](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-directories)、[文件](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-files)或者[字面值](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-literal-values)来创建，详细可参考这个[官方文档](https://kubernetes.io/zh/docs/tasks/configure-pod-container/configure-pod-configmap/#使用-kubectl-create-configmap-创建-configmap)。

创建成功后，我们可以通过如下方式来查看创建出来的对象。

```shell
$ kubectl get cm -n demo

NAME              DATA   AGE

cm-demo-all-env   2      30s

cm-demo-mix       4      2s

$ kubectl describe cm cm-demo-all-env -n demo

Name:         cm-demo-all-env

Namespace:    demo

Labels:       <none>

Annotations:  <none>

Data

====

SPECIAL_LEVEL:

----

very

SPECIAL_TYPE:

----

charm

Events:  <none>

$ kubectl describe cm cm-demo-mix -n demo

Name:         cm-demo-mix

Namespace:    demo

Labels:       <none>

Annotations:  <none>

Data

====

user-interface.properties:

----

color.good=purple

color.bad=yellow

allow.textmode=true

game.properties:

----

enemy.types=aliens,monsters

player.maximum-lives=5

player_initial_lives:

----

3

ui_properties_file_name:

----

user-interface.properties

Events:  <none>
```

下面我们看看怎么和 Pod 结合起来使用。在使用的时候，有几个地方需要特别注意：

- **Pod 必须和 ConfigMap 在同一个 namespace 下面；**
- **在创建 Pod 之前，请务必保证 ConfigMap 已经存在，否则 Pod 创建会报错。**

```yaml
$ cat cm-demo-pod.yaml

apiVersion: v1

kind: Pod

metadata:

  name: cm-demo-pod

  namespace: demo

spec:

  containers:

    - name: demo

      image: busybox:1.28

      command:

        - "bin/sh"

        - "-c"

        - "echo PLAYER_INITIAL_LIVES=$PLAYER_INITIAL_LIVES && sleep 10000"

      env:

        # 定义环境变量

        - name: PLAYER_INITIAL_LIVES # 请注意这里和 ConfigMap 中的键名是不一样的

          valueFrom:

            configMapKeyRef:

              name: cm-demo-mix         # 这个值来自 ConfigMap

              key: player_initial_lives # 需要取值的键

        - name: UI_PROPERTIES_FILE_NAME

          valueFrom:

            configMapKeyRef:

              name: cm-demo-mix

              key: ui_properties_file_name

      envFrom:  # 可以将 configmap 中的所有键值对都通过环境变量注入容器中

        - configMapRef:

            name: cm-demo-all-env

      volumeMounts:

      - name: full-config # 这里是下面定义的 volume 名字

        mountPath: "/config" # 挂载的目标路径

        readOnly: true

      - name: part-config

        mountPath: /etc/game/

        readOnly: true

  volumes: # 您可以在 Pod 级别设置卷，然后将其挂载到 Pod 内的容器中

    - name: full-config # 这是 volume 的名字

      configMap:

        name: cm-demo-mix # 提供你想要挂载的 ConfigMap 的名字

    - name: part-config

      configMap:

        name: cm-demo-mix

        items: # 我们也可以只挂载部分的配置

        - key: game.properties

          path: properties
```

在上面的这个例子中，几乎囊括了 ConfigMap 的几大使用场景：

- 命令行参数；
- 环境变量，可以只注入部分变量，也可以全部注入；
- 挂载文件，可以是单个文件，也可以是所有键值对，用每个键值作为文件名。

我们接着来创建：

```shell
$ kubectl create -f cm-demo-pod.yaml

pod/cm-demo-pod created
```

创建成功后，我们 exec 到容器中看看：

```shell
$ kubectl exec -it cm-demo-pod -n demo sh

kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.

/ # env

KUBERNETES_SERVICE_PORT=443

KUBERNETES_PORT=tcp://10.96.0.1:443

UI_PROPERTIES_FILE_NAME=user-interface.properties

HOSTNAME=cm-demo-pod

SHLVL=1

HOME=/root

SPECIAL_LEVEL=very

TERM=xterm

KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

KUBERNETES_PORT_443_TCP_PORT=443

KUBERNETES_PORT_443_TCP_PROTO=tcp

KUBERNETES_SERVICE_PORT_HTTPS=443

KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443

PLAYER_INITIAL_LIVES=3

KUBERNETES_SERVICE_HOST=10.96.0.1

PWD=/

SPECIAL_TYPE=charm

/ # ls /config/

game.properties            ui_properties_file_name

player_initial_lives       user-interface.properties

/ # ls -alh /config/

total 12

drwxrwxrwx    3 root     root        4.0K Aug 27 09:54 .

drwxr-xr-x    1 root     root        4.0K Aug 27 09:54 ..

drwxr-xr-x    2 root     root        4.0K Aug 27 09:54 ..2020_08_27_09_54_31.007551221

lrwxrwxrwx    1 root     root          31 Aug 27 09:54 ..data -> ..2020_08_27_09_54_31.007551221

lrwxrwxrwx    1 root     root          22 Aug 27 09:54 game.properties -> ..data/game.properties

lrwxrwxrwx    1 root     root          27 Aug 27 09:54 player_initial_lives -> ..data/player_initial_lives

lrwxrwxrwx    1 root     root          30 Aug 27 09:54 ui_properties_file_name -> ..data/ui_properties_file_name

lrwxrwxrwx    1 root     root          32 Aug 27 09:54 user-interface.properties -> ..data/user-interface.properties

/ # cat /config/game.properties

enemy.types=aliens,monsters

player.maximum-lives=5

/ # cat /etc/game/properties

enemy.types=aliens,monsters

player.maximum-lives=5
```

可以看到，环境变量都已经正确注入，对应的文件和目录也都挂载进来了。

在上面`ls -alh /config/`后，我们看到挂载的文件中存在软链接，都指向了`..data`目录下的文件。这样做的好处，是 kubelet 会定期同步检查已经挂载的 ConfigMap 是否是最新的，如果更新了，就是创建一个新的文件夹存放最新的内容，并同步修改`..data`指向的软链接。

一般我们只把一些非敏感的数据保存到 ConfigMap 中，敏感的数据就要保存到 Secret 中了。

### Secret

我们可以用 Secret 来保存一些敏感的数据信息，比如密码、密钥、token 等。在使用的时候， 跟 ConfigMap 的用法基本保持一致，都可以用来作为环境变量或者文件挂载。

Kubernetes 自身也有一些内置的 Secret，主要用来保存访问 APIServer 的 service account token，我们放到后面权限部分一起讲解，在此先略过。

除此之外，还可以用来保存私有镜像中心的身份信息，这样 kubelet 可以拉取到镜像。

> 注： 如果你使用的是 Docker，也可以提前在目标机器上运行`docker login yourprivateregistry.com`来保存你的有效登录信息。Docker 一般会将私有仓库的密钥保存在`$HOME/.docker/config.json`文件中，将该文件分发到所有节点即可。

我们看看如何通过 kubectl 来创建 secret，通过命令行 help 可以看到 kubectl 能够创建多种类型的 Secret。

```shell
$ kubectl create secret  -h

 Create a secret using specified subcommand.

Available Commands:

   docker-registry Create a secret for use with a Docker registry

   generic         Create a secret from a local file, directory or literal value

   tls             Create a TLS secret

Usage:

   kubectl create secret [flags] [options]

Use "kubectl  --help" for more information about a given command.

 Use "kubectl options" for a list of global command-line options (applies to all commands).
```

我们先来创建一个 Secret 来保存访问私有容器仓库的身份信息：

```shell
$ kubectl create secret -n demo docker-registry regcred \

   --docker-server=yourprivateregistry.com \

   --docker-username=allen \

   --docker-password=mypassw0rd \

--docker-email=allen@example.com

 secret/regcred created

 $ kubectl get secret -n demo regcred

 NAME      TYPE                             DATA   AGE

 regcred   kubernetes.io/dockerconfigjson   1      28s
```

这里我们可以看到，创建出来的 Secret 类型是`kubernetes.io/dockerconfigjson`：

```shell
$ kubectl describe secret -n demo regcred

Name:         regcred

Namespace:    demo

Labels:       <none>

Annotations:  <none>

Type:  kubernetes.io/dockerconfigjson

Data

====

.dockerconfigjson:  144 bytes
```

为了防止 Secret 中的内容被泄漏，`kubectl get`和`kubectl describe`会避免直接显示密码的内容。但是我们可以通过拿到完整的 Secret 对象来进一步查看其数据：

```yaml
$ kubectl get secret -n demo regcred -o yaml

apiVersion: v1

data: # 跟 configmap 一样，这块用于保存数据信息

  .dockerconfigjson: eyJhdXRocyI6eyJ5b3VycHJpdmF0ZXJlZ2lzdHJ5LmNvbSI6eyJ1c2VybmFtZSI6ImFsbGVuIiwicGFzc3dvcmQiOiJteXBhc3N3MHJkIiwiZW1haWwiOiJhbGxlbkBleGFtcGxlLmNvbSIsImF1dGgiOiJZV3hzWlc0NmJYbHdZWE56ZHpCeVpBPT0ifX19

kind: Secret

metadata:

  creationTimestamp: "2020-08-27T12:18:35Z"

  managedFields:

  - apiVersion: v1

    fieldsType: FieldsV1

    fieldsV1:

      f:data:

        .: {}

        f:.dockerconfigjson: {}

      f:type: {}

    manager: kubectl

    operation: Update

    time: "2020-08-27T12:18:35Z"

  name: regcred

  namespace: demo

  resourceVersion: "1419452"

  selfLink: /api/v1/namespaces/demo/secrets/regcred

  uid: 6d34123e-4d79-406b-9556-409cfb4db2e7

type: kubernetes.io/dockerconfigjson
```

这里我们发现`.dockerconfigjson`是一段乱码，我们用 base64 解压试试看：

```shell
$ kubectl get secret regcred -n demo --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode

{"auths":{"yourprivateregistry.com":{"username":"allen","password":"mypassw0rd","email":"allen@example.com","auth":"YWxsZW46bXlwYXNzdzByZA=="}}}
```

这实际上跟我们通过 docker login 后的`~/.docker/config.json`中的内容一样。
至此，我们发现 Secret 和 ConfigMap 在数据保存上的最大不同。**Secret 保存的数据都是通过 base64 加密后的数据**。

我们平时使用较为广泛的还有另外一种`Opaque`类型的 Secret：

```yaml
$ cat secret-demo.yaml

apiVersion: v1

kind: Secret

metadata:

  name: dev-db-secret

  namespace: demo

type: Opaque

data: # 这里的值都是 base64 加密后的

  password: UyFCXCpkJHpEc2I9

  username: ZGV2dXNlcg==
```

或者我们也可以通过如下等价的 kubectl 命令来创建出来：

```shell
$ kubectl create secret generic dev-db-secret -n demo \

  --from-literal=username=devuser \

  --from-literal=password='S!B\*d$zDsb='
```

或通过文件来创建对象，比如：

```shell
$ echo -n 'username=devuser' > ./db_secret.txt

$ echo -n 'password=S!B\*d$zDsb=' >> ./db_secret.txt

$ kubectl create secret generic dev-db-secret -n demo \

  --from-file=./db_secret.txt
```

有时候为了方便，你也可以使用`stringData`，这样可以避免自己事先手动用 base64 进行加密。

```yaml
$ cat secret-demo-stringdata.yaml

apiVersion: v1

kind: Secret

metadata:

  name: dev-db-secret

  namespace: demo

type: Opaque

stringData:

  password: devuser

  username: S!B\*d$zDsb=
```

下面我们在 Pod 中使用 Secret：

```yaml
$ cat pod-secret.yaml

apiVersion: v1

kind: Pod

metadata:

  name: secret-test-pod

  namespace: demo

spec:

  containers:

    - name: demo-container

      image: busybox:1.28

      command: [ "/bin/sh", "-c", "env" ]

      envFrom:

      - secretRef:

          name: dev-db-secret

  restartPolicy: Never

$ kubectl create -f pod-secret.yaml

pod/secret-test-pod created
```

创建成功后，我们来查看下：

```shell
$ kubectl get pod -n demo secret-test-pod

NAME              READY   STATUS      RESTARTS   AGE

secret-test-pod   0/1     Completed   0          14s

$ kubectl logs -f -n demo secret-test-pod

KUBERNETES_SERVICE_PORT=443

KUBERNETES_PORT=tcp://10.96.0.1:443

HOSTNAME=secret-test-pod

SHLVL=1

username=devuser

HOME=/root

KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1

PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

KUBERNETES_PORT_443_TCP_PORT=443

password=S!B\*d$zDsb=

KUBERNETES_PORT_443_TCP_PROTO=tcp

KUBERNETES_SERVICE_PORT_HTTPS=443

KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443

KUBERNETES_SERVICE_HOST=10.96.0.1

PWD=/
```

我们可以在日志中看到命令`env`的输出，看到环境变量`username`和`password`已经正确注入。类似地，我们也可以将 Secret 作为 Volume 挂载到 Pod 内，你~~大家~~可以课后实践一下。