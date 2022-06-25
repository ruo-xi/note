### hotspot目录结构

* cpu 
* os
* os_cpu
* share
  * adlc      (Architecture Description Language Compiler)      平台描述文件
  * aot        汇编器接口
  * c1           client编译器("又称c1")
  * ci            动态编译器的公共服务/从动态编译器到VM的接口
  * classfile  类文件的处理(类加载和系统符号表等等)   
  * code        动态生成的代码的管理
  * compiler   从VM调用动态编译器的接口
  * gc            GC的实现
  * include       比较重要的头文件,就三个jvm.h/jmm.h/cds.h
  * interpreter   解释器
  * jfr          //todo
  * jvmci     JVM Compiler Interface 
  * libadt    
  * logging
  * memory   内存管理
  * metaprogramming  
  * oops   HotSpot VM的对象系统的实现
  * opto  C2编译器(又称optp)
  * precompiled 预编译
  * prims    HotSpot VM对外接口,包括部分native和JVMTI的实现
  * runtime   运行时支持库
  * service    JMX
  * utilities  工具类