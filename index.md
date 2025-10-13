# DolphinDB redisCluster ���ʹ��˵��

ͨ�� DolphinDB �� redisCluster ������û��������ӵ� Redis Cluster ��Ⱥ��ֻ����������һ���ڵ㼴����ɼ�Ⱥ������·�ɡ�������� hash tag �� key �Զ��ֲ۲�·�ɡ������ӿ�֧�� pipeline ����߳���������¡�redisCluster ������� hiredis ��� redis-plus-plus �⿪����

ע��: 
- **Ŀǰ�ò��ֻ֧�� Redis Cluster, �ݲ�֧�ֵ�ʵ�� Redis��** ��Ե�ʵ�� redis �����ӺͲ�����ο� DolphinDB �ٷ� [redis ���](https://docs.dolphindb.cn/zh/plugins/redis.html)��
- �ڶ��̲߳���������ҵ������ʹ�ó����£��������ͬһ�� `connect` ������ Redis Cluster�����Ӿ���� Ϊ�����̰߳�ȫ������Դ���㵼�µĳ�����/�������⣬���鱣֤ **poolSize �� ����ҵ���������� �� ����ҵ numThreads + 2**��

## �ڲ���г���װ���

### �汾Ҫ��

- DolphinDB Server 3.00.4 �����߰汾��
- ֧�� Linux x64��

### ��װ����

1. �� DolphinDB �ͻ�����ʹ�� `listRemotePlugins` ����鿴����ֿ��еĲ����Ϣ��
```
login("admin", "123456")
listRemotePlugins()
```
2. ʹ�� `installPlugin` ��ɰ�װ��
```
installPlugin("redisCluster")
```
3. ʹ�� `loadPlugin` ���ز����
```
loadPlugin("redisCluster")
```

- �����ڱ���·������

```
loadPlugin("/path/to/PluginRedisCluster.txt")

```

## �����ӿ�

### connect

**�﷨**

```
redisCluster::connect(host, port, [password=""], [poolSize=3], [waitTimeoutMs=100])
```
**����**

ͨ������һ���ڵ����ӵ� Redis Cluster ���������Ӿ����

**����**
- host STRING ������Ŀ��ڵ� IP ����������
- port INT ������Ŀ��˿ڡ�
- password STRING ��������ѡ����֤���롣ע�⣺**����������, ���нڵ�Ӧʹ����ͬ����**��
- poolSize INT ��������ѡ���������ӳش�С, Ĭ�� 3��
- waitTimeoutMs INT ��������ѡ�����ӳؽ������ʱ����ȴ�ʱ�䣨���룩��Ĭ�� 100��������ʱ�����޿������ӽ��׳���ʱ�쳣������ wait_timeout <=0 ��ʾһֱ�ȡ�

**����ֵ**

���ش����� Redis Cluster ���Ӿ����

**ʾ��**

```
// Redis Cluster ������
conn = redisCluster::connect("127.0.0.1", 6379)
// �Զ������ӳ��볬ʱʱ�䣨���룩
conn = redisCluster::connect("127.0.0.1", 6379, "", 3, 200)
// Redis Cluster ������
conn = redisCluster::connect("127.0.0.1", 6379, YOUR_PASSWORD)
// ���������Զ������ӳ��볬ʱʱ��
conn = redisCluster::connect("127.0.0.1", 6379, YOUR_PASSWORD, 4, 500)
```

### run

**�﷨**

```
redisCluster::run(conn, routeKey, argv)
```

**����**

�� routeKey ָ����·�ɽڵ�ִ������ Redis ���

ע��:
- ��� Redis Cluster ���������룬��Ҫ������`redisCluster::connect`ʱָ����������ȡȨ�ޡ�����ֵ������ Redis ���Է��ص��κ��������ͣ��� string, list, �� set�ȡ�
- run ֻ���� routeKey ·�ɡ������������� key����ȷ����Щ key ������ͬ hash tag��
- argv �����������Ȳ��ܳ���4096��
**����**
- conn ͨ�� redisCluster::connect ��õ� Redis Cluster ���Ӿ����
- routeKey STRING ������������ hash tag ���������·�ɵ� key������ "tag=xxx", "tag:xxx", "key=xxx" �� "key:xxx" Ҳ�ɡ����磬"tag=good{morning}", "tag:{morning}", "key=morning" �� "morning" ��·�ɽڵ���ͬ��
- argv STRING �������������������� ["HSET", "profile:{user1}", "name", "kevin"]��

**����ֵ**

�����������н�������� Redis �ظ�ӳ�䵽 DolphinDB ���͡�

**ʾ��**
```
/* �� hash tag ·�ɵ�ͬһ�� */
rk = "{user:1}"
redisCluster::run(conn, rk, ["PING"])
redisCluster::run(conn, rk, ["SET", rk + ":name", "kevin"])
redisCluster::run(conn, rk, ["GET", rk + ":name"])
redisCluster::run(conn, rk, ["HSET", rk + ":info", "age", "25", "city", "nyc"])
redisCluster::run(conn, rk, ["HGETALL", rk + ":info"])
```

### set

**�﷨**

```
redisCluster::set(conn, key, value, [ttlSec=0])
```

**����**

���� STRING ����ֵ��

**����**

- conn ���Ӿ����
- key STRING ������
- value STRING ������
- ttlSec INT ��������ѡ������ʱ�䣨��������Ĭ�� 0����ʾ�����ù��ڡ�

**����ֵ**

���гɹ����� "OK"��

**ʾ��**

```
redisCluster::set(conn, "k1", "v1")
redisCluster::set(conn, "k2", "v2", 5)
```

### get

**�﷨**

```
redisCluster::get(conn, key)
```

**����**

��ȡ STRING ֵ���Զ�·�ɵ� key ���ڲۡ�

ע�⣺ֻ֧����ͨ STRING ��ֵ�ԡ�������Ի�ȡ�������͵� key������ Hash key�� ��ᱨ WRONGTYPE �Ĵ���

**����**
 
- conn ���Ӿ����
- key STRING ������֧��ʹ�� hash tag��

**����ֵ**

���� STRING �� NULL��

**ʾ��**

```
redisCluster::get(conn, "{acct:1001}:name")
redisCluster::get(conn, "user:1")
```

### batchSet

**�﷨**

```
redisCluster::batchSet(conn, keys, values, [batchWin=2048], [numThreads=1])
```

**����**

���� SET���Զ��ֲ۲�ʹ�� pipeline ����̣߳���ָ��������ִ�С�
�� keys �� values Ϊ����ʱ�˻�Ϊ���� SET��

ע�⣺�������ؼ�ʹ����ͬ hash tag ��ͬ�ۣ�slot�����и��������󻯵�·����Ч�ʡ�

**����**

- conn ���Ӿ����
- keys STRING ������������
- values STRING ������������
- batchWin INT ������ÿ�� pipeline �е���������Ĭ�� 2048��
- numThreads INT �����������߳�����Ĭ�� 1�������������̡߳�

**����ֵ**

���� "batchSet finish."��

**ʾ��**

```
keys = "{t:1}:" + string(1..100000)
vals = take("x", 100000)
redisCluster::batchSet(conn, keys, vals, 2048, 4)
```

### batchHashSet

**�﷨**

```
redisCluster::batchHashSet(conn, ids, fieldData, [batchWin=2048], [numThreads=1])
```

**����**

���� HSET��ÿ��дһ�� key �Ķ��ֶΡ��ֶ������� fieldData ���������ֶ�ֵ���Զ�Ӧ�����ڵ�ֵ��

ע�⣺�������ؼ�ʹ����ͬ hash tag ��ͬ�ۣ�slot�����и��������󻯵�·����Ч�ʡ�

**����**

- conn ���Ӿ����
- ids STRING ������ÿ�ж�Ӧ��һ�� key��
- fieldData �������б���Ϊ STRING��
- batchWin INT ������ÿ�� pipeline �е���������Ĭ�� 2048��
- numThreads INT �����������߳�����Ĭ�� 1�������������̡߳�

**����ֵ**

���гɹ����� "batchHashSet finish."��

**ʾ��**

```
ids = "{tick:}" + string(1..5)
t = table(`f1`f2`f3`f4`f5 as name, take(1,5) as f1, take(2,5) as f2)
t = select name, string(f1) as f1, string(f2) as f2 from t
redisCluster::batchHashSet(conn, ids, t, 2048, 3)
```

### batchGet

**�﷨**

```
redisCluster::batchGet(conn, keys)
```

**����**

���� GET������ֵ�� hash tag ��۷��鲢�ֱ��ÿ��ʹ�� MGET��

ע�⣺�������ؼ�ʹ����ͬ hash tag ��ͬ�ۣ�slot�����и��������󻯵�·����Ч�ʡ�

**����**

- conn ���Ӿ����
- keys STRING ��������������

**����ֵ**

���� STRING ��������������λ��Ϊ NULL��

**ʾ��**

```
keys = ["{u:1}:n","{u:2}:n","{u:3}:n"]
vals = redisCluster::batchGet(conn, keys)
```

### batchPush

**�﷨**

```
redisCluster::batchPush(conn, keys, values, [rightPush=true], [batchWin=2048], [numThreads=1])
```

**����**

���� RPUSH �� LPUSH��ÿ�� key ��Ӧһ�� STRING ������Ϊ������Ԫ�ء�
rightPush Ϊ true ִ�� RPUSH������ LPUSH��

ע�⣺�������ؼ�ʹ����ͬ hash tag ��ͬ�ۣ�slot�����и��������󻯵�·����Ч�ʡ�

**����**

- conn ���Ӿ����
- keys STRING ������
- values STRING ������
- rightPush BOOL ������Ĭ�� true��
- batchWin INT ������ÿ�� pipeline �е���������Ĭ�� 2048��
- numThreads INT �����������߳�����Ĭ�� 1�������������̡߳�

**����ֵ**

���гɹ����� "batchPush finish."��

**ʾ��**

```
ks = ["{q:1}", "{q:2}"]
vs = [ ["a","b"], ["c","d","e"] ]
redisCluster::batchPush(conn, ks, vs, true, 1024, 2)
```

### batchDel

**�﷨**

```
redisCluster::batchDel(conn, keys, [batchWin=2048], [numThreads=1], [useUnlink=true])
```

**����**

����ɾ�� key������ִ�� DEL�� useUnlink Ϊ true ʱʹ�� UNLINK �첽ɾ����

ע�⣺�������ؼ�ʹ����ͬ hash tag ��ͬ�ۣ�slot�����и��������󻯵�·����Ч�ʡ�

**����**
- conn ���Ӿ����
- keys STRING ������
- batchWin INT ������ÿ�� pipeline �е���������Ĭ�� 2048��
- numThreads INT �����������߳�����Ĭ�� 1�������������̡߳�
- useUnlink BOOL �������Ƿ����첽ɾ����Ĭ�� true��

**����ֵ**

���гɹ����� "batchDel finish."��

**ʾ��**

```
ks = "{tmp:}" + string(1..10000)
redisCluster::batchDel(conn, ks, 2048, 4)
```

### release

**�﷨**

```
redisCluster::release(conn)
```

**����**

�ͷ�һ�� Redis CLuster ���Ӿ����

**����**

conn ͨ�� connect ������ Redis Cluster ���Ӿ����

**ʾ��**

```
conn = redisCluster::connect("127.0.0.1",6379)
redisCluster::release(conn)
```

### releaseAll

**�﷨**

```
releaseAll()
```

**����**

�ͷŵ�ǰ�Ự�ڼ�¼������ Redis Cluster ���Ӿ����

**ʾ��**

```
redisCluster::releaseAll()
```

### getHandle

**�﷨**

```
redisCluster::getHandle(token)
```

**����**

��ȡ token ��Ӧ�� Redis Cluster ���Ӿ��������ͨ�� token �һؾ����

**����**

token STRING �����������ʶ��ͨ�� `redis::getHandleStatus` ���ر��еĵ�һ�е�֪������Ψһ��ʶһ�� Redis Cluster ���ӡ�

**����ֵ**

����token��Ӧ�����Ӿ����

**ʾ��**

```
conn = redisCluster::getHandle(token)
```

### getHandleStatus

**�﷨**

```
redisCluster::getHandleStatus()
```

**����**

�鿴��ǰ�Ự����ע�����Ӿ����Ϣ��

**����ֵ**

���ر�����Ϊ token, address, createdTime��

- token ���Ǹ����ӵ�Ψһ��ʶ����ͨ�� redisCluster::getHandle(token) ����ȡ�����
- address ���������� Redis Cluster ���ӵĽڵ�� "ip:port" ��ַ��
- createdTime �Ǹ����Ӵ�����ʱ�䡣

**ʾ��**

```
status = redis::getHandleStatus()
```

## ʹ��ʾ��

```
loadPlugin("path/to/PluginRedisCluster.txt");
go

try { unsubscribeTable(tableName=`table1, actionName=`testRedisClusterPlugin) } catch(ex) { print ex }
try { dropStreamTable(`table1) } catch(ex) {}

colName=["key", "value"]
colType=["string", "string"]
enableTableShareAndPersistence(table=streamTable(100:0, colName, colType), tableName=`table1, cacheSize=10000, asynWrite=false)

def myHandle(conn, msg) {
    keys = msg["key"]
    values = msg["value"]
	redisCluster::batchSet(conn, keys, values, 2048, 2)
}

conn = redisCluster::connect("YOUR_REDIS_CLUSTER_NODE_IP", 6379, "YOUR_PASSWORD", 3, false);

subscribeTable(
    tableName="table1", 
    actionName=`testRedisClusterPlugin,
    handler=myHandle{conn},
    msgAsTable=true, 
    batchSize=10000,
    throttle=1)

n=1000000
keys = take("{key}",n) + string(1..n)
values = string(1..n)
t1 = table(keys, values)
go

tableInsert(table1, t1)

valuesFetched = redisCluster::batchGet(conn, keys)
t2 = table(keys, valuesFetched)

redisCluster::batchDel(conn, keys, 2048, 3);

assert "test", n==t2.size()
```