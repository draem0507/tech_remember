有rd反馈部分机器jps命令时，显示的进程号与实际的java进程不一致，且该问题普遍发生在上海侧机器。



# 排查

### jps命令是干什么的，如何显示进程号

jps (Java Virtual Machine Process Status Tool)，可以显示当前运行的java进程以及相关参数，执行jps命令时，会在/tmp目录下查找hsperfdata_<username>目录，显示结果为这些目录下的文件名，只有当前用户有读权限才会被展示，例如sankuai用户，一般只能读取/tmp/hsperfdata_sankuai目录下文件的内容，而root用户却可以读取hsperfdata_*目录下的内容。



### hsperfdata_<username>目录下存放的是什么文件

jvm运行时会生成一个目录hsperfdata_<username>(username是启动java进程的用户),在linux中默认是/tmp,目录下会有些 pid文件,存放jvm进程信息,而jmap,jstack等工具会读取/tmp/hsperfdata_<username>下的pid文件获取连接信息



### 为什么jps显示的进程好和ps命令显示的不一致

了解了jps和hsperfdata后去hsperfdata_sankuai目录下查看文件列表，发现只有一个文件，也就是图中展示的306233，这样就能解释通了。查阅相关资料得知，jvm会在当前用户接下来的任何一个java进程(比如说我们执行jps)起来的时候会去做一个判断，遍历/tmp/hsperfdata_<user>下的进程文件，挨个看进程是不是还存在，如果不存在了就直接删除该文件，因此需要知道目录未更新的原因



### hsperfdata_sankuai目录下文件为什么没有更新

为什么这台机器没有更新？当时怀疑以下几个原因

1、和[上海侧启动用户切换](https://km.sankuai.com/page/16326655)有关

2、jdk版本问题

3、权限问题，当前用户没有该目录的写权限

怀疑第一个原因，找了一台测试机器(os-test-cs-web01.yp)和业务机器(mp-recommend-menu-service02.beta)做对比，在两台机器上分别重启tomcat

|                                          | mp-recommend-menu-service02.beta | os-test-cs-web01.yp |
| :--------------------------------------- | :------------------------------- | :------------------ |
|                                          | mp-recommend-menu-service02.beta | os-test-cs-web01.yp |
| jps执行结果                              | 异常                             | 正常                |
| hsperfdata_sankuai目录下文件内容         | 异常                             | 正常                |
| 重启tomcat是否更新hsperfdata_sankuai目录 | 否                               | 是                  |

接着对比了两台机器的jdk版本

|         | mp-recommend-menu-service02.beta | os-test-cs-web01.yp |
| :------ | :------------------------------- | :------------------ |
| jdk版本 | 1.7.0_80                         | 1.7.0_76            |

于是修改mp-recommend-menu-service02.beta机器上的jdk版本，重启tomcat后结果相同，jps命令还是异常，排除jdk版本问题。

然后对比两台机器的目录权限

|      | mp-recommend-menu-service02.beta | os-test-cs-web01.yp |
| :--- | :------------------------------- | :------------------ |
| 属主 | sankuai:sankuai                  | sankuai:sankuai     |
| mode | 775                              | 755                 |

因为一开始认为只要当前用户有目录的写权限就可以，没有怀疑是权限的问题，此时只是抱着尝试的心态修改了权限，测试发现jps能够正常运行了。于是将目光锁定在目录权限上继续查原因。

### hsperfdata_sankuai目录权限要求

Google后查到相关文档，jvm会对hsperfdata目录做权限检查，如果设置组用户或其他用户的写权限，jvm不会去修改改目录



# 结论

切换tomcat启动用户时，给/tmp目录下的文件都赋予了组用户的写权限，导致jps工作异常



# 解决方案

修改切换脚本，/tmp目录下的文件赋予组用户的写权限后，将hsperfdata_*目录的权限设置为755



# 参考资料

【1】https://mp.weixin.qq.com/s/gCE9eXbtMuze3jhuRm1YXA

【2】https://bugzilla.redhat.com/show_bug.cgi?id=1123870

【3】[http://hg.openjdk.java.net/jdk7u/jdk7u/hotspot/file/f8795cb3f5c1/src/os/linux/vm/perfMemory_linux.cpp#l200](http://hg.openjdk.java.net/jdk7u/jdk7u/hotspot/file/f8795cb3f5c1/src/os/linux/vm/perfMemory_linux.cpp#l200)