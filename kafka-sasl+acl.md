# 配置脚本
```
sudo ln -s /usr/local/kafka_2.11-2.3.0 /usr/local/kafka
echo "
listeners=SASL_PLAINTEXT://:9092
authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
#SASL_PLAINTEXT
sasl.enabled.mechanisms=PLAIN
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=PLAIN
allow.everyone.if.no.acl.found=true
super.users=User:admin

" >> /usr/local/kafka/config/server.properties
# 第44行插入
sed -i -e "44"i'\export KAFKA_OPTS="-Djava.security.auth.login.config=/usr/local/kafka/config/kafka_server_jaas.conf"' /usr/local/kafka/bin/kafka-server-start.sh
echo '
KafkaServer {
org.apache.kafka.common.security.plain.PlainLoginModule required
username="admin"
password="good$luck2019"
user_admin="good$luck2019"
user_reader="goodluck"
user_writer="goodluck2019";
};
' > /usr/local/kafka/config/kafka_server_jaas.conf

echo '
KafkaClient {
org.apache.kafka.common.security.plain.PlainLoginModule required
username="writer"
password="goodluck2019";
};
'> /usr/local/kafka/config/kafka_client_jaas.conf


echo 'security.protocol=SASL_PLAINTEXT' >> /usr/local/kafka/config/producer.properties
echo 'sasl.mechanism=PLAIN' >> /usr/local/kafka/config/producer.properties
echo 'security.protocol=SASL_PLAINTEXT' >> /usr/local/kafka/config/consumer.properties
echo 'sasl.mechanism=PLAIN' >> /usr/local/kafka/config/consumer.properties

sed -i -e "21"i'\export KAFKA_OPTS="-Djava.security.auth.login.config=/usr/local/kafka/config/kafka_client_jaas.conf"' /usr/local/kafka/bin/kafka-console-consumer.sh
sed -i -e "20"i'\export KAFKA_OPTS="-Djava.security.auth.login.config=/usr/local/kafka/config/kafka_client_jaas.conf"' /usr/local/kafka/bin/kafka-console-producer.sh
```

# 配置文件
`vim config/server.properties`
```
listeners=SASL_PLAINTEXT://:9092
advertised.listeners=SASL_PLAINTEXT://$advertised_hostname:9092
authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
#SASL_PLAINTEXT
sasl.enabled.mechanisms=PLAIN
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=PLAIN
allow.everyone.if.no.acl.found=true
super.users=User:admin
```

+ allow.everyone.if.no.acl.found=true
> 默认情况下，如果资源R没有关联acl，除了超级用户，没有用户允许访问。如果你想改变这种方式你可以做如下配置
> 什么意思呢？上面的配置已经启动了acl，除了超级用户之外，其他用户无法访问。那么问题就来了，在kafka集群中，其它节点需要同步数据，需要相互访问。
> 它默认会使用ANONYMOUS的用户名连接集群。在这种情况下，启动kafka集群，必然失败！所以这个参数一定要配置才行！

 

+ listeners=SASL_PLAINTEXT://:9092
> 这个参数，表示kafka监听的地址。此参数必须要配置，默认是注释掉的。默认会使用listeners=PLAINTEXT://:9092，但是我现在开启了SASL，必须使用SASL协议连接才行。
> //:9092 这里虽然没有写IP地址，根据官方解释，它会监听所有IP。注意：这里只能是IP地址，不能是域名。否则启动时，会提示无法绑定IP。

 
+ advertised.listeners 
> 这个参数，表示外部的连接地址。这里可以写域名，也可以写IP地址。建议使用域名，为什么呢？因为IP可能会变动，但是主机名是不会变动的。
> 所以在java代码里面写死，就可以了！注意：必须是SASL协议才行！

 
+ super.users=User:admin
>  表示启动超级用户admin，注意：此用户名不允许更改，否则使用生产模式时，会有异常！

# 启动脚本
`vim  bin/kafka-server-start.sh`
```
# 最上面添加
export KAFKA_OPTS="-Djava.security.auth.login.config=/usr/local/kafka/config/kafka_server_jaas.conf"
```
`vim /usr/local/kafka/config/kafka_server_jaas.conf`
```
KafkaServer {
org.apache.kafka.common.security.plain.PlainLoginModule required
username="admin"
password="good$luck2019"
user_admin="good$luck2019"
user_reader="goodluck"
user_writer="goodluck2019";
};
```
>user_reader:用户名reader的密码是123456

# 生产者
`vim bin/kafka-console-producer.sh`
```
# 最上面添加
export KAFKA_OPTS="-Djava.security.auth.login.config=/usr/local/kafka/config/kafka_client_jaas.conf"
```
`vim config/producer.properties`
```
# 末尾添加
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
```

# 消费者
`vim bin/kafka-console-consumer.sh`
```
# 最上面添加
export KAFKA_OPTS="-Djava.security.auth.login.config=/usr/local/kafka/config/kafka_client_jaas.conf"
```
`vim config/producer.properties`
```
# 末尾添加
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
```

# 配置ACL
> 允许writer用户有所有权限，访问所有topic
> --operation All 表示所有权限，
> --topic=* 表示所有topic
+ topic权限
`bin/kafka-acls.sh --authorizer kafka.security.auth.SimpleAclAuthorizer --authorizer-properties zookeeper.connect=127.0.0.1:2181 --add --allow-principal User:writer --operation All --topic=*`
`bin/kafka-acls.sh --authorizer kafka.security.auth.SimpleAclAuthorizer --authorizer-properties zookeeper.connect=127.0.0.1:2181 --add --allow-principal User:reader --operation Read --topic=*`

+ 组权限
`bin/kafka-acls.sh --authorizer kafka.security.auth.SimpleAclAuthorizer --authorizer-properties zookeeper.connect=127.0.0.1:2181 --add --allow-principal User:writer --operation All -group=*`


# 测试
+ 启动zk
`bin/zookeeper-server-start.sh  -daemon config/zookeeper.properties`
+ 启动kafka
`bin/kafka-server-start.sh  -daemon config/server.properties`
+ 创建topic
`bin/kafka-topics.sh --create --zookeeper 127.0.0.1:2181 --topic test --partitions 1 --replication-factor 1`
+ 启动生产者
`bin/kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic test --producer.config config/producer.properties`
+ 启动消费者
`bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic test --from-beginning --consumer.config config/consumer.properties`

# 参考资料
> https://www.cnblogs.com/xiao987334176/articles/10110389.html#autoid-2-0-0
