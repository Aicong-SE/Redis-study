# Redis主从复制

- **定义**

```python
1、一个Redis服务可以有多个该服务的复制品，这个Redis服务成为master，其他复制品成为slaves
2、master会一直将自己的数据更新同步给slaves，保持主从同步
3、只有master可以执行写命令，slave只能执行读命令
```

- **作用**

```python
分担了读的压力（高并发）
```

- **原理**

```python
从服务器执行客户端发送的读命令，比如GET、LRANGE、SMEMMBERS、HGET、ZRANGE等等，客户端可以连接slaves执行读请求，来降低master的读压力
```

- **两种实现方式**

  **方式一**（Linux命令行实现1）

  redis-server --slaveof <master-ip> <master-port>

```python
# 从服务端
redis-server --port 6300 --slaveof 127.0.0.1 6379
# 从客户端
redis-cli -p 6300
127.0.0.1:6300> keys * 
# 发现是复制了原6379端口的redis中数据
127.0.0.1:6300> set mykey 123
(error) READONLY You can't write against a read only slave.
127.0.0.1:6300> 
# 从服务器只能读数据，不能写数据
```

**方式一**（Redis命令行实现2）

```python
# 两条命令
1、>slaveof IP PORT
2、>slaveof no one
```

**示例**

```python
# 服务端启动
redis-server --port 6301
# 客户端连接
tarena@tedu:~$ redis-cli -p 6301
127.0.0.1:6301> keys *
1) "myset"
2) "mylist"
127.0.0.1:6301> set mykey 123
OK
# 切换为从
127.0.0.1:6301> slaveof 127.0.0.1 6379
OK
127.0.0.1:6301> set newkey 456
(error) READONLY You can't write against a read only slave.
127.0.0.1:6301> keys *
1) "myset"
2) "mylist"
# 再切换为主
127.0.0.1:6301> slaveof no one
OK
127.0.0.1:6301> set name hello
OK
```

**方式二**(修改配置文件)

```python
# 每个redis服务,都有1个和他对应的配置文件
# 两个redis服务
  1、6379 -> /etc/redis/redis.conf
  2、6300 -> /home/tarena/redis_6300.conf

# 修改配置文件
vi redis_6300.conf
slaveof 127.0.0.1 6379
port 6300
# 启动redis服务
redis-server redis_6300.conf
# 客户端连接测试
redis-cli -p 6300
127.0.0.1:6300> hset user:1 username guods
(error) READONLY You can't write against a read only slave.
```

**问题总结：master挂了怎么办？**

```python
1、一个Master可以有多个Slaves
2、Slave下线，只是读请求的处理性能下降
3、Master下线，写请求无法执行
4、其中一台Slave使用SLAVEOF no one命令成为Master，其他Slaves执行SLAVEOF命令指向这个新的Master，从它这里同步数据
# 以上过程是手动的，能够实现自动，这就需要Sentinel哨兵，实现故障转移Failover操作
```

**演示**

```python
1、启动端口6400redis，设置为6379的slave
   redis-server --port 6400
   redis-cli -p 6400
   redis>slaveof 127.0.0.1 6379
2、启动端口6401redis，设置为6379的slave
   redis-server --port 6401
   redis-cli -p 6401
   redis>slaveof 127.0.0.1 6379
3、关闭6379redis
   sudo /etc/init.d/redis-server stop
4、把6400redis设置为master
   redis-cli -p 6401
   redis>slaveof no one
5、把6401的redis设置为6400redis的salve
   redis-cli -p 6401
   redis>slaveof 127.0.0.1 6400
# 这是手动操作，效率低，而且需要时间，有没有自动的？？？
```

## **==官方高可用方案Sentinel==**

**Redis之哨兵 - sentinel**

```python
1、Sentinel会不断检查Master和Slaves是否正常
2、每一个Sentinel可以监控任意多个Master和该Master下的Slaves
```

**案例演示**

​	**1、**环境搭建

```python
# 共3台redis的服务器，如果是不同机器端口号可以是一样的
1、启动6379的redis服务器
   	sudo /etc/init.d/redis-server start
2、启动6380的redis服务器，设置为6379的从
    redis-server --port 6380
    tarena@tedu:~$ redis-cli -p 6380
    127.0.0.1:6380> slaveof 127.0.0.1 6379
    OK
3、启动6381的redis服务器，设置为6379的从
   	redis-server --port 6381
   	tarena@tedu:~$ redis-cli -p 6381
   	127.0.0.1:6381> slaveof 127.0.0.1 6379
```

​	**2、**安装并搭建sentinel哨兵

```python
# 1、安装redis-sentinel
sudo apt install redis-sentinel
验证: sudo /etc/init.d/redis-sentinel stop
# 2、新建配置文件sentinel.conf
port 26379
Sentinel monitor tedu 127.0.0.1 6379 1

# 3、启动sentinel
方式一: redis-sentinel sentinel.conf
方式二: redis-server sentinel.conf --sentinel

#4、将master的redis服务终止，查看从是否会提升为主
sudo /etc/init.d/redis-server stop
# 发现提升6381为master，其他两个为从
# 在6381上设置新值，6380查看
127.0.0.1:6381> set name tedu
OK

# 启动6379，观察日志，发现变为了6381的从
主从+哨兵基本就够用了
```

sentinel.conf解释

```python
# sentinel监听端口，默认是26379，可以修改
port 26379
# 告诉sentinel去监听地址为ip:port的一个master，这里的master-name可以自定义，quorum是一个数字，指明当有多少个sentinel认为一个master失效时，master才算真正失效
sentinel monitor <master-name> <ip> <redis-port> <quorum>
```

**生产环境中设置哨兵sentinel**

```python
1、安装sentinel
  sudo apt-get install redis-sentinel
2、创建配置文件 sentinel.conf
  port 26379
  Sentinel monitor 名字 IP PORT 投票数
3、启动sentinel开始监控
  redis-sentinel sentinel.conf
```

## 分布式锁

**原理**

```python
1、多个客户端先到redis数据库中获取一把锁,得到锁的用户才可以操作数据库
2、此用户操作完成后释放锁，下一个成功获取锁的用户再继续操作数据库
```

**实现**

```python
set key value nx ex 3
```

```python
import redis
    pool = redis.ConnectionPool(host='localhost', port=6379, db=0)
    r = redis.Redis(connection_pool=pool)
	try:
        with r.lock('key', blocking_timeout=3) as lock:
            代码块
    except Exception as e:
        print('lock is failed')
```

