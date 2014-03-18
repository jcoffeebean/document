macos 编译hadoop 2.3 64位版本
==


准备编译环境
===

安装java jdk

设置环境变量, 类似
	
```
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0_51.jdk/Contents/Home
```

安装ant 

```
brew install ant
ant -version
```

安装maven

下载maven最新版 http://maven.apache.org/download.cgi, 解压缩，安装。  

设置环境变量  
	
```
export M2_HOME=/usr/local/apache-maven
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

安装Protocol Buffers

```
sudo wget https://protobuf.googlecode.com/files/protobuf-2.5.0.tar.gz
tar -zxf protobuf-2.5.0.tar.gz
cd protobuf-2.5.0
./configure --prefix=/usr/local/protobuf
make
make install
protoc --version
```

安装ncurses

```
curl -O ftp://ftp.gnu.org/gnu/ncurses/ncurses-5.9.tar.gz
	tar -xzvf ncurses-5.9.tar.gz
	cd ./ncurses-5.9
	./configure --prefix=/usr/local \
  	--without-cxx --without-cxx-binding --without-ada --without-progs --without-curses-h \
  	--with-shared --without-debug \
  	--enable-widec --enable-const --enable-ext-colors --enable-sigwinch --enable-wgetch-events 
make
sudo make install
```

安装其他工具

```
brew install cmake
brew install openssl
brew install libtool
brew install autoconf
brew install automake
brew install pkg-config
```

编译
===

下载hadoop 2.3源代码, hadoop要在macos编译需要打补丁, 具体信息可参考https://issues.apache.org/jira/browse/HADOOP-9648

```
$wget https://issues.apache.org/jira/secure/attachment/12617363/HADOOP-9648.v2.patch
patch -p1 < HADOOP-9648.v2.patch
```

回退补丁可以执行

```
patch -RE -p1 < HADOOP-9648.v2.patch
```


在hadoop源代码根目录执行

```
mvn clean install –DskipTests
mvn package -Pdist,native -DskipTests -Dtar
```

编译后的文件`/hadoop-dist/target/hadoop-2.3.0`

编译后的native library在`lib／native`

验证编译结果

```
bin/hadoop version
```

编译maven插件  

```
cd hadoop-maven-plugins
mvn install
```

生成eclipse项目文件  

```
mvn eclipse:eclipse -DskipTests
```






