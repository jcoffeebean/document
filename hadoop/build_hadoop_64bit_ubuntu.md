ubuntu编译hadoop 2.3 64位 native code
==

官网下载的hadoop是32位版本，需要自己编译64位版本。  


准备编译环境
===

安装编译工具  

```
sudo apt-get install build-essential
sudo apt-get install openssl 
sudo apt-get install libssl-dev
sudo apt-get install zlib1g-dev
sudo apt-get install libtool
sudo apt-get install autoconf automake
sudo apt-get install cmake
sudo apt-get install libncurses5-dev
sudo apt-get install pkg-config
```

安装protobuf  

```
sudo wget https://protobuf.googlecode.com/files/protobuf-2.5.0.tar.gz
tar -zxf protobuf-2.5.0.tar.gz
cd protobuf-2.5.0
./configure --prefix=/usr/local/protobuf
make
make install

export PATH=/usr/local/protobuf/bin:$PATH
protoc --version
```

安装jdk   

安装jdk, 设置java环境变量  

```
export JAVA_HOME=/usr/jvm/jdk1.7.0_51
export CLASSPATH=".:$JAVA_HOME/lib:$JAVA_HOME/jre/lib$CLASSPATH"
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH
```

验证  

```
java -version
javac -version
```

安装ant

```
apt-get install ant
```

安装maven  

下载maven最新版 http://maven.apache.org/download.cgi, 解压缩，安装。  

设置maven环境变量

```
export M2_HOME=/usr/local/apache-maven/apache-maven-3.2.1
export M2=$M2_HOME/bin
export PATH=$M2:$PATH
```
验证 

```
mvn --version
```


修改配置文件conf/setting.xml, 增加国内的源。具体可参考http://maven.oschina.net/help.html  

`<mirrors>`配置节增加  

```
<mirror>
  	<id>nexus-osc</id>
  	<mirrorOf>*</mirrorOf>
		<name>Nexusosc</name>
		<url>http://maven.oschina.net/content/groups/public/</url>
</mirror>
```

`<profiles>`配置节增加

```
<profile>
   <id>jdk-1.7</id>
   <activation>
     <jdk>1.7</jdk>
   </activation>
   <repositories>
     <repository>
       <id>nexus</id>
       <name>local private nexus</name>
       <url>http://maven.oschina.net/content/groups/public/</url>
       <releases>
         <enabled>true</enabled>
       </releases>
       <snapshots>
         <enabled>false</enabled>
       </snapshots>
     </repository>
   </repositories>
   <pluginRepositories>
     <pluginRepository>
       <id>nexus</id>
      <name>local private nexus</name>
       <url>http://maven.oschina.net/content/groups/public/</url>
       <releases>
         <enabled>true</enabled>
       </releases>
       <snapshots>
         <enabled>false</enabled>
       </snapshots>
     </pluginRepository>
   </pluginRepositories>
 </profile>
```

编译hadoop
===

切换到hadoop源代码目录`hadoop-2.3.0-src`

用maven编译

```
mvn clean install –DskipTests
mvn compile -Pdist,native -Dskiptests -Dtar 
```

编译后的文件`/hadoop-dist/target/hadoop-2.3.0`。 

验证编译结果

```
bin/hadoop version
```

编译后的native library在`lib／native`, 把native目录下的文件复制到hadoop安装目录的对应子目录。  


编译真是浪费时间。  









