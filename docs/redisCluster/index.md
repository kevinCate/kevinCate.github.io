# DolphinDB redisCluster 插件使用说明

通过 DolphinDB 的 redisCluster 插件，用户可以连接到 Redis Cluster 集群。客户端只需连接任意一个节点，即可自动发现集群中的所有节点，并根据 key 的 hash 或 hash tag 将请求路由到对应槽位。批量接口支持 pipeline 与多线程，以提高吞吐量。该插件基于 hiredis 库和 redis-plus-plus 库开发。

注意: 
- **目前该插件只支持 Redis Cluster，暂不支持单实例 Redis。** 针对单实例 Redis 的连接和操作请参考 DolphinDB 官方 [redis 插件](https://docs.dolphindb.cn/zh/plugins/redis.html)。
- 在多线程并发、跨作业并发场景下，如果多个线程共用同一个 `connect` 创建的 Redis Cluster 的连接句柄， 可能会出现线程安全问题，或者因资源不足导致长时间阻塞或死锁。因此，建议设置 **poolSize ≥ 跨作业并发任务数 × 单作业 numThreads + 2**。

## 在插件市场安装插件

### 版本要求

- DolphinDB Server 3.00.4 及更高版本。
- 支持 Linux x64。

### 安装步骤

1. 在 DolphinDB 客户端中使用 `listRemotePlugins` 命令查看插件仓库中的插件信息。
```
login("admin", "123456")
listRemotePlugins()
```
2. 使用 `installPlugin` 完成安装。
```
installPlugin("redisCluster")
```
3. 使用 `loadPlugin` 加载插件。
```
loadPlugin("redisCluster")
```

## 函数接口

### connect

**语法**

```
redisCluster::connect(host, port, [password=""], [poolSize=3], [waitTimeoutMs=500])
```
**详情**

创建一个 Redis Cluster 的连接。

**参数**

**host** STRING 标量。目标节点 IP 或主机名。

**port** INT 标量。目标端口。 

**password** STRING 标量。可选。认证密码。注意：**如设置密码，所有节点应使用相同密码**。

**poolSize** INT 标量。可选。配置连接池大小，默认 3。取值必须小于等于 8192。

**waitTimeoutMs** INT 标量。可选。连接池借出连接时的最长等待时间（毫秒），默认 500；超过该时间仍无可用连接将抛出超时异常。设置 waitTimeoutMs ≤ 0，表示一直等待。取值必须小于等于 600000（10 min）。

**返回值**

返回创建的 Redis Cluster 连接句柄。

**示例**

```
// Redis Cluster 无密码
conn = redisCluster::connect("127.0.0.1", 6379)
// 自定义连接池与超时时间（毫秒）
conn = redisCluster::connect("127.0.0.1", 6379, "", 3, 200)
// Redis Cluster 有密码
conn = redisCluster::connect("127.0.0.1", 6379, YOUR_PASSWORD)
// 有密码且自定义连接池与超时时间
conn = redisCluster::connect("127.0.0.1", 6379, YOUR_PASSWORD, 4, 500)
```

### run

**语法**

```
redisCluster::run(conn, routeKey, argv)
```

**详情**

在 *routeKey* 指定的路由节点执行任意 Redis 命令。

注意:
- 如果 Redis Cluster 设置有密码，需要在运行`redisCluster::connect`时指定密码来获取权限。
- `run` 只依据 *routeKey* 路由。若命令包含多个 key，请确保这些 key 共用相同 hash tag。

**参数**

**conn** 通过 `redisCluster::connect` 获得的 Redis Cluster 连接句柄。

**routeKey** STRING 标量。可以是 hash tag 或任意 key。例如，"tag=good{morning}"，"tag:{morning}"，"key={morning}"，"key:{morning}"", "morning"或 {morning} 的路由节点相同。

**argv** STRING 向量。命令及其参数，形如 ["HSET", "profile:{user1}", "name", "kevin"]。长度 ≤ 4096。

**返回值**

返回命令运行结果。根据 Redis 回复映射到 DolphinDB 类型。

**示例**
```
/* 以 hash tag 路由到同一槽 */
rk = "{user:1}"
redisCluster::run(conn, rk, ["PING"])
redisCluster::run(conn, rk, ["SET", rk + ":name", "kevin"])
redisCluster::run(conn, rk, ["GET", rk + ":name"])
redisCluster::run(conn, rk, ["HSET", rk + ":info", "age", "25", "city", "nyc"])
redisCluster::run(conn, rk, ["HGETALL", rk + ":info"])
```

### set

**语法**

```
redisCluster::set(conn, key, value, [ttlSec=0])
```

**详情**

设置键值对。

**参数**

**conn** 连接句柄。 

**key** STRING 标量。

**value** STRING 标量。

**ttlSec** INT 标量。可选。过期时间（秒数），默认 0，表示不设置过期。

**返回值**

运行成功返回 "OK"。

**示例**

```
redisCluster::set(conn, "k1", "v1")
redisCluster::set(conn, "k2", "v2", 5)
```

### get

**语法**

```
redisCluster::get(conn, key)
```

**详情**

获取指定键对应的值。

注意：只支持普通键值对。如果尝试获取其他类型的键值对，例如 Hash key，则会报 WRONGTYPE 的错误。

**参数**
 
**conn** 连接句柄。

**key** STRING 标量。支持使用 hash tag。

**返回值**

返回 STRING 或 NULL。

**示例**

```
redisCluster::get(conn, "{acct:1001}:name")
redisCluster::get(conn, "user:1")
```

### batchSet

**语法**

```
redisCluster::batchSet(conn, keys, values, [batchWin=2048], [numThreads=1])
```

**详情**

批量设置键值对。自动分槽并使用 pipeline 与多线程（如指定）并发执行。
当 *keys* 与 *values* 为标量时退化为单次 SET。

注意：建议对包含相同 hash tag 或同槽（slot）的键运行该命令，以最大化单路管线效率。

**参数**

**conn** 连接句柄。 

**keys** STRING 标量或向量。

**values** STRING 标量或向量。

**batchWin** INT 标量。可选。每次 pipeline 中的命令数。默认 2048。取值必须小于等于 64000。

**numThreads** INT 标量。可选。并发线程数。默认 1，即不开启多线程。取值必须小于等于 4096。

**返回值**

返回 "batchSet finish."。

**示例**

```
keys = "{t:1}:" + string(1..100000)
vals = take("x", 100000)
redisCluster::batchSet(conn, keys, vals, 2048, 4)
```

### batchHashSet

**语法**

```
redisCluster::batchHashSet(conn, ids, fieldData, [batchWin=2048], [numThreads=1])
```

**详情**

批量执行 Redis 的 HSET 操作。*ids* 中每个元素是一个键，对应 *fieldData* 中相应的行。字段名来自 *fieldData* 的列名，字段值来自对应列名内的值。

注意：建议对包含相同 hash tag 或同槽（slot）的键运行该命令，以最大化单路管线效率。

**参数**

**conn** 连接句柄。

**ids** STRING 向量。*fieldData* 中每行对应的键。

**fieldData** 表。所有列必须为 STRING。每列列名作为 Redis HSET 中的 field，值作为 HSET 中的 value。列数必须小于等于 1024。

**batchWin** INT 标量。可选。每次 pipeline 中的命令数。默认 2048。取值必须小于等于 64000。

**numThreads** INT 标量。可选。并发线程数。默认 1，即不开启多线程。取值必须小于等于 4096。

**返回值**

运行成功返回 "batchHashSet finish."。

**示例**

```
ids = "{tick:}" + string(1..5)
t = table(`f1`f2`f3`f4`f5 as name, take(1,5) as f1, take(2,5) as f2)
t = select name, string(f1) as f1, string(f2) as f2 from t
redisCluster::batchHashSet(conn, ids, t, 2048, 3)
```

### batchGet

**语法**

```
redisCluster::batchGet(conn, keys)
```

**详情**

批量获取指定键对应的值。

注意：建议对包含相同 hash tag 或同槽（slot）的键运行该命令，以最大化单路管线效率。

**参数**

**conn** 连接句柄。

**keys** STRING 向量，要获取的键向量。

**返回值**

返回 STRING 向量。键不存在位置为 NULL。

**示例**

```
keys = ["{u:1}:n","{u:2}:n","{u:3}:n"]
vals = redisCluster::batchGet(conn, keys)
```

### batchPush

**语法**

```
redisCluster::batchPush(conn, keys, values, [rightPush=true], [batchWin=2048], [numThreads=1])
```

**详情**

批量执行 Redis 的 RPUSH 或 LPUSH 操作。*keys* 中每个键对应一个 *values* 中字符串向量。
*rightPush* 为 true 执行 RPUSH，否则 LPUSH。

注意：建议对包含相同 hash tag 或同槽（slot）的键运行该命令，以最大化单路管线效率。

**参数**

**conn** 连接句柄。

**keys** 要设置的键，为 STRING 类型向量。

**values** 要设置的值，为一个元组，其元素必须是 STRING 类型向量。

**rightPush** BOOL 标量。可选。默认 true。

**batchWin** INT 标量。可选。每次 pipeline 中的命令数。默认 2048。取值必须小于等于 64000。

**numThreads** INT 标量。可选。并发线程数。默认 1，即不开启多线程。取值必须小于等于 4096。

**返回值**

运行成功返回 "batchPush finish."。

**示例**

```
ks = ["{q:1}", "{q:2}"]
vs = [ ["a","b"], ["c","d","e"] ]
redisCluster::batchPush(conn, ks, vs, true, 1024, 2)
```

### batchDel

**语法**

```
redisCluster::batchDel(conn, keys, [batchWin=2048], [numThreads=1], [useUnlink=true])
```

**详情**

批量删除键值对。*useUnlink* 设置为 true 时执行异步删除。

注意：建议对包含相同 hash tag 或同槽（slot）的键运行该命令，以最大化单路管线效率。

**参数**

**conn** 连接句柄。

**keys** STRING 向量。要删除的键向量。

**batchWin** INT 标量。可选。每次 pipeline 中的命令数。默认 2048。取值必须小于等于 64000。

**numThreads** INT 标量。可选。并发线程数。默认 1，即不开启多线程。取值必须小于等于 4096。

**useUnlink** BOOL 标量。可选。是否开启异步删除。默认 true。

**返回值**

运行成功返回 "batchDel finish."。

**示例**

```
ks = "{tmp:}" + string(1..10000)
redisCluster::batchDel(conn, ks, 2048, 4)
```

### release

**语法**

```
redisCluster::release(conn)
```

**详情**

释放一个 Redis Cluster 连接句柄。

**参数**

**conn** 通过 `redisCluster::connect` 创建的 Redis Cluster 连接句柄。

**示例**

```
conn = redisCluster::connect("127.0.0.1",6379)
redisCluster::release(conn)
```

### releaseAll

**语法**

```
redisCluster::releaseAll()
```

**详情**

释放当前会话内记录的所有 Redis Cluster 连接句柄。

**示例**

```
redisCluster::releaseAll()
```

### getHandle

**语法**

```
redisCluster::getHandle(token)
```

**详情**

获取 *token* 对应的 Redis Cluster 连接句柄，用于通过 *token* 找回句柄。

**参数**

**token** STRING 标量。句柄标识。通过 `redisCluster::getHandleStatus` 返回表中的第一列得知，用于唯一标识一个 Redis Cluster 连接。

**返回值**

返回 *token* 对应的连接句柄。

**示例**

```
conn = redisCluster::getHandle(token)
```

### getHandleStatus

**语法**

```
redisCluster::getHandleStatus()
```

**详情**

查看当前会话内已注册连接句柄信息。

**返回值**

返回表。列名为 token，address，createdTime。

- token 列是该连接的唯一标识，可通过 `redisCluster::getHandle(token)` 来获取句柄。
- address 是用来建立 Redis Cluster 连接的节点的 "ip:port" 地址。
- createdTime 是该连接创建的时间, 格式为 DATETIME。

**示例**

```
status = redisCluster::getHandleStatus()
```

## 使用示例

```
loadPlugin("redisCluster");
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

[🏠 Return to Homepage](/)