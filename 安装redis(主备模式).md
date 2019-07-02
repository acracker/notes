# redis 主从部署

### 环境
* 系统版本:  Centos7.4
* Redis版本: Redis5.04
* 服务器：192.168.1.211, 192.168.1.212, 192.168.1.213     211为Master,三台机器都包含redis和sentinel 服务
 
### 说明
+ redis sentinel 为高可用性方案; redis cluster为分布式解决方案
+ sentinel 服务监控master-slave集群, 发现master宕机后能进行自动切换
+ sentinel本身是一个独立运行的进程, 也可以集群部署
+ redis master 宕机, sentinel选举出新的master, 会修改 redis和sentinel 配置文件
+ 旧的redis Master重新启动, 他会被设置会slave
### 配置
> `192.168.1.211` `redis.conf`
```
bind 0.0.0.0				# 侦听所有网卡
protected-mode yes		# 保护模式, 默认是开启状态, 只允许本地客户端连接, 可以设置密码或添加bind来连接
port 8394					# 侦听的端口
tcp-backlog 511			# TCP监听的最大容纳数量
timeout 0					# 指定在一个 client 空闲多少秒之后关闭连接（0表示永不关闭）
tcp-keepalive 300			# 单位是秒，表示将周期性的使用SO_KEEPALIVE检测客户端是否还处于健康状态
daemonize yes			# 守护进程运行
supervised systemd		# 可以通过upstart和systemd管理Redis守护进程
pidfile "/var/run/redis_8394.pid"
loglevel notice			# 日至级别
logfile "/var/log/redis/redis.log"
databases 16
save 900 1				# 900妙内至少1个KEY被修改则刷数据到磁盘
save 300 10			# 300妙内至少10个KEY被修改则刷数据到磁盘
save 60 10000
stop-writes-on-bgsave-error yes		# 快照刷盘失败停止接受新的请求
rdbcompression yes					# 快照文件压缩, 消耗CPU, 减小磁盘消耗
rdbchecksum yes						# 快照CRC64校念， 消耗CPU
dbfilename "dump.rdb"				# 快照名称
dir "/var/lib/redis"					# 快照目录
slave-serve-stale-data yes			# slave 数据过期处理方式, 返回错误或者过期数据
slave-read-only yes					# slave 是否只读
repl-diskless-sync no					# 主从数据复制是否使用无硬盘复制功能
repl-diskless-sync-delay 5			# 当启用无硬盘备份, 快照传送等待时间
repl-disable-tcp-nodelay no			# 同步之后是否禁用从站上的TCP_NODELAY
slave-priority 100						# sentinel 提升slave为master的优先级,越小优先级越高,但0为永不提升
requirepass "ctsec2019"				# redis 密码
appendonly no						# aof模式 Redis会把每次写入的数据在接收后都写入appendonly.aof文件
appendfilename "appendonly.aof"	# aof文件名
appendfsync everysec				# aof 持久化策略
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000		#  slog log是用来记录redis运行中执行比较慢的命令耗时
slowlog-max-len 128					#  慢查询日志长度
latency-monitor-threshold 0			# 延迟监控功能
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes			# 重新hash,降低內存使用. 延迟還內存
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10											#   redis执行任务的频率为1s除以hz
aof-rewrite-incremental-fsync yes
# slaveof 192.168.1.211 8394			# master 地址 端口 只需要在 slave 上配置
# masterauth "ctsec2019"				# master 密码
```
> `192.168.1.211``redis-sentinel.conf`
```
protected-mode no 	# 保护模式, 非单击模式一定要关闭, 且设置密码
port 26379 				# 当前Sentinel服务运行的端口
dir "/tmp" 				# Sentinel服务运行时使用的临时文件夹
sentinel monitor mymaster 192.168.1.211 8394 2 	# 名称, ip, 至少2个Sentinel判定为失效 
sentinel down-after-milliseconds mymaster 3000	# redis失连超过3秒,才视为失效
sentinel auth-pass mymaster ctsec2019			# redis 密码
logfile "/var/log/redis/sentinel.log"
supervised systemd
```
> `192.168.1.211` `192.168.1.212` `redis.conf`
>>  slave 额外添加以下配置, 其余可相同, 也可以将数据持久化之放在slave上.
```
slaveof 192.168.1.211 8394			# master 地址 端口 只需要在 slave 上配置
masterauth "ctsec2019"				# master 密码
```

> `192.168.1.211` `192.168.1.212` `redis-sentinel.conf`
>>  sentinel 的配置均相同,系统会生成部分的配置不同.


### 参考资料
> [redis.conf参数详细说明](https://www.cnblogs.com/pqchao/p/6558688.html) 

* 常见错误

>  错误日志:  33734:M 19 Mar 15:29:48.594 # Creating Server TCP listening socket 127.0.0.1:8394: bind: Permission denied
> >        解决方法: 关闭selinxu
------------------------------------------
> Sentinel在redis master 宕机之后不重新选举master
>>     设置redis-sentinel.conf: protected-mode no


