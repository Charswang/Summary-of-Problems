2020-12-5
- 配置的Hadoop版本为2.6.0
- Scala使用的2.11.8  下载地址：https://www.scala-lang.org/download/2.11.8.html
- Spark使用的2.4.7  下载地址：http://spark.apache.org/downloads.html

<font color="red">记得将Scala和Spark的安装配置，不仅要在主节点进行配置，从节点也要配置。就是说要配置三遍</font>

#### Scala安装
- 将下载好的Scala安装包
- 使用sudo rz命令传入到集群中相应的文件夹
- 解压scala安装包：

```
sudo tar -zxvf scala-2.11.8.tgz
```

- 配置环境变量：
```
sudo vi /etc/profile

//添加以下代码
export SCALA_HOME=/home/had_user/local/scala-2.11.8

//将Path更改为
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$SCALA_HOME/bin:$HIVE_HOME/bin
```

- 更新配置
```
source /etc/profile
```

- 验证scala是否安装成功
```
//查看版本号
scala -version
```

#### Spark安装配置

- 将下载好的Spark安装包
- 使用sudo rz命令传入到集群中相应的文件夹
- 解压Spark安装包：
```
sudo tar -zxvf spark-2.4.7-bin-hadoop2.6.tgz
```

- 配置环境变量：
```
sudo vi /etc/profile

//添加以下代码
export SPARK_HOME=/home/had_user/local/spark-2.4.7-bin-hadoop2.6

//更改Path
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$SCALA_HOME/bin:$SPARK_HOME/bin:$HIVE_HOME/bin
```

- 更新配置
```
source /etc/profile
```

- 配置spark-env.sh文件
```
//进入spark安装文件加下的conf目录
/home/had_user/local/spark-2.4.7-bin-hadoop2.6/conf

// 复制spark-env.sh
cp spark-env.sh.template spark-env.sh

// 修改spark-env.sh
// 添加以下代码
export SCALA_HOME=/home/had_user/local/scala-2.11.8
export JAVA_HOME=/home/had_user/local/jdk1.8.0_11
export HADOOP_HOME=/home/had_user/local/hadoop-2.6.0
export SPARK_WORKER_MEMORY=1g
export HADOOP_CONF_DIR=/home/had_user/local/hadoop-2.6.0/etc/hadoop
```

- 配置slaves
```
//进入spark安装文件加下的conf目录
/home/had_user/local/spark-2.4.7-bin-hadoop2.6/conf

// 复制slaves
cp slaves.template slaves

// 修改slaves文件
将localhost注释掉
// 然后添加主从节点的主机名称
master
slave1
slave2
```

- 启动Spark
```
//先启动hadoop  --  start-all.sh

//在进入到spark安装目录下的sbin下找到start-all.sh命令，进行启动
1、进入到/home/had_user/local/spark-2.4.7-bin-hadoop2.6/sbin

2、执行/home/had_user/local/spark-2.4.7-bin-hadoop2.6/sbin/start-all.sh进行启动

3、使用jps命令看是否有Master和Worker进程，有的话则启动成功
```

- Spark WebUI页面
```
// 8080：master的webUI，sparkwebUI的端口
http://192.168.246.130:8080/  || http://master:8080/

// 8081：worker的webUI的端口
http://192.168.246.130:8081/  || http://master:8081/

// 4040：启动Spark-Shell之后可以进入该页面， application的webUI的端口
// spark-shell命令在spark安装目录下的bin文件夹下进行启动
http://192.168.246.130:4040/  || http://master:4040/
```