# K8S 部署服务



# 单服务部署 （如 mail 应用部署在 cross 服）

## Docker

全部相同无需修改

```dockerfile
//Dockerfile

FROM alpine:latest

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
RUN apk add --no-cache tzdata
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
&& echo 'Asia/Shanghai' >/etc/timezone
RUN mkdir -p /opt/service/bin
WORKDIR /opt/service

COPY /cmd/cmd /opt/service/bin/
COPY /configs/* /opt/service/configs/

ENTRYPOINT ["/opt/service/bin/cmd", "-conf", "/opt/service/configs/"]
```



## CI

需修改应用名及其他情况

```yml
//.gitlab-ci.yml

variables:
  REPO_HARBOR: "repo.qianz.com"
  REPO_PATH:  "xy3"
  REPO_HARBOR_USER: "ci-user"
  REPO_HARBOR_PASS: "tGEVCyA56w%$"
  REPO_PROJECT: "mail"
  IMAGE_VERSION: "k8s"
  NAMESPACE: "xy3-cross"

stages:
- compile
- build
- deploy

compile:
  stage: compile
  tags:
  - middle-end-runner
  only:
  - master
  - k8s
  artifacts:
    paths:
    - cmd/
    - configs/
  image: repo.qianz.com/middle-end/golangandgit:ci-package
  before_script:
  - go version
  script:
  - echo "machine git.huoys.com login autocompile  password d9xNz4oGGmT4w9n2yiDH" > $HOME/.netrc
  - cd cmd && go build

build_image:
  stage: build
  tags:
  - xy3group
  only:
  - master
  - k8s
  before_script:
  - docker login -u $REPO_HARBOR_USER -p $REPO_HARBOR_PASS $REPO_HARBOR
  - ls -l
  script:
  - docker build -t ${REPO_HARBOR}/${REPO_PATH}/${REPO_PROJECT}:$IMAGE_VERSION .
  - docker push ${REPO_HARBOR}/${REPO_PATH}/${REPO_PROJECT}:${IMAGE_VERSION}


deploy_k8s:
  stage: deploy
  tags:
  - xy3group
  only:
  - master
  - k8s
  dependencies:
  - build_image
  image: git.huoys.com:9999/wuyc/kubectl-helm:huoys-dev
  script:
  - launcher_dev


.functions_init: &functions_init |

  function launcher_dev(){

    if [[ -z `kubectl get deploy ${REPO_PROJECT} -n ${NAMESPACE}` ]]
    then
        echo "delete deploy ${NAMESPACE} skip ."
    else
        echo "delete deploy ${NAMESPACE} ${REPO_PROJECT} list:"
        kubectl delete deploy -n ${NAMESPACE} ${REPO_PROJECT}
    fi

    kubectl apply -f ${REPO_PROJECT}.yaml

  }

before_script:
- *functions_init

```



## K8S

需修改应用名及其他情况

```yaml
//mail.yaml

apiVersion: apps/v1
kind: Deployment  // 无状态的应用
metadata:
  name: mail
  namespace: xy3-cross
spec:
  selector:
    matchLabels:
      app: mail
  replicas: 1
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
      labels:
        app: mail
    spec:
      containers:
        - name: mail
          image: repo.qianz.com/xy3/mail:k8s
          imagePullPolicy: "Always"
          ports:
            - containerPort: 8000
              name: http
            - containerPort: 9000
              name: grpc
          command:
            - /bin/sh
            - -c
            - |
              ./bin/cmd -conf ./configs/
          env:
            - name: APP_ID
              value: mail
            - name: LOG_DIR
              value: ./log
            - name: K8S_NAMESPACE
              value: xy3-{SRV_NO}
            - name: SERVERID
              value: "1"
            - name: K8S_CROSS_NAMESPACE
              value: xy3-cross
          volumeMounts:
            - mountPath: /opt/service/configs
              name: xyconfig
      imagePullSecrets:
        - name: qianzx
      volumes:
        - name: xyconfig
          persistentVolumeClaim:
            claimName: xy3-mail-cross

---
apiVersion: v1
kind: Service
metadata:
  name: mail
  namespace: xy3-cross
spec:
  selector:
    app: mail
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
    name: http
  - protocol: TCP
    port: 9000
    targetPort: 9000
    name: grpc
```



# 登录私有仓库

因为要拉取私有仓库的镜像，所以需登录私有仓库，执行以下命令：

```sh
kubectl create secret docker-registry qianzx \
--docker-server=repo.qianz.com \
--docker-username="ci-user" \
--docker-password="tGEVCyA56w%$" \
--docker-email="ci2user@gmail.com" \
-n <namespace>
```

如：

```sh
[root@daxuan-pressure-test xy3]# kubectl create secret docker-registry qianzx \
> --docker-server=repo.qianz.com \
> --docker-username="ci-user" \
> --docker-password="tGEVCyA56w%$" \
> --docker-email="ci2user@gmail.com" \
> -n xy3-1
secret/qianzx created
```



# 多服务部署 （如 scene 应用部署在 1,2,3,4 服）

## Docker

全部相同无需修改

```dockerfile
//Dockerfile

FROM alpine:latest

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories
RUN apk add --no-cache tzdata
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
&& echo 'Asia/Shanghai' >/etc/timezone
RUN mkdir -p /opt/service/bin
WORKDIR /opt/service

COPY /cmd/cmd /opt/service/bin/
COPY /configs/* /opt/service/configs/

ENTRYPOINT ["/opt/service/bin/cmd", "-conf", "/opt/service/configs/"]
```



## CI

需修改应用名及其他情况

```yml
//.gitlab-ci.yml

variables:
  REPO_HARBOR: "repo.qianz.com"
  REPO_PATH:  "xy3"
  REPO_HARBOR_USER: "ci-user"
  REPO_HARBOR_PASS: "tGEVCyA56w%$"
  REPO_PROJECT: "scene"
  IMAGE_VERSION: "k8s0.1"

stages:
- compile
- build
- deploy

compile:
  stage: compile
  tags:
  - middle-end-runner
  only:
  - master
  - k8s
  artifacts:
    paths:
    - cmd/
    - configs/
  image: repo.qianz.com/xy3/golangandgit:1.17
  before_script:
  - go version
  script:
  - echo "machine git.huoys.com login autocompile  password d9xNz4oGGmT4w9n2yiDH" > $HOME/.netrc
  - cd cmd && go build

build_image:
  stage: build
  tags:
  - xy3group
  only:
  - master
  - k8s
  before_script:
  - docker login -u $REPO_HARBOR_USER -p $REPO_HARBOR_PASS $REPO_HARBOR
  - ls -l
  script:
  - docker build -t ${REPO_HARBOR}/${REPO_PATH}/${REPO_PROJECT}:$IMAGE_VERSION .
  - docker push ${REPO_HARBOR}/${REPO_PATH}/${REPO_PROJECT}:${IMAGE_VERSION}


deploy_k8s:
  stage: deploy
  tags:
  - xy3group
  only:
  - master
  - k8s
  dependencies:
  - build_image
  image: git.huoys.com:9999/wuyc/kubectl-helm:huoys-dev
  script:
  - launcher_dev


.functions_init: &functions_init |

  function launcher_dev(){

    SRV_NO=0
    for NAMESPACE in 'xy3-1' 'xy3-2' 'xy3-3' 'xy3-4'
    do

      let SRV_NO+=1
      cp ${REPO_PROJECT}.yaml ${REPO_PROJECT}-${SRV_NO}.yaml
      sed -i "s#{SRV_NO}#${SRV_NO}#g" ${REPO_PROJECT}-${SRV_NO}.yaml

      if [[ -z `kubectl get sts ${REPO_PROJECT} -n ${NAMESPACE}` ]]
      then
          echo "delete sts ${NAMESPACE} skip ."
      else
          echo "delete deploy ${NAMESPACE} ${REPO_PROJECT} list:"
          kubectl delete sts ${REPO_PROJECT} -n ${NAMESPACE}
      fi

      kubectl apply -f ${REPO_PROJECT}-${SRV_NO}.yaml -n ${NAMESPACE}
    done
  }


before_script:
- *functions_init
```



## K8S

需修改应用名及其他情况

```yaml
//scene.yaml

apiVersion: apps/v1
kind: StatefulSet  // 有状态的应用
metadata:
  name: scene
spec:
  selector:
    matchLabels:
      app: scene
  replicas: 1
  serviceName: scene // 有状态的应用才需要
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
      labels:
        app: scene
    spec:
      containers:
      - name: scene
        image: repo.qianz.com/xy3/scene:k8s0.1
        imagePullPolicy: "Always"
        ports:
        - containerPort: 8000
          name: http
        - containerPort: 9000
          name: grpc
        - containerPort: 3101
          name: websocket
        command:
        - /bin/sh
        - -c
        - |
          ./bin/cmd -conf ./configs/
        env:
        - name: APP_ID
          value: scene
        - name: LOG_DIR
          value: ./log
        - name: K8S_NAMESPACE
          value: xy3-{SRV_NO}
        - name: SERVERID
          value: "1"
        - name: K8S_CROSS_NAMESPACE
          value: xy3-cross
        volumeMounts:
        - mountPath: /opt/service/configs
          name: xyconfig
      imagePullSecrets:
      - name: qianzx
      volumes:
      - name: xyconfig
        persistentVolumeClaim:
          claimName: xy3-scene-cfg-server-{SRV_NO}

---
apiVersion: v1
kind: Service
metadata:
  name: scene
spec:
  selector:
    app: scene
  ports:
  - protocol: TCP
    port: 9000
    targetPort: 9000
    name: grpc
  - protocol: TCP
    port: 80
    targetPort: 3101
    name: websocket
```



# NFS 配置文件挂载

**原理**：

Network File System (NFS) 网络文件系统

将 172.13.6.184:22（账号：hzy，密码：hyshzy123.com）SFTP 中的配置文件拉取到 Linux 中，修改 SFTP 中的配置 Linux 中内容会实时更新。



**实现**：

在 172.13.1.112:22（账号：root，密码：huoys.com）Linux 机器的 `/root/xy3-pv-pvc` 目录下，新建对应的文件夹和 yml 文件用于创建 pv,pvc。

如 1 服中的 scene 应用：

```yml
apiVersion: v1
kind: PersistentVolume
metadata:
    name: xy3-scene-cfg-server-1
    labels:
      app: xy3-scene-cfg-server-1
spec:
    capacity:
      storage: 1Gi
    accessModes:
      - ReadWriteMany
    persistentVolumeReclaimPolicy: Retain
    nfs:
      path: /data/nfs/config/1/scene/configs
      server: 172.13.6.184
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: xy3-scene-cfg-server-1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      app: xy3-scene-cfg-server-1
```

**用 k apply -f scene.yml -n xy3-1 生效：**

```sh
[root@daxuan-pressure-test xy3-pv-pvc]# k apply -f ./1/scene.yml -n xy3-1
persistentvolume/xy3-scene-cfg-server-1 created
persistentvolumeclaim/xy3-scene-cfg-server-1 created
```



用 k get pv,pvc -n xy3-1 查看 pv,pvc：



最终会将 SFTP 机器中 `/data/nfs/config/1/scene/configs` 路径下的配置文件拉取到 Linux 机器中 `/opt/service/configs` 路径下。

可以执行进入容器命令查看：

- 进入容器

kubectl exec -it <pod-name> -n <namespace> -- sh

k exec -it scene-0 -n xy3-1 -- sh

```sh
[root@daxuan-pressure-test xy3]# k exec -it scene-0 -n xy3-1 -- sh
/opt/service # ls
bin      configs  log
/opt/service # cd configs/
/opt/service/configs # ls

```

- 退出容器 exit



注意：配置文件修改后需重启服务。



# Docker 常用命令

- 列出容器

docker ps



- 列出镜像

docker images



- 查看镜像信息

docker inspect <image-name>

docker inspect repo.qianz.com/xy3/golangandgit:1.17.4



- 进入镜像

docker run -it -u root 镜像id /bin/bash



# kubectl 常用命令

经过 CI 的 compile--build--deploy 成功后，可在 Linux 中用 kubectl 命令查看：



>kubectl 可简写为 k
>
>pods 可简写为 po
>
>-n ：namespace



- 获取资源

kubectl get pods -n <namespace>

k get po -n xy3-1



- 删除deployment（pod也会被删除）

kubectl delete deployment <deployment-name>

k  delete deployment activity



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



- 重启容器

k delete po <pod-name> -n <namespace> 可重启单个pod

k get po -n xy3-1 |awk '{print $1}' |xargs kubectl delete -n xy3-1 po    这个命令可以重启xy3-1名字空间下的所有容器



# Linux 便捷命令

## 批量替换字符串

用 sed 命令可以批量替换多个文件中的 字符串。 

````
sed -i "s/原字符串/新字符串/g" `grep 原字符串 -rl 所在目录`
````

例如：我要把mahuinan替换 为huinanma，执行命令： 

```
sed -i "s/mahuinan/huinanma/g" `grep mahuinan -rl /www`
```



如：替换当前目录 `xy3-zl` 文件夹下 `cfg-server-1` 替换为 `cfg-server-zl`

```
[root@daxuan-pressure-test xy3-pv-pvc]# sed -i 's/cfg-server-1/cfg-server-zl/g' `grep "cfg-server-1" -rl ./xy3-zl/`
```



替换带有特殊字符 "/" 的字符串，在 / 前加 \ 

```
sed -i 's/data\/nfs\/config\/1/data\/nfs\/config\/3/g' `grep "data/nfs/config/1" -rl ./3`
```



## 批量应用 yaml 文件生成 pv,pvc

如果你碰巧在某个路径下的多个子路径中组织资源，那么也可以递归地在所有子路径上 执行操作，方法是在 `--filename,-f` 后面指定 `--recursive` 或者 `-R`。

例如，假设有一个目录路径为 `project/k8s/development`，它保存开发环境所需的 所有清单，并按资源类型组织：

```
project/k8s/development
├── configmap
│   └── my-configmap.yaml
├── deployment
│   └── my-deployment.yaml
└── pvc
    └── my-pvc.yaml
```

正确的做法是，在 `--filename,-f` 后面标明 `--recursive` 或者 `-R` 之后：

```shell
kubectl apply -f project/k8s/development --recursive
configmap/my-config created
deployment.apps/my-deployment created
persistentvolumeclaim/my-pvc created
```



项目中如：使用  **k apply -f ./xwt -n xy3-xwt -R**  批量生成 pv,pvc

```
[root@daxuan-pressure-test xy3-pv-pvc]# k apply -f ./xwt -n xy3-xwt -R
persistentvolume/xy3-activity-cfg-server-xwt created
persistentvolumeclaim/xy3-activity-cfg-server-xwt created
persistentvolume/xy3-arena-cfg-server-xwt created
persistentvolumeclaim/xy3-arena-cfg-server-xwt created
persistentvolume/xy3-battle-cfg-server-xwt created
persistentvolumeclaim/xy3-battle-cfg-server-xwt created
persistentvolume/xy3-coordinator-cfg-server-xwt created
persistentvolumeclaim/xy3-coordinator-cfg-server-xwt created
persistentvolume/xy3-guild-cfg-server-xwt created
persistentvolumeclaim/xy3-guild-cfg-server-xwt created
persistentvolume/xy3-login-cfg-server-xwt created
persistentvolumeclaim/xy3-login-cfg-server-xwt created
persistentvolumeclaim/xy3-rank-cfg-server-xwt created
persistentvolume/xy3-rank-cfg-server-xwt created
persistentvolume/xy3-scene-cfg-server-xwt created
persistentvolumeclaim/xy3-scene-cfg-server-xwt created
persistentvolume/xy3-task-cfg-server-xwt created
persistentvolumeclaim/xy3-task-cfg-server-xwt created
persistentvolumeclaim/xy3-world-cfg-server-xwt created
persistentvolume/xy3-world-cfg-server-xwt created
```





# BusyBox 常用命令

- 进入 busybox 容器 （退出 exit）

k exec -it busybox -n default -- sh



- nslookup：域名查询

nslookup scene-0.scene.xy3-1.svc.cluster.local

```shell
/ # nslookup scene-0.scene.xy3-1.svc.cluster.local
Server:    10.43.0.10
Address 1: 10.43.0.10 kube-dns.kube-system.svc.cluster.local

Name:      scene-0.scene.xy3-1.svc.cluster.local
Address 1: 10.42.10.70 scene-0.scene.xy3-1.svc.cluster.local
```



- telnet：远端登入

telnet scene-0.scene.xy3-1.svc.cluster.local 3101

```shell
// 连接成功
/ # telnet scene-0.scene.xy3-1.svc.cluster.local 3101

HTTP/1.1 400 Bad Request
Content-Type: text/plain; charset=utf-8
Connection: close

400 Bad RequestConnection closed by foreign host


// 连接失败 Connection refused
/ # telnet scene-0.scene.xy3-1.svc.cluster.local 66
telnet: can't connect to remote host (10.42.10.70): Connection refused

```



- 含 curl 的 busybox（也可用 telnet）

kubectl run --rm -it busybox --image sequenceiq/busybox --restart=Never



- curl：文件传输工具

curl scene-0.scene.xy3-1.svc.cluster.local:3101

```shell
[ root@busybox:/ ]$ curl scene-0.scene.xy3-1.svc.cluster.local:3101
Bad Request

[ root@busybox:/ ]$ curl scene-0.scene.xy3-1.svc.cluster.local:66
curl: (7) Failed to connect to scene-0.scene.xy3-1.svc.cluster.local port 66: Connection refused
```







































