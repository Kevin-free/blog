# Docker 常用命令

### 查看运行容器

```
docker ps
```

### 查看所有容器

```
docker ps -a
```

### 删除容器

```
docker rm [CONTAINER ID]
```

### 进入容器

```
docker exec -it [CONTAINER ID] bash
```





### 问题：

#### Conflict. The container name "XXX" is already in use by container 

```sh
docker: Error response from daemon: Conflict. The container name "/tars-framework" is already in use by container "6c7adba72dae1223b8c3b7d3001ef0390281aad10c9349788849053059642502". You have to remove (or rename) that container to be able to reuse that name.
See 'docker run --help'.
```

解决：

docker ps -a

docker rm [CONTAINER ID]