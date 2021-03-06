### Raspberry Pi에 OpenTSDB 설치와 분산환경 구성 안내

##### 설치 환경
* 하드웨어: Raspberry Pi 1 Model B+
* 운영체제: Raspbian Jessie Lite, 2016-03-18 release
* 사용 소프트웨어 버전
  - Oracle JDK 1.7
  - GnuPlot 4.6
  - Hadoop 1.2.1
  - HBase 0.94.27
  - OpenTSDB 2.2.0

##### 호스트 구성
* 호스트이름과 IP 주소 구성
  - server01 : 192.168.0.211
  - server02 : 192.168.0.212
  - server03 : 192.168.0.213
* 호스트별 서버 구성
  - server01 : (Master) Hadoop NameNode/SecondaryNameNode/DataNode, MapReduce JobTracker/TaskTracker, HBase Master/RegionServer, OpenTSDB
  - server02 : (Slave) Hadoop DataNode, MapReduce TaskTracker, HBase RegionServer
  - server03 : (Slave) Hadoop DataNode, MapReduce TaskTracker, HBase RegionServer

##### JDK(Java Development Kit) 설치하기
Hadoop, HBase, ZooKeeper, OpenTSDB 등 서버들이 모두 Java 기반으로 개발되어 있어서 실행할 때 JDK(혹은 JRE)가 필요하다. Open JDK는 Oracle JDK보다 성능이 떨어지고 설치할 프로그램들과 알 수 없는 버그를 만들기때문에 Oracle JDK를 사용한다.

1.JDK 설치 여부를 확인하고, Open JDK가 설치되어 있다면 제거한다.
```sh
java version "1.6.0_38"
OpenJDK Runtime Environment (IcedTea6 1.13.10) (6b38-1.13.10-1~deb7u1+rpi1)
OpenJDK Zero VM (build 23.25-b01, mixed mode)
$ sudo apt-get purge openjdk*
$ sudo apt-get autoremove
```

2.Oracle JDK가 설치되어 있지 않다면 설치한다.
```sh
$ sudo apt-get update
$ sudo apt-get install oracle-java7-jdk
```

3.JDK 버전을 확인한다.
```sh
$ java -version
java version "1.7.0_60"
Java(TM) SE Runtime Environment (build 1.7.0_60-b19)
Java HotSpot(TM) Client VM (build 24.60-b09, mixed mode)
```

4.편의를 위해서 Oracle JDK 디렉토리를 'default-java'로 링크한다.
```sh
$ sudo ln -s /usr/lib/jvm/jdk-7-oracle-arm-vfp-hflt /usr/lib/jvm/default-java
```

##### (공통) Hadoop 설치와 환경설정

* 아래 링크의 문서를 따라서 Hadoop을 설치한다.
  - Raspberry Pi에 Hadoop 설치하기
    - https://github.com/zerover0/projects/blob/master/hadoop/hadoop_install_on_rpi.md

##### (master) GnuPlot 설치하기

OpenTSDB Web UI 사이트에서 그래프를 그릴 때 사용된다.

1.패키지 설치 명령으로 GnuPlot을 설치한다.
```sh
$ sudo apt-get update 
$ sudo apt-get install gnuplot 
```

##### (master) HBase 설치와 환경설정

1.HBase 홈페이지에서 0.94.27 릴리즈 파일을 다운로드한다.
  - http://hbase.apache.org

2.다운로드한 파일을 hadoop 홈디렉토리 아래에 있는 'app' 디렉토리에 압축을 풀고, 생성된 디렉토리의 이름을 'hbase'로 바꾼다.
```sh
$ tar xzf hbase-0.94.27.tar.gz -C ~/app/
$ mv ~/app/hbase-0.94.27 ~/app/hbase
```

3.'regionservers' 파일을 열어서 RegionServer가 실행될 호스트이름을 한줄에 하나씩 입력하고 저장한다.
```sh
$ vi ~/app/hbase/conf/regionservers
server01
server02
server03
```

4.Hadoop HDFS 설정을 참조하기 위해서 'hdfs-site.xml' 파일을 Hadoop 디렉토리에서 HBase 디렉토리로 복사한다.
```sh
$ cp ~/app/hadoop/conf/hdfs-site.xml ~/app/hbase/conf/
```

5.'hbase-env.sh' 파일을 열어서 'JAVA_HOME', 'HBASE_CLASSPATH', 'HBASE_HEAPSIZE', 'HBASE_LOG_DIR', 'HBASE_MANAGES_ZK'을 찾아서 수정한 후 저장한다.
  - JAVA_HOME : JDK 홈디렉토리
  - HBASE_CLASSPATH : HADOOP_CONF_DIR을 가리키도록 설정($HADOOP_CONF_DIR/hadoop-env.sh 이용)
  - HBASE_HEAPSIZE : Raspberry Pi의 메모리가 작으므로 heap 용량을 제한한다.
  - HBASE_LOG_DIR : HBase server 로그 저장 위치
  - HBASE_MANAGES_ZK : HBase가 내장 ZooKeeper를 이용할 지 여부 설정
```sh
$ vi ~/app/hbase/conf/hbase-env.sh
export JAVA_HOME=/usr/lib/jvm/default-java
export HBASE_CLASSPATH=${HBASE_CLASSPATH}:${HADOOP_HOME}/conf
export HBASE_HEAPSIZE=250
export HBASE_LOG_DIR=/home/hadoop/data/hbase/logs
export HBASE_MANAGES_ZK=true
```

6.'hbase-site.xml' 파일을 열어서 아래 내용을 추가한다.
  - hbase.rootdir : HDFS의 HBase 루트 디렉토리 URI
  - hbase.cluster.distributed : true = 분산환경에서 동작을 의미
  - hbase.zookeeper.quorum : ZooKeeper 프로세스가 실행 중인 호스트이름 목록
  - hbase.zookeeper.property.dataDir : ZooKeepr 데이터를 저장할 로컬 디렉토리
```sh
$ vi ~/app/hbase/conf/hbase-site.xml
<configuration>
  <property>
    <name>hbase.rootdir</name>
    <value>hdfs://server01:9000/hbase</value>
  </property>
  <property>
    <name>hbase.cluster.distributed</name>
    <value>true</value>
  </property>
  <property>
    <name>hbase.zookeeper.quorum</name>
    <value>server01,server02,server03</value>
  </property>
  <property>
    <name>hbase.zookeeper.property.dataDir</name>
    <value>/home/hadoop/data/zookeeper</value>
  </property>
</configuration>
```

7.HBase Master 프로세스가 실행될 때 ZooKeeper 서버를 실행되는데(HBASE_MANAGES_ZK=true 설정에 의해서), Raspberry Pi의 성능문제로 ZooKeepr 서버가 실행되는 시간을 잠시 기다린 후에 HBase Master 프로세스를 실행하도록 'start-hbase.sh' 파일을 수정한다. 아래와 같이 ZooKeepr와 Master 프로세스를 실행하는 코드에 'sleep'을 추가해서 지연시간을 둔다.
```sh
$ vi ~/app/hbase/bin/start-hbase.sh
  "$bin"/hbase-daemons.sh --config "${HBASE_CONF_DIR}" start zookeeper
  echo "Waiting for ZooKeeper ready..."
  sleep 10
  "$bin"/hbase-daemon.sh --config "${HBASE_CONF_DIR}" start master
  echo "Waiting for HBase Master ready..."
  sleep 10
```

8.master 노드에 있는 HBase 디렉토리 전체를 slave 노드로 복사한다. 원격으로 slave 노드의 프로세스를 실행하려면 master 노드와 디렉토리 구조가 동일해야한다.
```sh
$ scp -r ~/app/hbase hadoop@server02:~/app/
$ scp -r ~/app/hbase hadoop@server03:~/app/
```

9.HBase 프로세스를 실행한다.
```sh
$ /home/hadoop/app/hbase/bin/start-hbase.sh
```

10.HBase 프로세스가 작동하는지 확인한다.
```sh
server01:~$ jps
6432 TaskTracker
6070 DataNode
21987 HQuorumPeer
6183 SecondaryNameNode
22153 HRegionServer
5959 NameNode
6327 JobTracker
22054 HMaster
```
```sh
server02:~$ jps
11594 HRegionServer
2786 TaskTracker
11522 HQuorumPeer
2691 DataNode
```

11.HBase 모니터링 사이트를 열어서 동작 상태를 확인한다.
  - HBase Master Web UI : http://server01:60010
  - HBase RegionServer Web UI : http://server01:60030
  - HBase RegionServer Web UI : http://server02:60030
  - HBase RegionServer Web UI : http://server03:60030

12.HBase 프로세스를 종료하려면, 아래 명령을 실행한다.
```sh
$ /home/hadoop/app/hbase/bin/stop-hbase.sh
```

##### (master) OpenTSDB 설치와 환경설정

1.다음 주소에서 OpenTSDB의 릴리즈 파일을 다운로드한다.
  - https://github.com/OpenTSDB/opentsdb/releases

2.다운로드한 파일의 압축을 푼 후, 프로그램을 빌드한다.
```sh
$ tar xzf opentsdb-2.2.0.tar.gz -C ~/app
$ mv ~/app/opentsdb-2.2.0 ~/app/opentsdb
$ cd ~/app/opentsdb
$ ./build.sh
```

3.환경설정파일 'opentsdb.conf'을 수정한다.
  - tsd.network.port = TSD 연결 포트
  - tsd.http.staticroot = OpenTSDB 홈페이지 파일 위치
  - tsd.http.cachedir = TSD 임시 파일 저장 위치
  - tsd.core.auto_create_metrics : true = 레코드의 metric이 데이터베이스에 존재하지 않을 때, 자동으로 metric 추가
  - tsd.storage.fix_duplicates : true = 같은 시간에 중복된 데이터가 존재하는 경우 마지막 입력된 데이터만 쓰임
```sh
$ sudo vi ~/app/opentsdb/src/opentsdb.conf
tsd.network.port = 4242
tsd.http.staticroot = /home/hadoop/app/opentsdb/build/staticroot
tsd.http.cachedir = /home/hadoop/data/opentsdb
tsd.core.auto_create_metrics = true
tsd.storage.fix_duplicates = true
```

4.로그설정파일 'logback.xml'을 수정한다. 로그파일이 저장되는 위치를 바꾸어준다. root 권한으로 OpenTSDB를 실행한다면 기본값을 사용해도 문제없다.
```sh
$ sudo vi ~/app/opentsdb/src/logback.xml
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/home/hadoop/data/opentsdb/opentsdb.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
      <fileNamePattern>/home/hadoop/data/opentsdb/opentsdb.log.%i</fileNamePattern>
  <appender name="QUERY_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/home/hadoop/data/opentsdb/queries.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
      <fileNamePattern>/home/hadoop/data/opentsdb/queries.log.%i</fileNamePattern>
```

5.OpenTSDB를 설치한 후, 최초로 한번 데이터베이스 테이블을 구성하는 명령을 실행한다.
```sh
$ export JAVA_HOME=/usr/lib/jvm/default-java
$ export HBASE_HOME=/home/hadoop/app/hbase
$ export COMPRESSION=NONE 
$ ~/app/opentsdb/src/create_table.sh
2016-04-15 11:24:19,339 WARN  [main] util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 0.94.27, r14c0e77956f9bb4c6edf0378474264843e4a82c3, Wed Mar 16 21:18:26 PDT 2016

create 'tsdb-uid',
  {NAME => 'id', COMPRESSION => 'NONE', BLOOMFILTER => 'ROW'},
  {NAME => 'name', COMPRESSION => 'NONE', BLOOMFILTER => 'ROW'}
0 row(s) in 2.3620 seconds

Hbase::Table - tsdb-uid

create 'tsdb',
  {NAME => 't', VERSIONS => 1, COMPRESSION => 'NONE', BLOOMFILTER => 'ROW'}
0 row(s) in 1.3680 seconds

Hbase::Table - tsdb
  
create 'tsdb-tree',
  {NAME => 't', VERSIONS => 1, COMPRESSION => 'NONE', BLOOMFILTER => 'ROW'}
0 row(s) in 1.3270 seconds

Hbase::Table - tsdb-tree
  
create 'tsdb-meta',
  {NAME => 'name', COMPRESSION => 'NONE', BLOOMFILTER => 'ROW'}
0 row(s) in 1.3340 seconds

Hbase::Table - tsdb-meta
```

6.TSD 데몬을 실행한다. TSD 데몬을 실행할 때 프로세스 ID를 파일로 저장해두면 TSD 데몬을 종료할 때 이용할 수 있다.
```sh
$ /home/hadoop/app/opentsdb/build/tsdb tsd --config=/home/hadoop/app/opentsdb/src/opentsdb.conf&
$ echo $! > /home/hadoop/data/opentsdb/opentsdb.pid
```

7.OpenTSDB 서비스가 정상적으로 작동하는지 OpenTSDB Web UI 사이트를 통해서 확인한다.
  - 호스트 주소가 'server01'인 경우 : http://server01:4242

8.TSD 데몬을 종료할 때는 아래의 명령을 실행한다.
```sh
$ kill -HUP `cat /home/hadoop/data/opentsdb/opentsdb.pid`
```
