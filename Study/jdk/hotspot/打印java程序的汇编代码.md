# 打印Java程序的汇编代码

### 环境

> jdk version : openjdk 11.0.10-internal 2021-01-19

### 查看相关文档

#### 命令

```
man java
```

**-XX:+PrintAssembly**
           Enables printing of assembly code for bytecoded and native methods by using the external **disassembler.so** library. This enables you to see the generated code, which may help you to diagnose performance issues.
           By default, this option is disabled and assembly code is not printed. The **-XX:+PrintAssembly** option has to be used together with the -**XX:UnlockDiagnosticVMOptions** option that unlocks diagnostic JVM options.



### 操作

直接添加以下参数

```
-XX:+UnlockDiagnosticVMOptions -XX:+PrintAssembly
```

当前未遇到没有disassembler.so的情况

未遇到报错