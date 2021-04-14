* linux

  * arch  

    这个为目录是硬件架构相关每个CPU的子目录，又进一步分解为boot，mm，kernel等子目录，分别控制系统引导，内存管理，系统调用，动态调频，主频率设置部分等

  * block

    linux存储体系中关于块设备管理的代码。

  * certs

    认证相关

  * crypto

    各种的加密算法.

  * documentation

    文档

  * drivers

    驱动目录

  * fs

    文件系统

  * include

    头文件目录,公共（各种CPU架构共用的）的头文件都在这里每种CPU架构特有的一些头文件在arch/arm/include目录及其子目录下

  * init

    linux内核启动时,初始化内核的代码

  * ipc(inter process commuication)

    进程间通信的代码实现

  * kernel

    内核本身需要的文件

  * lib

    lib是库的意思，这里面都是一些公用的有用的库函数，注意这里的库函数和C语言的库函数不一样的。在内核编程中是不能用C语言标准库函数，这里的lib目录下的库函数就是用来替代那些标准库函数的。譬如在内核中要把字符串转成数字用atoi，但是内核编程中只能用lib目录下的atoi函数，不能用标准C语言库中的atoi。譬如在内核中要打印信息时不能用printf，而要用printk，这个printk就是我们这个lib目录下的。

  * LICENSES

  * mm

    内存管理

  * net

    网络相关

  * samples

    内核编程的范例

  * scripts

    辅助对linux内核进行配置编译生产的脚本文件

  * security

    安全相关

  * sound

    音频处理

  * tools

    linux中用到的有用的工具

  * usr

    目录下是initramfs相关的，和linux内核的启动有关

  * virt

    内核虚拟机相关