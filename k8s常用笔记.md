# **k8s学习笔记**持续更新中...

## **k8s基本概念**

* node是k8s集群中的一个节点
* pod是k8s集群中最小的单位，一个pod可以包含任意多个容器，但总是运行同一个工作节点，绝对不会跨多个工作节点。每个pod都会有自己的ip和主机名 
* ReplicaSet pod副本的抽象，用于解决pod的扩容和伸缩
* Deployment副本集，可以始终保持pod的数量，一个副本集可以控制多个rs
* namespace命名空间，一个集群内部的逻辑隔离机制(鉴权、资源额度)。用来区分master，test环境的，不同命名空间，可以有相同的名字
* service负载集群的网络和负载

## **命令简写**
* pods --> po
* services --> svc
* replicationcontroller --> rc
* replicaset --> rs
* HorizontalPodAutoscaler --> hpa
* configmap --> cm

## **Service 概念**
* LoadBalancer服务，可以创建一个外部的负载均衡，通过负载均衡的公用ip访问pod
  
* ```bash
  kubectl expose rc nginx_demo --type=LoadBalancer --name my-http
  kubectl get svc #查看刚创建的网络对象
  ```
  
* NodePort服务，可以创建一个让外网访问到内部pod的服务，在node节点上分配一个端口号，访问nodeip+端口范围内部pod服务。

## Ingress 概念

- Ingress公开了从集群外部到群集内service的HTTP和HTTPS路由。 流量路由由Ingress资源上定义的规则控制。简单点说就是暴露service可以被外部访问！

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: example-app
  namespace: kube-system  #指定命名空间创建
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: example.cby.com
    http:
      paths:
      - path: /s1
        backend:
          serviceName: app1 #service名字
          servicePort: service-port #service端口,可以使用service端口别名，以后service端口改变了也无需更改Ingress配置
      - path: /s2
        backend:
          serviceName: app2
          servicePort: service-port2
      - path: / #根要写在最后匹配
        backend:
          serviceName: app3
          servicePort: service-port3
```

## **CNI网络插件**

### Flannel

推荐使用VXLAN。建议有经验的用户使用host-gw来提高性能并获得基础架构支持（通常不能在云环境中使用）。建议将UDP仅用于调试或不支持VXLAN的非常老的内核。AWS，GCE和AliVPC是实验性的，不受支持。如使用需自担风险。

flannel SNAT规则优化，容器之间访问使用容器ip，而不是宿主机ip。 https://blog.51cto.com/12965094/2469161

配置
`'{ "Network": "172.7.0.0/16", "Backend": { "Type": "vxlan", "DirectRouting": true}}'`  #DirectRouting默认为false，host-gw当主机位于同一子网中时，启用直接路由（如）。VXLAN仅用于将数据包封装到不同子网中的主机

flannel的host-gw效率最高（增加静态路由，基本没有任何消耗）
`'{ "Network": "172.7.0.0/16", "Backend": { "Type": "host-gw"}}'`  #使用host-gw通过远程计算机IP创建到子网的IP路由，具有良好的性能，几乎没有依赖关系，并且易于设置



## **服务输出参数介绍**
查看rc

```bash
kubectl get rc 
NAME    DESIRED CURRENT READY AGE
my-http    1       1      1   17m
```
 DESIRED: 希望rc保持的个数
 CURRENT: 当前运行的pod数 
 READY: 运行的pod数

## **k8s常用命令** 
#### apply 与 replace 的区别

kubectl replace 的执行过程，是使用新的 YAML 文件中的 API 对象，替换原有的 API 对象；而 kubectl apply，则是执行了一个对原有 API 对象的 PATCH 操作。类似地，kubectl set image 和 kubectl edit 也是对已有 API 对象的修改。更进一步地，这意味着 kube-apiserver 在响应命令式请求（比如，kubectl replace）的时候，一次只能处理一个写请求，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），一次能处理多个写操作，并且具备 Merge 能力。

#### 将文件复制到容器或从容器中复制出来

`kubectl cp pod_name:/etc/hosts hosts` **把容器中的文件复制到本机**
`kubectl cp test.txt -c container_name pod_name:/tmp/` **把文件复制到容器中(当有多个容器时使用-c指定具体容器)**

`kubectl explain pods 或者 pod.spec` **查看属性对象的使用方法**
`kubectl run nginx_demo --image=nginx:v1 --port=80 --generator=run/v1` **generator参数创建ReplicationController**
`kubectl expose` **开放端口**
`kubectl edit rc rc_name` **编辑rc的yaml配置**
`kubectl version`     **查看版本**
`kubectl get nodes` **查看所有node节点**

#### kubectl命令操作pod

`kubectl get pods`   **查看所有pod**
`kubectl get pods -o wide` **更详细的输出**
`kubectl get pods pod_name -o yaml` **查看pod的yaml定义,json同理**
`kubectl get pods --all-namespaces`  **查看所有命名空间的pod**
`kubectl get pods -n namespace_name -o wide` **查看命名空间详细信息**
`kubectl describe pods pods_name` **查看pods信息**
`kubectl delete pods pods_name`  **删除一个pods**

#### 查看容器日志

`kubectl logs pod_name --previous` **这个参数可以查看前一个容器终止时的日志，而不是当前的容器日志**`kubectl logs pod_name -c 容器名` **当pod包含多个容器时，查看指定容器的日志**

#### 操作标签(labels)

`kubectl get pods --show-labels` **输出信息列出labels选项**
`kubectl get pods -L lab_name1,lab_name2` **查看具有这2个标签的所有pod**
`kubectl get pods -l app=nginx` **根据标签查看pods信息**
`kubectl get pods -l '!env'` **列出所有没有env标签的pod**
`kubectl label pods pod_name name=cby` **添加标签**
`kubectl label pods pod_name name=test --overwrite` **更新已有的标签**
`kubectl label pods pod_name name-` **删除name标签**

#### kubectl命令操作deployments

`kubectl delete deployments deploy_name`  **删除一个deployments**
`kubectl get deployments` **查看deployments控制器**
`kubectl describe deploy deploy_name` **查看描述**
`kubectl scale deploy deploy_name --replicas=4` **扩缩容 注意：应用本身需要支持水平伸缩**
`kubectl rolling-update rc-demo --image=nginx:v3` **滚动更新**

#### kubectl命令操作容器

`kubectl exec pod_name -i -t /bin/bash`  **进入pod终端** 
`kubectl exec pod_name -- cat /1.txt `  **不进入容器查看配置**
`kubectl run testname --image=nginx:v1 --port=8080`  **启动一个容器**

#### 端口转发

`kubectl port-forward pod_name 80:8080` **将本地的80端口转发到pod的8080端口，可以通过ip:80访问pod**

## **资源服务**

### NameSpaces

- 删除一个 namespace 会自动删除所有属于该 namespace 的资源。
- `default` 和 `kube-system` 命名空间不可删除。
- PV 是不属于任何 namespace 的，但 PVC 是属于某个特定 namespace 的。
- Event 是否属于 namespace 取决于产生 event 的对象。

`kubectl create namespace test`  **创建test命名空间**

### Pod

**删除命名空间下所有的pod**

`kubectl delete pod --all`

**Pod在整个生命周期过程中被系统定义为各种状态**

| 状态值 | 描述 |
| --- | --- |
| Pending | API Server已经创建该Pod，但Pod内还有一个或多个容器的镜像没有创建成功，包括正在下载的镜像的过程。|
| Running | Pod内所有容器均已创建，且至少有一个容器处于运行状态、正在启动或正在重启状态。|
| Succeeded | Pod内所有容器均已成功执行退出，且不会再重启。cronjob |
| Failed | Pod内所有容器均已退出，但至少有一个容器退出为失败状态。cronjob |
| Unknown | 由于某种原因无法获取该Pod的状态，可能由于网络通信不畅导致。|
| Ready | 长期运行的pod，如果配置健康检查并且通过健康检查的状态。 |
| CrashLoopBackOff | 长期运行的pod，如果配置健康检查但没有通过健康检查的状态。启动失败。 |
| ContainerCreating | pod正在创建当中。 |

**Pod的重启策略**

* 注意，这里的重启是指在Pod所在Node上面本地重启，并不会调度到其他Node上去。
* Pod的重启策略包括Always、OnFailure和Never，默认值为Always
* Always: 当容器失效时，由kubelet自动重启该容器
* OnFailure: 当容器终止运行切退出代码不为0时，由kubelet自动重启该容器
* Never: 不论容器运行状态如何，kubelet都不会重启该容器
* RC和DaemonSet: 必须设置为Always，需要保证该容器持续运行
* Job: OnFailure或Never，确保容器执行完成后不再重启。
* Kubelet: 在Pod失效时自动重启它，不论将RestartPolicy设置为什么值，也不会对Pod进行健康检查

**镜像拉取策略**

- Always：不管本地镜像是否存在都会去仓库进行一次镜像拉取。校验如果镜像有变化则会覆盖本地镜像，否则不会覆盖。
- Never：只是用本地镜像，不会去仓库拉取镜像，如果本地镜像不存在则Pod运行失败。
- IfNotPresent：只有本地镜像不存在时，才会去仓库拉取镜像。ImagePullPolicy的默认值。

**探针介绍**
* livenessProbe: 存活探针: 检查容器是否有问题，杀掉容器重启。

* readinessProbe: 可读性探针作用: 用于判断容器是否完成(Ready状态)，可以接收请求。如果ReadinessProbe探针检测到失败，则Pod的状态将被修改。Endpoint Controller将从Service的Endpoint中删除包含该容器所在的Pod的Endpoint。

**pod的3种健康检查方法**

* 基于cmd shell脚本命令的检查
* 基于http请求的检查
* 基于tcp端口的检查

```yaml
apiVersion: v1
kind: Pod
metdata: #元信息
  name: check-liveness
  labels:
    app: liveness
spec:
  nodeSelector:      #节点选择器,选择节点标签disk为ssd的节点部署pod
    disk: "ssd"
  hostAliases:       #配置容器里的hosts文件,容器里修改hosts文件不生效,需要在pod配置文件里修改
  - ip: "10.120.1.10"
    hostnames:       #可以配置多个hostname
    - "www.dame.com"
  hostNetwork: true  #与宿主机共享网络,可以看到宿主机上的网络连接
  hostPID: true      #与宿主机共享进程,可以看到宿主机上的进程
  initContainers:    #初始化容器，在主容器运行之前完成工作
  - name: init-demo
    image: busybox
    command:
    - curl
    - "http://www.baidu.com"
  terminationGracePeriodSeconds: 30 #默认30s,给老pod发送SIGTERM信号,并等待30秒,超过等待时间会强制结束老pod
  containers:
  - name: liveness
    images: nginx
    imagePullPolicy: IfNotPresent #镜像拉取策略
    args:                    #容器启动后可以执行
    - /nginx                 #执行的命令
    livenessProbe:           #存活探针,如果检测失败,k8s会杀掉pod重启
      httpGet:               #基于http的探针,发送http get请求
        path: /index.html    #检测的url
        port: 80             #检测的容器端口
        scheme: HTTP
      initalDelaySeconds: 10 #表示第一次执行探针的时候等待10s
      periodSeconds: 5       #每隔5s执行一次健康检查
      failureThreshold: 2    #表示健康检查2次失败,则不检查了
      successThreshold: 1    #表示健康检查从失败到检查成功1次，则恢复
      timeoutSeconds: 5      #表示健康检查执行的等待时间
    readinessProbe:          #可读性探针
      tcpSocket:             #基于tcp的探针,发送tcp请求
        port: 8080           #检测tcp端口
      exec:                  #基于命令的探针
        command:
        - /bin/sh
        - -c
        - ps -ef|grep java|grep -v grep
      initalDelaySeconds: 5
      periodSeconds: 5
---
apiVersion: v1
kind: Pod
metdata:
  name: hook-demo1
  labels:
    app: hook
  spec:
    affinity: #亲和性，比nodeSelector更灵活
      nodeAffinity: #节点亲和性
        requiredDuringSchedulingIgnoredDuringExecution: #硬策略
          nodeSelectorTerms:  #多个nodeSelectorTerms是或的关系
          - matchExpressions: #多个matchExpressions是并且的关系
            - key: kubernetes.io/hostname
              operator: NotIn #表达式
              values:
              - node01
        preferredDuringSchedulingIgnoredDuringExecution: #软策略
        - weight: 1 #权重，占比
          preference:
            matchExpressions:
            - key: web
              operator: In
              values:
              - nginx
    containers:
    - name: hook-demo1
      images: nginx
      ports:
      - name: webport
        containerPort: 80
      volumeMounts:
      - name: data
        mountPath: /usr/share    #挂载指定容器里的路径
        readOnly: true #表示只读
      lifecycle:    #pod的生命周期
        postStart:  #钩子函数,容器创建后立即执行。preStop是容器删除之前调用
          exec:
            command: ["/bin/bash", "-c", "echo 'My name is cby' > /usr/share/test.txt"]
        preStop:    #容器停止之前执行
          exec:
            command: ["/bin/bash", "-c", "df -h"]
    volumes:
    - name: data    #此名字要和挂载容器里的name一致
      hostPath:
        path: /tmp #宿主机的挂载路径，相当于把本机的/tmp目录挂载到容器里的/usr/share目录里
```
#### 资源限制

- cpu: "700m" 相当于0.7，最好写成700m的方式！
- 内存的单位则包括 `E, P, T, G, M, K, Ei, Pi, Ti, Gi, Mi, Ki` 等
- 使用容器的时候，你可以通过设置 cpuset 把容器绑定到某个 CPU 的核上，而不是像 cpushare 那样共享 CPU 的计算能力。这种情况下，由于操作系统在 CPU 之间进行上下文切换的次数大大减少，容器里应用的性能会得到大幅提升。事实上，cpuset 方式，是生产环境里部署在线应用类型的 Pod 时，非常常用的一种方式。
- 首先，你的 Pod 必须是 Guaranteed 的 QoS 类型；然后，你只需要将 Pod 的 CPU 资源的 requests 和 limits 设置为同一个相等的整数值即可。
- 当 Pod 里的每一个 Container 都同时设置了 requests 和 limits，并且 requests 和 limits 值相等的时候，这个 Pod 就属于 Guaranteed 类别。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  containers:
    - image: nginx
      name: nginx
      resources:
        requests: #起始使用资源
          cpu: "300m"
          memory: "56Mi" #内存请求，也是调度内存资源的依据，可以超过；但如果超过，容器可能会在 Node 内存不足时清理.
        limits:   #最大限制资源
          cpu: "500m"      #cpu上限，可以短暂超过，容器也不会被停止.
          memory: "128Mi"  #内存上限,不可以超过,如果超过,容器可能会被终止或调度到其他资源充足的机器上
```

#### pod时区

很多容器都是配置了 UTC 时区，与国内集群的 Node 所在时区有可能不一致，可以通过 HostPath 存储插件给容器配置与 Node 一样的时区：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sh
  namespace: default
spec:
  containers:
  - image: alpine
    stdin: true
    tty: true
    volumeMounts:
    - mountPath: /etc/localtime
      name: time
      readOnly: true
  volumes:
  - hostPath:
      path: /etc/localtime
      type: ""
    name: time
```

#### 亲和性调度(affinity)

如果nodeSelectorTerms下面有多个选项的话，满足任何一个条件就可以了。如果matchExpressions有多个选项的话，则必须同时满足这些条件才能正常调度pod。

operator表达式：

- In：label 的值在某个列表中

- NotIn：label 的值不在某个列表中

- Gt：label 的值大于某个值

- Lt：label 的值小于某个值

- Exists：某个 label 存在

- DoesNotExist：某个 label 不存在

- Equal：label的值等于某个值

  

#### pod亲和性与非亲和性

```yaml
  spec:
    affinity:
      podAffinity: #pod亲和性。podAntiAffinity反亲和性，相反的
        requiredDuringSchedulingIgnoredDuringExecution: #硬策略
        - labelSelector:  #多个labelSelector是或的关系
          - matchExpressions: #多个matchExpressions是并且的关系
            - key: app  #表示要和app=web-demo的pod运行在一起
              operator: In #表达式
              values:
              - web-demo
          topologyKey: kubernetes.io/hostname #范围限制
        preferredDuringSchedulingIgnoredDuringExecution: #软策略
        - weight: 1 #权重，占比
          preference:
            matchExpressions:
            - key: web
              operator: In
              values:
              - nginx
```



#### 污点(Taints)

**节点标记为污点后，新创建的pod将不会分配到有污点的node上**

查看污点设置

`kubectl describe node node01`  #输出找到 Taints 列

设置污点（kv的设置模式）

`kubectl taint nodes node01 demotaint=node01:NoSchedule`

删除污点

`kubectl taint nodes node01 demotaint-`

污点三种策略:

- NoSchedule：调度器不会把pod调度到这个节点上。
- NoExecute：不仅不会调度，还会驱逐这个节点上以有的pod。也可以设置容忍时间，多长时间把pod驱逐。
- PreferNoSchedule：尽量不要把pod调度到这个节点上。

#### 容忍(tolerations)

**将具有污点标记的node上允许运行pod**

```yaml
spec:
  containers:
  #省略
  tolerations:
  - key: "demotaint" #设置污点的key
    operator: "Equal" #表达式，Equal代表等于
    value: "node01"  #设置污点的value
    effect: "NoSchedule" #和污点设置的策略一样才行
```

### PodPreset(给Pod 注入额外的信息,如环境变量、存储卷等)

PodPreset 用来给指定标签的 Pod 注入额外的信息，如环境变量、存储卷等。这样，Pod 模板就不需要为每个 Pod 都显式设置重复的信息。

当然，你也可以给 Pod 增加注解 `podpreset.admission.kubernetes.io/exclude: "true"` 来避免它们被 PodPreset 修改。

开启PodPreset

- 开启 API `kube-apiserver --runtime-config=settings.k8s.io/v1alpha1=true`
- 开启准入控制 `--enable-admission-plugins=..,PodPreset`

增加环境变量和存储卷的 PodPreset:

```yaml
kind: PodPreset
apiVersion: settings.k8s.io/v1alpha1
metadata:
  name: allow-database
  namespace: myns
spec:
  selector:
    matchLabels: #匹配标签为app=sit-nginx的pod，增加环境变量和存储。
      app: sit-nginx 
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```

### ConfigMap(存储非安全的配置信息)

保存单个属性、也可以保存整个配置文件，使用kv存储

- 命令行创建ConfigMap对象

`kubectl create configmap config-demo-1 --from-file=data` **把data目录下配置文件写入configmap**
`kubectl create configmap config-demo-2 --from-file=redis.conf --from-file=db.conf` **写入多个配置文件**

- yaml创建ConfigMap对象

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap-demo
data:
  data.1: hello
  data.2: world
  master.cnf: |  #配置文件方式
    port=80
    host=127.0.0.1
```
* 使用ConfigMap

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo-pod
spec:
  containers:
  - name: demo-pod
    image: busybox
    command: [ "/bin/sh", "-c", "env" ] #容器里执行命令，可以执行 echo $DB_HOST 输出变量
    env:
    - name: DB_HOST
      valueFrom:             #环境变量值的来源
        configMapKeyRef:     #从ConfigMap对象获取
          name: config-demo2 #指定config对象名
          key: db.host       #获取config对象的值赋予DB_HOST
    - name: DB_PORT
      valueFrom:
        configMapKeyRef:
          name: config-demo2
          key: db.port
    envFrom:                 #获取config对象整个值
    - configMapRef:
        name: configmap-demo
```
* 通过数据卷挂载配置文件(最常用)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-demo-pod
spec:
  containers:
  - name: demo-pod
    image: busybox
    command: [ "/bin/sh", "-c", "cat /etc/config/redis.conf" ]
    volumeMounts:
    - name: config-volume #标记1
      mountPath: /etc/config  #挂载的数据路径
  volumes:
    - name: config-volume #和标记1的名字要一致
      configMap:
        name: config-redis-demo #configmap对象名
```
* ConfigMap值被映射的数据卷里去控制路径

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo-pod
spec:
  containers:
  - name: test
    image: busybox
    command: [ "/bin/sh","-c","cat /etc/config/path/to/msyql.conf" ]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: config-demo #configmap对象，假如有mysql和redis2个配置文件
        items:
        - key: mysql.conf #指定配置文件
          path: path/to/msyql.conf #把上面的配置文件映射到这个路径。也就是/etc/config/path/to/msyql.conf
        - key: redis.conf
          path: path/to/redis.conf
```
### Secret(存储安全的配置信息)

**secret可以存储 docker register 的鉴权信息，ImagePullSecret参数，用于拉取私有仓库的镜像**

创建docker register secret对象

`kuberctl create secret docker-registry register-name --docker-server=xxx.com --docker-username=cby --docker-password=123456 --docker-email=xxx.qq.com`

pod yaml 中使用

```yaml
spec:
  containers:
  #省略
  imagePullSecrets:
  - name: register-name #创建docker register的secret对象的名字
```

#### 创建secret对象

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-demo
type: Opaque #不透明，一般都使用这个模式，一共有三种模式
data:
  username: aW1vb2M=  #value值需base64编码，通过 echo -n xxx|base64 编码
  passwd: aW1vb2MxMjM=
```

#### pod使用secret存储环境变量

```yaml
spec:
  containers:
  - name: test
    image: busybox
    command: [ "/bin/sh", "-c", "env" ]
    env:
    - name: USERNAME
      valueFrom:            #环境变量值的来源
        secretKeyRef:       #从secret对象获取
          name: secret-demo #指定secret对象名
          key: username     #获取secret对象的username值赋予USERNAME
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: secret-demo
          key: password
```

#### pod使用secret存储文件

```yaml
spec:
  containers:
  - name: test
    image: busybox
    command: [ "/bin/sh","-c","cat /etc/config/passwd" ]
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/config/
  volumes:
    - name: secret-volume
      secret:
        defaultMode: 420 #文件权限
        secretName: secret-demo #secret对象名字
```

### downwardAPI(获取pod里的配置信息)

```yaml
spec:
  containers:
  - name: test
    image: busybox
    command: [ "/bin/sh","-c","cat /etc/config/pidinfo" ]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
    - name: config-volume
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels #获取pod的标签
              - path: "name"
                fieldRef:
                  fieldPath: metadata.name
              - path: "mem-request"
                resourceFieldRef:
                  containerName: web
                  resource: limits.memory #获取pod的内存信息
```

### ReplicationController

```yaml
---          
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc
  labels:
    app: rc
spec:
  replicas: 3 #期望的副本数
  selector: #选择器，选择标签app等于nginx的pod
     app: nginx
  template:   #模板，用于生成pod
    metadata:
      #不需要定义pod_name，如果跑在一台机子就冲突了，让它自动生成。
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx       #容器名称
        image: nginx:1.10 #镜像版本
        ports:
        - containerPort: 80 #容器的端口
```
### ReplicaSet

```yaml
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: replicaset-example
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.10
```
### Service

* ClusterIP：通过集群的内部 IP 暴露服务，选择该值，服务只能够在集群内部可以访问，这也是默认的ServiceType。
* NodePort：通过每个 Node 节点上的 IP 和静态端口（NodePort）暴露服务。NodePort 服务会路由到 ClusterIP 服务，这个 ClusterIP 服务会自动创建。通过请求，可以从集群的外部访问一个 NodePort 服务。
* LoadBalancer：使用云提供商的负载局衡器，可以向外部暴露服务。外部的负载均衡器可以路由到 NodePort 服务和 ClusterIP 服务，这个需要结合具体的云厂商进行操作。
* ExternalName：通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容（例如，foo.bar.example.com）。没有任何类型代理被创建，这只有 Kubernetes 1.7 或更高版本的 kube-dns 才支持。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service-demo
spec:
  selector:
    app: nginx
    version: v1.0
  ports:
    - protocol: TCP  #默认TCP
      port: 80       #service的端口
      targetPort: 80 #匹配pod容器端口
      name: service-name #给service端口设置别名
  type: LoadBalancer #指定类型，默认为ClusterIP
```
#### 将 service 转发到 k8s 集群外部的服务(而不是 Pod)

自定义 endpoint，即创建同名的 service 和 endpoint，在 endpoint 中设置外部服务的 IP 和端口。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
---
kind: Endpoints
apiVersion: v1
metadata:
  name: my-service
subsets:
  - addresses:
      - ip: 1.2.3.4
    ports:
      - port: 9376
```

### Deployment

* RC的全部功能：Deployment具备上面描述的RC的全部功能
* 事件和状态查看：可以查看Deployment的升级详细进度和状态
* 回滚：当升级Pod的时候如果出现问题，可以使用回滚操作回滚到之前的任一版本
* 版本记录：每一次对Deployment的操作，都能够保存下来，这也是保证可以回滚到任一版本的基础
* 暂停和启动：对于每一次升级都能够随时暂停和启动

#### get输出

```bash
# kubectl get deployment
NAME     DESIRED CURRENT UP-TO-DATE AVAILABLE AGE
demo-ng  3       3       3          3         80m
```

DESIRED: 期望pod数量
CURRENT: 当前运行的pod数量
UP-TO-DATE: 到达期望版本的pod数量
AVAILABLE: 运行中并可用的pod数量 
AGE: deployment创建的时间

#### kubectl 命令操作

**rollout命令参数支持3种资源对象**

- deployments
- daemonsets
- statefulsets

```bash
kubectl set image deploy deploy_name deploy_name=nginx:v2 #更新镜像
kubectl rollout history deploy deploy_name #查看升级历史,可以在执行部署的时候加上--record=true,可以记录历史执行的命令
kubectl rollout status deploy deploy_name #查看更新结果
kubectl rollout pause deploy deploy_name  #暂停升级
kubectl rollout resume deploy deploy_name #恢复升级
kubectl rollout undo deploy deploy_name   #快速回滚镜像，回滚到上一个版本
kubectl rollout undo deploy deploy_name --to-revision=3  #快速回滚到指定版本,查看REVISION版本号
```

#### yaml定义

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-demo
spec:
  strategy: #策略，默认是RollingUpdate滚动部署策略。
    type: Recreate #类型重建，当更新时，会终止之前运行的pod，然后重新创建新版本。优点节约资源。
  replicas: 1
  revisionHistoryLimit: 10 #保留最近10个rs更新版本,默认为10
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
        image: nginx:1.13.5-alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```
#### 滚动部署

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-demo
spec:
  strategy: #策略
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25% #可以同时启动pod数量的百分比
      maxUnavailable: 25% #不可用pod的百分比
```



### PV

- pv相当于volume的一个插件。

pv的访问模式：

1. ReadWriteOnce（单个节点挂载读写）
2. ReadOnlyMany（多节点挂载只读）
3. ReadWriteMany（多节点挂载读写）

volume的生命周期5个阶段

1. Provisioning，即 PV 的创建，可以直接创建 PV（静态方式），也可以使用 StorageClass 动态创建
2. Binding，将 PV 分配给 PVC
3. Using，Pod 通过 PVC 使用该 Volume，并可以通过准入控制 StorageObjectInUseProtection（1.9 及以前版本为 PVCProtection）阻止删除正在使用的 PVC
4. Releasing，Pod 释放 Volume 并删除 PVC
5. Reclaiming，回收 PV，可以保留 PV 以便下次使用，也可以直接从云存储中删除
6. Deleting，删除 PV 并从云存储中删除后段存储

pv的状态：

1. Available（pv就绪，且没有和任何pvc绑定）
2. Bound（pv已经和某个pvc绑定了）
3. Released（与pv绑定的pvc已删除，但pv资源还没有被重新声明Available，PVC 解绑但还未执行回收策略）
4. Failed（该pv卷自动回收处理失败）

当删除pvc时触发pv回收的三种策略：

1. Retain（保持Released状态，而不是Available状态，pv数据仍然保留，需手动清理）
2. Recycle（新版已备用，自动删除pv上的数据，然后设置为Available状态）
3. Delete（当绑定pv的pvc被删除后，会将pv和pv关联的存储都删除掉）

`kubectl get pv` **查看创建的pv**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-demo
spec:
  capacity:
    storage: 10Gi   #存储容量
  accessModes:
    - ReadWriteOnce #访问模式
  persistentVolumeReclaimPolicy: Recycle #pv回收策略
  nfs:
    path: "/tmp"
    server: 172.16.1.2
```

### PVC

- 一个pvc只能绑定一个pv，一个pv只能对应一种后端存储。

pvc是对pv资源的请求：

1. pvc负责请求pv的大小和访问方式
2. pvc将与满足请求的pv资源一对一绑定
3. 使用：像Volume一样

`kubectl get pvc` **查看创建的pvc**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: glusterfs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: glusterfs-storage-class #根据名字找到StorageClass
  resources:
    requests:
      storage: 10Gi
```

### StorageClass

pvc按"Class"匹配pv

1. pvc负责请求pv的大小和访问方式。
2. 可以为pv指定storageClassName属性，标识pv归属哪一个Class。
3. 绑定：一个请求绑定特定class的pvc只能绑定具有该属性的pv，没有指定class的pvc仅可以绑定没有特定claas属性字段的pv。

- aws ebs存储插件配置


```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: class-demo
provisioner: kubernetes.io/aws-ebs #使用aws ebs存储插件
parameters: #下面是存储插件参数，每个插件的参数都不一样
  type: io1
  zone: us-east-1d
  iopsPerGB: "10"
```

- glusterfs存储插件配置


```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs-storage-class
provisioner: kubernetes.io/glusterfs #使用存储插件
parameters:
  resturl: "http://ip:port" #glusterfs server的请求地址
  restauthenabled: "false"
```

### 挂载pvc的pod配置

```yaml
  spec:
    containers:
    - name: web-deploy
      images: nginx
      ports:
      - name: webport
        containerPort: 80
      volumeMounts:
      - name: gluster-volume
        mountPath: /usr/share #挂载指定容器里的路径
        readOnly: false  #设置为false，表示可读可写
    volumes:
    - name: gluster-volume    #此名字要和挂载容器里的name一致
      persistentVolumeClaim:
        claimName: glusterfs-pvc #pvc的名字
```

### StatefulSet(有状态的资源对象)

- 推荐在k8s v1.9或以上版本中使用

- 稳定的持久化存储，即 Pod 重新调度后还是能访问到相同的持久化数据，基于 PVC 来实现

- 稳定的网络标志，即 Pod 重新调度后其 PodName 和 HostName 不变，基于 Headless Service（即没有 Cluster IP 的 Service）来实现

- 有序部署，有序扩展，即 Pod 是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依序进行（即从 0 到 N-1，在下一个 Pod 运行之前所有之前的 Pod 必须都是 Running 和 Ready 状态），基于 init containers 来实现

- 有序收缩，有序删除（即从 N-1 到 0）

  ####  先定义headless service(通过dns访问pod的ip地址)


```yaml
apiVersion: v1
kind: Service
metadata:
  name: springboot-web-svc
spec:
  ports: 
  - port: 80
    targetPort: 8080
    protocol: TCP
  clusterIP: None #不使用vip，使用域名解析endpoint ip，保证通过dns找到对应的pod，而不是ip
  selector:
    app: springboot-web
```

#### StatefulSet yaml定义


```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: springboot-web
spec:
  serviceName: springboot-web-svc # serviceName 这个重要
  replicas: 2
  selector:
    matchLabels:
      app: springboot-web
  template:
    metadata:
      labels:
        app: springboot-web
    spec:
      containers:
      - name: springboot-web
        image: hub.com/springboot-web:v1
        imagePullPolicy: IfNotPresent #镜像拉取策略，默认值:本地有则使用本地镜像，没有则拉取镜像
        ports:
        - containerPort: 80
        volumeMounts:
        - name: gluster-volume
          mountPath: /data   #挂载指定容器里的路径
    volumeClaimTemplates:    #会自动创建pvc
    - metadata:
        name: gluster-volume #此名字要和挂载容器里的name一致
      spec:
        accessModes:
        - ReadWriteOnce
        storageClassName: glusterfs-storage-class #根据名字找到StorageClass
        resources:
          requests:
            storage: 10Gi #创建几个副本，就会自动创建几个pv，磁盘使用量=副本数*10G
```

### DaemonSet

- 可以让集群内的每个节点都运行一个相同的pod
- 强烈建议将 DaemonSet 的 Pod 都设置为 Guaranteed 的 QoS 类型，否则，一旦 DaemonSet 的 Pod 被回收，它又会立即在原宿主机上被重建出来，这就使得前面资源回收的动作，完全没有意义了

#### get输出

```bash
# kubectl get ds
NAME     DESIRED CURRENT READY UP-TO-DATE AVAILABLE NODE-SELECTOR AGE
demo-ng  3       3       3     3          3         <none>        4s
```

DESIRED: 期望pod数量
CURRENT: 当前运行的pod数量
READY: 就绪的个数
UP-TO-DATE: 到达期望版本的pod数量
AVAILABLE: 运行中并可用的pod数量
NODE SELECTOR: 节点选择标签 
AGE: deployment创建的时间

#### yaml定义

更新策略:

1. RollingUpdate: 默认的更新策略。当更新模板后，老的pod会被先删除，然后再去创建新的pod。
2. OnDelete: 当DaemonSet模板更新后，只有手动的删除某一个对应pod，此节点pod才会被更新。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: deamonset-example
  labels:
    app: daemonset
spec:
  selector:
    matchLabels:
      name: deamonset-example
  template:
    metadata:
      labels:
        name: deamonset-example
    spec:
      containers:
      - name: daemonset-example
        image: zabbix/zabbix-agent:v1
```

### HorizontalPodAutoscaler(水平伸缩)

* 命令行执行

```kubectl autoscale deployment nginx-deployment --min=1 --max=10 --cpu-percent=85```

* yaml格式

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo
  namespace： default #命名空间，默认为default
spec:
  maxReplicas: 10 #最大扩展到10
  minReplicas: 1  #最新为1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: deploy-demo #指定deployment名字
  targetCPUUtilizationPercentage: 85 #当cpu达到百分之85触发扩容
```
### LimitRange(管理命名空间下pod和container的计算资源)

`kubectl get limitrange` **查看**
`kubectl describe namespaces test` **查看limitrange对象对test命名空间的资源限制**

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range-example
spec:
  limits:
  #针对容器的资源限制
  - default:        #对limit的限制
      cpu: 1
      memory: 200Mi
    defaultRequest: #对request的限制
      cpu: 0.2
      memory: 100Mi
    max:
      cpu: 2
      memory: 2Gi
    min:
      cpu: 0.1
      memory: 50Mi
    type: Container
  #针对pod的资源限制
  - max:
      cpu: 4
      memory: 3Gi
    min:
      cpu: 0.1
      memory: 60Mi
    type: Pod
```

### ResourceQuota(管理命名空间下的总体资源配额)

可以为计算资源、存储资源、对象资源设置配额。

- 资源配额应用在 Namespace 上，并且每个 Namespace 最多只能有一个 ResourceQuota 对象
- 开启计算资源配额后，创建容器时必须配置计算资源请求或限制（也可以用 LimitRange 设置默认值）
- 用户超额后禁止创建新的资源
- ResourceQuota 对象限制当前命名空间下所有 pod 资源 requests 和 limits 的总量，而 LimitRange 是限制每个单独的 pod 或容器

#### 资源配额的类型

计算资源，包括 cpu 和 memory

- cpu, limits.cpu, requests.cpu
- memory, limits.memory, requests.memory

存储资源，包括存储资源的总量以及指定 storage class 的总量

- requests.storage：存储资源总量，如 500Gi
- persistentvolumeclaims：pvc 的个数
- .storageclass.storage.k8s.io/requests.storage
- .storageclass.storage.k8s.io/persistentvolumeclaims
- requests.ephemeral-storage 和 limits.ephemeral-storage （需要 v1.8+）

对象数，即可创建的对象的个数

- pods, replicationcontrollers, configmaps, secrets
- resourcequotas, persistentvolumeclaims
- services, services.loadbalancers, services.nodeports

`kubectl describe quota` **查看配额和配额的使用情况**

以计算资源配置为例：

```yaml
apiVersion: v1
kind: ResourceQuota
metdata:
  name: resources-example
spec:
  hard:
    request.cpu: "1"
    request.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

以对象数可创建对象的个数为例：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resources-example
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
```

### PriorityClass(优先级调度)

```yaml
apiVersion: v1
kind: PriorityClass
metadata:
  name: high-demo
value: 1000000 #对于没有声明 PriorityClass 的 Pod 来说，它们的优先级就是 0。
globalDefault: false #false需pod定义spec层级下添加"priorityClassName: high-demo"，true全局pod使用，只能有一个。
description: "描述信息"
```

### Job(执行一次任务)

`kubectl get jobs job-demo`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-demo
spec:
  template：
    metdata:
      name: job-demo
    spec:
      restartPolicy: Never #pod重启策略
      containers:
      - name: test
        image: busybox
```
### CronJob(定时执行任务)

`kubectl get cronjob cronjob-demo`

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: cronjob-demo
spec:
  schedule: "*/1 * * * *"
  succeessfulJobsHistoryLimit: 3 #保留成功的定时任务数量限制，运行完pod不会马上删除，保留3个
  suspend: false #false表示立即运行定时任务策略
  concurrencyPolicy: Forbid
  failedJobsHistorryLimit: 1
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: cronjob-demo
        spec:
          restartPolicy: Never #必须要配置这个，pod重启策略
          containers:
          - name: cronjob-demo
            image: cronjob-demo:latest
```

### RBAC(基于角色的控制)

1. Role：角色，它其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限。
2. Subject：被作用者，既可以是“人”，也可以是“机器”，也可以是你在 Kubernetes 里定义的“用户”。
3. RoleBinding：定义了“被作用者”和“角色”的绑定关系。

- 在使用 RBAC 时，只需要在启动 kube-apiserver 时配置 `--authorization-mode=RBAC` 即可。
- Kubernetes 里的"内置用户"：ServiceAccount。
- 对于非 Namespaced 对象(比如: Node)，或者某一个 Role 想要作用于所有的 Namespace 的时候，就可以使用 ClusterRole 和 ClusterRoleBinding。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: demo-role
  namespace: sit-demo
rules:
- apiGroups: ["apps", "extensions"] #组
  resources: ["deployments", "replicasets"] #资源对象
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"] #*代替所有操作权限
- apiGroups: [""] #""代表核心组
  resources: ["pods"]
  resourceNames: ["nginx-prod"] #只对名字叫nginx-prod的pod生效
  verbs: ["get", "list"] #操作权限
```

创建基于用户的权限绑定

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: demo-rolebinding
  namespace: sit-demo
subjects: #操作集群的对象
- kind: User
  name: cby
  apiGroup: ""
roleRef: #角色
- kind: Role
  name: demo-role #Role的名字
  apiGroup: ""
```

创建ServiceAccount对象

`kubectl create sa demo-sa -n sit-demo` **创建**

创建基于ServiceAccount权限绑定

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: demo-rolebinding
  namespace: sit-demo
subjects: #操作集群的对象
- kind: ServiceAccount
  name: cby-sa
  namespace: sit-demo
roleRef:
- kind: Role
  name: demo-sa-role
  apiGroup: rbac.authorization.k8s.io
```

创建一个可以访问所有namespace的ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo-cluster
  namespace: kube-system
  
---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: demo-clusterrolebinding
subjects: 
- kind: ServiceAccount
  name: demo-cluster #匹配需绑定的ServiceAccount名字
  namespace: kube-system #要和需要绑定的ServiceAccount一样的命名空间
roleRef:
- kind: ClusterRole
  name: cluster-admin #最高权限的集群角色,系统默认自带
  apiGroup: rbac.authorization.k8s.io
```

## 收集、获取实际资源使用情况

### Heapster(目前已弃用)

#### 部署文档

https://github.com/kubernetes-retired/heapster

**请使用metrics-server和第三方指标管道来手机prometheus格式的指标**
https://github.com/kubernetes-sigs/metrics-server

#### 基本操作

查看集群节点的cpu和内存使用量
`kubectl top node`

查看单独pod的cpu和内存使用量
`kubectl top pod --all-namespaces`

如果要查看容器而不是pod的资源使用情况，可以使用 `--container` 选项
