## registry深度调研



使用registry可以搭建本地镜像注册中心（为啥又叫注册中心了）

registry既可以作为私有仓库也可以作为镜像加速器，那么他到底是怎么在这两种模式下进行切换的呢？我的理解就是本地不改mirror那就在本地直接拉，改了Mirror那就Mirror地址拉

需要搭建一套完整的仓库系统，然后去

registry镜像适用于建立私有镜像仓库的

 Registry在github上有两份代码：老代码库和新代码库。老代码是采用python编写的，存在pull和push的性能问题，出到0.9.1版本之后就标志为deprecated，不再继续开发。从2.0版本开始就到在新代码库进行开发，新代码库是采用go语言编写，修改了镜像id的生成算法、registry上镜像的保存结构，大大优化了pull和push镜像的效率

文档参考地址https://docs.docker.com/registry/

部署文档参看这里：https://docs.docker.com/registry/deploying/

## What it is

The Registry is a stateless, highly scalable server side application that stores and lets you distribute Docker images. The Registry is open-source, under the permissive [Apache license](http://en.wikipedia.org/wiki/Apache_License).

## Why use it

You should use the Registry if you want to:

- tightly control where your images are being stored （控制镜像存储的位置）
- fully own your images distribution pipeline（）
- integrate image storage and distribution tightly into your in-house development workflow

A registry is a storage and content delivery system, holding named Docker images, available in different tagged versions

在配置项中有一项是Pushing to a registry configured as a pull-through cache is unsupported

registry只能作为Pull的缓存，不能用于推

![1578540053122](C:\Users\fishing\AppData\Roaming\Typora\typora-user-images\1578540053122.png)

为了能够拉取私有仓库，必须写明用户名和密码。注意点：这些私有仓库会被存储在代理缓存的存储中，采取适当的措施来保护对代理缓存的访问。（这边适当的措施来保护代理缓存的访问是什么意思，到底在缓存节点上能不能通过docker images命令看到呢？）

但我不知道如果remoteurl填的阿里云加速器的链接还能不能拉取到。这个需要试一下。

# Registry as a pull through cache（这边讲的就是镜像加速器啦）

## Use-case（使用案例/场景）

If you have multiple instances of Docker running in your environment, such as multiple physical or virtual machines all running Docker, each daemon goes out to the internet and fetches an image it doesn’t have locally, from the Docker repository. You can run a local registry mirror and point all your daemons there, to avoid this extra internet traffic.

### Gotcha

It’s currently not possible to mirror another private registry. Only the central Hub can be mirrored.

The Registry can be configured as a pull through cache. In this mode a Registry responds to all normal docker pull requests but stores all content locally.

## How does it work?

The first time you request an image from your local registry mirror, it pulls the image from the public Docker registry and stores it locally before handing it back to you. On subsequent requests, the local registry mirror is able to serve the image from its own storage（第一次拉取，会向公共的dockerhub仓库拉取，然后存储在本地，之后的请求就会用本地的已有镜像存储直接服务）

## Run a Registry as a pull-through cache

The easiest way to run a registry as a pull through cache is to run the official Registry image. At least, you need to specify `proxy.remoteurl` within `/etc/docker/registry/config.yml` as described in the following subsection.（需要修改proxy.remoteurl）

Multiple registry caches can be deployed over the same back-end. A single registry cache ensures that concurrent requests do not pull duplicate data, but this property does not hold true for a registry cache cluster.(多个registry缓存可以部署在一个后端，单个regsitry可以保证多个并发请求不会拉重复的数据，但此属性在regsitry集群中不成立)

### Configure the cache

To configure a Registry to run as a pull through cache, the addition of a `proxy` section is required to the config file.

To access private images on the Docker Hub, a username and password can be supplied.（用户名和密码是用于拉取私有镜像的）

```
proxy:
  remoteurl: https://registry-1.docker.io
  username: [username]
  password: [password]
```

> **Warning**: If you specify a username and password, it’s very important to understand that private resources that this user has access to Docker Hub is made available on your mirror. **You must secure your mirror** by implementing authentication if you expect these resources to stay private!（什么叫实现一些认证来保护你的Mirror）

> **Warning**: For the scheduler to clean up old entries, `delete` must be enabled in the registry configuration. See [Registry Configuration](https://docs.docker.com/registry/configuration/) for more details.

### Configure the Docker daemon

Either pass the `--registry-mirror` option when starting `dockerd` manually, or edit [`/etc/docker/daemon.json`](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file) and add the `registry-mirrors` key and value, to make the change persistent.

```
{
  "registry-mirrors": ["https://<my-docker-mirror-host>"]
}
```

Save the file and reload Docker for the change to take effect.



# Garbage collection

*Estimated reading time: 3 minutes*

As of v2.4.0 a garbage collector command is included within the registry binary. This document describes what this command does and how and why it should be used.

## About garbage collection

In the context of the Docker registry, garbage collection is the process of removing blobs from the filesystem when they are no longer referenced by a manifest. Blobs can include both layers and manifests.

Registry data can occupy considerable amounts of disk space. In addition, garbage collection can be a security consideration, when it is desirable to ensure that certain layers no longer exist on the filesystem.

## Garbage collection in practice

Filesystem layers are stored by their content address in the Registry. This has many advantages, one of which is that data is <u>stored once</u> and referred to by manifests. See [here](https://docs.docker.com/registry/compatibility/#content-addressable-storage-cas) for more details.

Layers are therefore shared amongst manifests; each manifest maintains a reference to the layer. <u>As long as a layer is referenced by one manifest, it cannot be garbage collected</u>.

Manifests and layers can be `deleted` with the registry API (refer to the API documentation [here](https://docs.docker.com/registry/spec/api/#deleting-a-layer) and [here](https://docs.docker.com/registry/spec/api/#deleting-an-image) for details). This API removes references to the target and makes them eligible for garbage collection. It also makes them unable to be read via the API.

If a layer is deleted, it is removed from the filesystem when garbage collection is run. If a manifest is deleted the layers to which it refers are removed from the filesystem if no other manifests refers to them.

### Example

In this example manifest A references two layers: `a` and `b`. Manifest `B` references layers `a` and `c`. In this state, nothing is eligible for garbage collection:

```
A -----> a <----- B
    \--> b     |
         c <--/
```

Manifest B is deleted via the API:

```
A -----> a     B
    \--> b
         c
```

In this state layer `c` no longer has a reference and is eligible for garbage collection. Layer `a` had one reference removed but not garbage collected as it is still referenced by manifest `A`. The blob representing manifest `B` is eligible for garbage collection.

After garbage collection has been run, manifest `A` and its blobs remain.

```
A -----> a
    \--> b
```

### More details about garbage collection

Garbage collection runs in two phases. First, in the ‘mark’ phase, the process scans all the manifests in the registry. From these manifests, it constructs a set of content address digests. This set is the ‘mark set’ and denotes the set of blobs to *not* delete. Secondly, in the ‘sweep’ phase, the process scans all the blobs and if a blob’s content address digest is not in the mark set, the process deletes it.(垃圾回收两个阶段，第一步标记，第二部清除)

> **Note**: You should ensure that the registry is in read-only mode or not running at all. If you were to upload an image while garbage collection is running, there is the risk that the image’s layers are mistakenly deleted leading to a corrupted image.（需要是read-only模式做垃圾回收，不会可能存在误删除的风险。）

This type of garbage collection is known as stop-the-world garbage collection.

## Run garbage collection

Garbage collection can be run as follows

```
bin/registry garbage-collect [--dry-run] /path/to/config.yml
```

The garbage-collect command accepts a `--dry-run` parameter, which prints the progress of the mark and sweep phases without removing any data. Running with a log level of `info` gives a clear indication of items eligible for deletion.

The config.yml file should be in the following format:

```
version: 0.1
storage:
  filesystem:
    rootdirectory: /registry/data
```

*Sample output from a dry run garbage collection with registry log level set to info*

```
hello-world
hello-world: marking manifest sha256:fea8895f450959fa676bcc1df0611ea93823a735a01205fd8622846041d0c7cf
hello-world: marking blob sha256:03f4658f8b782e12230c1783426bd3bacce651ce582a4ffb6fbbfa2079428ecb
hello-world: marking blob sha256:a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
hello-world: marking configuration sha256:690ed74de00f99a7d00a98a5ad855ac4febd66412be132438f9b8dbd300a937d
ubuntu

4 blobs marked, 5 blobs eligible for deletion
blob eligible for deletion: sha256:28e09fddaacbfc8a13f82871d9d66141a6ed9ca526cb9ed295ef545ab4559b81
blob eligible for deletion: sha256:7e15ce58ccb2181a8fced7709e9893206f0937cc9543bc0c8178ea1cf4d7e7b5
blob eligible for deletion: sha256:87192bdbe00f8f2a62527f36bb4c7c7f4eaf9307e4b87e8334fb6abec1765bcb
blob eligible for deletion: sha256:b549a9959a664038fc35c155a95742cf12297672ca0ae35735ec027d55bf4e97
blob eligible for deletion: sha256:f251d679a7c61455f06d793e43c06786d7766c88b8c24edf242b2c08e3c3f599
```





加了username后：

![1578552637030](C:\Users\fishing\AppData\Roaming\Typora\typora-user-images\1578552637030.png)

查不出有多少镜像



![1578564002916](C:\Users\fishing\AppData\Roaming\Typora\typora-user-images\1578564002916.png)

1、我们在配置加速器是有代理Url（图中remoteurl），用的是阿里的加速器地址和azure的，不配置用户名和密码。能直接拉取公有镜像，私有镜像不行

2、在1的基础之上，在加速器配置文件中加上Username,pwd，remoteurl用的阿里或者亚马逊的加速器地址。是不能拉取私有镜像的。必须改掉url为dockerhub的url才能拉（即https://registry-1.docker.io）。

3、使用dockerhub的url,配上username,pwd。（用户A的密码）。用户A能从加速器拉dockerhub私有镜像。用户B不能从加速器拉自己的私有镜像。同时用户A的私有镜像会被缓存在加速器本地：

![1578564186613](C:\Users\fishing\AppData\Roaming\Typora\typora-user-images\1578564186613.png)

圈出来就是fishingfly用户的私有镜像，被存在镜像加速器里，下次Pull可以很快被拉下来