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



#### 第6章 深入分析集群安全机制



#### 第7章 网络原理



#### 第8章 存储原理和应用



## 第三部份 Kubernetes应用

#### 第9章 Kubernetes开发指南



#### 第10章 Kubernetes运维管理



#### 第11章 TroubleShooting指南



#### 第12章 Kubernetes开发中的新功能