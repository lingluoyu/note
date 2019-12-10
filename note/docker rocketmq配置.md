#### docker rocketmq配置
```shell
docker run -e "JAVA_OPTS=-Drocketmq.namesrv.addr=192.168.138.128:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false" -p 8080:8080 --name rmqconsole -t styletang/rocketmq-console-ng
```

```shell
docker run -d -p 9876:9876 -v /home/lingluoyu/docker/rocketmq/logs:/root/logs -v /home/lingluoyu/docker/rocketmq/store:/root/store --name rmqnamesrv  apacherocketmq/rocketmq:4.5.2-alpine sh mqnamesrv
```

```shell
docker run -d -p 10911:10911 -p 10909:10909 -v /home/lingluoyu/docker/rocketmq/broker/logs:/root/logs -v /home/lingluoyu/docker/rocketmq/broker/store:/root/store -e "JAVA_OPT_EXT=-server -Xms128m -Xmx128m -Xmn128m" --name rmqbroker --link rmqnamesrv:namesrv -e "NAMESRV_ADDR=namesrv:9876" apacherocketmq/rocketmq:4.5.2-alpine sh mqbroker -c ../conf/broker.conf
```

关闭防火墙

```shell
systemctl stop firewalld
```


