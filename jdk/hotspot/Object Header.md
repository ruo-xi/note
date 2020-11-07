# Object Header

## Define

```c
//"src/hotspot/share/oops/markOop.hpp"


// The markOop describes the header of an object.
//
// Note that the mark is not a real oop but just a word.
// It is placed in the oop hierarchy for historical reasons.
//
// Bit-format of an object header (most significant first, big endian layout below):
//
//  32 bits:
//  --------
//  hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//  size:32 ------------------------------------------>| (CMS free block)
//  PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
//
//  - hash contains the identity hash value: largest value is
//    31 bits, see os::random().  Also, 64-bit vm's require
//    a hash value no bigger than 32 bits because they will not
//    properly generate a mask larger than that: see library_call.cpp
//    and c1_CodePatterns_sparc.cpp.
//
//  - the biased lock pattern is used to bias a lock toward a given
//    thread. When this pattern is set in the low three bits, the lock
//    is either biased toward a given thread or "anonymously" biased,
//    indicating that it is possible for it to be biased. When the
//    lock is biased toward a given thread, locking and unlocking can
//    be performed by that thread without using atomic operations.
//    When a lock's bias is revoked, it reverts back to the normal
//    locking scheme described below.
//
//    Note that we are overloading the meaning of the "unlocked" state
//    of the header. Because we steal a bit from the age we can
//    guarantee that the bias pattern will never be seen for a truly
//    unlocked object.
//
//    Note also that the biased state contains the age bits normally
//    contained in the object header. Large increases in scavenge
//    times were seen when these bits were absent and an arbitrary age
//    assigned to all biased objects, because they tended to consume a
//    significant fraction of the eden semispaces and were not
//    promoted promptly, causing an increase in the amount of copying
//    performed. The runtime system aligns all JavaThread* pointers to
//    a very large value (currently 128 bytes (32bVM) or 256 bytes (64bVM))
//    to make room for the age bits & the epoch bits (used in support of
//    biased locking), and for the CMS "freeness" bit in the 64bVM (+COOPs).
//
//    [JavaThread* | epoch | age | 1 | 01]       lock is biased toward given thread
//    [0           | epoch | age | 1 | 01]       lock is anonymously biased
//
//  - the two lock bits are used to describe three states: locked/unlocked and monitor.
//
//    [ptr             | 00]  locked             ptr points to real header on stack
//    [header      | 0 | 01]  unlocked           regular object header
//    [ptr             | 10]  monitor            inflated lock (header is wapped out)
//    [ptr             | 11]  marked             used by markSweep to mark an object
//                                               not valid at any other time
//
//    We assume that stack/thread pointers have the lowest two bits cleared.

unused:25     hashcode:31   unused:1    age:4    baisedlock:1(0)     lock:2(01)            normal
threadinfo*:54   epoch:2   unused:1     age:4    baisedlock:1(1)     lock:2 (01)          baisedlcok
ptr_to_lock_record(ptr points to real header on stack)          00      轻量级锁
ptr_to_monitorObject(inflated lock (header is wapped out))        10      重量级锁
ptr                                              11      gc

```

* **object header **     (12 Byte)

  ​	**Common structure at the beginning of every GC-managed heap object**. (**Every oop points to an object header**.) Includes **fundamental information about the heap object's layout**, **type**, **GC state**, **synchronization state**, and **identity hash code**. **Consists of two words**. In arrays it is immediately followed by a length field. Note that both Java objects and VM-internal objects have a common object header format.

* **mark word**          (8 Byte)

  ​	The **first** word of every object header. Usually a set of bitfields including synchronization state and identity hash code. May also be a pointer (with characteristic low bit encoding) to synchronization related information. During GC, may contain GC state bits.

* **klass pointer**       (4 Byte)

  ​	The **second** word of every object header. Points to another object (a metaobject) which describes the layout and behavior of the original object. For Java objects, the "klass" contains a C++ style "vtable".

## 代码测试(工具 jol)

### 普通对象(无任何操作)

```java
package top.laonaailifa.jdk.jol;

import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.vm.VM;

public class JolExample1 {
    public static void main(String[] args) {
        System.out.println(VM.current().details());
        System.out.println("-----------------------------");
        System.out.println(ClassLayout.parseInstance(new JolExample1()).toPrintable());
    }
}

//output:
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# WARNING | Compressed references base/shifts are guessed by the experiment!
# WARNING | Therefore, computed addresses are just guesses, and ARE NOT RELIABLE.
# WARNING | Make sure to attach Serviceability Agent to get the reliable addresses.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
    //oop(Ordinary Object Pointer), boolean,byte,char,short,int,float,long,double
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
    
-----------------------------
    
top.laonaailifa.jdk.jol.JolExample1 object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           40 70 06 00 (01000000 01110000 00000110 00000000) (421952)
     12     4        (loss due to the next object alignment)	   
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```





### **对象的hashcode需要时才会加载**

```java
package top.laonaailifa.jdk.jol;

import org.openjdk.jol.info.ClassLayout;

public class JolExample2 {
    public static void main(String[] args) {
        A a = new A();
        System.out.println("before hash");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());

        System.out.println("jvm----0x"+Integer.toHexString(a.hashCode()));
        System.out.println();
        System.out.println("after hash");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());
    }
}

//output
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)   //todo ?偏向锁
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

//小端存储----低位存在低地址
jvm----0x2667f029

after hash
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 29 f0 67 (00000001 00101001 11110000 01100111) (1743792385)
      4     4        (object header)                           26 00 00 00 (00100110 00000000 00000000 00000000) (38)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

### 偏向锁

#### 无sleep,hashcode

```java
package top.laonaailifa.jdk.jol;

import org.openjdk.jol.info.ClassLayout;

public class JolExample3 {
    public static void main(String[] args) {
        A a = new A();

        System.out.println("before lock");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());
        synchronized (a){
            System.out.println("lock");
            System.out.println(ClassLayout.parseInstance(a).toPrintable());

        }
        System.out.println("after lcok");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());
    }
}


//output
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

lock
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 48 02 b4 (00000101 01001000 00000010 10110100) (-1274918907)
      4     4        (object header)                           05 7f 00 00 (00000101 01111111 00000000 00000000) (32517)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

after lcok
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 48 02 b4 (00000101 01001000 00000010 10110100) (-1274918907)
      4     4        (object header)                           05 7f 00 00 (00000101 01111111 00000000 00000000) (32517)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

#### 无sleep有hashcode

```java
package top.laonaailifa.jdk.jol;

import org.openjdk.jol.info.ClassLayout;

public class JolExample3 {
    public static void main(String[] args) throws InterruptedException {
        A a = new A();

        a.hashCode();

        System.out.println("before lock");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());
        synchronized (a){
            System.out.println("locking");
            System.out.println(ClassLayout.parseInstance(a).toPrintable());
        }
        System.out.println("after lcok");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());
    }
}

//output
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 64 05 14 (00000001 01100100 00000101 00010100) (335897601)
      4     4        (object header)                           48 00 00 00 (01001000 00000000 00000000 00000000) (72)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

locking
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           e8 88 90 63 (11101000 10001000 10010000 01100011) (1670416616)
      4     4        (object header)                           70 7f 00 00 (01110000 01111111 00000000 00000000) (32624)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

after lcok
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 64 05 14 (00000001 01100100 00000101 00010100) (335897601)
      4     4        (object header)                           48 00 00 00 (01001000 00000000 00000000 00000000) (72)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total


Process finished with exit code 0

```



#### 有sleep无hashcode

```java
package top.laonaailifa.jdk.jol;

import org.openjdk.jol.info.ClassLayout;

public class JolExample3 {
    public static void main(String[] args) throws InterruptedException {
        A a = new A();
        Thread.sleep(1000);

        System.out.println("before lock");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());
        synchronized (a){
            System.out.println("locking");
            System.out.println(ClassLayout.parseInstance(a).toPrintable());
        }
        System.out.println("after lcok");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());
    }
}

//output
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

locking
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 48 02 6c (00000101 01001000 00000010 01101100) (1812088837)
      4     4        (object header)                           3e 7f 00 00 (00111110 01111111 00000000 00000000) (32574)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

after lcok
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 48 02 6c (00000101 01001000 00000010 01101100) (1812088837)
      4     4        (object header)                           3e 7f 00 00 (00111110 01111111 00000000 00000000) (32574)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```



#### 有sleep有hashcode

```java
package top.laonaailifa.jdk.jol;

import org.openjdk.jol.info.ClassLayout;

public class JolExample3 {
    public static void main(String[] args) throws InterruptedException {
        A a = new A();
        Thread.sleep(1000);
        a.hashCode();

        System.out.println("before lock");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());
        synchronized (a){
            System.out.println("locking");
            System.out.println(ClassLayout.parseInstance(a).toPrintable());
        }
        System.out.println("after lcok");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());
    }
}


//output
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 64 05 14 (00000001 01100100 00000101 00010100) (335897601)
      4     4        (object header)                           48 00 00 00 (01001000 00000000 00000000 00000000) (72)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

locking
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           e8 f8 48 9b (11101000 11111000 01001000 10011011) (-1689716504)
      4     4        (object header)                           2a 7f 00 00 (00101010 01111111 00000000 00000000) (32554)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

after lcok
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 64 05 14 (00000001 01100100 00000101 00010100) (335897601)
      4     4        (object header)                           48 00 00 00 (01001000 00000000 00000000 00000000) (72)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total


Process finished with exit code 0

```

#### 产生差异的原因分析

jvm目前默认不执行hashcode方法则默认为偏向锁,若执行则为普通对象.

对象若为普通对象,进入同步方法后,直接进入轻量级锁的状态.

对象若是偏向锁,进入同步方法后,若无竞争则一直是偏向锁.

### 轻量级锁

#### 单线程不可偏向

```java
package top.laonaailifa.jdk.jol;

import org.openjdk.jol.info.ClassLayout;

public class JolExample3 {
    public static void main(String[] args) throws InterruptedException {
        A a = new A();
        a.hashCode();

        System.out.println("before lock");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());
        synchronized (a){
            System.out.println("locking");
            System.out.println(ClassLayout.parseInstance(a).toPrintable());
        }
        System.out.println("after lcok");
        System.out.println(ClassLayout.parseInstance(a).toPrintable());
    }
}

//output:
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 64 05 14 (00000001 01100100 00000101 00010100) (335897601)
      4     4        (object header)                           48 00 00 00 (01001000 00000000 00000000 00000000) (72)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4    int A.i                                       0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

locking
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           e8 d8 ba 1f (11101000 11011000 10111010 00011111) (532338920)
      4     4        (object header)                           e9 7f 00 00 (11101001 01111111 00000000 00000000) (32745)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4    int A.i                                       0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

after lcok
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 64 05 14 (00000001 01100100 00000101 00010100) (335897601)
      4     4        (object header)                           48 00 00 00 (01001000 00000000 00000000 00000000) (72)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4    int A.i                                       0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```



#### 多线程无竞争的情况

##### 上一线程死亡

```java
package top.laonaailifa.jdk.jol;

import org.openjdk.jol.info.ClassLayout;

public class JolExample6 {
    public static void main(String[] args) throws InterruptedException {
        A a = new A();
        Thread t1 = new Thread(() -> {
            synchronized (a) {
                System.out.println(ClassLayout.parseInstance(a).toPrintable());
            }
        });
        t1.start();
        t1.join();//等待线程死亡
        Thread t2 = new Thread(() -> {
            synchronized (a) {
                System.out.println(ClassLayout.parseInstance(a).toPrintable());
            }
        });
        t2.start();
    }
}


//output:
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 d0 44 f0 (00000101 11010000 01000100 11110000) (-263925755)
      4     4        (object header)                           af 7f 00 00 (10101111 01111111 00000000 00000000) (32687)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4    int A.i                                       0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 d0 44 f0 (00000101 11010000 01000100 11110000) (-263925755)
      4     4        (object header)                           af 7f 00 00 (10101111 01111111 00000000 00000000) (32687)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4    int A.i                                       0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total


```
##### 上一线程未死亡
```java
package top.laonaailifa.jdk.jol;

import org.openjdk.jol.info.ClassLayout;

public class JolExample6 {
    public static void main(String[] args) throws InterruptedException {
        A a = new A();
        Thread t1 = new Thread(() -> {
            synchronized (a) {
                System.out.println(ClassLayout.parseInstance(a).toPrintable());
            }
            while (true){

            }
        });
        t1.start();
        Thread.sleep(2000);
        Thread t2 = new Thread(() -> {
            synchronized (a) {
                System.out.println(ClassLayout.parseInstance(a).toPrintable());
            }
        });
        t2.start();
    }
}

//output:
top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 48 42 64 (00000101 01001000 01000010 01100100) (1682065413)
      4     4        (object header)                           b3 7f 00 00 (10110011 01111111 00000000 00000000) (32691)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4    int A.i                                       0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

top.laonaailifa.jdk.jol.A object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           78 a7 80 2e (01111000 10100111 10000000 00101110) (780183416)
      4     4        (object header)                           b3 7f 00 00 (10110011 01111111 00000000 00000000) (32691)
      8     4        (object header)                           48 72 06 00 (01001000 01110010 00000110 00000000) (422472)
     12     4    int A.i                                       0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total


```



### 三种锁的速度对比

```java
package top.laonaailifa.jdk.jol;

import org.openjdk.jol.info.ClassLayout;

import java.util.concurrent.CountDownLatch;

/**
 * 偏向锁 14
 * 轻量级锁 145ms
 * 重量级锁 845
 */
public class JolExample4 {

    static CountDownLatch countDownLatch = new CountDownLatch(10000000);

    public static void main(String[] args) throws InterruptedException {
        A a = new A();
        /**
         * 执行该方法为轻量级锁
         * 不执行则是偏向锁
         */
//        a.hashCode();
        long start = System.currentTimeMillis();


//        for (int i = 0; i < 10000000L; i++) {
//            a.parse();
//        }

        /**
         * 重量级锁方法
         */
//        for (int i = 0; i < 10; i++) {
//            new Thread(() -> {
//                while (countDownLatch.getCount() > 0){
//                    a.parse();
//                    countDownLatch.countDown();
//                }
//            }).start();
//        }
//        countDownLatch.await();

        long end = System.currentTimeMillis();
        System.out.println(String.format("%sms", end - start));
    }
}

```

cp