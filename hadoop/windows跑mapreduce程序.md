#### 下载hadoop-2.6.0.tar.gz，解压

#### 配置环境变量
- HADOOP_HOME：D:\WangHao\Tools\hadoop-2.6.0
- %HADOOP_HOME%\bin
- %HADOOP_HOME%\sbin


### 解决报错问题
#### 替换hadoop-2.6.0的bin目录，要包含winutils.exe文件的

#### 将hadool.dll和winutils.exe文件放到C:\Windows\System32下

#### 下载hadoop-2.6.0-src.tar.gz，解压
- hadoop-2.6.0-src\hadoop-common-project\hadoop-common\src\main\java\org\apache\hadoop\io\nativeio下NativeIO.java 复制到对应的idea-project-src-main-java下要创建org.apache.hadoop.io.nativeio包下面。
- 更改557行的return true

##### 问题解答
https://blog.csdn.net/congcong68/article/details/42043093

##### hadoop相关下载 
http://archive.apache.org/dist/hadoop/core/hadoop-2.6.0/
##### 替换hadoop-2.6.0的bin目录 
https://github.com/steveloughran/winutils