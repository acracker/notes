# MongoDB Replica-Set 部署

### 环境
* 系统版本:  Centos7.4
* Mongodb版本: v4.0.6
* 服务器：192.168.1.211(node-1), 192.168.1.212(node-2), 192.168.1.213 (node-3)

### 说明
* Relica-Set 数据冗余, Sharding集群扩展
* Relica-Set 包含三类角色(Primary, Secondary, Arbiter)
> `Primary` 接收所有的写请求, 然后把修改同步到所有Secondary. 一个Replica Set只能有一个Primary节点, 当Primary挂掉后, 其他Secondary或者Arbiter节点会重新选举出来一个主节点. 默认读请求也是发到Primary节点处理的, 需要转发到Secondary需要客户端修改一下连接配置. 
> `Secondary`与主节点保持同样的数据集. 当主节点挂掉的时候, 参与选主. 
> `Arbiter`不保有数据, 不参与选主, 只进行选主投票. 使用Arbiter可以减轻数据存储的硬件需求, Arbiter跑起来几乎没什么大的硬件资源需求, 但重要的一点是, 在生产环境下它和其他数据节点不要部署在同一台机器上. 一个自动failover的Replica Set节点数必须为奇数, 目的是选主投票的时候要有一个大多数才能进行选主决策


### 配置
`vim /etc/mongod.conf`
```
net:
  port: 33017
  bindIp: 0.0.0.0 
replication:
  replSetName: ctsec	# 需要设置分片名称

```
在node-1上执行:
```
mongo --port 33017
rs.initiate()
rs.add("node-2:33017")
rs.add("node-3:33017")
rs.isMaster()
rs.status()
```
### Python操作MongoDB
> 错误日志: pymongo.errors.ServerSelectionTimeoutError: No replica set members match selector "Primary()"
>>  原因是因为配置里面是主机名,但是客户端不能解析主机名,在hosts文件添加好 IP 映射即可.
```
from pymongo import MongoClient
conn = MongoClient(['node-1:33017', 'node-2:33017', 'node-3:33017'])
print(conn['test']['test'].insert_one({'a': 1, 'b': 2}))
cur = conn['test']['test'].find()
for doc in cur:
    print(doc)


```