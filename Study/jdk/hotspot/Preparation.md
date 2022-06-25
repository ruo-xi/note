### Enviroment

> 操作系统:      Linux yu 5.14.10-arch1-1 #1 SMP PREEMPT Thu, 07 Oct 2021 20:00:23 +0000 x86_64 GNU/Linux
>
> bootjdk:        openjdk 17 2021-09-14
>
> jdk源码:         openjdk 17-internal 2021-09-14
>
> 阅读工具:       vscode

### Source Download

* openjdk [https://hg.openjdk.java.net/](https://hg.openjdk.java.net/)

### Compile

#### compile tools

* basic tools

* bear    

  用于生成**compile_commands.json**文件

#### compile

```bash
cd ${source}

# --with-boot-jdk 指定bootjdk目录

# jdk 11
bash configure --disable-warnings-as-errors --with-boot-jdk=/usr/lib/jvm/java-11-openjdk --with-toolchain-type=clang

bear -- make JOBS=16

# jdk 17
bash configure --disable-warnings-as-errors --with-boot-jdk=/usr/lib/jvm/java-17-openjdk --with-toolchain-type=clang 

bear -- make JOBS=16
```

### Tools

* vscode
  * Plugin
    * clangd
    * Java Extension Pack

* jps  

  * -l 查看java线程

* jhsdb 

  * sudo jhsdb hsdb 打开ui界面

  需要安装[Courier Prime](https://quoteunquoteapps.com/courierprime/)  字体

  ```
  yay -S ttf-courier-prime
  ```

