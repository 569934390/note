
编译:
http://blog.csdn.net/wusihang9/article/details/77619735
集群模式安装:
机器信息（郴州双线）
113.219.128.100               42.49.186.206 （同时也是zk节点）
113.219.128.103               42.49.186.209 （同时也是zk节点）
113.219.128.106               42.49.186.212 （同时也是zk节点）
113.219.128.107               42.49.186.213
4台独立安装clickhouse，并且选了3台安装zk3.4.5+
相关配置
每台机子上修改  /etc/clickhouse-server/config.xml     增加配置项 <include_from>/etc/clickhouse-server/metrika.xml</include_from>   （当然也可以不增加，如果不增加那么默认配置文件需要放置在/etc/目录下）
添加配置后，在/etc/clickhouse-server/目录下新建metrika.xml
其中一台配置内容如下：
<yandex>
<clickhouse_remote_servers>
   <!---测试集群，副本由分布式表管理->
    <test_cluster>
        <shard>
            <replica>
                <host>113.219.128.100</host>
                <port>19008</port>
            </replica>
            <replica>
                <host>113.219.128.103</host>
                <port>19008</port>
            </replica>
        </shard>
        <shard>
            <replica>
                <host>113.219.128.106</host>
                <port>19008</port>
            </replica>
            <replica>
                <host>113.219.128.107</host>
                <port>19008</port>
            </replica>      
    </shard>
    </test_cluster>
    <!--测试带副本的表的集群，副本有副本表自己管理，集群本身不参与管理副本-->
    <cluster_for_replicated_table>
        <shard>
         <!--该选项的含义是，只要找到底下声明的任意一个replica，那么就只往这个节点写入数据，另外一个节点不写，另外一个节点的副本有副本表自身管理-->
         <!--该选项不声明时默认为false，表示对于底下声明的每一个replica，集群都需要写一份数据-->
        <internal_replication>true</internal_replication>
            <replica>
                <host>113.219.128.100</host>
                <port>19008</port>
            </replica>
            <replica>
                <host>113.219.128.103</host>
                <port>19008</port>
            </replica>
        </shard>
        <shard>
        <internal_replication>true</internal_replication>
            <replica>
                <host>113.219.128.106</host>
                <port>19008</port>
            </replica>
            <replica>
                <host>113.219.128.107</host>
                <port>19008</port>
            </replica>
        </shard>
    </cluster_for_replicated_table>
</clickhouse_remote_servers>
<!--这个地方的配置四个节点都不一样，这个用于带副本的表的建立，命名规则任意，此处的命名是shard表示互为备份的两台机，replica是机器名称-->
<macros>
         <shard>100_103</shard>
        <replica>PShnczsjzxul100</replica>
</macros>
<!--zk节点信息-->
<zookeeper-servers>
        <node index="1">
                <host>113.219.128.100</host>
                <port>7072</port>
        </node>
     <node index="2">
                <host>113.219.128.103</host>
                <port>7072</port>
        </node>
         <node index="3">
                <host>113.219.128.106</host>
                <port>7072</port>
        </node>
</zookeeper-servers>
<!--数据压缩算法和一些阈值-->
<clickhouse_compression>
<case>
  <min_part_size>10000000000</min_part_size>
  <min_part_size_ratio>0.01</min_part_size_ratio>
  <method>lz4</method>
</case>
</clickhouse_compression>
</yandex>
集群基本测试（以带replica的MergeTree为例）

###创建带副本的基础表，每一台都要创建
create table test_distributed.test_replica(
id UInt64,
name String,
date Date
)Engine=ReplicatedMergeTree('/clickhouse/tables/{shard}/test_replica','{replica}',date,(id,date),8192);

#### {shard}对应于每台机子上metrika.xml中配置 的macros中的shard值，replica也一样

###创建分布式表，任意一台上创建即可

create table IF NOT EXISTS all_distributed ON Cluster cluster_for_replicated_table (id UInt64,name String,date Date) Engine=Distributed(cluster_for_replicated_table,test_distributed,test_replica,rand());
#####删除一个分布式表，任意一台上操作即可
drop table all_distributed on cluster cluster_for_replicated_table ;

####插入数据测试，任意一台上操作

insert into test_distributed.all_distributed values (1,'wusihang','2017-09-11'),(3,'hahngsiw','2017-09-10'),(4,'sihangwu','2017-09-11'),(2,'siwuhang','2017-09-12')
 
