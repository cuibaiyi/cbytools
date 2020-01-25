# **k8s学习笔记**持续更新中...

## **k8s基本概念**

* node是k8s集群中的一个节点
* pod是k8s集群中最小的单位，一个pod可以包含任意多个容器，但总是运行同一个工作节点，绝对不会跨多个工作节点。每个pod都会有自己的ip和主机名 
* ReplicaSet pod副本的抽象，用于解决pod的扩容和伸缩
* Deployments副本集，可以始终保持pod的数量，一个副本集可以控制多个rs
* namespace命名空间，用来区分master，test环境的，不同命名空间，可以有相同的名字
* service负载集群的网络和负载

## **命令简写**
* pods --> po
* services --> svc
* replicationcontroller --> rc
* replicaset --> rs
* HorizontalPodAutoscaler --> hpa

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
```



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
`kubectl explain pods 或者 pod.spec` **查看属性对象的使用方法**
`kubectl  run nginx_demo --image=nginx:v1 --port=80 --generator=run/v1` **generator参数创建ReplicationController**
`kubectl expose` **开放端口**
`kubectl edit rc rc_name` **编辑rc的yaml配置**
`kubectl version`     **查看版本**
`kubectl logs pod_name --previous` **这个参数可以查看前一个容器终止时的日志，而不是当前的容器日志**
`kubectl get nodes` **查看所有node节点**
`kubectl get pods`   **查看所有pod**
`kubectl get pods -o wide` **更详细的输出**
`kubectl get pods pod_name -o yaml` **查看pod的yaml定义,json同理**
`kubectl logs pod_name -c 容器名` **当pod包含多个容器时，查看指定容器的日志**
`kubectl get pods --show-labels` **输出信息列出labels选项**
`kubectl get pods -L lab_name1,lab_name2` **查看具有这2个标签的所有pod**
`kubectl get pods -l app=nginx` **根据标签查看pods信息**
`kubectl get pods -l '!env'` **列出所有没有env标签的pod**
`kubectl label pods pod_name name=cby` **添加标签**
`kubectl label pods pod_name name=test --overwrite` **更新已有的标签**
`kubectl get pods --all-namespaces`  **查看所有命名空间的pod**
`kubectl get pods -n namespace_name -o wide` **查看命名空间详细信息**
`kubectl describe pods pods_name` **查看pods信息**
`kubectl run testname --image=nginx:v1 --port=8080`  **启动一个容器**
`kubectl delete pods pods_name`  **删除一个pods**
`kubectl delete deployments deploy_name`  **删除一个deployments**
`kubectl get deployments` **查看deployments控制器**
`kubectl describe deploy deploy_name` **查看描述**
`kubectl scale deploy deploy_name --replicas=4` **扩缩容 注意：应用本身需要支持水平伸缩**
`kubectl set image deploy deploy_name deploy_name=nginx:v2`  **更新镜像**
`kubectl rollout history deploy deploy_name` **查看升级历史，可以在执行部署的时候加上--record=true，可以记录历史执行的命令**
`kubectl rollout status deploy deploy_name` **查看更新结果**
`kubectl rollout pause deploy deploy_name` **暂停升级**
`kubectl rollout resume deploy deploy_name` **恢复升级**
`kubectl rollout undo deploy deploy_name` **快速回滚镜像**
`kubectl rollout undo deploy deploy_name --to-revision=3` **快速回滚到指定版本，查看REVISION版本号**
`kubectl rolling-update rc-demo --image=nginx:v3` **滚动更新**
`kubectl exec pod_name -i -t /bin/bash`  **进入pod终端** 
`kubectl port-forward pod_name 80:8080` **将本地的80端口转发到pod的8080端口，可以通过ip:80访问pod**

## **资源服务**
### Pod
**Pod在整个生命周期过程中被系统定义为各种状态**

| 状态值 | 描述 |
| --- | --- |
| Pending | API Server已经创建该Pod，但Pod内还有一个或多个容器的镜像没有创建成功，包括正在下载的镜像的过程。|
| Running | Pod内所有容器均已创建，且至少有一个容器处于运行状态、正在启动或正在重启状态。|
| Succeeded | Pod内所有容器均已成功执行退出，且不会再重启。|
| Failed | Pod内所有容器均已退出，但至少有一个容器退出为失败状态。|
| Unknown | 由于某种原因无法获取该Pod的状态，可能由于网络通信不畅导致。|

**Pod的重启策略**
* 注意，这里的重启是指在Pod所在Node上面本地重启，并不会调度到其他Node上去。
* Pod的重启策略包括Always、OnFailure和Never，默认值为Always
* Always: 当容器失效时，由kubelet自动重启该容器
* OnFailure: 当容器终止运行切退出代码不为0时，由kubelet自动重启该容器
* Never: 不论容器运行状态如何，kubelet都不会重启该容器
* RC和DaemonSet: 必须设置为Always，需要保证该容器持续运行
* Job: OnFailure或Never，确保容器执行完成后不再重启。
* Kubelet: 在Pod失效时自动重启它，不论将RestartPolicy设置为什么值，也不会对Pod进行健康检查

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
metdata:
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
  containers:
  - name: liveness
    images: nginx
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
    containers:
    - name: hook-demo1
      images: nginx
      ports:
      - name: webport
        containerPort: 80
      volumeMounts:
      - name: data
        mountPath: /usr/share    #挂载指定容器里的路径
      lifecycle:    #pod的生命周期
        postStart:  #钩子函数,容器创建后立即执行。preStop是容器删除之前调用
          exec:
            command: ["/bin/bash", "-c", "echo 'My name is cby' > /usr/share/test.txt"]
    volumes:
    - name: data    #此名字要和挂载容器里的name一致
      hostPath:
        path: /tmp #宿主机的挂载路径，相当于把本机的/tmp目录挂载到容器里的/usr/share目录里
```
### ConfigMap

```

```
* 使用

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
            configMapKeyRef:     #从configmap对象获取
              name: config-demo2 #指定config对象名
              key: db.host       #获取config对象的值赋予DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: config-demo3
              key: db.port
      envFrom:
        - configMapRef:
            name: cm-demo1
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
  ports:
    - protocol: TCP  #默认TCP
      port: 80       #service的端口
      targetPort: 80 #匹配pod容器端口
      name: service-name #给service端口设置别名
  type: LoadBalancer #指定类型，默认为ClusterIP
```
### Deployment

* RC的全部功能：Deployment具备上面描述的RC的全部功能
* 事件和状态查看：可以查看Deployment的升级详细进度和状态
* 回滚：当升级Pod的时候如果出现问题，可以使用回滚操作回滚到之前的任一版本
* 版本记录：每一次对Deployment的操作，都能够保存下来，这也是保证可以回滚到任一版本的基础
* 暂停和启动：对于每一次升级都能够随时暂停和启动

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-demo
spec:
  replicas: 1
  revisionHistoryLimit: 10 #保留最近10个rs更新版本
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
### DaemonSet



```yaml

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
### Job

```yaml

```
### CronJob

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
          restartPolicy: Never #必须要配置这个
          containers:
          - name: cronjob-demo
            image: cronjob-demo:latest
```

### StatefulSet(有状态的服务对象)

- 稳定的持久化存储，即 Pod 重新调度后还是能访问到相同的持久化数据，基于 PVC 来实现
- 稳定的网络标志，即 Pod 重新调度后其 PodName 和 HostName 不变，基于 Headless Service（即没有 Cluster IP 的 Service）来实现
- 有序部署，有序扩展，即 Pod 是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依序进行（即从 0 到 N-1，在下一个 Pod 运行之前所有之前的 Pod 必须都是 Running 和 Ready 状态），基于 init containers 来实现
- 有序收缩，有序删除（即从 N-1 到 0）

```yaml

```
