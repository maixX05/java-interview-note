# 1. JVM优化-day01

**主要内容：**
* 了解下我们为什么要学习 JVM优化
* 掌握 jvm的运行参数以及参数的设置
* 掌握 jvm的内存模型（堆内存）
* 掌握 jamp命令的使用以及通过MAT工具进行分析
* 掌握定位分析内存溢出的方法
* 掌握 jstack命令的使用
* 掌握 VisualJVM工具的使用

## 1.1. 我们为什么要对jvm做优化？
在本地开发环境中我们很少会遇到需要对jvm进行优化的需求，但是到了生产环境，我们
可能将有下面的需求：
* 运行的应用 “卡住了”，日志不输出，程序没有反应
* 服务器的 CPU负载突然升高
* 在多线程应用下，如何分配线程的数量？
* ……
说明：使用的jdk版本为1.8。

## 1.2. jvm的运行参数

在jvm中有很多的参数可以进行设置，这样可以让jvm在各种环境中都能够高效的运行。
绝大部分的参数保持默认即可。

### 1.2.1. 三种参数类型
jvm的参数类型分为三类，分别是：
* 标准参数
-help
-version
* -X 参数 （非标准参数）
-Xint
-Xcomp
* -XX 参数（使用率较高）
-XX:newSize
-XX:+UseSerialGC

### 1.2.2. 标准参数
jvm的标准参数，一般都是很稳定的，在未来的JVM版本中不会改变，可以使用java -help
检索出所有的标准参数。
```
  [root@node01 ~]# java ‐help
用法: java [‐options] class [args...]
           (执行类)
   或  java [‐options] ‐jar jarfile [args...]
           (执行 jar 文件)
其中选项包括:
    ‐d32   使用 32 位数据模型 (如果可用)    
    ‐d64   使用 64 位数据模型 (如果可用)    
    ‐server   选择 "server" VM 
                  默认 VM 是 server,
                  因为您是在服务器类计算机上运行。
    ‐cp <目录和 zip/jar 文件的类搜索路径>
    ‐classpath <目录和 zip/jar 文件的类搜索路径>
                  用 : 分隔的目录, JAR 档案
                  和 ZIP 档案列表, 用于搜索类文件。
    ‐D<名称>=<值>
                  设置系统属性
    ‐verbose:[class|gc|jni]
                  启用详细输出
    ‐version      输出产品版本并退出
    ‐version:<值>
                  警告: 此功能已过时, 将在
                  未来发行版中删除。
                  需要指定的版本才能运行
    ‐showversion  输出产品版本并继续
    ‐jre‐restrict‐search | ‐no‐jre‐restrict‐search
                  警告: 此功能已过时, 将在
                  未来发行版中删除。
                  在版本搜索中包括/排除用户专用 JRE
    ‐? ‐help      输出此帮助消息
    ‐X            输出非标准选项的帮助
    ‐ea[:<packagename>...|:<classname>]
    ‐enableassertions[:<packagename>...|:<classname>]
                  按指定的粒度启用断言
    ‐da[:<packagename>...|:<classname>]
    ‐disableassertions[:<packagename>...|:<classname>]
                  禁用具有指定粒度的断言
    ‐esa | ‐enablesystemassertions
                  启用系统断言
    ‐dsa | ‐disablesystemassertions
                  禁用系统断言
    ‐agentlib:<libname>[=<选项>]
                  加载本机代理库 <libname>, 例如 ‐agentlib:hprof
                  另请参阅 ‐agentlib:jdwp=help 和 ‐agentlib:hprof=help
    ‐agentpath:<pathname>[=<选项>]
                  按完整路径名加载本机代理库
    ‐javaagent:<jarpath>[=<选项>]
                  加载 Java 编程语言代理, 请参阅 java.lang.instrument
    ‐splash:<imagepath>
                  使用指定的图像显示启动屏幕
```
#### 1.2.2.1. 实战
> 实战1：查看jvm版本
```
[root@node01 ~]# java ‐version
java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141‐b15)
Java HotSpot(TM) 64‐Bit Server VM (build 25.141‐b15, mixed mode)
# ‐showversion参数是表示，先打印版本信息，再执行后面的命令，在调试时非常有用，后面会使用到。
```

> 实战2：通过-D设置系统属性参数
```
public class TestJVM {
    public static void main(String[] args) {
        String str = System.getProperty("str");
        if (str == null) {
            System.out.println("hello");
        } else {
            System.out.println(str);
        }
    }
}
```
进行编译、测试：
```
#编译
[root@node01 test]# javac TestJVM.java
#测试
[root@node01 test]# java TestJVM
hello
[root@node01 test]# java ‐Dstr=123 TestJVM
123
```

#### 1.2.2.2. -server与-client参数

可以通过-server或-client设置jvm的运行参数。
* 它们的区别是 Server VM的初始堆空间会大一些，默认使用的是并行垃圾回收器，启动慢运行快。
* Client VM 相对来讲会保守一些，初始堆空间会小一些，使用串行的垃圾回收器，它的目标是为了让JVM的启动速度更快，但运行速度会比Serverm模式慢些。
* JVM 在启动的时候会根据硬件和操作系统自动选择使用Server还是Client类型的JVM。
* 32 位操作系统
    * 如果是 Windows系统，不论硬件配置如何，都默认使用Client类型的JVM。
    * 如果是其他操作系统上，机器配置有 2GB以上的内存同时有2个以上CPU的话默认使用server模式，否则使用client模式。
* 64 位操作系统
    * 只有 server类型，不支持client类型。

测试：
```
[root@node01 test]# java ‐client ‐showversion TestJVM
java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141‐b15)
Java HotSpot(TM) 64‐Bit Server VM (build 25.141‐b15, mixed mode)
hello
[root@node01 test]# java ‐server ‐showversion TestJVM
java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141‐b15)
Java HotSpot(TM) 64‐Bit Server VM (build 25.141‐b15, mixed mode)
hello
#由于机器是64位系统，所以不支持client模式
```



### 1.2.3. -X参数

jvm的-X参数是非标准参数，在不同版本的jvm中，参数可能会有所不同，可以通过java -X查看非标准参数。

```
[root@node01 test]# java ‐X
    ‐Xmixed           混合模式执行 (默认)
    ‐Xint             仅解释模式执行
    ‐Xbootclasspath:<用 : 分隔的目录和 zip/jar 文件>
                      设置搜索路径以引导类和资源
    ‐Xbootclasspath/a:<用 : 分隔的目录和 zip/jar 文件>
                      附加在引导类路径末尾
    ‐Xbootclasspath/p:<用 : 分隔的目录和 zip/jar 文件>
                      置于引导类路径之前
    ‐Xdiag            显示附加诊断消息
    ‐Xnoclassgc       禁用类垃圾收集
    ‐Xincgc           启用增量垃圾收集
    ‐Xloggc:<file>    将 GC 状态记录在文件中 (带时间戳)
    ‐Xbatch           禁用后台编译
    ‐Xms<size>        设置初始 Java 堆大小
    ‐Xmx<size>        设置最大 Java 堆大小
    ‐Xss<size>        设置 Java 线程堆栈大小
    ‐Xprof            输出 cpu 配置文件数据
    ‐Xfuture          启用最严格的检查, 预期将来的默认值
    ‐Xrs              减少 Java/VM 对操作系统信号的使用 (请参阅文档)
    ‐Xcheck:jni       对 JNI 函数执行其他检查
    ‐Xshare:off       不尝试使用共享类数据
    ‐Xshare:auto      在可能的情况下使用共享类数据 (默认)
    ‐Xshare:on        要求使用共享类数据, 否则将失败。
    ‐XshowSettings    显示所有设置并继续
    ‐XshowSettings:all
                      显示所有设置并继续
    ‐XshowSettings:vm 显示所有与 vm 相关的设置并继续
    ‐XshowSettings:properties
                      显示所有属性设置并继续
    ‐XshowSettings:locale
                      显示所有与区域设置相关的设置并继续
‐X 选项是非标准选项, 如有更改, 恕不另行通知。
```

#### 1.2.3.1. -Xint、-Xcomp、-Xmixed

* 在解释模式 (interpreted mode)下，-Xint标记会强制JVM执行所有的字节码，当然这会降低运行速度，通常低10倍或更多。

* -Xcomp 参数与它（-Xint）正好相反，JVM在第一次使用时会把所有的字节码编译成本地代码，从而带来最大程度的优化。
    * 然而，很多应用在使用 -Xcomp也会有一些性能损失，当然这比使用-Xint损失的少，原因是-xcomp没有让JVM启用JIT编译器的全部功能。JIT编译器可以对是否需要编译做判断，如果所有代码都进行编译的话，对于一些只执行一次的代码就没有意义了。
* -Xmixed 是混合模式，将解释模式与编译模式进行混合使用，由jvm自己决定，这是jvm默认的模式，也是推荐使用的模式。

> 示例：强制设置运行模式
```
#强制设置为解释模式
[root@node01 test]# java  ‐showversion ‐Xint TestJVM
java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141‐b15)
Java HotSpot(TM) 64‐Bit Server VM (build 25.141‐b15, interpreted mode)
hello
#强制设置为编译模式
[root@node01 test]# java  ‐showversion ‐Xcomp TestJVM
java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141‐b15)
Java HotSpot(TM) 64‐Bit Server VM (build 25.141‐b15, compiled mode)
hello
#注意：编译模式下，第一次执行会比解释模式下执行慢一些，注意观察。
#默认的混合模式
[root@node01 test]# java  ‐showversion TestJVM
java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141‐b15)
Java HotSpot(TM) 64‐Bit Server VM (build 25.141‐b15, mixed mode)
hello
```

### 1.2.4. -XX参数
-XX 参数也是非标准参数，主要用于jvm的调优和debug操作。
-XX参数的使用有2种方式，一种是boolean类型，一种是非boolean类型：
* boolean 类型
    * 格式： -XX:[+-]
    * 如： -XX:+DisableExplicitGC 表示禁用手动调用gc操作，也就是说调用System.gc()无效
* 非 boolean类型
    * 格式： -XX:
    * 如： -XX:NewRatio=1 表示新生代和老年代的比值
> 用法：
```
[root@node01 test]# java ‐showversion ‐XX:+DisableExplicitGC TestJVM
java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141‐b15)
Java HotSpot(TM) 64‐Bit Server VM (build 25.141‐b15, mixed mode)

hello
```

### 1.2.5. -Xms与-Xmx参数
-Xms与-Xmx分别是设置jvm的堆内存的初始大小和最大大小。
-Xmx2048m：等价于-XX:MaxHeapSize，设置JVM最大堆内存为2048M。
-Xms512m：等价于-XX:InitialHeapSize，设置JVM初始堆内存为512M。
适当的调整jvm的内存大小，可以充分利用服务器资源，让程序跑的更快。
> 示例:
```
[root@node01 test]# java ‐Xms512m ‐Xmx2048m TestJVM
hello
```

### 查看jvm的运行参数

#### 运行java命令时打印参数

#### 查看正在运行的jvm参数

## jvm内存模型

### jdk1.7的堆内存模型

### jdk1.8的堆内存模型

### 为甚要废弃1.7中的永久区

### 通过jstat命令查看堆内存的使用情况

#### 查看class加载统计

#### 查看编译统计

#### 垃圾回收统计

## jmap的使用以及内存溢出分析

### 查看内存使用情况
### 查看内存中对象数量以及大小
### 将内存使用情况dump到文件中
### 通过jhat堆dump文件进行分析
### 通过MAT工具对dump文件进行分析
#### MAT工具介绍
#### 下载安装
#### 使用

## 实战：内存溢出的定位和分析
### 模拟内存溢出
### 运行测试
### 导入到MAT工具中进行分析
