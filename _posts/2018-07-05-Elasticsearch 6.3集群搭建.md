###准备

- 两台CentOS 6.5
  - MemTotal: 132,146,064 kB
- JDK 8.0
- ES 6.3
- 使用两台物理机，每台2个节点，搭建四节点集群

### 安装

#### JDK

- 准备jdk-8u171-linux-x64.tar.gz
- 安装目录:`opt/java/jdk1.8.0_171`
- 设置环境变量
  - `vi/etc/profile`
  - export JAVA_HOME=/opt/java/jdk1.8.0_171
    export PATH=\$PATH:\$JAVA_HOME/bin
  - `source /etc/profile`

#### Elasticsearch

- 官网下载elasticsearch-6.3.0.tar.gz并通过xshell发送至服务器

  ```shell
  cd /opt
  mkdir es
  mv elasticsearch-6.3.0.tar.gz es
  tar -zxvf elasticsearch-6.3.0.tar.gz
  ```

- 新建用户及组

  ```shell
  groupadd esgroup
  useradd esuser -g esgroup
  chown -R esuser:esgroup es/
  ```

- 配置系统参数

  ```shell
  #Ubantu 14系统下该配置无法生效
  #不修改报错：max file descriptors [65535] for elasticsearch process is too low
  vi /etc/security/limits.conf
  *       hard    nofile  655340
  *       soft    nofile  655340
  *       hard    nproc   327670
  *       soft    nproc   327670
  *       hard    core    unlimited
  *       soft    core    unlimited
  
  #修改可使用的最大线程数，否则启动失败
  vi /etc/security/limits.d/90-nproc.conf
  * soft nproc 1028
  修改为
  * soft nproc 4096
  
  vi /etc/sysctl.conf
  #末尾行添加
  vm.max_map_count=262144
  
  /sbin/sysctl -p
  ```

- 配置es参数

  ```shell
  #es必需使用非root权限用户启动
  su esuser
  cd /opt/es/elasticsearch-6.3.0/config
  vi elasticsearch.yml
  
  #注意,yml文件下格式要求 ：后面一定要空一格，否则无法加载
  #集群名称，同一个集群下的所有节点需保持名称一致
  cluster.name: es
  #当前节点的名称
  node.name: node1
  #是否有资格成为master节点,默认为true，通常master节点不负责存储数据,data节点只存储数据,client节点只负责处理用户请求，实现转发，负载均衡,client节点的node.master,node.data均为false
  node.master: true
  #是否为数据节点，默认为true
  node.data: true
  #数据存放目录，多个目录用,分隔
  path.data: /opt/es/es_data/data1
  #日志存放目录
  path.logs: /opt/es/es_log/log1
  #绑定的IP
  network.host: 192.168.21.204
  #客户端通过HTTP访问的端口号设置
  http.port: 9200
  #集群间通过tcp传递端口号
  transport.tcp.port: 9301
  #组建集群的所有地址列表，可以通过这些节点来自动发现新加入集群的节点，IP地址或者主机名
  discovery.zen.ping.unicast.hosts:["192.168.21.204:9301","192.168.21.204:9302","135.251.209.27:9301","135.251.209.27:9302"]
  #待研究
  cluster.routing.allocation.same_shard.host: true
  #为了避免脑裂，该值最少为（有资格进行master选举的节点数n/2+1），所以说该值应该根据你设置的vnode.master: true的节点数来确定，假设你设置的有资格选举为master的节点为1，该值设2则会报错not enough master nodes discovered during pinging
  discovery.zen.minimum_master_nodes: 2    
  
  #es选举master节点的超时时间，可使用默认值，不做添加
  discovery.zen.ping_timeout: 3s
  
  #CentoOS 6.5需要如下配置，否则会报内核版本过低的错误
  bootstrap.memory_lock: false
  bootstrap.system_call_filter: false
  
  #下面配置jvm.options
  vi jvm.options
  #JVM堆空间设置25g
  -Xms25g
  -Xmx25g
  # -XX:+HeapDumpOnOutOfMemoryError  # 注释掉这一行
  # 修改heapDump日志路径至足够大的目录
  -XX:HeapDumpPath=/data/HeapDumpPath
  ```

- 启动

  ```shell
  cd /opt/es/elasticsearch-6.3.0
  ./bin/elasticsearch
  ```

- 拷贝当前配置完成的节点，启动第二个节点

  ```shell
  cd /opt/es
  cp -r elasticsearch-6.3.0 elasticsearch2-6.3.0
  
  cd elasticsearch2-6.3.0/config
  vi elasticsearch.yml
  
  #由于是拷贝自之前的配置，需要修改部分参数，没有列出的参数不用修改
  node.name: node2
  node.master: false
  path.data: /opt/es/es_data/data2
  path.logs: /opt/es_es_log/log2
  network.host: 192.168.21.204
  http.port: 9201
  transport.tcp.port: 9302
  
  cd ..
  ./bin/elasticsearch
  ```

- 另一台物理机操作同上

  - 其中一个节点配置文件中`node.master: false`
  - 另一个节点配置文件中注释掉`node.master: false`,因为如上参数中为避免脑裂最少master数设置为2

- 第二台物理机在启动时报错`failed to obtain node locks`

  - 删除第一台物理机master节点所在安装目录下的**data**文件夹

- 查看当前es集群节点信息

  ```shell
  curl -XGET 'http://192.168.21.204:9200/_cat/nodes?pretty'
  135.251.209.27 6 56 2 0.39 0.33 0.21 di  - node-4
  192.168.21.204 5 59 1 0.62 0.72 0.42 di  - host-2
  135.251.209.27 8 56 2 0.39 0.33 0.21 mdi * node-3 #*表示master节点
  192.168.21.204 3 59 1 0.62 0.72 0.42 mdi - host-1
  
  #查看更多集群相关信息
  curl -XGET 'http://172.18.68.11:9200/_cat?pretty'
  ```



### 一些重要的参数配置

通过jvm.options文件进行设置，设置的值依赖于机器的内存大小，下面是一些设置规则

- 堆的最大值和最小值保持一致

- 为Es配置的堆值越大，用于缓存的内存越多，同时也会导致JVM进行GC的时间过长

- Xmx不要超过物理内存的50%

- 不要把Xmx的值设置超过JVM可以进行指针压缩的上限—32G，如下

  `heap size [34.8gb], compressed ordinary object pointers [false]`

  `heap size [29.8gb], compressed ordinary object pointers [true]`

- 最好能够保证Xmx位于JVM的指针零压缩阈值下，通常为26G或者30G,可以通过增加启动参数`-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompressedOopsMode `通过打印的日志判断，经过测试，基于以上硬件和配置可以设置为30G,当设置为31G时达到非零压缩模式：`heap address: 0x00007f083c000000, size: 31744 MB, Compressed Oops mode: Non-zero based:0x00007f083bfff000, Oop shift amount: 3`



- `XX:HeapDumpPath=`用于配置存放heap dump的目录，如果这里设置的是一个文件名，JVM会重复利用该文件从而避免heap dumps聚集过多

