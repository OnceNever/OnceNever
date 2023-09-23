---
title: JDK工具使用指南
date: 2023/9/23 14:27:26
categories:
- [JDK, 工具]
tags:
- 调优
---

### JDK工具使用指南

#### jps

**该命令用于打印当前JVM中运行的Java进程状态信息**

```bash
jps [options] [hostid]
```

`options`表示命令可选参数：

- -q：仅显示pid
- -m：显示进程pid及main方法参数
- -l：打印进程对应JAR文件所在的完整路径名
- -v：查看进程pid及JVM参数
- -h：打印帮助信息



#### jinfo

**用于打印全部参数和系统属性**

```bash
jinfo [options] <pid>
```

options表示命令可选参数：

- -[pid]：输出全部参数
- -flag [name] pid：打印对应名称的参数内容
- -flag [+|-] [name] pid：不重启虚拟机的情况下，动态修改JVM参数
- -flag [name]=[value]：不重启虚拟机的情况下，动态修改JVM参数



#### jstat

**对Java应用程序的性能和资源进行实时监控，包括Heap和垃圾回收状态的监控**

```bash
jstat -[options] [-t] [-h<lines>] <vmid> [<interval> [<count>]]
```

其中，vmid为虚拟机id，Liunx/Unix一般为pid，interval为采样间隔（ms），count为采样数

`jstat -gc 54231 1000 3` 表示每隔1000ms打印一次pid为54231的进程GC信息，总共打印三次

options表示命令可选参数：

- -class：显示加载class的数量及所占空间等信息

- -compiler：JIT即时编译器相关的统计信息

- -gc：打印GC信息（展开查看参数说明）

- -gccapacity：打印各个内存池分代空间的容量

  | 参数  | 说明                                           |
  | ----- | ---------------------------------------------- |
  | NGCMN | 年轻代(young)中初始化(最小)的大小(字节)        |
  | NGCMX | 年轻代(young)的最大容量(字节)                  |
  | NGC   | 年轻代(young)中当前的容量(字节)                |
  | S0C   | 年轻代中第一个survivor（幸存区）的容量(字节)   |
  | S1C   | 年轻代中第二个survivor（幸存区）的容量(字节)   |
  | EC    | 年轻代中Eden（伊甸园）的容量(字节)             |
  | OGCMN | old代中初始化(最小)的大小(字节)                |
  | OGCMX | old代的最大容量(字节)                          |
  | OGC   | old代当前新生成的容量(字节)                    |
  | OC    | Old代的容量(字节)                              |
  | MCMN  | Metaspace 区初始化 ( 最小 ) 容量 (字节)        |
  | MCMX  | Metaspace 区的最大容量(字节)                   |
  | MC    | 当前 Metaspace 的容量(字节)                    |
  | CCSMN | 最小压缩类空间大小(字节)                       |
  | CCMSX | 最大压缩类空间大小(字节)                       |
  | CCSC  | 当前压缩类空间大小(字节)                       |
  | YGC   | 从应用程序启动到采样时年轻代中GC(minor GC)次数 |
  | FGC   | 从应用程序启动到采样时old代GC(full GC)次数     |

- -gcutil：GC相关区域的使用率统计

  | 参数 | 说明                                                     |
  | ---- | -------------------------------------------------------- |
  | S0   | 年轻代中第一个survivor（幸存区）已使用的占当前容量百分比 |
  | S1   | 年轻代中第二个survivor（幸存区）已使用的占当前容量百分比 |
  | E    | 年轻代中Eden（伊甸园）已使用的占当前容量百分比           |
  | O    | old代已使用的占当前容量百分比                            |
  | M    | Metaspace代已使用的占当前容量百分比                      |
  | CCS  | 压缩区已使用的占当前容量百分比                           |
  | YGC  | 从应用程序启动到采样时年轻代中gc次数                     |
  | YGCT | 从应用程序启动到采样时年轻代中gc所用时间(s)              |
  | GCT  | 从应用程序启动到采样时gc用的总时间(s)                    |
  | FGC  | 从应用程序启动到采样时old代(全gc)gc次数                  |
  | FGCT | 从应用程序启动到采样时old代(全gc)gc所用时间(s)           |

- -gccause：查看上次GC和本次GC（如果正在GC）产生的原因，参数同-gcutil

- -gcnew：年轻代的统计信息

  | 参数 | 说明                                                         |
  | ---- | ------------------------------------------------------------ |
  | S0C  | 年轻代中第一个survivor（幸存区）的容量(字节)                 |
  | S1C  | 年轻代中第二个survivor（幸存区）的容量(字节)                 |
  | S0U  | 年轻代中第一个survivor（幸存区）目前已使用空间(字节)         |
  | S1U  | 年轻代中第二个survivor（幸存区）目前已使用空间(字节)         |
  | TT   | 持有次数限制(默认15)                                         |
  | MTT  | Maximum tenuring threshold，可以通过-XX:MaxTenuringThreshold=N设置，默认为15 |
  | DSS  | Desired survivor size，单位为KB，这个值默认为Survivor区的50% |
  | EC   | 年轻代中Eden（伊甸园）的容量(字节)                           |
  | EU   | 年轻代中Eden（伊甸园）目前已使用空间(字节)                   |
  | YGC  | 从应用程序启动到采样时年轻代中gc次数                         |
  | YGCT | 从应用程序启动到采样时年轻代中gc所用时间(s)                  |

- -gcoldcapacity：老年代空间大小统计

  | 参数  | 说明                   |
  | ----- | ---------------------- |
  | OGCMN | 老年代最小容量         |
  | OGCMX | 老年代最大容量         |
  | OGC   | 当前老年代大小         |
  | OC    | 老年代大小             |
  | YGC   | 年轻代垃圾回收次数     |
  | FGC   | 老年代垃圾回收次数     |
  | FGCT  | 老年代垃圾回收消耗时间 |
  | GCT   | 垃圾回收消耗总时间     |

- -gcmetacapacity：meta区大小统计

  | 参数  | 说明                   |
  | ----- | ---------------------- |
  | MCMN  | 最小元数据容量         |
  | MCMX  | 最大元数据容量         |
  | MC    | 当前元空间大小         |
  | CCSMN | 最小压缩类空间大小     |
  | CCSMX | 最大压缩类空间大小     |
  | CCSC  | 当前压缩类空间大小     |
  | YGC   | 年轻代垃圾回收次数     |
  | FGC   | 老年代垃圾回收次数     |
  | FGCT  | 老年代垃圾回收消耗时间 |
  | GCT   | 垃圾回收消耗总时间     |

- -printcompilation：打印JVM编译统计信息



#### jmap

**用于查看堆内存的情况，或者将堆内存输出为二进制文本**

```bash
jmap [option] <pid>
```

options表示命令可选参数：

- -heap：打印堆内存配置和使用情况

- -histo：打印堆内存中的对象大小、数量

  | 参数       | 说明     |
  | ---------- | -------- |
  | num        | 序号     |
  | instances  | 实例个数 |
  | bytes      | 字节数   |
  | class_name | 类名     |

- `-dump:format=b,file=[filename].hprof` ：dump堆内存到hprof文件



#### jhat

**用于访问jmap命令生产的dump文件，通过浏览器访问**

```bash
jhat [options] <dump-file>
```

options表示命令可选参数：

- -port：指定jhat的http服务端口，默认7000



#### jstack

**用于查看指定Java进程pid的堆栈信息**

```bash 
jstack [options]
```

options表示命令可选参数：

- -F：强制线程转储。当jstack 没有响应(进程挂起)时使用
- -m：打印Java和底层C/C++框架所有栈信息
- -l(小写L)：长列表模式，打印锁的附加信息

