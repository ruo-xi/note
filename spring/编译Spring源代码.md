# 编译Spring源代码

### 环境

> 操作系统:  Linux yu 5.11.11-arch1-1 #1 SMP PREEMPT Tue, 30 Mar 2021 14:10:17 +0000 x86_64 GNU/Linux
>
> Spring版本: spring-framework 5.3.5
>
>  源码阅读工具:  Idea 2021.1
>
> jdk版本:     openjdk 11.0.10 2021-01-19         和   我自己编译的openjdk 11.0.10-internal 2021-01-19

### 编译过程

* 进入工程目录执行

  `./gradlew :spring-oxm:compileTestJava`

* 将工程导入到Idea中

* 由Gradle 切换为 Idea  build 工程

### 报错

* jdk有问题

  ```
  java: package jdk.jfr does not exist
  ```

  切换jdk为自己编译的jdk后解决   应该archlinux安装的openjdk的问题


* AspectJ 有问题

  ```
  java: cannot find symbol
    symbol:   class AnnotationBeanConfigurerAspect
    location: package org.springframework.beans.factory.aspectj
  ```

  删除spring-aspectJ可以解决问题

  或者自行下载AspectJ包添加到相关位置

