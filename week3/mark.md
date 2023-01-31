##### 遇到的问题：(问题已经解决)

试过好多办法 目前卡在prometheus无法添加检测点，但是直接访问端口+ip是可以拿到信息的。

![无法添加](.\picture\无法添加.png)



访问 `http://IP:9001/actuator/prometheus`

![接口响应](.\picture\接口响应.png)



在prometheus 页面上 Status - Targets 看不到添加的数据，但是在 Status - Service Discovery 是能看到的。

![disc](.\picture\disc.png)



由于检测不到，添加模板也无数据。

![无数据](.\picture\无数据.png)







##### GC实验

目前以课件里GC日志与视频内容作为参考了。

GC日志解读官网：https://gceasy.ycrash.cn/



**JVM 配置参数的素材：**

```
# 吞吐量优先策略： 
JAVA_OPT="${JAVA_OPT} -Xms256m -Xmx256m -Xmn125m -XX:MetaspaceSize=128m - Xss512k" 
JAVA_OPT="${JAVA_OPT} -XX:+UseParallelGC -XX:+UseParallelOldGC "
JAVA_OPT="${JAVA_OPT} -XX:+PrintGCDetails -XX:+PrintGCTimeStamps - XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:${BASE_DIR}/logs/gc-ps- po.log" 

# 响应时间优先策略 
JAVA_OPT="${JAVA_OPT} -Xms256m -Xmx256m -Xmn125m -XX:MetaspaceSize=128m - Xss512k" 
JAVA_OPT="${JAVA_OPT} -XX:+UseParNewGC -XX:+UseConcMarkSweepGC "
JAVA_OPT="${JAVA_OPT} -XX:+PrintGCDetails -XX:+PrintGCTimeStamps - XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:${BASE_DIR}/logs/gc-parnew- cms.log" 

# 全功能垃圾收集器 
JAVA_OPT="${JAVA_OPT} -Xms256m -Xmx256m -XX:MetaspaceSize=128m -Xss512k"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:MaxGCPauseMillis=100"
JAVA_OPT="${JAVA_OPT} -XX:+PrintGCDetails -XX:+PrintGCTimeStamps - XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:${BASE_DIR}/logs/gc-g- one.log"
```

先对JVM参数作一个描述，对于第一种PS+PO的组为吞吐量优先，第二种为ParNew+CMS是以响应时间优先， 第三种为G1是一种多功能的场景GC策略。



###### 吞吐量优先策略

![ps+po](.\picture\ps+po.png)

这里看到GC清理的情况垃圾回收的大小，不过吞吐量显示80%，是因为有些情况下，对比垃圾收集次数与时间是不同的，由于影响因素比较大，所以不能直接以百分比作依据。







###### 响应时间优先策略



###### ![pnew+cms](.\picture\pnew+cms.png)

![cms](.\picture\cms.png)

对于ParNew+CMS来讲是以响应时间为主导，可以看出请求响应时间明显较少。



对比：

![gra](.\picture\gra.png)



通过对比图可以看到左面的QPS对比，可以看到使用ParNew+CMS 会比PS+PO的情况好一点点，右面的请求耗时可以看到，ParNew+CMS 更加稳定且响应时间更小。





###### 全功能垃圾收集器

![G1g](.\picture\G1g.png)

G1

###### ![G1t](.\picture\G1t.png)



图可看出老年代GC只发生了一次。对于G1来讲 使用场景更广泛，不过基于新的策略算法建议在高内存场景下使用。





##### 问题已经解决

![大盘](.\picture\大盘.png)

原因是容器环境下的容器名称填写的不对，已经修改正确的就可以了。