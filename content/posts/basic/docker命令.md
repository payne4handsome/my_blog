---
layout: post
title: "docker命令"
date: 2021-9-16
categories: 
    - "docker"
tags: [docker]
author: "pan"
---


## 基础命令

+ 查看有那些镜像

```sh
xxxxx:5000/v2/_catalog
```

+ 查看具体项目的tag列表

```sh
xxxx:5000/v2/project/repo/tags/list
```

+ 启动一个镜像

```sh
docker run -it --rm -v $PWD:/tmp -w /tmp self_image_name self_command
```

其中

+ -v: 将宿主的目录挂载到容器内部
+ -w: 指定工作目录
+ 启动一个镜像(web应用，需要端口映射)

```sh
docker run -it --rm -v $PWD:/tmp -w /tmp -p 5000:5000 self_image_name self_command
```

+ 查看容器内部的标准输出

```sh
docker logs -f bf08b7f2cd89
```

+ 查看容器内部运行的进程

```sh
docker top wizardly_chandrasekhar
```

+ 查看容器的配置和状态信息

```sh
docker inspect wizardly_chandrasekhar
```

+ 更新镜像

```sh
docker commit -m="has update" -a="runoob" e218edb10161 runoob/ubuntu:v2
```

+ 打tag（push到仓库）

```sh
docker tag 860c279d2fec runoob/centos:dev

docker login xxxx.com

docker push a/b/c/image_name:v1.0.0
```


## GPU 环境安装

### NVIDIA Docker 安装

 如需在 Linux 上启用 GPU 支持，请[安装 NVIDIA Docker 支持](https://github.com/NVIDIA/nvidia-docker)
验证 nvidia-docker 安装效果

```sh
docker run --gpus all --rm nvidia/cuda nvidia-smi

```

### tensorflow环境安装

1. 拉取镜像（版本自已指定）

    ```sh
    docker pull tensorflow/tensorflow:1.14.0-gpu
    ```

2. 启动

    ```sh
    docker run --gpus all -it tensorflow/tensorflow:1.14.0-gpu bash
    ```

或者，指定特定的那一块GPU

    ```sh
    docker run --gpus '"device=2"' -it tensorflow/tensorflow:1.14.0-gpubash
    ```

### pytorch环境安装

1. 拉取镜像（版本自已指定）

    ```sh
    docker pull pytorch/pytorch:1.6.0-cuda10.1-cudnn7-runtime
    ```

2. 启动

    ```sh
    docker run -it --rm --init --gpus all  pytorch/pytorch:1.6.0-cuda10.1-cudnn7-runtime bash
    ```

或者，指定特定的那一块GPU

```sh
 docker run -it --rm --init --gpus '"device=2"'  pytorch/pytorch:1.6.0-cuda10.1-cudnn7-runtime bash
```
