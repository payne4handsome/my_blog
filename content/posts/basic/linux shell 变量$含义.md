---
layout: post
title: "linux shell 变量$含义"
date: 2021-9-16
categories: 
    - "linux"
tags: [shell]
author: "pan"
---


1. $开头的变量

    测试代码，sh文件名：params.sh

    ```sh
    #!/bin/bash
    # $$ Shell本身的PID（ProcessID） 
    printf "The complete list is %s\n" "$$"
    # $! Shell最后运行的后台Process的PID 
    printf "The complete list is %s\n" "$!"
    # $? 最后运行的命令的结束代码（返回值） 
    printf "The complete list is %s\n" "$?"
    # $* 所有参数列表
    printf "The complete list is %s\n" "$*"
    # $@ 所有参数列表
    printf "The complete list is %s\n" "$@"
    # $# 添加到Shell的参数个数 
    printf "The complete list is %s\n" "$#"
    # $0 Shell本身的文件名 
    printf "The complete list is %s\n" "$0"
    # $1 第一个参数
    printf "The complete list is %s\n" "$1"
    # $2 第二个参数
    printf "The complete list is %s\n" "$2
    ```

    执行命令

    ```sh
    bash params.sh 123456 QQ
    ```

    以上shell的输出

    ```sh
    The complete list is 24249
    The complete list is
    The complete list is 0
    The complete list is 123456 QQ
    The complete list is 123456
    The complete list is QQ
    The complete list is 2
    The complete list is params.sh
    The complete list is 123456
    The complete list is QQ
    ```
