---
layout: post
title: "nvidia smi docker usage"
date: 2021-9-22
categories: 
    - "docker"
tags: [docker]
author: "pan"
---

<!-- ---
layout: post
title: "matplotlib画图"
date: 2021-12-19
categories: 
    - "python"
tags: [matplotlib]
author: "pan"
--- -->

# 正文

1. docker 和 nvidia-docker命令的区别

   如果容器中需要用到cuda，但是使用docker 启动是找不到cuda的，nvidia-smi命令也无法使用。必须使用nvidia-docker启动。经试验，如下命令也是ok的，docker指定参数 --gpus

   ```sh
   docker run --rm --gpus all nvidia/cuda nvidia-smi
   ```

2. 指定使用那一块GPU

    + 使用全部的gpu

    ```sh
    docker run --rm --gpus all nvidia/cuda nvidia-smi
    ```

    + 使用环境变量NVIDIA_VISIBLE_DEVICES来指定使用那一个GPU（必须指定runtime，--runtime=nvidia）
  
    ```sh
    docker run --rm --runtime=nvidia -e NVIDIA_VISIBLE_DEVICES=all nvidia/cuda nvidia-smi

    ```

    + 容器内可以使用两块gpu
  
    ```sh
    docker run --rm --gpus 2 nvidia/cuda nvidia-smi
    ```

    + 指定gpu编号
  
    ```sh
    docker run --gpus '"device=1,2"' nvidia/cuda nvidia-smi --query-gpu=uuid --format-csv

    ```


# [参考文档]
1. https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/user-guide.html
2. [nvidia-docker2 安装](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker)