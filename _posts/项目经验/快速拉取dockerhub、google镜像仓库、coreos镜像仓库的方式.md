快速拉取dockerhub、google镜像仓库、coreos镜像仓库的方式

常见的镜像仓库地址：

1. DockerHub镜像仓库:https://hub.docker.com/
2. 阿里云镜像仓库：https://cr.console.aliyun.com
3. google镜像仓库：https://console.cloud.google.com/gcr/images/google-containers/GLOBAL
4. coreos镜像仓库：https://quay.io/repository/
5. RedHat镜像仓库：https://access.redhat.com/containers

1、快速拉取dockerhub公共镜像，私有镜像不行哦

平时我们拉取dockerhub镜像比较慢的时候，可以使用如下方式去拉镜像：

```
docker pull dockerhub.azk8s.cn/library/<imagename>:<version> 
#例子
docker pull dockerhub.azk8s.cn/library/centos
```

拉下来的镜像名字是dockerhub.azk8s.cn/library/centos:latest，但他和dockerhub上的centos:latest是一样的，就是走的微软加速器地址拉的，所以拉下来的镜像名称加了一个前缀dockerhub.azk8s.cn。你要是觉得别扭就把他重新tag一下：

```
docker tag dockerhub.azk8s.cn/library/centos:latest centos:latest
```

2、快速拉取谷歌镜像

```
docker pull gcr.azk8s.cn/google_containers/<imagename>:<version>
#例子
docker pull gcr.azk8s.cn/google_containers/kubedns-amd64:1.7
```

