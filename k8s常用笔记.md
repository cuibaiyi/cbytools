## **k8s基本概念**

* node是k8s集群中的一个节点
* pod是k8s集群中最小的单位，一个pod可以包含任意多个容器
* ReplicaSet pod副本的抽象，用于解决pod的扩容和伸缩
* Deployments副本集，可以始终保持pod的数量
* namespace命名空间，用来区分master，test环境的，不同命名空间，可以有相同的名字
* service负载集群的网络和负载

## **k8s常用命令**

`kubectl version`     **查看版本** <br>
`kubectl get nodes` **查看所有node节点** <br>
`kubectl get pods`   **查看所有pod** <br>
`kubectl get pods -o wide` **更详细的输出** <br>
`kubectl get pods -l app=nginx` **根据标签查看pods信息 <br>
`kubectl get pods --all-namespaces`  **查看所有命名空间**
`kubectl get pods -n namespace_name -o wide` **查看命名空间详细信息**
`kubectl describe pods pods_name` **查看pods信息**
`kubectl run testname --image=nginx:v1 --port=8080`  **启动一个容器**
`kubectl delete pods pods_name`  **删除一个pods**
`kubectl delete deployments deploy_name`  **删除一个deployments**
`kubectl get deployments` **查看deployments控制器**
`kubectl describe deploy deploy_name` **查看描述**
`kubectl scale deploy deploy_name --replicas=4` **扩缩容**
`kubectl set image deploy deploy_name deploy_name=nginx:v2`  **更新镜像**
`kubectl rollout status deploy deploy_name` **查看更新 结果**
`kubectl rollout undo deploy deploy_name` **快速回滚镜像**
`kubectl exec pod_name -i -t /bin/bash`  **进入pod终端**
