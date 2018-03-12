
1.编译:
http://blog.csdn.net/wusihang9/article/details/77619735
2.单机安装:
###clickhouse装机后（集群模式）的配置步骤
以下步骤都是对每一台机器操作过去
1. service clickhouse-server stop #先停掉进程
2. touch /etc/clickhouse-server/metrika.xml
   chown clickhouse:clickhouse /etc/clickhouse-server/metrika.xml
再写入以下内容（范例）：
<yandex>
    <clickhouse_remote_servers>
    <!--集群配置，按照实际需要来配置，一般使用机器名配置-->
        <local_cover_for_vod_cluster>
            <shard>
        <weight>1</weight>
                <replica>
                    <host>zhx152</host>
                    <port>19008</port>
                </replica>
            </shard>
        <shard>
                <weight>1</weight>
                <replica>
                    <host>zhx153</host>
                    <port>19008</port>
                </replica>
            </shard>
        </local_cover_for_vod_cluster>
    </clickhouse_remote_servers>
 
    <!--和副本相关的配置，推荐是互备两台的机器名作为shard，本机机器名作为replica，当然，此处可以任意自定义，也可以加其他参数-->
    <macros>
    <shard>zhx152_zhx153</shard>
    <replica>zhx152</replica>
    </macros>
    <!--zk集群信息，这个可以和其他业务共用，但是注意集群名称配置不要和其他重复了！-->
    <zookeeper-servers>
        <node index="1">
            <host>zk1-jstorm-suqian.qoss.haplat.net</host>
            <port>7072</port>
        </node>
    <node index="2">
            <host>zk2-jstorm-suqian.qoss.haplat.net</host>
            <port>7072</port>
        </node>
        <node index="3">
            <host>zk3-jstorm-suqian.qoss.haplat.net</host>
            <port>7072</port>
        </node>
    <node index="4">
            <host>zk4-jstorm-suqian.qoss.haplat.net</host>
            <port>7072</port>
        </node>
    <node index="5">
            <host>zk5-jstorm-suqian.qoss.haplat.net</host>
            <port>7072</port>
        </node>
    <node index="6">
            <host>zk6-jstorm-suqian.qoss.haplat.net</host>
            <port>7072</port>
        </node>
    <node index="7">
            <host>zk7-jstorm-suqian.qoss.haplat.net</host>
            <port>7072</port>
        </node>
    </zookeeper-servers>
    <!--配置lz4压缩，这里还有其他压缩方式，具体参考文档-->
    <clickhouse_compression>
        <case>
            <min_part_size>10000000000</min_part_size>
            <min_part_size_ratio>0.01</min_part_size_ratio>
            <method>lz4</method>
        </case>
    </clickhouse_compression>
</yandex>

3. 在config.xml 加 <include_from>/etc/clickhouse-server/metrika.xml</include_from>
4. hostname -f 测试机器名 ，如果返回正确机器名，进入下一步，否则需要在config.xml添加(值是机器名) <interserver_http_host>zhx152</interserver_http_host>
5. 确认一下互为集群的几台机器能够互相ping的通（例如 ping zhx152），如果ping不通，需要编辑 /etc/hosts 添加对应机器和ip
6. 在现有的clickhouse机器上选一个users.xml将当前的uses.xml替换掉
7. 确定一下config.xml中的path和tmp_path节点，根据节点配置信息，将对应目录的owner改成clickhouse(例如：chown -R clickhouse:clickhouse /cache/clickhouse/)
以下是范例内容：
<!-- Path to data directory, with trailing slash. -->
<path>/cache/clickhouse/data</path>
<!-- Path to temporary data for processing hard queries. -->
<tmp_path>/cache/clickhouse/data/tmp/</tmp_path>

3.集群模式安装:
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
 
