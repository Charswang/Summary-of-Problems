### Hadoop集群配置解决
第一次配置所用包及版本（jdk-8u11-linux-x64.tar ， hadoop-2.6.0.tar）

##### 按照网上的ubuntu系统配置Hadoop环境的csdn博客(https://blog.csdn.net/u014636511/article/details/80171002)从头到脚走一遍，但是其中会遇到一些问题.
1. 最后利用scp命令将主节点(master)中更改的hadoop中的配置文件信息复制到子节点(slave1,slave2)中，**按照他的命令是不对的**  
**解决** ：不如不使用scp命令，直接将在master上更改的配置文件信息，**原样的在子节点中更改**。直接在master中先配置好，然后子节点直接赋值master中的配置信息即可。
2. 启动hadoop集群，发现报出找不到JAVA_HOME的错误，但是测试java，java是安装好了的。利用echo $JAVA_HOME也能得出JAVA_HOME的路径。  
**解决：**更改/home/had_user/local/hadoop-2.6.0/etc/hadoop下的hadoop-env.sh中的**export JAVA_HOME=${JAVA_HOME}**更改为**export JAVA_HOME=/home/had_user/local/jdk1.8.0_11**
3. hadoop namenode -format只使用一次就好。如果多次使用之后，发现子节点没有了datanode的话，那就将/home/had_user/local/hadoop-2.6.0下的**logs和tmp目录删除，子节点的也删除掉**。然后**再执行一遍hadoop namenode -format**就好了。  
4. 如果多次执行hadoop namenode -format之后，/home/had_user/local/hadoop-2.6.0/hdfs/name/current下的VERSION文件中的clusterID只会保留最新一次执行hadoop namenode -format的clusterID，但是/home/had_user/local/hadoop-2.6.0/hdfs/data/current下的VERSION文件是不会变的还是停留在第一次执行hadoop namenode -format的结果，所以要讲name中VERSION中的clusterID复制到data中VERSION文件中去。

##### 第一次配置hadoop集群遇到的问题及解决到此完成。

1. IDEA写的hdfs代码，说没有权限  “Permission denied: user=wanghao, access=WRITE, inode="/":root:supergroup:drwxr”  
**解决：**修改 HADOOP 配置文件 hdfs-site.xml 文件

```
添加
<property>
    <name>dfs.permissions</name>
    <value>false</value>
</property>
```
2. 在 IDEA 中添加运行用户,Run/Debug Configurations -> Configurations -> VM options 添加 -DHADOOP_USER_NAME=hdfs  （Edit 项目，run绿色那里 -- VM options）