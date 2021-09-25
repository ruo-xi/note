# Jdk源码阅读准备

#### 环境准备

> 操作系统:      Linux version 5.6.13-arch1-1 (linux@archlinux) (gcc version 10.1.0 (GCC)) #1 SMP PREEMPT
>
> bootjdk:        java version "11.0.8" 2020-07-14 LTS
>
> jdk源码:         openjdk-jdk11
>
> 阅读工具:       vscode

#### 源码下载

* openjdk [https://hg.openjdk.java.net/](https://hg.openjdk.java.net/)
* github  [https://github.com/openjdk/jdk11u/tags](https://github.com/openjdk/jdk11u/tags)

#### 阅读工具

* bear
  生成**compile_commands.json**文件

* vscode
  * Plugin
    * C/C++
    * Java Extension Pack

  * 操作方法

    Ctrl + p 打开命令面板

    * ​        跳转到相应的文件
    * **@**    可以查找文件内的变量和函数
    * ***>***      可以执行命令
    * **:**       跳转到相应的行

* jps  

  * -l 查看java线程

* jhsdb 

  * sudo jhsdb hsdb 打开ui界面

  需要安装[Courier Prime](https://quoteunquoteapps.com/courierprime/)  字体

  ```
  yay -S ttf-courier-prime
  ```

