[TOC]

Android 崩溃

Android 的崩溃分为两种

- Java 崩溃
- Native 崩溃

## 1. 客观衡量崩溃和稳定性

### 1.1 崩溃
UV 崩溃率可以评估崩溃造成的用户影响范围

**UV 崩溃率 = 发生崩溃的 UV/ 登录 UV**

启动崩溃需要单独统计，因为启动崩溃对用户带来的伤害最大，应用无法启动，热修复也无法拯救。可以考虑监控客户端启动失败后，启动安全模式，保障客户端的启动流程。

### 1.2 稳定性

用异常率来衡量应用的稳定性

**UV 异常率 = 发生异常退出或崩溃的 UV / 登录 UV**

**发生异常退出**的情况分为两种

- **前台异常退出**
    更加关注的情况，一般与 ANR , OOM 等异常情况下有更大的关联
    
- **后台异常退出**
    后台异常退出的主要原因是被系统杀死

Android 中应用退出的情形：

- 自动自杀。 Process.killProcess() 、 exit() 等
- 崩溃。出现 Java 或 Native 崩溃
- 系统重启。系统出现异常、断电、用户主动重启等，可以通过比较应用开机运行时间是否比之前记录的值更小来判断是否属于系统重启导致的退出
- 被系统杀死。 被 low memory killer 杀掉、从系统的任务管理中划掉等
- ANR

## 2. Java 崩溃

Java 崩溃就是在 Java　代码中，出现了未捕获异样，导致程序异常退出。
Java 崩溃比较容易捕获简单


## 3. Native 崩溃

Native 崩溃一般是在 Native 代码中访问非法地址，也可以能是地址对齐出现问题，或者程序主动 abort, 这些产生相应的 signal 信号，导致程序异常退出。


## 4. 崩溃捕获

崩溃捕获主要是崩溃现场的信息，这些信息对解决崩溃是至关重要的。

### 4.1 需要采集崩溃现场的信息
#### 4.1.1 崩溃信息 

- 进程名、线程名
    崩溃的进程是前台进程合适后台进程，崩溃是不是发生在 UI 线程
- 崩溃堆栈和类型
    崩溃是属于 Java 崩溃、Native 崩溃，还是 ANR, 对于不同类型的崩溃关注的点也不一样。特别需要看崩溃堆栈的栈顶，看具体崩溃在系统的代码，还是自己写的代码里面

#### 4.1.2 系统信息
系统信息也已有一些关键的线索

- Logcat 日志
    Logcat 日志包含应用、系统的运行日志。

- 机型、系统、厂商、CPU、ABI、Linux 版本等。
    采集这些数据可以对寻找共性有帮助，例如是不是只有某个品牌的手机才用、是不是某个版本才有
    
- 设备状态，是否 root、是否是模拟器
    一些问题是由 Xposed 或者软件多开造成的
    
#### 4.1.3 内存信息
很多崩溃和内存有着直接的关系，所以，内存信息也是很重要的。

- 系统剩余内存
    对于系统内存状态，可以直接读取文件 /proc/meminfo 获取。当系统可以用内存小（低于 MemTotal 10%）时，OOM、大量 GC、系统频繁自杀拉起等问题都非常容易出现
- 应用使用内存
    包括 Java 内存、RSS(Resident Set Size)、PSS(Proportional Set Size)， 可以计算出已用本身内存的占用和分布。 PSS 和 RSS 通过 /proc/self/smap 计算，可以近一些获取 apk、dex、so 等更加详细的分类统计
- 虚拟内存
  虚拟内存可以通过 /proc/self/status 得到，通过 /proc/self/maps 文件可以得到具体的内存分布情况。很多问题、例如 OOM, tgkill 问题都是虚拟内存不足导致的
    
#### 4.1.4 资源信息

在应用堆内存和设备内存非常充足，还是出现内存分配失败的情况，这跟资源泄露可能有比较大的关系。

- 文件句柄 fd
    文件句柄的限制可以通过 /proc/self/limits 获取，一般单个进程允许打开的最大文件句柄数是 1024.但是文件句柄超过 800 个就是比较危险的，需要将所有的 fd 以及对应的文件名输出到日志中，进一步排查是否出现了文件或线程泄露。

- 线程数
  当前线程数可以通过 status 文件得到，一个线程可能占 2 MB 的虚拟内存，过多的线程会对虚拟内存和文件句柄带来压力。需要将所有的线程 id 以及对应的线程名输出到日志中，进一步排查是否出现线程相关的问题。

- JNI
    使用 JNI 时，如果不注意很容易出现引用无效、引用爆表等一些崩溃。可以通过 DumpReferenceTables 统计 JNI 的引用表，进一步分析是否出现 JNI 泄露等问题。

#### 4.1.5 应用信息
崩溃场景，关键操作路径，其他定义信息


### 4.2 Native 崩溃的捕获

#### 4.2.1 难点
- **文件句柄泄露，导致创建日志文件失败**
  方法：我们需要提前申请文件句柄预留，防止出现这种情况
  
- **栈溢出了，导致日志生成失败**
  方法：为了防止栈溢出导致进程没有空间创建调用栈执行处理函数，通常会使用 signalstack。 在一些特殊的情况，还需要替换当前栈，所以也这里需要在堆中预留部分空间
  
- **整个堆的内存都耗尽了，导致日志生成失败**
  方法：在堆的内存耗尽的情况下，无法安全地分配内存，也不敢使用 stl 或者 libc 的函数，因为它们内部实现会分配堆内存。这个时候如果继续分配内存，会导致出现堆破坏或者二次崩溃的情况。常见的解决方法可以参考 Breakpad 的做法，就是重新封装 Linux Syscall Support, 来避免直接调用 libc.
  
- **堆破坏或二次崩溃导致日志生成失败**
    方法：Breakpad 会从原进程 fork 出子进程去收集崩溃现场，此外涉及与 Java 相关的，一般也会使用子进程的去操作。这样即使出现二次崩溃，只是部分的信息丢失，我们的父进程后面还可以继续获取其他的信息。在特殊的情况，还可以从子进程 fork 出孙进程。
    

## 5 崩溃分析

步骤：
### 5.1 第一步：确定重点
- **1.确认严重程度**
    解决崩溃要看性价比，优先解决 Top 崩溃或对业务有重大影响，例如启动、支付过程的崩溃

- **2.崩溃基本信息**
    确定崩溃的类型和异常描述，对崩溃有大致的判断。
    - Java 崩溃
    - Native 崩溃
        需要观察 signal,code,fault addr 等内容，以及崩溃时 Java 的堆栈。比较常见的有 SIGSEGV 和 SIGABRT, 前者一般是由于空指针、非法指针造成， 后者主要因为 ANR 和调用 abort() 退出所导致
    - ANR
      先看主线程堆栈，是否因为等锁等待导致。接着看 ANR  日志中 iowait, CPU, GC, system server 等信息，进一步确认是 I/O 问题，或是 CPU 竞争问题，还是大量 GC 导致卡死

- **3.Logcat**
    日志一般会有价值的线索。如果从一条崩溃日志中无法看出问题的原因，或者等不到有用信息，可以查看相同崩溃点下的更多崩溃日志，查找共性
    
- **4.各个资源情况**
    结合崩溃信息，看看是不是和内存信息相关或者资源信息相关。

### 5.2 第二步：查找共性
当第一步不能有效定位问题，可以查找这类问题的共性。

机型、系统、ROM、厂商、ABI 等这些维度的信息可以聚合，查找共性

### 5.3 第三步：尝试复现

## 6 崩溃解决
如果是 Android 版本的 bug 或者 厂商修改 ROM  导致的,这种

-  查找可能的原因
-  尝试规避
-  Hook 解决
    hook 可以用 Java hook 或者 Native hook 解决

崩溃优化不是孤立的，它与内存、卡顿、 I/O 等内容有关
## 参考
https://jsonchao.github.io/2019/11/24/%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%B4%A2Android%E7%A8%B3%E5%AE%9A%E6%80%A7%E4%BC%98%E5%8C%96/
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  
  


