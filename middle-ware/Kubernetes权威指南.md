## 第一部分 Kubernetes基础

#### 第1章 Kubernetes入门

###### Kubernetes简介

- **背景与起源**
  - **Borg系统的开源版本**：Kubernetes的思想源于Google内部运行了十几年的大规模集群管理系统Borg，Google将Borg的理念和经验付诸实践，并将其开源，这就是Kubernetes（常简称为K8s）。
  - **CNCF与云原生**：Kubernetes是**云原生计算基金会（CNCF）** 的旗舰项目，并且已经成为云原生时代的**操作系统**。
- **官方定义**
  - Kubernetes是一个开源的**容器编排引擎**，用于自动化部署、扩展和管理容器化应用。
  - 它提供了一个以**容器为中心**的管理环境，旨在实现生产级别的容器编排和运维能力。

###### 为什么要用Kubernetes？

- **演进历程**：
  - **传统部署时代**：应用直接运行在物理服务器上，资源分配难以控制，应用之间容易产生资源争用，扩容成本高、周期长。
  - **虚拟化部署时代**：通过虚拟机（VM）在单台物理机上运行多个操作系统。提供了更好的资源隔离和安全性，并且更容易扩容。但每个VM都包含一整套操作系统，体积庞大、笨重，资源开销大。
  - **容器化部署时代**：容器与宿主机共享操作系统内核，因此更轻量、启动更快、资源效率更高。Docker使容器技术普及，但当容器数量增多、分布 across 多台主机时，如何管理它们的生命周期、网络互联、存储挂载、故障恢复和扩容？**这就需要容器编排**。
- **Kubernetes的核心价值（解决了哪些痛点）**：
  - **服务发现与负载均衡**：K8s可以为容器组提供唯一的域名和虚拟IP地址（Service），并自动将请求负载均衡到多容器实例上。
  - **存储编排**：可以自动挂载选择的存储系统，无论是本地存储、云提供商（如AWS EBS）还是网络存储系统（如NFS, Ceph）。
  - **自动部署和回滚**：可以描述应用的期望状态，K8s会以可控的速率将实际状态改变为期望状态，如果部署出错，可以轻松回滚。
  - **自动调度**：K8s调度器会依据每个容器对CPU和内存（RAM）的资源需求，自动安排到合适的节点上运行以最大化资源利用率。
  - **自我修复**：这是最关键的能力之一。K8s能**持续监控**集群状态。
    - 如果某个容器故障**崩溃**，它会**重启**该容器。
    - 如果某个**节点宕机**，它会将在该节点上运行的容器**重新调度**到其他健康节点上。
    - 如果服务**不响应健康检查**，它会**终止**该容器并在别处重新启动它。
  - **密钥与配置管理**：可以存储和管理敏感信息（如密码、OAuth令牌）和应用程序的配置信息（如配置文件），并能在不重建容器镜像的情况下部署和更新这些secret和配置。

###### Kubernetes入门实例

- **启动一个Deployment**：`kubectl create deployment nginx --image=nginx`
  - **意图**：告诉K8s运行一个名为`nginx`的应用，它使用`nginx`这个Docker镜像。
  - **K8s的行动**：K8s会创建一个`Deployment`对象，它负责保证Nginx容器的运行。随后，`Deployment`会创建一个`Pod`（容器组）来实际运行Nginx容器。
- **将服务暴露给外部**：`kubectl expose deployment nginx --port=80 --type=NodePort`
  - **意图**：前文的Nginx只在集群内部可以访问，现在让外部用户也能访问到它。
  - **K8s的行动**：K8s会创建一个`Service`对象，其类型为`NodePort`。这意味着K8s会在集群的每个节点上打开一个端口（如30080），并将访问该端口的所有流量转发到后台的Nginx Pod上。
- **访问应用**：用户可以通过`<任意节点的IP地址>:30080`来访问Nginx的欢迎页面。

###### Kubernetes的基本概念和术语

- **集群类**：

  - **Master（控制平面）**：集群的“大脑”。负责管理、调度、决策和暴露API。通常包含以下核心组件：

    > **API Server**：所有资源操作的唯一入口，是各个组件之间通信的枢纽。
    >
    > **Scheduler**：负责根据资源情况将新创建的Pod调度到合适的Node上。
    >
    > **Controller Manager**：运行着各种控制器，负责维护集群的状态（如故障检测、自动扩展、滚动更新）。
    >
    > **etcd**：一个高可用的键值数据库，持久化存储整个集群的状态和配置数据。

  - **Node（工作节点）**：集群的“工作肌肉”。负责运行容器。每个Node上至少运行：

    > **Kubelet**：负责与Master通信，管理当前节点上Pod的生命周期（如创建、销毁容器）。
    >
    > **Kube-proxy**：负责维护节点上的网络规则，实现Service的负载均衡和流量转发。
    >
    > **容器运行时**：如Docker或containerd，负责真正运行容器。

- **应用类（最核心的一组概念）**：

  - **Pod**：**K8s调度和管理的最小单位**。一个Pod可以包含一个或多个紧密关联的容器（如主应用容器和日志收集Sidecar容器）。这些容器共享网络命名空间、IP地址、端口空间和存储卷。Pod是 ephemeral（短暂的），会被频繁地创建和销毁。

  - **Label**：**标签**，是附加到K8s对象（如Pod）上的键值对，用于标识对象的特定属性，是进行**筛选和分组**的核心手段。

  - **Controller（控制器）**：通过管理Pod模板来维护应用的**期望状态**。

    > **Deployment**：最常用的控制器，用于部署**无状态应用**，它管理ReplicaSet，并提供滚动更新、回滚等强大功能。
    >
    > **ReplicaSet**：确保指定数量的Pod副本始终在运行通常由Deployment自动创建和管理，一般不需要直接操作。
    >
    > **StatefulSet**：用于部署**有状态应用**（如MySQL），它为Pod提供**稳定的标识符、有序的部署和扩展、稳定的持久化存储**。
    >
    > **DaemonSet**：确保**每个Node上都运行一个**指定的Pod副本，用于运行集群级别的守护进程，如日志收集器（Fluentd）。
    >
    > **Job/CronJob**：用于运行**一次性任务**或**定时任务**，任务完成后Pod即退出。

  - **Service**：**服务发现与负载均衡**的核心，定义一个访问Pod的逻辑集合（通常由Label选择器确定）的策略，Service有稳定的IP地址和DNS名称，无论后端的Pod如何重启、迁移，访问方式都保持不变。类型包括：

    > **ClusterIP**：默认类型，仅在集群内部可访问。
    >
    > **NodePort**：通过每个节点的IP和静态端口暴露服务。
    >
    > **LoadBalancer**：使用云服务商提供的负载均衡器对外暴露服务。

  - **Ingress**：管理**外部访问**集群服务的API对象，通常是HTTP/HTTPS流量，提供了比Service `LoadBalancer`更强大的功能，如基于域名和路径的路由、SSL终止等，需要配合**Ingress Controller**（如Nginx, Traefik）使用。

- **存储类**：

  - **Volume**：卷，解决了Pod内容器共享数据以及数据持久化的问题，但Volume的生命周期与Pod绑定。
  - **PersistentVolume**：集群级别的存储资源，由管理员预先配置（如NFS卷、云存储盘）。
  - **PersistentVolumeClaim**：用户对存储资源的**申请**，Pod通过PVC来使用PV，从而实现了**存储与Pod的解耦**，使得Pod可以随意调度而不担心数据丢失。

- **配置与安全类**：

  - **Namespace**：命名空间。在物理集群内部提供**虚拟的隔离**，用于将资源划分到不同的项目、团队或环境（如`dev`, `prod`）。
  - **ConfigMap**：用于将**非机密**的配置数据（如配置文件、环境变量）与应用代码分离。
  - **Secret**：用于存储**敏感信息**，如密码、令牌、密钥。数据以Base64编码存储，提供一定的安全性。

#### 第2章 Kubernetes安装配置指南

###### 核心安装方案详解

- **创建TLS证书和秘钥**：需要为etcd、API Server、kubelet等各个组件以及管理员分别创建证书，并确保它们被正确的CA签名。。
- **部署高可用的etcd集群**：在多个节点上部署etcd，配置其对等证书加密通信，并组成集群，这是整个集群的“数据库”，必须先部署。
- **部署Master组件（控制平面）**：
  - **API Server**：配置其连接etcd的证书、服务端证书、以及用于认证的token文件等。
  - **Controller Manager**：配置其kubeconfig文件以安全地访问API Server。
  - **Scheduler**：同样配置其kubeconfig文件。
  - 这些组件通常通过`systemd`守护进程来管理。
- **部署Node组件**：
  - **kubelet**：配置最复杂。需要其bootstrap kubeconfig文件（用于首次申请证书）、证书轮换、连接容器运行时等参数。
  - **kube-proxy**：配置其kubeconfig文件以访问API Server。
- **部署集群插件**：同样需要手动部署DNS和网络插件。

###### 关键配置与运维指南

- **使用私有镜像库**：企业环境中通常使用私有Harbor或Nexus仓库。
  - 需要在所有节点上**登录私有仓库**（`docker login`）。
  - 创建**Kubernetes Secret**，并在Pod定义中通过`imagePullSecrets`字段引用它，这样kubelet才有权限拉取私有镜像。
- **Kubernetes的版本升级**：
  - 介绍如何安全地进行集群升级，通常遵循“先Master，后Node”的顺序。
  - 对于kubeadm集群，会介绍`kubeadm upgrade`命令的使用。
  - 强调升级前一定要**备份etcd**和数据。
- **CRI详解**：
  - 解释**容器运行时接口**的概念。Kubernetes并不直接操作Docker，而是通过CRI这个抽象接口与容器运行时交互。
  - 这使得Kubernetes可以支持多种运行时（Docker, containerd, CRI-O）。书中会解释如何配置kubelet来使用不同的CRI。
- **kubectl命令行工具用法详解**：
  - **语法格式**：`kubectl [command] [TYPE] [NAME] [flags]`
  - **常用命令**：`get`, `describe`, `create`, `apply`, `delete`, `logs`, `exec`。
  - **输出格式**：`-o wide`, `-o yaml`, `-o json`, `-o name` 等。
  - **kubectl补全**：如何启用bash/zsh的自动命令补全功能，极大提升效率。

## 第二部分 Kubernetes原理

#### 第3章 深入掌握Pod

###### Pod的基本概念与本质

- **“逻辑主机”模型**

  - **核心思想**：Pod的设计源于一个简单的观察：在现实应用中，**多个进程往往需要紧密协作**才能提供一个完整的服务。例如，一个主应用进程和一个日志收集进程、或者一个主Web服务器和一个同步本地文件的内容管理进程。

  - **解决问题**：Docker提倡“一个容器一个进程”，但紧密协作的进程需要共享某些资源（如网络、存储空间）。如果将它们强行分散到多个隔离的容器中，共享和通信会变得非常复杂。

  - **Pod的解决方案**：Pod就像一个**逻辑主机**，它模拟了一个传统的虚拟机环境。在这个环境里，**多个“进程”（由容器实现）** 可以：

    > **共享同一个IP地址和端口空间**：它们可以通过`localhost`直接通信，不会发生端口冲突。
    >
    > **共享相同的网络命名空间**：它们看到的网络设备、路由表完全相同。
    >
    > **共享存储卷**：Pod级别的Volume可以挂载到所有容器中，使它们能够共享文件。

- **Pod vs. 容器**

  - **容器**：是镜像的运行实例，是隔离和打包的单元。
  - **Pod**：是Kubernetes的调度和管理的单元，是一个或多个容器的**分组**。Kubernetes不直接调度容器，而是调度整个Pod。

- **Pod的实现机制**

  - Pod本身只是一个**逻辑概念**，在节点上，Pod的实现依赖于一个基础的“基础设施容器”（在早期使用Docker时，是`pause`容器）。这个容器非常轻量，它只负责持有Pod的**网络命名空间**和**IPC命名空间**，并提供Pod的**IP地址**。
  - 用户定义的业务容器则通过Docker的`--net=container:<id>`参数加入到这个基础设施容器的网络空间中。

###### Pod的定义详解（YAML/JSON）

```yaml
apiVersion: v1       # Kubernetes API的版本，Pod是v1核心API
kind: Pod            # 资源类型，这里是Pod
metadata:            # 元数据，描述Pod本身的信息
  name: nginx-fa35fx # Pod的名称，在同一命名空间内必须唯一
  namespace: depart  # 所属的命名空间，默认是default
  labels:            # 标签，用于标识和选择Pod
    app: nginx
    env: production
  annotations:       # 注解，存储非标识性元数据（如构建信息、配置说明），可供工具使用
    description: "Our main website backend"
spec:                # 规约，这是Pod的核心，描述了期望的状态
  containers:        # 容器列表，定义Pod中包含的一个或多个容器
  - name: nginx      # 容器的名称
    image: nginx:1.19-alpine # 容器镜像
    imagePullPolicy: IfNotPresent # 镜像拉取策略（Always, Never, IfNotPresent）
    ports:
    - containerPort: 80       # 容器暴露的端口（主要是文档性作用，实际暴露取决于镜像）
      protocol: TCP
    env:                      # 注入到容器的环境变量
    - name: LOG_LEVEL
      value: "debug"
    resources:                # 资源请求和限制，是调度和运维的关键
      requests:               # 请求的资源，调度器根据此值选择节点
        cpu: "250m"           # 250 milliCPU cores (0.25 cores)
        memory: "64Mi"        # 64 Mebibytes
      limits:                 # 资源上限，超过此限制容器会被终止或重启
        cpu: "500m"
        memory: "128Mi"
    volumeMounts:             # 将Pod级别的卷挂载到容器内的特定路径
    - name: html-volume
      mountPath: /usr/share/nginx/html
  - name: log-sync            # 第二个容器：日志收集Sidecar
    image: busybox
    command: ['sh', '-c', 'tail -f /var/log/nginx/access.log'] # 覆盖默认的启动命令
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/nginx
  volumes:                    # 定义在Pod级别的存储卷，可供所有容器挂载
  - name: html-volume         # 卷名称
    emptyDir: {}              # 卷类型：临时空目录，生命周期与Pod一致
  - name: log-volume
    emptyDir: {}
  restartPolicy: Always       # 容器失败时的重启策略（Always, OnFailure, Never）
  nodeSelector:               # 节点选择器，将Pod调度到带有特定标签的节点上
    disktype: ssd
  schedulerName: default-scheduler # 指定调度器
```

###### Pod的核心使用模式与高级特性

- **静态Pod**

  - **概念**：由特定节点上的`kubelet`直接管理的Pod，不通过API Server进行管理。
  - **用途**：常用于部署Kubernetes自身的核心组件，如API Server、Scheduler等（它们本身也是以Pod形式运行）。`kubelet`会监视一个静态Pod文件目录（如`/etc/kubernetes/manifests`），并自动创建和运行其中的Pod定义。
  - **与DaemonSet的区别**：DaemonSet由Controller Manager管理，是集群级别的；静态Pod由节点级别的`kubelet`管理。

- **Pod的配置管理**

  - **ConfigMap**：将配置数据（如配置文件、环境变量）与Pod定义解耦，Pod可以通过环境变量或卷挂载的方式使用ConfigMap。
  - **Secret**：以安全的方式（Base64编码）存储敏感信息（如密码、令牌），用法与ConfigMap类似。

- **Downward API**

  - **问题**：容器内的进程有时需要知道自身运行环境的信息（如Pod的IP、名称、所在节点名称、资源限制等）。
  - **解决方案**：Downward API允许容器以环境变量或文件（通过Volume）的方式**向下**获取Pod自身的元数据信息。

- **Pod的生命周期与重启策略**

  - **生命周期阶段**：`Pending` -> `Running` -> `Succeeded/Failed`。

  - **重启策略**：由`spec.restartPolicy`控制，决定了容器退出后是否重启。

    > **Always**：总是重启（最适合长期运行的Web服务）。
    >
    > **OnFailure**：仅在失败（非0退出码）时重启（适合Job）。
    >
    > **Never**：从不重启。

- **Pod的健康检查**：这是保障应用稳定性的**最关键机制**，Kubernetes通过**探针**来检查容器的健康状态。

  - **存活探针**：检查容器**是否还在正常运行**，如果检查失败，kubelet会**杀死并重启**容器。用于解决应用程序运行但已死锁的情况。

  - **就绪探针**：检查容器**是否已准备就绪，可以接收流量**，如果检查失败，Pod会被从Service的负载均衡端点列表中**移除**。用于解决应用程序正在启动、负载过高或依赖服务未就绪等情况。

  - **启动探针**：检查容器**内的应用程序是否已启动完成**。在启动探针成功之前，其他所有探针都会被禁用。用于保护慢启动容器。

  - **探针的检查机制**：

    > **exec**：在容器内执行命令，看退出码是否为0。
    >
    > **httpGet**：对容器的IP和端口发送HTTP GET请求，看状态码是否为2xx/3xx。
    >
    > **tcpSocket**：尝试与容器的指定端口建立TCP连接，看是否成功。

- **Init Containers**

  - **概念**：在应用容器（`spec.containers`）启动之前**必须运行并完成**的一个或多个初始化容器。
  - **特性**：它们按顺序执行，只有前一个成功完成后，下一个才会启动，如果任何Init Container失败，Pod会根据`restartPolicy`重启，直到所有Init Container都成功。
  - **典型用途**：等待其他依赖服务就绪，从远程仓库或安全工具中下载配置或密钥，为应用容器预先生成配置文件或进行数据迁移。

###### Pod的调度

- **NodeSelector**：最简单的调度方式，将Pod调度到带有指定标签的节点上。
- **资源请求与限制**：调度器根据`spec.containers[].resources.requests`来计算节点剩余资源，并决定Pod放在哪个节点上，`limits`则由节点上的`kubelet`执行，防止容器耗尽资源。

#### 第4章 深入掌握Service

###### Service的核心价值与要解决的问题

- **问题：Pod的动态性与不可靠性**
  - **动态IP**：Pod的IP地址不是固定的，当一个Pod被重新调度或替换时，它会获得一个新的IP地址。
  - **多个副本**：一个应用通常由多个Pod副本（Deployment管理）组成，客户端需要与所有这些Pod通信，而不是只盯着某一个。
- **解决方案：Service的引入**
  - **稳定的IP地址**：Service会获得一个在整个集群内唯一的、固定的虚拟IP（VIP），也称为ClusterIP。客户端只需要记住这个IP（或对应的DNS名称），无需关心后端有多少个Pod或它们的IP是什么。
  - **稳定的DNS名称**：集群内的DNS服务（如CoreDNS）会为Service自动创建一个DNS记录，格式为`<service-name>.<namespace>.svc.cluster.local`。集群内其他Pod只需通过服务名即可访问。
  - **负载均衡**：Service自动将发送到其虚拟IP的请求，以负载均衡的方式分发到所有健康的后端Pod上。

###### Service的定义与工作机制

- **Service的定义（YAML）详解**

  ```yaml
  apiVersion: v1
  kind: Service       # 资源类型为Service
  metadata:
    name: my-service  # Service的名称，用于DNS解析
    namespace: default
  spec:
    selector:         # 标签选择器！这是Service与Pod关联的核心
      app: MyApp      # 选择所有具有标签`app: MyApp`的Pod
    ports:
      - protocol: TCP # 协议类型（TCP, UDP, SCTP）
        port: 80      # Service自身暴露的端口
        targetPort: 9376 # 后端Pod监听的端口（也可以是端口名称）
    type: ClusterIP   # Service的类型，默认是ClusterIP
  ```

  - **spec.selector**：这是Service的**灵魂**。它通过标签（Label）来识别哪些Pod属于它的后端端点。Service控制器会持续监控与选择器匹配的Pod列表的变化。
  - **ports.port**：Service的虚拟IP上暴露的端口。
  - **ports.targetPort**：请求被转发到Pod上的目标端口，如果`targetPort`是字符串，则引用Pod定义中`containerPort`的**名称**。

- **核心工作机制：kube-proxy与Endpoints**

  - **Endpoints对象**：当你创建一个Service时，Kubernetes会自动创建一个同名的**Endpoints**对象。这个对象是一个动态列表，包含了所有匹配`selector`的、状态为Ready的Pod的IP地址和端口，可以`kubectl get endpoints <service-name>`查看。

  - **kube-proxy**：每个Node上都运行着一个`kube-proxy`守护进程，它负责实现Service的虚拟IP功能，它监视API Server上Service和Endpoints的变化，并**动态更新本机的网络规则**，确保发往Service VIP的流量能被正确转发到后端的Pod。

  - **流量转发模式**：`kube-proxy`有三种工作模式，决定了流量如何被路由：

    > **iptables模式（默认）**：使用Linux内核的`iptables`规则配置负载均衡，高性能、高可靠，但规则多了之后排查复杂。
    >
    > **ipvs模式**：使用Linux内核的IP Virtual Server功能。为大型集群提供了更好的可扩展性和性能，支持更多的负载均衡算法。
    >
    > **userspace模式（已弃用）**：流量在用户空间被代理，性能较差，仅用于历史兼容。

###### Service的类型（Type）及其应用场景

- **ClusterIP（默认类型）**

  - **暴露范围**：仅在**Kubernetes集群内部**可访问。
  - **IP地址**：分配一个集群内部的虚拟IP。
  - **用途**：这是最常用的类型，用于让集群内部的其他Pod（如前端Pod访问后端API）访问服务，它是微服务间内部通信的基础。

- **NodePort**

  - **暴露范围**：暴露到**集群每个节点的特定端口**上，因此可以从集群外部通过`<任何节点的IP>:<NodePort>`访问。

  - **工作机制**：NodePort建立在ClusterIP之上，它会在**每个Node**上打开一个静态端口（`30000-32767`范围），并将访问该端口的流量路由到背后的ClusterIP Service，最终再到Pod。

  - **YAML示例**：

    ```yaml
    spec:
      type: NodePort
      ports:
      - port: 80     # Service的ClusterIP端口
        targetPort: 9376
        nodePort: 30007 # 可选：手动指定节点端口，必须在范围内。不指定则随机分配。
    ```

  - **用途**：用于开发测试环境，或者让外部系统（如无法使用Ingress的传统系统）直接访问服务，**不适合生产环境直接暴露服务**。

- **LoadBalancer**

  - **暴露范围**：暴露到**集群外部**，通常是公有云提供的**网络负载均衡器**（如AWS的ELB/ALB、GCP的Load Balancer、Azure的LB）。
  - **工作机制**：当声明一个`type: LoadBalancer`的Service时，Kubernetes会向云提供商发起API调用，自动创建并配置一个外部的负载均衡器，这个负载均衡器会将流量引导到集群各个节点的NodePort，最终再到Pod。
  - **用途**：在公有云上暴露服务的最直接、最标准的方式，云负载均衡器自带健康检查、高可用、SSL终止等企业级功能，但每个Service都会创建一个独立的云LB，成本较高。

- **ExternalName**

  - **暴露范围**：集群内部。
  - **工作机制**：它不选择任何Pod，而是通过返回一个`CNAME`记录，将集群内部的Service映射到一个**外部域名**。
  - **用途**：让集群内的Pod能够通过Kubernetes Service的DNS机制访问集群外部的服务，实现解耦。

###### 高级模式与服务发现机制

- **Headless Service（无头服务）**

  - **定义**：将Service的`spec.clusterIP`设置为`None`。

  - **行为**：它**不会分配ClusterIP**，也不会提供负载均衡代理。

  - **DNS解析**：

    > 对Headless Service的DNS查询会返回**所有后端Pod的IP地址列表**（而不是一个Service的VIP）。
    >
    > 如果Service定义了标签选择器，则返回的是所有被选中的Pod的IP。
    >
    > 如果Service**没有**定义标签选择器，但手动配置了Endpoints，则返回这些Endpoints的IP。

  - **StatefulSet的最佳搭档**：用于StatefulSet中，每个Pod会有稳定的DNS名称（`<pod-name>.<headless-svc-name>`），非常适合有状态应用如ZooKeeper、Etcd、MySQL主从，它们需要直接互相发现和通信。

  - **自定义服务发现**：客户端可以自己获取所有后端地址，并实现自己的负载均衡策略。

- **服务发现机制**

  - **环境变量**：kubelet在启动Pod时，会将当前命名空间下所有活跃Service的IP和端口信息以环境变量的形式注入到Pod中。
  - **DNS（推荐）**：集群的CoreDNS服务会为每个Service创建DNS记录。这是**最主流和最推荐**的方式。应用程序只需通过服务名即可访问，实现了完全的松耦合。

###### Ingress - 超越Service的7层流量管理

- **与Service的关系**：Ingress**不是**Service的一种类型。它位于多个Service之前，充当**智能的7层（HTTP/HTTPS）路由器和入口点**。
- **工作原理**：你需要部署一个**Ingress Controller**，它本身通常通过`LoadBalancer`或`NodePort`类型的Service暴露，然后创建**Ingress资源**来定义路由规则。
- **优势**：一个Ingress Controller（一个LoadBalancer IP）可以代理**成百上千个不同的内部Service**，极大地节省成本和简化管理。

#### 第5章 核心组件的运行机制

###### 架构总览与核心交互模式

- **声明式API (Declarative API)**：

  - 用户向系统提交一个“期望状态”（如：要有3个Nginx副本）。
  - 系统持续工作，驱动当前状态向期望状态无限接近，用户无需发出“创建”、“扩容”等命令式指令。

- **监听-调和循环 (Reconcile Loop)**：

  - 这是所有控制器的核心工作模式。每个控制器都通过**Informer/List-Watch机制**监听它们所关心资源（Pod、Service）的变化。
  - 当监听到变化时，控制器将**当前状态**与**API中声明的期望状态**进行比对。
  - 如果发现差异，控制器就会执行一系列操作（如创建新Pod），试图消除差异，使当前状态符合期望状态，这个过程就叫**调和**。

- **核心工作流**

  ```mermaid
  sequenceDiagram
      actor User
      participant API Server
      participant etcd
      participant Controller Manager
      participant Scheduler
      participant Kubelet
      participant Container Runtime
  
      User->>API Server: 1. kubectl apply -f deployment.yaml
      Note over API Server, etcd: 认证、授权、准入控制
      API Server->>etcd: 2. 校验并存储资源对象
      etcd-->>API Server: 存储成功
  
      par Controller Manager 循环
          Controller Manager->>API Server: 3. List-Watch监听Deployment变化
          API Server-->>Controller Manager: 通知有新Deployment
          Controller Manager->>Controller Manager: 4. 调和：期望副本数 vs 当前副本数
          Controller Manager->>API Server: 5. 创建ReplicaSet及对应Pod资源 (期望状态)
          API Server->>etcd: 存储Pod资源
      end
  
      par Scheduler 循环
          Scheduler->>API Server: 6. List-Watch监听未调度Pod (spec.nodeName为空)
          API Server-->>Scheduler: 通知有待调度Pod
          Scheduler->>Scheduler: 7. 调度决策 (过滤、评分)
          Scheduler->>API Server: 8. 绑定Pod到选定Node (更新Pod的nodeName字段)
          API Server->>etcd: 存储绑定信息
      end
  
      par Kubelet 循环
          Kubelet->>API Server: 9. List-Watch监听属于本Node的Pod
          API Server-->>Kubelet: 通知有Pod被绑定到本Node
          Kubelet->>Container Runtime: 10. 创建容器
          Container Runtime-->>Kubelet: 容器创建成功
          Kubelet->>API Server: 11. 更新Pod状态为Running
      end
  ```

###### API Server - 集群的网关与交通枢纽

- **核心功能**：所有RESTful操作的**唯一入口**，是其他所有组件交互的中枢。
- **认证 (Authentication)**：确认用户身份，支持客户端证书、Bearer Token等，API Server会依次尝试所有配置的认证方式。
- **授权 (Authorization)**：已认证的用户是否有权限执行操作，常用 **RBAC** 模式，判断用户是否拥有对某资源的权限（get, create）。
- **准入控制 (Admission Control)**：在授权之后、对象被持久化之前，对请求进行拦截和修改。
  - **修改型**：例如 `MutatingAdmissionWebhook`，可以自动为Pod注入Sidecar容器（如Istio）或修改资源限制。
  - **验证型**：例如 `ValidatingAdmissionWebhook`，可以基于自定义策略拒绝请求（如要求所有Pod都必须有`app`标签）。
- **List-Watch**：API Server支持客户端（如Controller）建立长连接来监听资源变化，是所有控制器能够实现实时调和的基石。

###### etcd：集群的状态真相之源

- **核心功能**：分布式、高可用的键值存储数据库，**持久化存储整个集群的所有配置数据和状态**。
- **数据模型**：采用层次化的键空间，例如 `/registry/pods/<namespace>/<pod-name>` 存储了一个Pod的完整JSON定义。
- **一致性协议**：使用 **Raft共识算法**来保证集群中多个etcd实例之间的数据强一致性。任何写操作都必须经过集群多数节点确认，这意味着它通常是集群高可用的瓶颈，生产环境必须部署3个或以上奇数节点。
- **Watch机制**：API Server会监听etcd的变化，并将这些事件再转发给各个客户端（Controller），etcd是事件的最终来源。

###### Controller Manager：集群的自动化控制中心

- **核心功能**：运行着一系列**控制器**，每个控制器都是一个独立的调和循环，负责管理某种资源。
- **详细机制（以Deployment控制器为例）**：
  - 通过Informer监听Deployment和ReplicaSet的变化。
  - 比较**当前状态**（实际的ReplicaSet状态）与**期望状态**（Deployment中定义的`replicas`）。
  - 如果不一致，则调用API Server的接口，**创建或删除ReplicaSet**，使状态一致。
- **其他重要控制器**：
  - **ReplicaSet控制器**：监听ReplicaSet和Pod，确保运行的Pod副本数符合期望。
  - **Node控制器**：监控Node状态，当Node失联时，负责标记Node为`NotReady`并驱逐其上的Pod。
  - **Endpoint控制器**：监听Service和Pod的变化，维护Service的后端端点列表（Endpoints对象）。
  - **ServiceAccount控制器**：确保每个命名空间都存在一个默认的ServiceAccount。

###### Scheduler：集群的智能调度大脑

- **核心功能**：为新创建的、未调度的Pod（`spec.nodeName`为空的Pod）自动选择一个最合适的Node来运行。
- **过滤 (Filtering / Predicates)**：
  - 调度器基于所有Node的当前状态，过滤出所有能够运行该Pod的节点，过滤完成后，得到一个**可用节点列表**。
  - **检查策略**：节点资源是否足够（CPU、内存）、端口冲突、节点选择器匹配、亲和性与反亲和性、卷拓扑等。
- **评分 (Scoring / Priorities)**：
  - 调度器为过滤后的每个节点进行打分（0-100分），选择**综合得分最高**的节点。
  - **评分策略**：选择资源最空闲的节点（便于资源均衡）、选择与Pod亲和性高的节点、选择镜像已存在的节点等。
- **绑定 (Binding)**：确定最优节点后，调度器调用API Server的接口，将Pod的`spec.nodeName`字段更新为该节点名称，写操作会触发etcd的更新，之后该节点上的kubelet就会检测到这个Pod被分配给了自己，并开始创建容器。

###### Kubelet：节点上的 Pod 生命周期管理者

- **核心功能**：运行在每个Node上的代理，负责保证Pod中的容器**健康运行**，它是Master和Node之间的桥梁。
- **Pod来源**：
  - **来自API Server**：监听被调度到本节点的Pod（主要来源）。
  - **静态Pod (Static Pod)**：通过本地配置文件（如`/etc/kubernetes/manifests`）启动的Pod，kubelet会自动监控并启动它们，并为其在API Server创建一个镜像Pod对象，k8s控制平面组件（如API Server、Scheduler）通常就是以静态Pod方式运行的。
- **容器管理**：通过 **CRI** 接口与容器运行时（Docker, containerd）交互，执行创建、删除容器等操作。
- **探针管理**：执行用户配置的**存活探针**和**就绪探针**，并根据结果重启容器或通知Service更新端点。
- **资源监控**：通过 **cAdvisor** 收集本节点和容器的CPU、内存、磁盘、网络使用情况等监控数据。

###### Kube Proxy：集群的网络代理

- **核心功能**：维护节点上的网络规则，实现Service的**负载均衡**和**服务发现**功能。
- **监听**：通过List-Watch监听API Server中Service和Endpoints的变化。
- **规则更新**：当Service或后端Pod发生变化时，会更新本机的网络规则，确保发往Service虚拟IP的流量能被转发到实际的后端Pod。
- **三种模式**：
  - **iptables模式（默认）**：使用Linux内核的iptables规则实现负载均衡。高性能，但规则链过长时性能会下降。
  - **ipvs模式**：使用Linux内核的IP Virtual Server模块，为大型集群提供了更好的可扩展性和性能，支持更多的负载均衡算法。
  - **userspace模式（已弃用）**：流量在用户空间被转发，性能差。

#### 第6章 深入分析集群安全机制

###### 认证 (Authentication) 

- **X509客户端证书认证 (最常用、最安全)**
  - **工作原理**：客户端使用由集群证书颁发机构签名的SSL证书来证明自己的身份，证书中的`subject`字段就是用户名和组信息。
  - **流程**：在`kubectl`与API Server建立TLS连接时，`kubectl`会提供自己的证书（通常位于`~/.kube/config`中），API Server使用集群的CA证书来验证该客户端证书的有效性和真实性。
  - **适用场景**：适用于**人类用户**和**机器用户**（如其他组件、CI/CD系统）的认证，是管理员访问集群的标准方式。
- **静态令牌文件 (Static Token File)**
  - **工作原理**：API Server启动时指定一个包含`token`的CSV文件，客户端在请求头中携带`Authorization: Bearer <token>`。
  - **缺点**：需要重启API Server来更新令牌，非常不灵活，**不推荐用于生产环境**。
- **ServiceAccount令牌 (Pod身份认证)**
  - **工作原理**：这是**Pod与API Server通信**的主要方式，当你创建一个ServiceAccount时，Kubernetes会自动为其创建一个Secret，其中包含一个签名的JWT令牌。
  - **Pod中的使用**：该Secret会被自动挂载到Pod的`/var/run/secrets/kubernetes.io/serviceaccount/token`路径下，`kubectl`或Pod内的应用程序可以使用这个令牌来认证。
  - **流程**：Pod中的进程使用此令牌与API Server交互，API Server通过验证JWT签名的有效性来认证Pod的身份。
- **其他认证方式**
  - **Bootstrap Tokens**：用于节点加入集群时的引导认证，是一种短期、预定义的令牌。
  - **OpenID Connect (OIDC)**：与第三方身份提供商集成，实现单点登录（SSO），这是企业级部署的推荐方式。
  - **Webhook Token Authentication**：将令牌验证委托给外部的webhook服务，实现自定义认证逻辑。

###### 授权 (Authorization) 

- **RBAC - 基于角色的访问控制 (现行标准，最常用)**

  - **Rule**：规则，定义一组操作权限（如`apiGroups: [""], resources: ["pods"], verbs: ["get", "list", "create"]`）。
  - **Role** 和 **ClusterRole**：角色，是规则的集合。`Role`作用于特定命名空间，`ClusterRole`是集群范围的。
  - **Subject**：主体，即被授予权限的对象，可以是`User`、`Group`或`ServiceAccount`。
  - **RoleBinding** 和 **ClusterRoleBinding**：绑定，将`Role/ClusterRole`中定义的权限授予一个`Subject`，`RoleBinding`可以将`ClusterRole`的权限限制在特定命名空间内。

  - **优势**：高度可配置、意图明确、易于审计和管理。

- **ABAC - 基于属性的访问控制 (旧式，复杂)**

  - **工作原理**：基于静态策略文件，策略文件包含一系列JSON格式的规则，每条规则指定了主体属性（如用户`alice`）、资源属性（如资源`pods`）和操作属性（如动词`get`）之间的匹配关系。
  - **缺点**：每次修改策略都需要重启API Server，难以管理和审计，**基本已被RBAC取代**。

- **Node授权 - 节点专用**：授权kubelet对Node、Pod、Endpoint等资源的操作，确保kubelet只能操作绑定到其所在节点的Pod。

- **Webhook授权**：将授权决策委托给外部的RESTful服务，实现自定义授权逻辑。

###### 准入控制 (Admission Control)

- **工作机制**：API Server接收到请求并通过认证授权后，会将请求对象依次发送给一系列准入控制器。每个控制器都可以：
  - **验证 (Validating)**：检查对象是否符合某些标准，如果不符合则拒绝请求。
  - **变更 (Mutating)**：修改传入的对象（例如，注入一个Sidecar容器或设置默认的资源限制）。
- **重要内置控制器**：
  - **NamespaceLifecycle**：防止在正在终止的命名空间中创建新资源。
  - **LimitRanger**：强制执行命名空间中的`LimitRange`对象，为Pod设置默认的计算资源请求和限制。
  - **ResourceQuota**：强制执行命名空间中的`ResourceQuota`对象，限制命名空间的总资源消耗。
  - **DefaultStorageClass**：如果用户没有指定存储类，则为PersistentVolumeClaim设置一个默认的StorageClass。
- **动态准入控制 (Webhook) - 扩展性的核心**
  - **ValidatingAdmissionWebhook**：API Server将请求发送到外部webhook进行校验。
  - **MutatingAdmissionWebhook**：API Server将请求发送到外部webhook进行修改，这是**服务网格（如Istio）自动注入Sidecar容器**的实现原理，它在用户不知情的情况下修改Pod Spec，注入`istio-proxy`容器。
- **ServiceAccount的自动化**
  - 每个命名空间都有一个默认的`default` ServiceAccount。
  - Pod可以通过`spec.serviceAccountName`字段指定要使用的ServiceAccount。如果不指定，则使用默认的。
  - 对应的Secret令牌会被自动挂载到Pod的`/var/run/secrets/kubernetes.io/serviceaccount`目录下。
- **Secret对象**
  - 用于存储敏感信息（密码、令牌、密钥）。
  - 数据以Base64编码存储（**并非加密！**），提供基本的数据混淆。
  - **注意**：默认情况下，Secret以明文形式存储在etcd中，任何有etcd访问权限的人都能看到，**必须启用静态加密**才能保证其安全。

#### 第7章 网络原理

###### Kubernetes 网络模型与基础

- **Kubernetes网络模型的三条核心法则**：
  - **法则一**：所有Pod都可以在不使用NAT的情况下与所有其他Pod通信。
  - **法则二**：所有节点都可以在不使用NAT的情况下与所有Pod通信。
  - **法则三**：Pod自己看到的IP地址（`ip addr show`）与别人看到的它的IP地址是同一个。
- **网络命名空间**：每个Pod都拥有自己独立的网络命名空间，拥有自己独立的IP地址、端口空间、路由表和防火墙规则。
- **容器运行时网络**：Docker的网络基础，包括`docker0`网桥、`veth pair`（虚拟以太网设备对），一个`veth pair`像一根网线，一端插在Pod的网络命名空间里（命名为`eth0`），另一端插在宿主机的`docker0`网桥上。

###### Pod网络解决方案 (CNI)

- **CNI简介：**
  - **容器网络接口**是一个标准的插件规范。它定义了一套简单的命令（如`ADD`, `DEL`, `CHECK`）和配置格式。
  - kubelet会在创建Pod时（`pause`容器启动后）调用配置的CNI插件，为Pod配置网络，销毁Pod时也会调用插件进行清理。
- **主流网络方案实现原理**：
  - **Overlay Network **：在底层网络之上再构建一个虚拟网络，将原始Pod数据包封装在另一个网络包（如UDP）进行隧道传输。
  - **纯三层路由**：要求底层网络设备支持路由。**每个节点都知道其他节点Pod网段的路由**，数据包通过路由直接转发，无封装。
  - **Underlay Network**：Pod直接使用底层网络的IP地址，与主机网络处于同一层级。

###### 外部接入网络 (Ingress)

- **Ingress 不是 Service**：它是一个API对象，描述的是**流量路由规则**，本身不具备流量处理能力。
- **Ingress Controller**：这是**流量的实际处理者**，它是一个 Deployment/Pod，通常以`DaemonSet`或`Deployment`形式运行在集群内，它负责监听Ingress对象的变化，并动态配置自己（如Nginx的配置文件）来实现这些规则。
- **工作原理：**
  - 用户创建Ingress资源，定义“将`foo.example.com`的流量路由到`my-service:80`”。
  - Ingress Controller检测到这个变化。
  - Ingress Controller（如Nginx）重新加载配置，使得访问`foo.example.com`的请求被反向代理到`my-service`的ClusterIP上。
  - Ingress Controller自身需要通过一个`LoadBalancer`或`NodePort`类型的Service暴露到集群外部。

#### 第8章 存储原理和应用

###### Volume 详解 - Pod 内的存储抽象

- **emptyDir**
  - **Volume类型**：本地Volume类型（临时存储）。
  - **工作原理**：Pod被调度到节点时，在节点上创建一个空目录；Pod被删除时，目录及其内容也被清除。
  - **用途**：用于同一个Pod内多个容器之间的**临时数据共享**（如一个容器生产日志，另一个容器处理日志），不适合存储重要数据。
- **nfs**
  - **Volume类型**：网络Volume类型（持久化存储）。
  - **工作原理**：将远程NFS服务器（Network File System）上的目录挂载到Pod中。
  - **用途**：经典的共享存储方案，允许多个Pod以读写模式共享同一存储空间，简单但性能和生产环境高可用性需谨慎考量。
- **configMap / secret**
  - **Volume类型**：网络Volume类型（持久化存储）。
  - **工作原理**：将Kubernetes的ConfigMap或Secret对象中的数据以文件形式挂载到容器内，更新ConfigMap/Secret，已挂载的文件内容可以自动更新（依赖缓存时效性）。
  - **用途**：向容器注入配置信息和敏感数据，实现配置与镜像解耦。
- **云厂商Volume类型**：`awsElasticBlockStore` (EBS), `azureDisk`, `gcePersistentDisk`等。
  - **工作原理**：将云平台提供的块存储设备挂载到Pod中。
  - **特点**：通常是**RWO（ReadWriteOnce）** 模式，即只能被单个节点同时挂载进行读写，适合数据库等需要块设备特性的应用。

###### PV 和 PVC - 存储与计算的解耦

- **PersistentVolume (PV) - 集群级别的存储资源**

  - **定义**：由集群管理员**预先配置好**的一块网络存储，它是集群中的**资源**，就像节点（Node）一样。
  - **创建者**：管理员。
  - **关注点**：定义存储的**具体细节**，如容量、访问模式（RWO, ROX, RWX）、存储类型（NFS, Ceph RBD）、以及如何连接到该存储的具体参数（如NFS path, server）。

  ```yaml
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: pv-nfs-example
  spec:
    capacity:
      storage: 10Gi # 存储容量
    accessModes: # 访问模式
      - ReadWriteMany # 可被多个节点读写
    persistentVolumeReclaimPolicy: Retain # 回收策略，删除PVC后，PV如何处理
    nfs: # 具体的存储类型和参数
      path: /data/nfs
      server: nfs-server.example.com
  ```

- **PersistentVolumeClaim (PVC) - 用户对存储的请求**

  - **定义**：是用户（应用开发者）对存储的**请求**，它类似于Pod，Pod消耗节点资源，而PVC消耗PV资源。
  - **创建者**：用户/开发者。
  - **关注点**：声明所需的存储**规格**，如大小、访问模式，不关心底层实现细节。

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: my-app-pvc
  spec:
    accessModes:
      - ReadWriteMany
    resources:
      requests:
        storage: 5Gi # 请求5G存储
    # storageClassName: "" # 可选，指定存储类名称
  ```

- **绑定与使用流程**

  - 用户创建一个PVC，指定需要5Gi的RWX存储。
  - Kubernetes存储控制器会寻找一个满足要求的PV（如上面创建的10Gi的NFS PV）。
  - 找到后，将PVC和PV**绑定**。绑定是排他的，一个PV只能绑定一个PVC。
  - 用户在Pod中通过`volumes`字段引用PVC名称（而不是PV名称）。
  - Pod被调度时，系统会确保Pod能访问到与其PVC绑定的PV所代表的实际存储。

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: my-pod
  spec:
    containers:
      - name: app
        image: nginx
        volumeMounts:
          - mountPath: "/var/www/html"
            name: app-storage
    volumes:
      - name: app-storage
        persistentVolumeClaim:
          claimName: my-app-pvc # 关键：引用PVC名称
  ```

- **生命周期与回收策略 - persistentVolumeReclaimPolicy**

  - **Retain（保留）**：删除PVC后，PV仍然存在，数据被保留，管理员可以手动回收卷，**最安全**。
  - **Delete（删除）**：删除PVC后，自动删除后端存储资产（如云盘），**最方便但也最危险**。
  - **Recycle（回收**）- **已弃用**：删除数据并让PV可被新的PVC使用。已被Dynamic Provisioning取代。

###### Dynamic Provisioning - 存储的自动化

- **核心概念**：

  - **StorageClass (SC)**：定义了“存储类”，描述了可以动态创建PV的模板和创建者（Provisioner）。
  - **Provisioner**：负责真正创建底层存储设备的组件（如`nfs-client-provisioner`, `ebs.csi.aws.com`）。

- **工作流程**：

  - 管理员创建一个StorageClass，指定provisioner和参数。

  - 用户创建一个PVC，并在其中指定`storageClassName`为上述SC的名字。

  - **关键**：这个PVC**不需要绑定一个现有的PV**。

  - SC对应的provisioner会监听到这个“未绑定”的PVC，并**自动地、按需地**创建对应的后端存储和PV。

  - 新创建的PV会自动绑定到请求它的PVC上。

  ```yaml
  # 1. 管理员创建StorageClass
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: fast-ssd
  provisioner: ebs.csi.aws.com # 指定使用AWS EBS的CSI驱动
  parameters:
    type: gp3
  ---
  # 2. 用户创建PVC，指定StorageClass
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: my-dynamic-pvc
  spec:
    accessModes:
      - ReadWriteOnce
    storageClassName: fast-ssd # 关键：指定存储类名
    resources:
      requests:
        storage: 100Gi
  ```

###### CSI - 容器存储接口

- **CSI**：一个标准化的接口，允许存储厂商编写自己的插件而无需将代码合并到Kubernetes核心代码中。
- **工作原理**：
  - CSI驱动通常以DaemonSet和StatefulSet的形式部署在集群中。
  - 它向kubelet和Controller Manager注册自己。
  - 当需要操作存储时（create/delete/mount），kubelet会通过**Unix Domain Socket**调用本机的CSI驱动来执行具体操作。
- **带来的好处**：
  - **生态繁荣**：存储厂商可以独立开发和发布驱动。
  - **功能扩展**：支持了诸如**卷快照**、**卷克隆**、**卷扩容**等高级特性。

## 第三部份 Kubernetes应用

#### 第9章 Kubernetes开发指南

###### 理解Kubernetes API - 一切的基石

- **API的基本特征**：

  - **RESTful**：Kubernetes API是符合REST规范的，每个资源（如Pod、Service）都有一个唯一的URL路径，通过标准的HTTP方法（GET、POST、PUT、DELETE、PATCH等）进行操作。
  - **声明式**：API资源对象描述了“期望的状态”，系统会不断地调整实际状态以匹配期望状态。
  - **强一致性**：一旦API Server持久化了一个对象的更新，所有后续的读操作都会看到这个最新的版本。

- **API Group**：为了更好组织和扩展，API被划分成多个组。例如：

  - **/api/v1**：核心组，包含Pod、Service、Node等最核心的资源。
  - **/apis/apps/v1**：apps组，包含Deployment、StatefulSet、DaemonSet等。
  - **/apis/batch/v1**：batch组，包含Job、CronJob。
  - **/apis/networking.k8s.io/v1**：networking组，包含Ingress。

- **API Version**：每个API组都有版本，通常有：`v1` (稳定版本)；`v1beta1`, `v1alpha1` (测试版本，功能可能变动或移除)。

- **资源路径**：`/apis/<API_GROUP>/<API_VERSION>/namespaces/<NAMESPACE>/<RESOURCE>/<NAME>`。

- **API的探索与发现**：

  - **使用`kubectl proxy`**：在本地启动一个到API Server的代理，然后可以直接用`curl`或浏览器访问API。

    ```bash
    kubectl proxy --port=8080 &
    curl http://localhost:8080/api/v1/pods
    curl http://localhost:8080/apis/apps/v1/deployments
    ```

  - **使用`kubectl get --raw`**：直接获取原始API信息。

    ```bash
    kubectl get --raw /apis/apps/v1
    ```

  - **Swagger/OpenAPI文档**：Kubernetes API Server暴露了完整的Swagger规范，可以通过`/openapi/v2`端点获取。

###### 使用客户端库 - 编程式交互

- **官方Go客户端库 (client-go)**：

  - **地位**：这是最强大、功能最全的客户端库，也是Kubernetes自身组件（如Controller Manager）使用的库。
  - **List-Watch**：客户端首先列出(`List`)所有相关资源，然后从返回的`resourceVersion`开始监听(`Watch`)后续的变化事件（ADDED, MODIFIED, DELETED）。这是一个长连接。
  - **Informer**：`client-go`提供了一个高级的Informer抽象，它内部实现了List-Watch，并将获取到的对象存储在一个**本地内存缓存**中，你的代码只需要从本地缓存读取数据，速度极快，并注册事件处理函数（`AddFunc`, `UpdateFunc`）来响应变化。
  - **Workqueue**：Informer通常与工作队列结合使用，事件处理函数不直接处理业务逻辑，而是将对象的Key（`namespace/name`）加入到工作队列中，由多个工作线程从队列中取出并处理，实现了异步和批处理，提高了可靠性。

- **Go代码示例（简化）**

  ```go
  package main
  
  import (
      "context"
      "fmt"
      metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
      "k8s.io/client-go/kubernetes"
      "k8s.io/client-go/tools/clientcmd"
  )
  
  func main() {
      // 1. 加载kubeconfig文件，创建配置
      config, err := clientcmd.BuildConfigFromFlags("", "/path/to/your/kubeconfig")
      if err != nil {
          panic(err.Error())
      }
  
      // 2. 创建Clientset
      clientset, err := kubernetes.NewForConfig(config)
      if err != nil {
          panic(err.Error())
      }
  
      // 3. 使用Clientset操作API
      pods, err := clientset.CoreV1().Pods("default").List(context.TODO(), metav1.ListOptions{})
      if err != nil {
          panic(err.Error())
      }
      fmt.Printf("There are %d pods in the default namespace\n", len(pods.Items))
  
      for _, pod := range pods.Items {
          fmt.Printf("- %s\n", pod.Name)
      }
  }
  ```

###### 扩展Kubernetes API - 自定义资源（CRD）和控制器

- **CustomResourceDefinitions (CRD)**：

  - **定义**：用户自定义自己的资源类型，就像内置的Pod、Deployment一样，这个自定义资源可以有自己的`spec`和`status`字段。
  - **场景**：用于声明性地定义你想要管理的任何自定义对象。

- **编写自定义控制器 (Controller/Operator)**：

  - **定义**：CRD定义了“是什么”，而控制器负责“怎么做”。控制器持续监听自定义资源的变化，并根据其`spec`中声明的期望状态，执行具体的操作来驱动当前状态向期望状态收敛。

  - **Operator模式**：Operator是控制器的一种，其核心思想是**将运维人员的知识编码到软件中**，用于管理和自动化有状态应用（如数据库、中间件）的整个生命周期（安装、升级、备份、恢复、扩缩容）。

  - **工作原理**：

    > 使用`client-go`库，为你的CRD创建Informer，监听自定义资源的变化。
    >
    > 当CR被创建、更新或删除时，事件处理函数将被触发。
    >
    > 控制器对比CR的`spec`（期望状态）和当前集群的实际状态（通过查询其他API获得）。
    >
    > 如果状态不一致，控制器就通过调用Kubernetes API或其他外部API执行操作（例如，创建Deployment、创建Secret、调用云服务API等）。
    >
    > 操作完成后，更新CR的`status`字段，反映当前状态。

  - **开发框架**：为了简化控制器的开发，社区有多个高级框架：

    > **Kubebuilder**：Kubernetes SIG项目，当前的主流和官方推荐。
    >
    > **Operator SDK**：由Red Hat主导，基于Kubebuilder，提供了更多开箱即用的工具。

- **API Aggregation (AA) - API聚合**：编写独立的API Server，Kubernetes主API Server会将特定API组的请求转发到服务器上处理。

  - **与CRD的区别**：CRD的数据由Kubernetes API Server存储在其etcd中。而AA允许你完全控制API的端点和存储后端。
  - **适用场景**：需要非常特殊的存储或行为，或者需要将现有系统深度集成到Kubernetes API中。

#### 第10章 Kubernetes运维管理

###### 资源管理

- **资源请求和限制**：为Pod中的容器设置`requests`和`limits`，这是资源调度和质量保证的基石。
- **命名空间资源配额**：使用`ResourceQuota`限制一个命名空间所能使用的总计算资源（CPU/内存）和API对象数量。
- **限制范围**：使用`LimitRange`为命名空间中的容器设置默认的`requests`和`limits`，自动为没有设置的容器加上限制，避免遗忘。

###### 高可用与容灾

- **Master节点高可用**：部署多Master节点，通过负载均衡器暴露API Server，使用多个实例保障控制平面组件（API Server, Scheduler, Controller Manager）和`etcd`集群的可用性。
- **工作节点与Pod分布**：使用`PodAntiAffinity`避免将相同的Pod调度到同一个节点或机柜，分散故障风险。
- **Pod中断预算**：使用`PodDisruptionBudget`在**主动驱逐**时保证应用的高可用。例如，告诉Kubernetes：“我的这个应用，任何时候至少要有3个副本可用，最多只能同时停掉1个。” 这在节点维护或滚动更新时至关重要。

###### 监控和日志

- **监控体系**：
  - **资源指标**：使用`metrics-server`提供基础的CPU/内存监控，供`kubectl top`和HPA使用。
  - **全面监控**：使用**Prometheus**作为监控系统的事实标准，采集所有组件（节点、Pod、Service）的详细指标。
  - **告警**：使用**Alertmanager**根据规则发送告警信息。
- **日志体系**：使用`Fluentd`或`Filebeat`作为日志采集Agent，收集每个容器的日志并发送到`Elasticsearch`。

###### 应用维护与发布

- **Deployment的滚动更新**：Kubernetes的核心功能，逐步用新Pod替换旧Pod，实现零停机部署。
- **Helm**：作为“Kubernetes的包管理器”，通过**Chart**来定义、安装和升级复杂的应用，是管理多资源应用的利器。

###### 安全与权限

- **RBAC**：使用`Role`和`RoleBinding`/`ClusterRole`和`ClusterRoleBinding`为调用方分配精确的权限，杜绝权限过大。
- **Pod安全策略**：控制Pod能否使用特权模式、挂载主机目录等安全敏感操作。

###### 日常故障排查

- **查看Pod状态和事件**：`kubectl describe pod <pod-name>`
- **查看Pod日志**：`kubectl logs <pod-name> [-c <container>]`
- **进入Pod排查**：`kubectl exec -it <pod-name> -- sh`
- **查看集群组件状态**：`kubectl get componentstatuses`
- **查看集群事件**：`kubectl get events --all-namespaces --sort-by='.lastTimestamp'`