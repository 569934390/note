1.producer-->broker(kafka实力)-->topic-->consumer
2.结构 
1.索引文件----存储消息的位置
2.数据文件----存储消息的实际信息以及长度
3.page cache
4.客户端读写都是操作的leader（follower同步数据）
