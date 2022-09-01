# CI & CD



## 简介

1. GitLab

   > 是一套基于Ruby开发的开源Git项目管理应用，其提供的功能和Github类似，不同的是GitLab提供一个GitLab CE社区版本，用户可以将其部署在自己的服务器上，这样就可以用于团队内部的项目代码托管仓库。

2. GitLab CI（Continuous Integration）

   >是GitLab 提供的持续集成服务(从8.0版本之后，GitLab CI已经集成在GitLab中了)，只要在你的仓库根目录下创建一个.gitlab-ci.yml 文件， 并为该项目指派一个Runner，当有合并请求或者Push操作时，你写在.gitlab-ci.yml中的构建脚本就会开始执行。

3. GitLab CD（Continuous Delivery/Continuous Deployment）

   >持续集成是程序开发人员在频繁的提交代码之后，能有相应的环境能对其提交的代码自动执行构建(Build)、测试(Test),然后根据测试结果判断新提交的代码能否合并加入主分支当中,而持续部署也就是在持续集成之后自动将代码部署(Deploy)到生产环境上

4. GitLab Runner

   > 是配合GitLab CI进行构建任务的应用程序，GitLab CI负责yml文件中各种阶段流程的执行，而GitLab Runner就是具体的负责执行每个阶段的脚本执行。一般来说GitLab Runner需要安装在单独的机器上通过其提供的注册操作跟GitLab CI进行绑定，当然，你也可以让其和GitLab安装在一起，只是有的情况下，你代码的构建过程对资源消耗十分严重的时候，会拖累GitLab给其他用户提供政策的Git服务。



![Deeper look into the basic CI/CD workflow](https://docs.gitlab.com/ee/ci/introduction/img/gitlab_workflow_example_extended_v12_3.png)



## 开启 GitLab CI 功能

在GitLab 8.0版本之后,你可以通过如下两部启用GitLab CI功能

1. 新建一个`.gitlab-ci.yml`文件在你项目的根目录
2. 为你的项目配置一个GitLab Runner



### 配置一个`.gitlab-ci.yml`文件

`.gitlab-ci.yml`文件是用来配置GitLab CI进行构建流程的配置文件，其采用YAML语法,所以你需要额外注意要用空格来代替缩进，而不是Tabs。下面通过我自己项目中的`.gitlab-ci.yml`文件来具体介绍其规则

```
stages:
  - init
  - check
  - build
  - deploy

cache:
  key: ${CI_BUILD_REF_NAME}
  paths:
  - node_modules/
  - dist/

#定义一个叫init的Job任务
init:
  stage: init
  script:
  -  cnpm install

#master_check Job:检查master分支上关键内容
master_check:
  stage: check
  script:
  - echo "Start Check Eros Config ..."
  - bash check.sh release 
  only:
  - master
  
#dev_check Job: 检查dev分支上关键内容
dev_check:
  stage: check
  script:
  - echo "Start Check Eros Config ..."
  - bash check.sh debug
  only:
  - dev

js_build:
  stage: build
  script:
  - eros build

master_deploy:
  stage: deploy
  script:
  - bash deploy.sh release
  only:
  - master

dev_deploy:
  stage: deploy
  script:
  - bash deploy.sh debug
  only:
  - dev

复制代码
```

在上面的例子中，我们利用`stages`关键字来定义持续构建过程中的四个阶段init、chec、build、deploy

> 关于GitLab CI中的stages,有如下几个特点:

```
1. 所有 Stages 会按照顺序运行，即当一个 Stage 完成后，下一个 Stage 才会开始
2. 只有当所有 Stages 完成后，该构建任务 (Pipeline) 才会成功
3. 如果任何一个 Stage 失败，那么后面的 Stages 不会执行，该构建任务 (Pipeline) 失败
复制代码
```

然后我们利用`caches`字段来指定下面将要进行的job任务中需要共享的文件目录,如果没有，每个Job开始的时候，GitLab Runner都会删掉`.gitignore`里面的文件

紧接着，我们定义了一个叫做`init`的job，其通过stage字段声明属于`init`阶段，因此，这个job会第一个执行，我们在这个job当中，执行一些环境的初始化工作。

接下来是`check`阶段,用来检查代码的一些基础错误(代码规范之类不会被编译器发现的问题)，以及一些配置文件的检查，我将其命名为`master_check`和`dev_check`,通过only字段来告诉GitLab CI 只有当对应的分支有push操作的时候才会触发这个job。

然后就是代码的`build`阶段，由于此阶段不像上个极端，没有需要区分不同分支的命令，所以就只需要定义一个job就够了

最后的`deploy`，因为不同的分支需要发布到不同的环境，所以依然通过only来区分两个job。

> 关于GitLab CI中的Jobs,也有如下几个特点:

```
1. 相同 Stage 中的 Jobs 会并行执行
2. 相同 Stage 中的 Jobs 都执行成功时，该 Stage 才会成功
3. 如果任何一个 Job 失败，那么该 Stage 失败，即该构建任务 (Pipeline) 失败
复制代码
```

在我的这个构建任务当中，根据我的业务情况只用到了少许关键字,还有更多的类似于`before_script`、`after_script`等关键字，具体的可以参阅[GitLab的官方文档](https://link.juejin.cn?target=https%3A%2F%2Fdocs.gitlab.com%2Fce%2Fci%2Fyaml%2FREADME.html)

在我们完成`.gitlab-ci.yml`的流程编写之后，就可以将其放在项目的根目录下，然后push到我们的GitLab上，这时，如果你打开项目首页的Piplines标签页，会发现一个状态标识为`pending`的构建任务，如下图所示：
 ![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/4/19/162dd40d3b222beb~tplv-t2oaga2asx-watermark.awebp)
 这时由于这个构建任务还有找到可用的GitLab Runner来执行其构建脚本，等我们接下来为我们的项目接入GitLab Runner之后，这些任务的状态就会由`pendding`变成`running`了

### 安装GitLab Runner

找一台适合安装GitLab Runner的机器，无论是Windows或者Mac还是Linux都行，最好是那种比较空闲的能24小时开启的机器，我们在[GitLab Runner的官网](https://link.juejin.cn?target=https%3A%2F%2Fdocs.gitlab.com%2Frunner%2Finstall%2F)找到我们平台的安装文件，以及对应的安装流程。由于笔者我准备安装GitLab Runner是一台闲置的iMac电脑，因此我就在演示MacOS下GitLab Runner的安装：

> GitLab Runner 在macOS和Linux/UNIX下安装流程是一样的，都是直接下载已编译好的二进制包

1. 下载对应GitLab 版本的GitLab Runner

   - 如果你的GitLab是10.0之后的版本，GitLab Runner可执行文件改名为`gitlab-runner`

   ```
   sudo curl --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-darwin-amd64
   sudo chmod +x /usr/local/bin/gitlab-runner
   复制代码
   ```

   - 9.0~10.0之间的版本

   ```
   sudo curl --output /usr/local/bin/gitlab-ci-multi-runner https://gitlab-ci-multi-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-ci-multi-runner-darwin-amd64
   sudo chmod +x /usr/local/bin/gitlab-ci-multi-runner
   复制代码
   ```

   - 9.0 之前,由于9.0之后启用全新的API4接口，所以如果你的GitLab是9.0以前的版本,需要下载下面的版本,否则会导致你的GitLab Runner注册不上

   ```
   sudo curl --output /usr/local/bin/gitlab-ci-multi-runnerhttps://gitlab-ci-multi-runner-downloads.s3.amazonaws.com/v1.11.1/binaries/gitlab-ci-multi-runner-darwin-amd64
   sudo chmod +x /usr/local/bin/gitlab-ci-multi-runner
   复制代码
   ```

   >  不知道各位有没有注意到上面下载地址链接当中的v1.11.1,这个就是对应的Gitlab Runner,如果你的GitLab是9.0之前的版本，使用GitLab Runner v1.11.1这个版本仍然注册不上，可以尝试使用降几个版本的GitLab Runner,所有GitLab Runner发行的版本可以在[GitLab Runner Tags](https://link.juejin.cn?target=https%3A%2F%2Fgitlab.com%2Fgitlab-org%2Fgitlab-runner%2Ftags)找到 

   假如你遇到不能通过登录服务器来确定GitLab版本号时，可以通过直接访问gitlab的首页，后面加上help，如下图：
    ![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/4/19/162dd40d3c69dc4b~tplv-t2oaga2asx-watermark.awebp)

2. 注册GitLab Runner

   执行下面命令，

   ```
   sudo gitlab-ci-multi-runner register
   复制代码
   ```

   如果你的终端提示找不到命令，请通过`export PATH=/usr/local/bin:$PATH`将/usr/local/bin目录加入环境变量,或者你遗漏了上面的chmod命令导致文件不可执行。

   执行完上面的命令之后，会让你输入下面的信息:

   - Please enter the gitlab-ci coordinator URL:
   - Please enter the gitlab-ci token for this runner:
   - Please enter the gitlab-ci description for this runner
   - Please enter the gitlab-ci tags for this runner (comma separated):
   - Whether to run untagged builds [true/false]:
   - Please enter the executor:

   其中coordinator URL和token可以在你需要进行持续集成的项目的Runner标签页中找到
    ![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/4/19/162dd40d3ba05b0e~tplv-t2oaga2asx-watermark.awebp)
    description和tags可以自己定义，是否build没有tag的提交这个也是根据你自己的需求来选择，默认是false，executer选择shell

   填写完成之后如果提示`Registering runner... succeeded`表明这个Runner已经被注册成功了，之后你在返回进入项目的Runners页面，会发现下面多了一个处于Active状态的Runner

   紧接着最后一步，启动我们刚注册的Runner

   ```
   sudo  gitlab-ci-multi-runner start
   复制代码
   ```

   现在，我们切回项目的Pipelines当中，肯定会发现之前处于peding状态的任务已经开始running了，我们可以通过点击这个状态按钮来实时查看每个阶段的输出日志
    ![img](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/4/19/162dd40d3d3753f8~tplv-t2oaga2asx-watermark.awebp)






### 备注：

注册 runner 所选项：

![image-20211108175743651](C:\Users\xiongwt\AppData\Roaming\Typora\typora-user-images\image-20211108175743651.png)



runners.docker 修改：

问题：

```sh
$ docker login -u $REPO_HARBOR_USER -p $REPO_HARBOR_PASS $REPO_HARBOR
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
error during connect: Post http://docker:2375/v1.40/auth: dial tcp: lookup docker on 172.13.0.7:53: server misbehaving
```

解决：

```sh
[[runners]]
  name = "chain-game-group runner"
  url = "https://git.huoys.com/"
  token = "RZQXGkiEphPSRvsvM27P"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "repo.qianz.com/middle-end/docker:stable-dind"
    privileged = false
    disable_cache = false
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"] # 修改处
    pull_policy = "if-not-present" # 修改处
    shm_size = 0
  [runners.cache]
```





## GolangandGit

```dockerfile
//cd /root/golanggit
- Dockerfile
// Dockerfile
FROM golang:1.17.4-alpine3.15

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories

RUN apk update && \
    apk upgrade && \
    apk add --no-cache bash git openssh

RUN apk add build-base

RUN apk add vim

RUN go env -w GOPROXY=https://goproxy.cn,direct && \
    go env -w GONOPROXY=git.huoys.com && \
    go env -w GOSUMDB=off
```

（repo 中先新建 chain-game-rowing 项目）

- docker build -t repo.qianz.com/chain-game-rowing/golangandgit:1.17.0 .

- docker push repo.qianz.com/chain-game-rowing/golangandgit:1.17.0



## Kubectl

```dockerfile
//cd /home/helm
- config
- dockerfile
- kubectl

// config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJlRENDQVIrZ0F3SUJBZ0lCQURBS0JnZ3Foa2pPUFFRREFqQWtNU0l3SUFZRFZRUUREQmx5YTJVeUxYTmwKY25abGNpMWpZVUF4TmpNM05Ua3hOVE0zTUI0WERUSXhNVEV5TWpFME16SXhOMW9YRFRNeE1URXlNREUwTXpJeApOMW93SkRFaU1DQUdBMVVFQXd3WmNtdGxNaTF6WlhKMlpYSXRZMkZBTVRZek56VTVNVFV6TnpCWk1CTUdCeXFHClNNNDlBZ0VHQ0NxR1NNNDlBd0VIQTBJQUJNMERsSkhRaDBrd1hnQXVUSVBKVWNocHgrYmkrQ2lPc0RTUWN6cmoKZUlQNzdTeDZkeEdNeFhTVWtTUGdaQTRscUpBVnVCYWxUbXhRN3lnY3M1Z1B3aXVqUWpCQU1BNEdBMVVkRHdFQgovd1FFQXdJQ3BEQVBCZ05WSFJNQkFmOEVCVEFEQVFIL01CMEdBMVVkRGdRV0JCUnMyaWQ4VGlxVmNJY2JEOHNlCldvWmhRVUd4WHpBS0JnZ3Foa2pPUFFRREFnTkhBREJFQWlBUUh1WmFZWU5abFl3RGZzZXFKcitBd2pVMTNhZmsKd0hGV0QyY1libWxoQ3dJZ1MycStXVFg3SW55Wjd5T09xaVRqbmszbmVnTUJUZE51aHcyK1NITEVOSFU9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://172.18.6.111:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJrekNDQVRpZ0F3SUJBZ0lJSXNGdDMxM0o2QWd3Q2dZSUtvWkl6ajBFQXdJd0pERWlNQ0FHQTFVRUF3d1oKY210bE1pMWpiR2xsYm5RdFkyRkFNVFl6TnpVNU1UVXpOekFlRncweU1URXhNakl4TkRNeU1UZGFGdzB5TWpFeApNakl4TkRNeU1UZGFNREF4RnpBVkJnTlZCQW9URG5ONWMzUmxiVHB0WVhOMFpYSnpNUlV3RXdZRFZRUURFd3h6CmVYTjBaVzA2WVdSdGFXNHdXVEFUQmdjcWhrak9QUUlCQmdncWhrak9QUU1CQndOQ0FBUUhQMGxKVXBMVFp4bnYKNkQzSWNabHR4Mm5CSjZ1T1VVL1BwR3hPaXdMWGQ2TTcwTWVrYnh4a3pldk1lM1E5eVVEUE8wTy9VMzh3c1hOSApQY2ptbzg2em8wZ3dSakFPQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUhBd0l3Ckh3WURWUjBqQkJnd0ZvQVUrTEhNeEZmY3hhVHJ3clRPWHN2R004VEwwaUF3Q2dZSUtvWkl6ajBFQXdJRFNRQXcKUmdJaEFPSk1tTzV4bi9BQXJieVBrNnQ0eFBPanJab0dqOXBScW1xU2VoL09VRktXQWlFQXQ3eXpvdUNiNkVEMQpEQUUrVjdld1UzUHJpelJyZ2tyTm9MK3Y0R1V5QitjPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCi0tLS0tQkVHSU4gQ0VSVElGSUNBVEUtLS0tLQpNSUlCZURDQ0FSK2dBd0lCQWdJQkFEQUtCZ2dxaGtqT1BRUURBakFrTVNJd0lBWURWUVFEREJseWEyVXlMV05zCmFXVnVkQzFqWVVBeE5qTTNOVGt4TlRNM01CNFhEVEl4TVRFeU1qRTBNekl4TjFvWERUTXhNVEV5TURFME16SXgKTjFvd0pERWlNQ0FHQTFVRUF3d1pjbXRsTWkxamJHbGxiblF0WTJGQU1UWXpOelU1TVRVek56QlpNQk1HQnlxRwpTTTQ5QWdFR0NDcUdTTTQ5QXdFSEEwSUFCSFZrRHcrbU43TFJnQ3dxNGt0c09HTjV4N0tGalNuSWRJLzkzYkRUCmdjQzA3U1VkUGMxR2F4QkdiTGV0RWQyYzUxVFJmVTBIK2o2UTVDa0FxNDQvTWZDalFqQkFNQTRHQTFVZER3RUIKL3dRRUF3SUNwREFQQmdOVkhSTUJBZjhFQlRBREFRSC9NQjBHQTFVZERnUVdCQlQ0c2N6RVY5ekZwT3ZDdE01ZQp5OFl6eE12U0lEQUtCZ2dxaGtqT1BRUURBZ05IQURCRUFpQXVXSG1KWUJpV3llM2tRdkNYdW41QWR6b3c3U1k3CnpuM01vc3orSE1zR1VnSWdBTW5ad3JlQVJXNFNwQ2FoM2N1YVJxTW9BOUZHZVNaTjkwK0xtUkcxekFVPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBFQyBQUklWQVRFIEtFWS0tLS0tCk1IY0NBUUVFSUN2NG9RR3dUd2ZtVmxTVC9FdENpdnhYRGI1ME14OHNkS085Si9mQzNiY2xvQW9HQ0NxR1NNNDkKQXdFSG9VUURRZ0FFQno5SlNWS1MwMmNaNytnOXlIR1piY2Rwd1NlcmpsRlB6NlJzVG9zQzEzZWpPOURIcEc4YwpaTTNyekh0MFBjbEF6enREdjFOL01MRnpSejNJNXFQT3N3PT0KLS0tLS1FTkQgRUMgUFJJVkFURSBLRVktLS0tLQo=


// dockerfile 
FROM alpine:latest

ADD ./config /opt/

RUN  mkdir -p ~/.kube && \
     mv /opt/config ~/.kube/

ADD ./kubectl /bin/
```

- docker build -t repo.qianz.com/chain-game-rowing/kubectl-111:v2 .
- docker push repo.qianz.com/chain-game-rowing/kubectl-111:v2





## Reference

[GitLab 官方文档](https://docs.gitlab.com/ee/ci/)

[GitLab 中文文档](https://www.bookstack.cn/books/gitlab-doc-zh)

[利用GitLab提供的GitLab-CI以及GitLab-Runner搭建持续集成/部署环境](https://juejin.cn/post/6844903593259040782)

[简析 GitLab Runner](https://qtozeng.top/2019/07/25/%E7%AE%80%E6%9E%90-GitLab-Runner/)

[Gitlab CI Multi Runner搭建CI持续集成环境](https://www.cxymm.net/article/u010893333/52685796)

