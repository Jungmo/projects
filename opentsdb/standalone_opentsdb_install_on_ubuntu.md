### Ubuntu에 OpenTSDB 설치와 단독실행형(Standalone) 구성 안내

##### 설치 환경
* 운영체제: Ubuntu 14.04, x86 64-bit
* 사용 소프트웨어 버전
  - Oracle JDK 1.7
  - GnuPlot 4.6
  - HBase 0.94.27
  - OpenTSDB 2.2.0

##### JDK(Java Development Kit) 설치하기
HBase, OpenTSDB 서버는 모두 Java 기반으로 개발되어 있어서 실행할 때 JDK(혹은 JRE)가 필요합니다. Open JDK는 Oracle JDK보다 성능도 나쁘고 설치할 프로그램들과 여러가지 버그를 만들기때문에 Oracle JDK를 사용합니다.

1.JDK 설치 여부를 확인합니다.
```sh
$ java -version
java version "1.7.0_95"
OpenJDK Runtime Environment (IcedTea 2.6.4) (7u95-2.6.4-0ubuntu0.14.04.2)
OpenJDK 64-Bit Server VM (build 24.95-b01, mixed mode)
```

2.OpenJDK가 이미 설치되어 있다면, 제거합니다. JDK가 설치되어 있지 않다면, PPA repository를 추가하고 Oracle JDK를 설치합니다.
  - Open JDK 제거
```sh
$ sudo apt-get purge openjdk*
```
  - Oracle JDK 설치
```sh
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo apt-get update
$ sudo apt-get install oracle-java7-installer
```

3.JDK 버전을 확인합니다.
```sh
$ java -version
java version "1.7.0_80"
Java(TM) SE Runtime Environment (build 1.7.0_80-b15)
Java HotSpot(TM) 64-Bit Server VM (build 24.80-b11, mixed mode)
```

4.편의를 위해서 Oracle JDK를 'default-java'로 링크합니다.
```sh
$ sudo ln -s /usr/lib/jvm/java-7-oracle /usr/lib/jvm/default-java
```

##### GnuPlot 설치하기
OpenTSDB에서 그래프 그릴 때 사용됩니다.

1.패키지 설치 명령으로 GnuPlot을 설치합니다.
```sh
$ sudo apt-get update 
$ sudo apt-get install gnuplot 
```

##### localhost 주소 변경하기
Linux에서는 '/etc/hosts' 파일에 localhost의 주소로 '127.0.0.1' 뿐만아니라 '127.0.1.1'도 사용하는데, 이것을 '127.0.0.1'로 바꾸어줍니다. '127.0.1.1'을 사용하면 HBase에서 오류가 나는 경우가 있어서 수정이 필요합니다.

1.'/etc/hosts' 파일에서 '127.0.1.1'을 찾아서 '127.0.0.1'로 수정합니다.
```
$ sudo vi /etc/hosts
127.0.0.1       server01
```

##### 프로그램 설치와 데이터 저장을 위한 디렉토리 생성
서버 프로그램을 사용자 홈디렉토리에 설치하고 데이터 저장 디렉토리를 사용자 홈디렉토리로 지정하면 root 권한없이도 프로그램을 실행할 수 있어서 관리가 편리합니다.

1.프로그램 설치와 데이터 저장을 위한 디렉토리를 사용자 홈디렉토리('/home/tinyos')에 생성합니다.
```sh
$ mkdir /home/tinyos/app
$ mkdir /home/tinyos/data
```

##### HBase 단독실행형으로 설치하기
HBase를 단독실행형으로 설치하게 되면, HDFS 서버와 ZooKeeper 서버가 별도로 필요하지 않으며 하나의 호스트에서 실행됩니다.

1.HBase 홈페이지에서 0.94.27 릴리즈 파일을 다운로드한다.
  - http://hbase.apache.org

2.다운로드한 파일을 hadoop 홈디렉토리 아래에 있는 'app' 디렉토리에 압축을 풀고, 생성된 디렉토리의 이름을 'hbase'로 바꾼다.
```sh
$ tar xzf hbase-0.94.27.tar.gz -C ~/app/
$ mv ~/app/hbase-0.94.27 ~/app/hbase
```

3.'hbase-env.sh' 파일을 열어서 'JAVA_HOME', 'HBASE_CLASSPATH', 'HBASE_HEAPSIZE', 'HBASE_LOG_DIR', 'HBASE_MANAGES_ZK'을 찾아서 수정한 후 저장한다.
  - JAVA_HOME : Java 홈디렉토리
  - HBASE_LOG_DIR : HBase server 로그 저장 위치
  - HBASE_MANAGES_ZK : HBase가 내장 ZooKeeper를 이용할 지 여부 설정
```sh
$ vi ~/app/hbase/conf/hbase-env.sh
export JAVA_HOME=/usr/lib/jvm/default-java
export HBASE_LOG_DIR=/home/tinyos/data/hbase/logs
```

4.'hbase-site.xml' 파일에 HBase rootdir 설정과 ZooKeeper 데이터 디렉토리를 설정합니다.
다음 설정은 사용자 홈디렉토리를 데이터 디렉토리로 이용하는 HBase 설정의 예입니다.
```sh
$ vi ~/app/hbase/conf/hbase-site.xml
<configuration> 
  <property> 
    <name>hbase.rootdir</name> 
    <value>file:///home/tinyos/data/hbase</value> 
  </property> 
  <property> 
    <name>hbase.zookeeper.property.dataDir</name> 
    <value>/home/tinyos/data/zookeeper</value> 
  </property> 
</configuration> 
```

5.다음 명령을 실행해서 HBase 서버를 실행합니다. HBase 서버를 실행할 때, Raspberry Pi의 성능문제로 초기화에 상당한 시간이 소요되므로 약 1분 정도 기다렸다가 이후 과정을 진행합니다.
```sh
$ /home/tinyos/app/hbase/bin/start-hbase.sh
```

6.HMaster 프로세스가 실행되었는지 확인합니다.
```sh
$ jps
1150 HMaster
```

7.HBase shell을 이용해 HBase에 연결되는지 확인합니다. 시스템 성능에 따라서 다소 시간이 걸리므로 메시지가 나올 때까지 계속 기다립니다.
```sh
$ /home/tinyos/app/hbase/bin/hbase shell
2016-04-15 20:45:40,800 WARN  [main] util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
HBase Shell; enter 'help<RETURN>' for list of supported commands.
Type "exit<RETURN>" to leave the HBase Shell
Version 0.94.27, r14c0e77956f9bb4c6edf0378474264843e4a82c3, Wed Mar 16 21:18:26 PDT 2016

hbase(main):001:0> 
```

8.HBase 서버를 종료할 때는 아래 명령을 실행합니다.
```sh
$ /home/tinyos/app/hbase/bin/stop-hbase.sh
```

##### OpenTSDB 설치하기

1.다음 주소에서 OpenTSDB의 릴리즈 파일을 다운로드한다.
  - https://github.com/OpenTSDB/opentsdb/releases

2.다운로드한 파일의 압축을 푼 후, 프로그램을 빌드한다.
```sh
$ tar xzf opentsdb-2.2.0.tar.gz -C ~/app
$ mv ~/app/opentsdb-2.2.0 ~/app/opentsdb
$ cd ~/app/opentsdb
$ ./build.sh
```
**javacc error 발생시**
```sh
cd opentsdb-2.3.0

mkdir build

cp -r third_party ./build

./build.sh
```sh

3.환경설정파일 'opentsdb.conf'을 수정한다.
  - tsd.network.port = TSD 연결 포트
  - tsd.http.staticroot = OpenTSDB 홈페이지 파일 위치
  - tsd.http.cachedir = TSD 임시 파일 저장 위치
  - tsd.core.auto_create_metrics : true = 레코드의 metric이 데이터베이스에 존재하지 않을 때, 자동으로 metric 추가
  - tsd.storage.fix_duplicates : true = 같은 시간에 중복된 데이터가 존재하는 경우 마지막 입력된 데이터만 쓰임
```sh
$ sudo vi ~/app/opentsdb/src/opentsdb.conf
tsd.network.port = 4242
tsd.http.staticroot = /home/tinyos/app/opentsdb/build/staticroot
tsd.http.cachedir = /home/tinyos/data/opentsdb
tsd.core.auto_create_metrics = true
tsd.storage.fix_duplicates = true
```

4.로그설정파일 'logback.xml'을 수정한다. 로그파일이 저장되는 위치를 바꾸어준다. root 권한으로 OpenTSDB를 실행한다면 기본값을 사용해도 문제없다.
```sh
$ sudo vi ~/app/opentsdb/src/logback.xml
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/home/tinyos/data/opentsdb/opentsdb.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
      <fileNamePattern>/home/tinyos/data/opentsdb/opentsdb.log.%i</fileNamePattern>
  <appender name="QUERY_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/home/tinyos/data/opentsdb/queries.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
      <fileNamePattern>/home/tinyos/data/opentsdb/queries.log.%i</fileNamePattern>
```

5.OpenTSDB를 설치한 후, 최초로 한번 데이터베이스 테이블을 구성하는 명령을 실행한다.
```sh
$ export JAVA_HOME=/usr/lib/jvm/default-java
$ export HBASE_HOME=/home/tinyostinyos/app/hbase
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
$ /home/tinyos/app/opentsdb/build/tsdb tsd --config=/home/tinyos/app/opentsdb/src/opentsdb.conf&
$ echo $! > /home/tinyos/data/opentsdb/opentsdb.pid
```

7.OpenTSDB 서비스가 정상적으로 작동하는지 OpenTSDB Web UI 사이트를 통해서 확인한다.
  - 호스트 주소가 'server01'인 경우 : http://server01:4242

8.TSD 데몬을 종료할 때는 아래의 명령을 실행한다.
```sh
$ kill -HUP `cat /home/tinyos/data/opentsdb/opentsdb.pid`
```

##### Grafana에서 OpenTSDB lookup API 사용을 위한 설정

1.Grafana에서 템플릿을 만들 때 변수가 자동으로 나타나도록 하려면 'opentsdb.conf'에 아래 설정을 추가하여 활성화해야 합니다.
  - tsd.core.meta.enable_realtime_ts = true

2.OpenTSDB에 있는 time series 데이터의 메타데이터를 나타나도록 하려면 아래 명령을 OpenTSDB 서버가 실행 중인 곳에서 실행합니다.
```sh
$ /home/tinyos/app/opentsdb/build/tsdb uid metasync
```
