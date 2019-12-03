# python-redis

[toc]

### **模块(redis)**

Ubuntu

```python
sudo pip3 install redis
```

Windows

```python
# 方法1. python -m pip install redis
# 方法2. 以管理员身份打开cmd命令行
        pip install redis
```

### **使用流程**

```python
import redis
# 创建数据库连接对象
r = redis.Redis(host='127.0.0.1',port=6379,db=0,password='123456')
```

### **通用命令代码示例**

```python
import redis

# 创建连接对象
r = redis.Redis(host='192.168.153.146',port=6379,db=0)

# r.keys('*') -> 列表
key_list = r.keys('*')
for key in key_list:
  print(key.decode())

# b'list'
print(r.type('mylist'))
# 返回值: 0 或者 1
print(r.exists('spider:urls'))
# 删除key
r.delete('mylist2')
```

### **python操作list**

```python
import redis

r = redis.Redis(host='192.168.153.146',port=6379,db=0)

# pylist: ['pythonweb','socket','pybase']
r.lpush('pylist','pybase','socket','pythonweb')
# pylist: ['spider','pythonweb','socket','pybase']
r.linsert('pylist','before','pythonweb','spider')
# 4
print(r.llen('pylist'))
# [b'spider', b'pythonweb', b'socket', b'pybase']
print(r.lrange('pylist',0,-1))
# b'pybase'
print(r.rpop('pylist'))
# [b'spider', b'pythonweb']
r.ltrim('pylist',0,1)

while True:
  # 如果列表中为空时,则返回None
  result = r.brpop('pylist',1)
  if result:
    print(result)
  else:
    break


r.delete('pylist')
```

### **Python操作string**

```python
import redis

r = redis.Redis(host='192.168.153.146',port=6379,db=0)

r.set('username','guods')
print(r.get('username'))
# mset参数为字典
r.mset({'username':'xiaoze','password':'123456'})
# 列表: [b'xiaoze', b'123456']
print(r.mget('username','password'))
# 6
print(r.strlen('username'))

# 数值操作
r.set('age','25')
r.incrby('age',10)
r.decrby('age',10)
r.incr('age')
r.decr('age')
r.incrbyfloat('age',3.3)
r.incrbyfloat('age',-3.3)
print(r.get('age'))

r.delete('username')
```





