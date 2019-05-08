## **k8s基本概念**

* node是k8s集群中的一个节点
* pod是k8s集群中最小的单位，一个pod可以包含任意多个容器，但总是运行同一个工作节点，绝对不会跨多个工作节点。每个pod都会有自己的ip和主机名 <br>
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

## Service 概念
* LoadBalancer服务，可以创建一个外部的负载均衡，通过负载均衡的公用ip访问pod
  `kubectl expose rc nginx_demo --type=LoadBalancer --name my-http` <br>
  `kubectl get svc`  *查看刚创建的网络对象*<br>
* NodePort服务，可以创建一个让外网访问到内部pod的服务，在node节点上分配一个端口号，访问nodeip+端口范围内部pod服务。

## 服务输出参数介绍
*查看rc*
```
kubectl get rc 
NAME    DESIRED CURRENT READY AGE
my-http    1       1      1   17m
```
 DESIRED: 希望rc保持的个数
 CURRENT: 当前运行的pod数 
 READY: 运行的pod数
 
## **k8s常用命令** 
`kubectl explain pods 或者 pod.spec` **查看属性对象的使用方法**<br>
`kubectl  run nginx_demo --image=nginx:v1 --port=80 --generator=run/v1` **generator参数创建ReplicationController**<br>
`kubectl expose` **开放端口**<br>
`kubectl edit rc rc_name` **编辑rc的yaml配置**<br>
`kubectl version`     **查看版本**<br>
`kubectl logs pod_name --previous` **这个参数可以查看前一个容器终止时的日志，而不是当前的容器日志**<br>
`kubectl get nodes` **查看所有node节点**<br>
`kubectl get pods`   **查看所有pod**<br>
`kubectl get pods -o wide` **更详细的输出**<br>
`kubectl get pods pod_name -o yaml` **查看pod的yaml定义,json同理**<br>
`kubectl logs pod_name -c 容器名` **当pod包含多个容器时，查看指定容器的日志**<br>
`kubectl get pods --show-labels` **输出信息列出labels选项**<br>
`kubectl get pods -L lab_name1,lab_name2` **查看具有这2个标签的所有pod**<br>
`kubectl get pods -l app=nginx` **根据标签查看pods信息**<br>
`kubectl get pods -l '!env'` **列出所有没有env标签的pod**<br>
`kubectl label pods pod_name name=cby` **添加标签**<br>
`kubectl label pods pod_name name=test --overwrite` **更新已有的标签**<br>
`kubectl get pods --all-namespaces`  **查看所有命名空间的pod**<br>
`kubectl get pods -n namespace_name -o wide` **查看命名空间详细信息**<br>
`kubectl describe pods pods_name` **查看pods信息**<br>
`kubectl run testname --image=nginx:v1 --port=8080`  **启动一个容器**<br>
`kubectl delete pods pods_name`  **删除一个pods**<br>
`kubectl delete deployments deploy_name`  **删除一个deployments**<br>
`kubectl get deployments` **查看deployments控制器**<br>
`kubectl describe deploy deploy_name` **查看描述**<br>
`kubectl scale deploy deploy_name --replicas=4` **扩缩容 注意：应用本身需要支持水平伸缩**<br>
`kubectl set image deploy deploy_name deploy_name=nginx:v2`  **更新镜像**<br>
`kubectl rollout history deploy deploy_name` **查看升级历史，可以在执行部署的时候加上--record=true，可以记录历史执行的命令**<br>
`kubectl rollout status deploy deploy_name` **查看更新结果**<br>
`kubectl rollout pause deploy deploy_name` **暂停升级**<br>
`kubectl rollout resume deploy deploy_name` **恢复升级**<br>
`kubectl rollout undo deploy deploy_name` **快速回滚镜像**<br>
`kubectl rollout undo deploy deploy_name --to-revision=3` **快速回滚到指定版本，查看REVISION版本号**<br>
`kubectl rolling-update rc-demo --image=nginx:v3` **滚动更新**<br>
`kubectl exec pod_name -i -t /bin/bash`  **进入pod终端**<br>
`kubectl port-forward pod_name 80:8080` **将本地的80端口转发到pod的8080端口，可以通过ip:80访问pod**<br>
## 资源服务YAML定义
###Pod
```

```
###ConfigMap
```

```
* 使用

```
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

```
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

```
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
```
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
###ReplicaSet
```
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
###Service
* ClusterIP：通过集群的内部 IP 暴露服务，选择该值，服务只能够在集群内部可以访问，这也是默认的ServiceType。
* NodePort：通过每个 Node 节点上的 IP 和静态端口（NodePort）暴露服务。NodePort 服务会路由到 ClusterIP 服务，这个 ClusterIP 服务会自动创建。通过请求，可以从集群的外部访问一个 NodePort 服务。
* LoadBalancer：使用云提供商的负载局衡器，可以向外部暴露服务。外部的负载均衡器可以路由到 NodePort 服务和 ClusterIP 服务，这个需要结合具体的云厂商进行操作。
* ExternalName：通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容（例如，foo.bar.example.com）。没有任何类型代理被创建，这只有 Kubernetes 1.7 或更高版本的 kube-dns 才支持。

```
kind: Service
apiVersion: v1
metadata:
  name: service-demo
spec:
  selector:
    app: nginx
  ports:
    - name: http
      protocol: TCP  #默认TCP
      port: 80       #service的端口
      targetPort: 80 #匹配pod容器端口
  type: LoadBalancer #指定类型，默认为ClusterIP
```
###Deployment
* RC的全部功能：Deployment具备上面描述的RC的全部功能
* 事件和状态查看：可以查看Deployment的升级详细进度和状态
* 回滚：当升级Pod的时候如果出现问题，可以使用回滚操作回滚到之前的任一版本
* 版本记录：每一次对Deployment的操作，都能够保存下来，这也是保证可以回滚到任一版本的基础
* 暂停和启动：对于每一次升级都能够随时暂停和启动

```
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
###HorizontalPodAutoscaler(水平伸缩)
```
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
###Job
```

```
###CronJob
```

```
