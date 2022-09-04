## 常用



>kubectl 可简写为 k
>
>pods 可简写为 po
>
>-n ：namespace



- 获取资源

kubectl get pods -n <namespace>

k get po -n xy3-1



- 描述资源

kubectl describe pods <pod-name> -n <namespace>

k describe po scene-0 -n xy3-1



- 查看日志

kubectl logs -f  <pod-name> -n <namespace>

k logs -f scene-0 -n xy3-1



- 删除 yaml 文件

kubectl delete -f <APP>.yaml -n <namespace>

k delete -f scene.yaml -n xy3-1



- 应用 yaml 文件

kubectl  apply -f <APP>.yaml -n <namespace>

k apply -f scene.yaml -n xy3-1



- 进入容器

kubectl exec -it <pod-name> -n <namespace> -- sh

k exec -it scene-0 -n xy3-1 -- sh





# 创建资源

kubectl create -f service.yaml -f rc.yaml

# 获取资源
kubectl get pods

kubectl get rc, service


# 描述资源
kubectl describe nodes <node-name>

kubectl describe pods/<pod-name>

kubectl describe pods <rc-name>


# 删除资源
kubectl delete -f pod.yaml

kubectl delete pods,services -l name=<label-name>

kubectl delete pods --all

# 删除节点
kubectl delete nodes <node-name>


# 进入容器
kubectl exec <pod-name> date

kubectl exec -it <pod-name> -c <container-name> /bin/bash



# 进入镜像

docker run -it -u root 镜像id /bin/bash




# 查看日志
kubectl logs <pod-name>

kubectl logs -f <pod-name> -c <container-name>

# 其他 

#修改副本数
kubectl scale rc redis-slave --replicas=3

#容器暴露的ip:port
kubectl get endpoints

kubectl get namespaces

kubectl create namespace quota-object-example

# 修改对外端口范围 apiserver 的配置文件， 新增 command 参数

vim /etc/kubernetes/manifests/kube-apiserver.yaml

--service-node-port-range=1-65535

# 登录私有仓库
# https://kubernetes.io/zh/docs/tasks/configure-pod-container/pull-image-private-registry/

kubectl create secret docker-registry qianzx \
--docker-server=repo.qianz.com \
--docker-username="ci-user" \
--docker-password="tGEVCyA56w%$" \
--docker-email="ci2user@gmail.com"

kubectl get secret qianzd --output=yaml

kubectl get secret qianzd --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode

