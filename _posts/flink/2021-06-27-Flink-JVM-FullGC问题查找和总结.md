---
layout: post
title: 记一次线上Flink-JVM-FullGC问题总结
categories: [Flink,Java]
description: Flink Taskmanager JVM FullGC
keywords: Flink JVM FullGC
---

# 背景和现象
目前公司线上数据量小，线上环境采用standlone模式部署的taskmanager，通过zabbix监控看到，taskmanager每5-10分钟有一次fullgc问题。
推测： 
1. 代码问题，有对象使用后没有做进行关闭或回收
2. JVM参数配置问题

# 结论
先说结论，通过阅读项目代码，发现写入mysql和clickhouse的api代码存在问题， statement，connection再使用完毕后没有关闭，以及类似的问题。
因为插入数据库时使用的是批量插入，这部分存在很大问题。（备注：刚进入公司，代码非本人编写。） 这证明了推测1。

观察线上环境的jvm参数配置 ，发现jvm中新生代内存配置不合理。8G的堆内存，只有332M的新生代，不符合3/8常规配比，证明了推测2.

JVM配置参考 [https://www.huaweicloud.com/articles/b86de23d6c3d5a161b25b1013a388d8d.html](https://www.huaweicloud.com/articles/b86de23d6c3d5a161b25b1013a388d8d.html)
# 解决步骤
1. 代码部分：
只展示部分
**Connection和PreparedStatement**

``` java

Connection connection = null;
try {
    connection = pool.getConnection();
    connection.setAutoCommit(false);
    PreparedStatement prest = connection.prepareStatement(sql, ResultSet.TYPE_SCROLL_SENSITIVE, ResultSet.CONCUR_READ_ONLY);
    for(Map.Entry<String, Long> entry : metrics.entrySet()) {
        prest.setLong(1, entry.getValue()/1000);
        prest.setString(2, entry.getKey());
        prest.setInt(3, Integer.parseInt(productId));
        prest.addBatch();
    }
    prest.executeBatch();
    connection.commit();
} catch (Exception e) {
    connection.rollback();
    System.err.println(sql);
    System.err.println(e);
    throw new Exception(e.getMessage());
} finally {
    if (connection != null) {
        try {
            connection.close();
        } catch (SQLException throwables) {
            throwables.printStackTrace();
        }
    }
}

```

修改后：

``` java 

Connection connection = null;
PreparedStatement prest = null;
try {
    connection = pool.getConnection();
    connection.setAutoCommit(false);
    prest = connection.prepareStatement(sql, ResultSet.TYPE_SCROLL_SENSITIVE, ResultSet.CONCUR_READ_ONLY);
    for (Map.Entry<String, Long> entry : metrics.entrySet()) {
        prest.setLong(1, entry.getValue() / 1000);
        prest.setString(2, entry.getKey().substring(3));
        prest.setInt(3, Integer.parseInt(productId));
        prest.addBatch();
    }
    prest.executeBatch();
    connection.commit();
    prest.clearParameters();
} catch (Exception e) {
    connection.rollback();
    System.err.println(sql);
    System.err.println(e);
    throw new Exception(e.getMessage());
} finally {
    if (prest != null && !prest.isClosed()) {
        try {
            prest.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    if (connection != null && !connection.isClosed()) {
        try {
            connection.close();
        } catch (SQLException throwables) {
            throwables.printStackTrace();
        }
    }
}
```

**StringBuilder部分**

阅读源码和网络文章

源码：
``` java
java.lang.StringBuilder

//404行
@Override
public String toString() {
    // Create a copy, don't share the array
    return new String(value, 0, count);
}

```
当StringBuilder.toString()时，会创建一个新的String对象。
当你的String很长时，多次toString，对产生很多大对象，容易把新生代使用满，但是对象还未回收进入到老年代(特别是当前项目中，批量插入数据库中时，数据足够大，基本上千升值上万条批次插入clickhouse，且数据都放在String中)。
应该尽量避免tostring，在StringBuilder完成后toString一次，用一个对象去接受；不要多次toString（这部分的代码优化来源于jmap -heap pid | jmap -dump:format=b,file=xxx.hprof pid 统计得出。
）

``` bash

jmap -dump:format=b,file=./06281330.hprof 10058
#将06281330.hprof 文件下载后 使用VirsualVM分析 hprof文件，发现有char[]中有很多StringBuilder中的字符串数据，本应该回收的，但是一直在jvm内存中。
#目前造成这种现象一个是因为多次引用StringBuilder中的内容时，都toString造成，还有另外一方面是因为新生代内存太小。

```

使用参考 [https://www.yiibai.com/java_data_type/java_stringbuilder_stringbuffer.html](https://www.yiibai.com/java_data_type/java_stringbuilder_stringbuffer.html)

代码部分优化后可以看到FullGC频率降低了。

2. 分析过程

通过将线上环境的堆内存dump文件下载下来分析。（因本地环境的数据量小不是很容易复现）
``` bash
jmap -dump:format=b,file=xxx.hprof pid
```
将上述中获取的xxx.hprof文件下载到本地。

使用VisualVM工具分析对象在堆内存中的占用情况，发现char[]中有大量的批量插入语句，占比达到了惊人的54.8%。
这其中的插入语句不应该有这么多存在。
这部分的主要原因应该是代码中StringBuilder多次toString和PreparedStatement未关闭造成；
还有部分原因是因为新生代内存太小，太大对象产生后直接进入到了老年代。
至此也证实代码中造成FullGC的一些问题。


3. JVM参数部分：

观察线上环境的jvm参数配置:

发现jvm中新生代内存配置不合理。8G的堆内存，只有332M的新生代，不符合3/8常规配比

优化前参数：
``` bash
jmap -heap pid
                                                                                                
Debugger attached successfully.                                                                  
Server compiler detected.                                                                        
JVM version is 25.251-b08                                                                        
                                                                                                 
using parallel threads in the new generation.                                                    
using thread-local object allocation.                                                            
Concurrent Mark-Sweep GC                                                                         
                                                                                                 
Heap Configuration:                                                                              
   MinHeapFreeRatio         = 40                                                                 
   MaxHeapFreeRatio         = 70                                                                 
   MaxHeapSize              = 8455716864 (8064.0MB)                                              
   NewSize                  = 348913664 (332.75MB)                                               
   MaxNewSize               = 348913664 (332.75MB)                                               
   OldSize                  = 8106803200 (7731.25MB)                                                                                                                                               
   NewRatio                 = 2                                                                  
   SurvivorRatio            = 8                                                                  
   MetaspaceSize            = 536870912 (512.0MB)                                                
   CompressedClassSpaceSize = 536870912 (512.0MB)                                                
   MaxMetaspaceSize         = 1073741824 (1024.0MB)                                              
   G1HeapRegionSize         = 0 (0.0MB)                                                          

Heap Usage:                                                                                      
New Generation (Eden + 1 Survivor Space):                                                        
   capacity = 314048512 (299.5MB)                                                                
   used     = 139638808 (133.1699447631836MB)                                                    
   free     = 174409704 (166.3300552368164MB)                                                    
   44.46408840173075% used                                                                       
Eden Space:                                                                                      
   capacity = 279183360 (266.25MB)                                                               
   used     = 104773656 (99.9199447631836MB)                                                     
   free     = 174409704 (166.3300552368164MB)                                                    
   37.52861775143046% used                       
From Space:                                                                                      
   capacity = 34865152 (33.25MB)                                                                 
   used     = 34865152 (33.25MB)                                                                 
   free     = 0 (0.0MB)                                                                          
   100.0% used                                                                                   
To Space:                                                                                        
   capacity = 34865152 (33.25MB)                                                                 
   used     = 0 (0.0MB)                                                                          
   free     = 34865152 (33.25MB)                                                                 
   0.0% used                                                                                     
concurrent mark-sweep generation:                                                                
   capacity = 8106803200 (7731.25MB)                                                             
   used     = 5230841296 (4988.518997192383MB)                                                   
   free     = 2875961904 (2742.731002807617MB)                                                   
   64.52409373894756% used                                                                       

32186 interned Strings occupying 3597144 bytes.

```
优化后JVM参数：
``` bash

Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 8455716864 (8064.0MB)
   NewSize                  = 2147483648 (2048.0MB)
   MaxNewSize               = 2147483648 (2048.0MB)
   OldSize                  = 6308233216 (6016.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 536870912 (512.0MB)
   CompressedClassSpaceSize = 536870912 (512.0MB)
   MaxMetaspaceSize         = 1073741824 (1024.0MB)
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 1932787712 (1843.25MB)
   used     = 1211894112 (1155.7522888183594MB)
   free     = 720893600 (687.4977111816406MB)
   62.70187379999237% used
Eden Space:
   capacity = 1718091776 (1638.5MB)
   used     = 1165076592 (1111.1036224365234MB)
   free     = 553015184 (527.3963775634766MB)
   67.81224427442926% used
From Space:
   capacity = 214695936 (204.75MB)
   used     = 46817520 (44.64866638183594MB)
   free     = 167878416 (160.10133361816406MB)
   21.80643046731914% used
To Space:
   capacity = 214695936 (204.75MB)
   used     = 0 (0.0MB)
   free     = 214695936 (204.75MB)
   0.0% used
concurrent mark-sweep generation:
   capacity = 6308233216 (6016.0MB)
   used     = 1113470792 (1061.8884963989258MB)
   free     = 5194762424 (4954.111503601074MB)
   17.651072081099166% used

```
优化的Flink-TaskManager启动参数
flink-conf.yml
``` yml
#  -Xmn2G 设置新生代为2G ，当启动Taskmanager后，没有启动任务时，发现Taskmanager的新生代已经使用了983M，占比53%，所以原来的300多M是很不合理的。也是造成FullGC的最主要原因.
env.java.opts.taskmanager: -Djava.util.Arrays.useLegacyMergeSort=true -XX:NativeMemoryTracking=detail -Xmn2G  -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=70  -XX:+UseCompressedClassPointers -XX:CompressedClassSpaceSize=512M -XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=1024m

```

# JVM调优常用参数

**参数说明** **[参考华为jvm调优文档](https://www.huaweicloud.com/articles/b86de23d6c3d5a161b25b1013a388d8d.html)**
```
参数名称	含义	默认值	描述
-Xms	初始堆大小	物理内存的1/64(<1gb)<td>	默认(MinHeapFreeRatio参数可以调整)空余堆内存小于40%时，JVM就会增大堆直到-Xmx的最大限制.
-Xmx	最大堆大小	物理内存的1/4(<1gb) <td>	默认(MaxHeapFreeRatio参数可以调整)空余堆内存大于70%时，JVM会减少堆直到 -Xms的最小限制
-Xmn	年轻代大小(1.4or lator)	 	注意：此处的大小是（eden+ 2 survivor space).与jmap -heap中显示的New gen是不同的。
整个堆大小=年轻代大小 + 年老代大小 + 持久代大小.
增大年轻代后,将会减小年老代大小.此值对系统性能影响较大,Sun官方推荐配置为整个堆的3/8
-XX:NewSize	设置年轻代大小(for 1.3/1.4)	 	 
-XX:MaxNewSize	年轻代最大值(for 1.3/1.4)	 	 
-XX:PermSize	设置持久代(perm gen)初始值	物理内存的1/64	 
-XX:MaxPermSize	设置持久代最大值	物理内存的1/4	 
-Xss	每个线程的堆栈大小	 	JDK5.0以后每个线程堆栈大小为1M,以前每个线程堆栈大小为256K.更具应用的线程所需内存大小进行 调整.在相同物理内存下,减小这个值能生成更多的线程.但是操作系统对一个进程内的线程数还是有限制的,不能无限生成,经验值在3000~5000左右
一般小的应用， 如果栈不是很深， 应该是128k够用的 大的应用建议使用256k。这个选项对性能影响比较大，需要严格的测试。（校长）
和threadstacksize选项解释很类似,官方文档似乎没有解释,在论坛中有这样一句话:"”
-Xss is translated in a VM flag named ThreadStackSize”
一般设置这个值就可以了。
-XX:ThreadStackSize	Thread Stack Size	 	(0 means use default stack size) [Sparc: 512; Solaris x86: 320 (was 256 prior in 5.0 and earlier); Sparc 64 bit: 1024; Linux amd64: 1024 (was 0 in 5.0 and earlier); all others 0.]
-XX:NewRatio	年轻代(包括Eden和两个Survivor区)与年老代的比值(除去持久代)	 	-XX:NewRatio=4表示年轻代与年老代所占比值为1:4,年轻代占整个堆栈的1/5
Xms=Xmx并且设置了Xmn的情况下，该参数不需要进行设置。
-XX:SurvivorRatio	Eden区与Survivor区的大小比值	 	设置为8,则两个Survivor区与一个Eden区的比值为2:8,一个Survivor区占整个年轻代的1/10
-XX:LargePageSizeInBytes	内存页的大小不可设置过大， 会影响Perm的大小	 	=128m
-XX:+UseFastAccessorMethods	原始类型的快速优化	 	 
-XX:+DisableExplicitGC	关闭System.gc()	 	这个参数需要严格的测试
-XX:MaxTenuringThreshold	垃圾最大年龄	 	如果设置为0的话,则年轻代对象不经过Survivor区,直接进入年老代. 对于年老代比较多的应用,可以提高效率.如果将此值设置为一个较大值,则年轻代对象会在Survivor区进行多次复制,这样可以增加对象再年轻代的存活 时间,增加在年轻代即被回收的概率
该参数只有在串行GC时才有效.
-XX:+AggressiveOpts	加快编译	 	 
-XX:+UseBiasedLocking	锁机制的性能改善	 	 
-Xnoclassgc	禁用垃圾回收	 	 
-XX:SoftRefLRUPolicyMSPerMB	每兆堆空闲空间中SoftReference的存活时间	1s	softly reachable objects will remain alive for some amount of time after the last time they were referenced. The default value is one second of lifetime per free megabyte in the heap
-XX:PretenureSizeThreshold	对象超过多大是直接在旧生代分配	0	单位字节 新生代采用Parallel Scavenge GC时无效
另一种直接在旧生代分配的情况是大的数组对象,且数组中无外部引用对象.
-XX:TLABWasteTargetPercent	TLAB占eden区的百分比	1%	 
-XX:+CollectGen0First	FullGC时是否先YGC	false
```