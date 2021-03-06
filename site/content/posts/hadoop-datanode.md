---
author: "Kim, Geunho"
date: 2019-07-29
title: "Hadoop DataNode"
---


# Architecture 
하둡 클러스터에서 DataNode는 파일 블록이 저장되는 노드이다. 십 수대의 서버에서 수백, 수천대의 서버에 DataNode가 기동되어 네트워크 상에서 거대한 파일 시스템의 블록 구조를 가지게 된다. 단일 디스크를 위한 파일 시스템의 블록 구조가 네트워크를 통해 확장, 추상화 되었다.

![FAT 파일 시스템 구조](/hadoop-datanode-1.png) _그림 1. FAT 파일 시스템 구조. 첫 번째 블록은 파일 할당 테이블(FAT) 정보를 담고 있다. [NameNode](/posts/hadoop-namenode/)가 하둡 파일 시스템의 전체 블록 정보를 가지고 있는 것과 유사하다._

단일 디스크 파일 시스템의 기본 블록 크기가 4KB 정도 인것에 비해,[^1] DataNode에 저장되는 블록의 크기는 64MB~128MB의 크기를 가진다. HDFS가 저장하고 처리하고자 하는 파일 하나의 크기가 작게는 수 백 메가바이트에서 크게는 테라, 페타 바이트 만큼 크기 때문이다.[^2] 
단일 디스크 파일 시스템과는 다르게, 블록 크기보다 작은 파일이 전체 블록 크기만큼의 공간을 차지하지는 않는다. 하지만 NameNode는 파일 시스템의 메타데이터를 전부 메모리에 관리하기 때문에 블록 크기 보다 작은 많은 수의 파일을 HDFS에 저장하는 것은 NameNode에 부담을 주고, 따라서 HDFS에 적합하지 않다.

HDFS의 NameNode가 여러가지 보조 장치를 통해 파일 시스템의 네임스페이스와 메타 데이터를 유실하지 않도록 발전한것 처럼, DataNode도 이러한 분산 환경에서 파일 블록이 언제든 유실되거나 접근할 수 없는 상황을 가정하고 설계되었는데, 분산된 노드에 파일 블록의 복제본을 두는 방식이다. HDFS에서는 기본적으로 세 개의 파일 블록 복제본을 서로 다른 DataNode에서 보관하도록 처리한다. 하나의 DataNode 데이터가 모두 유실되더라도 다른 DataNode에 저장된 복제본 블록에 접근해서 온전한 파일을 읽을 수 있다. 

![HDFS 블록 구조](/hadoop-datanode-2.png) _그림 2. HDFS 블록 구조. 여러 대의 DataNode가 서로 다른 파일 블록의 복제본을 저장하고 있다. 그리고 분산된 파일 블록의 구조 덕분에 단일 디스크의 용량보다 큰 파일을 다룰 수 있다._

NameNode는 DataNode가 전송[^3]하는 블록 리포트를 통해 분산 환경에 흩어져있는 블록의 메타 정보에 대한 최신 상태를 유지한다. 만약 특정 DataNode 장애로 인해 장애가 발생한 DataNode의 블록에 접근할 수 없다면 정상적인 파일 읽기/쓰기가 불가능할 것이다. 이러한 DataNode의 liveness를 알기 위해 DataNode는 블록 리포트보다 훨씬 짧은 간격으로 heartbeat을 NameNode로 보낸다.[^4]

<br />
# Read and Write
클라이언트는 HDFS의 파일을 읽거나 쓸때, 먼저 NameNode로부터 파일 블록들의 위치를 응답받은 다음, 그 정보를 가지고 DataNode로 직접 읽거나 쓰기 요청을 한다. NameNode는 파일의 IO에 직접적인 관여를 하지 않고 분산되어 있는 파일 블록들을 병행(parallel)해서 읽어들인다. 만액 어떤 파일의 읽기 요청이 많이 몰린다면, 이 파일 혹은 디렉토리에 대해 복제 계수(replication factor)를 기본 값보다 늘려서 파일 블록을 더 많은 노드에 분산시킬 수 있다. 블록이 여러 노드에 위치하기 때문에 읽기 요청에 대한 부하를 나눌 수 있는 것이다.

![HDFS Read](/hadoop-datanode-3.png) _그림 3. HDFS 파일 읽기 과정. `DistributedFileSystem`가 NameNode로 RPC 통신을 하고, `FSDataInputStream`는 파일을 읽어들인다._

HDFS에 파일을 읽고 쓰는 클라이언트는 일반적으로 하둡의 MapReduce 애플리케이션과 같이 하둡 클러스터의 워커 노드(DataNode)에서 동작하는 프로그램이다.  
하둡 2 부터는 하둡 클러스터 위에서 동작하는 애플리케이션을 [YARN](/posts/hadoop-yarn/)이 관리하는데, 이때 처리하려는 파일 블록의 위치에 따라 그 블록이 위치한 노드에서 애플리케이션이 실행될 수 있도록 자원을 할당한다.



[^1]: 파일 시스템의 종류와 설정에 따라 다르다. 디스크의 크기가 훨씬 작았던 과거에는 512B 정도로 작았다.
[^2]: 그렇기 때문에 작은 크기로 나뉘어진 여러 개의 파일을 처리하는 용도로는 부적합하다.
[^3]: DataNode가 최초 동작할때 즉시 한번 보내고, 그 후 기본 값으로 6시간에 한번씩 보낸다. [hdfs-default.xml](https://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml) 파일의 `dfs.blockreport.intervalMsec` 참고
[^4]: 기본 값은 3초이다. [hdfs-default.xml](https://hadoop.apache.org/docs/r2.9.1/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml) 파일의 `dfs.heartbeat.interval` 참고