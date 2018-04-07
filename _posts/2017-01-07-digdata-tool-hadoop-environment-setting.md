---
layout: post
title: hadoop 环境搭建
key: 20170107
tags: hadoop环境搭建
---

### 一、安装JDK

在linux命令行中，先敲入 命令 来查看系统上是否有安装 jak

```
javac
```

如果 没有 ，我们就要安装JDK
ubuntu下

```
apt-get install openjdk-7-jdk
```

centOS下

```
yum install *jdk-7*
```

安装完成后，我们开始配置jdk 的环境变量

     打开 /etc/profile  进行修改，添加以下信息【注意 安装目录可能会有所不同】

```
export JAVA_HOME=/usr/lib/jvm/java-7-openjdk-amd64
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=.:$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH
export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
```

保存， 输入命令 生效

```
source /etc/profile
```

### 二、 配置hadoop

    1 下载hadoop 安装包
          【
                    由于库中没有，我们使用源来下载 ：  http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-1.2.1/hadoop-1.2.1.tar.gz
           】

```
wget http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-1.2.1/hadoop-1.2.1.tar.gz
```


       我们把这个压缩包一到/opt/目录下

```
mv hadoop-1.2.1.tar.gz /opt/
```
      来到 /opt/目录下后  我们解压缩

```
tar -zxvf hadoop-1.2.1.tar.gz
```

   2  打开解压缩的目录进行配置
         我们要配置的文件 在conf 目录下，要配置的有4个文件
		![这里写图片描述](https://app.yinxiang.com/shard/s5/res/12797eb2-32aa-48e5-bed6-fb1f70368ad4)
   1） 打开 hadoop-env.sh 文件，配置  JAVA_HOME 的更新, 要和上文中配置的环境变量 JAVA_HOME 一致
      【指定Hadoop要用的JDK 环境变量，守护进程JDK 选项，pid文件和log文件夹】

vim hadoop-env.sh  //进行配置

   2） 打开 core-site.xml 文件，
     【指定与Hadoop守护进程和客户端相关参数的XML文件】【主要配置 】

```
<configuration>
<!-- 设置hadoop的工作目录 -->
<property>
     <name>hadoop.tmp.dir</name>
     <value>/hadoop</value>
</property>

<!--  fs.default.name - 这是一个描述集群中NameNode结点的URI(包括协议、主机名称、端口号)，集群里面的每一台机器都需要知道NameNode的地址。DataNode结点会先在NameNode上注册，这样它们的数据才可以被使用。独立的客户端程序通过这个URI跟DataNode交互，以取得文件的块列表。-->
<property>
     <name>fs.default.name</name>
     <value >hdfs://localhost:9000</value>     <!-- 这里 localhost 应填写当前主机的hostname， 请根据现实情况进行配置，这里我配置为本地-->
</property>

</configuration>
```

   3） 打开hdfs-site.xml
     【指定HDFS守护进程和客户端要用的参数的XML文件】【主要配置 】

```
<configuration>
<!-- 配置文件系统的数据存放目录 -->
<property>
     <name>dfs.daya.dir</name>
     <value >/hadoop/data</value>
</property>

</configuration>
```

    4）打开mapred-site.xml
     【指定MapReduce守护进程和客户端要用的参数的XML文件 】【主要配置 】

```
<configuration>
<!-- 设置HDFS中每个Block块被复制的次数 -->
<property>
     <name>dfs.replication</name>
     <value>1</value>
</property>

<!-- 设置将HDFS文件系统的元信息的保存目录 -->
<property>
     <name>dfs.name.dir</name>
     <value>/hadoop/name</value>
</property>

<!-- 配置任务调度器的访问  -->
<property>
     <name>mapred.job.tracker</name>
      <value>localhost:9001</value >
</property>

</configuration>
```

下面是其他配置文件的说明
     log4j.properties
     【一个包含所有日志配置信息的java属性文件】

     masters
     【在新行中列出运行 次NameNode 的机器，只会被satrt-*。sh类的脚本调用】

     slavers
     【在新行中列出运行DataNode / tasktracker进程对的服务器名，只会被satrt-*。sh类的脚本调用】

     fair-scheduler
     【用来指定资源库，设置MapRduce的Fari Scheduler 任务调度器插件】

     capacity-scheduker
     【曾经用来指定MaoReduce Caoacity Scheduler 任务调度插件的队列和设置】

     dfs.include
     【在新行中列出允许连接NameNode的服务器名】

     hadoop-policy
     【用来定义和Hadoop通信时，哪个用户和哪个组允许调用指定的RPC功能的XML文件】

     mapred-queue-acls
     【定义哪个用户和哪个组被允许提交作业到哪个MapReduce作业队列的XML文件】

     taskcontroller.cfg
     【类似于Java属性风格的文件，定义了在安全模式下操作时所用到的MapReduce辅助程序 setuid task-controller 要用的值】

    5）配置hadoop的环境变量，打开/etc/profile 文件进行添加HADOOP的安装路径 配置，配置完成后，使它生效

```
export HADOOP_HOME=/opt/hadoop-1.2.1
在PATH 里面加入了 :$HADOOP_HOME/bin
```

**#让配置生效**

```
 source /etc/profile
```

### 3 对namenode 进行格式化操作

```
hadoop namenode -format
```

输入 ：start-all.sh  命令后可能会出现 【root@localhost's password:localhost:permission denied,please try again  错误】
解决方案，使用试试免密登录，设置收重新 格式化 namenode ,start-all

设置ssh 免密,移步下面这篇博客
http://www.cnblogs.com/qiangweikang/p/4740936.html

4 使用 jps 查看前所有java进程pid的，如果有以下进程， 说明我们的hadoop启动成功

jps是JDK 1.5提供的一个显示当前所有java进程pid的命令，简单实用，非常适合在linux/unix平台上简单察看当前java进程的一些简单情况。
![这里写图片描述](http://img.blog.csdn.net/20170109105429578?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbmcxNDgz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20170109105453391?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbmcxNDgz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![这里写图片描述](http://img.blog.csdn.net/20170109105517062?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2hlbmcxNDgz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
小结

1） 在Linux中安装JDK,  并设置环境变量
2）下载Hadoop，并设置Hadoop的环境变量
3）修改Hadoop的配置文件