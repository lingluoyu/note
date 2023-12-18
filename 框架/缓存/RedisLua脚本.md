Redis提供了丰富的指令集，但是仍然不能满足所有场景，在一些特定场景下，需要自定义一些指定来完成某些功能。因此，Redis提供了Lua脚本支持，用户可以自己编写脚本来实现想要的功能。

### Redis中使用Lua

在 Redis 中执行 Lua 脚本有两种方法：`eval`和`evalsha`。EVAL 和 EVALSHA 命令是从 Redis 2.6.0 版本开始的，使用内置的 Lua 解释器，可以对 Lua 脚本进行求值。

#### EVAL命令

通过内置的 `Lua` 解释器，可以使用 `EVAL` 命令（也可以使用`redis-cli` 的`--eval` 参数）对 `Lua` 脚本进行解析。需要注意的点是执行`Lua`也会使`Redis`阻塞。

```bash
## 格式
eval  脚本内容  key个数  key列表  参数列表
## 使用了key列表和参数列表来为Lua脚本提供更多的灵活性
127.0.0.1:6379> eval 'return "hello " .. KEYS[1] .. ARGV[1]' 1 redis world
"hello redisworld"
```

#### EVALSHA命令

根据给定的 SHA1 校验码，对缓存在服务器中的脚本进行求值。 将脚本缓存到服务器的操作可以通过 SCRIPT LOAD 命令进行。 这个命令的其他地方，比如参数的传入方式，都和`EVAL`命令一样。

`evalsha` 使用拆分成两个步骤：

1. 首先要将`Lua`脚本加载到`Redis`服务端，得到该脚本的`SHA1`校验码。
2. 然后使用`evalsha`命令使用`SHA1`作为参数可以直接执行对应`Lua`脚本。

这样做的好处是可以避免每次发送`Lua`脚本的开销，而脚本也会常驻在服务端，脚本功能得到了复用。缺点是要怎么管理这些脚本和命令过多的话会占用`Redis`的内存。

```bash
## 在当前目录定义一个Lua文件
vim myLua.lua
## 在文件中添加命令并保存
return "hello " ..KEYS[1].." "..ARGV[1]
## 1. script load命令可以将脚本内容加载到Redis内存中
redis-cli script load "$(cat myLua.lua)"
## 2. 进入Redis客户端
redis-cli
## 3. evalsha执行脚本格式
evalsha  脚本SHA1值 key个数 key列表 参数列表
## 4. 执行myLua.lua
127.0.0.1:6379> evalsha 5ea77eda7a16440abe244e6a88fd9df204ecd5aa 1 redis world
"hello redis world"
```

### Lua 的 Redis API

`Lua` 可以使用`redis.call` 和 `redis.pcall` 两个函数实现对`Redis`的访问。

`redis.call()` 与 `redis.pcall()`很类似，他们唯一的区别是当redis命令执行结果返回错误时， `redis.call()`会停止脚本执行，将返回给调用者一个错误：

```bash
> del foo
(integer) 1
> lpush foo a
(integer) 1
> eval "return redis.call('get','foo')" 0
(error) ERR Error running script (call to f_6b1bf486c81ceb7edf3c093f4c48582e38c0e791): ERR Operation against a key holding the wrong kind of value
```

而`redis.pcall()`会将捕获的错误以Lua表的形式返回（忽略错误继续执行脚本）:

```bash
redis 127.0.0.1:6379> EVAL "return redis.pcall('get', 'foo')" 0
(error) ERR Operation against a key holding the wrong kind of value
```

### 其他命令

#### SCRIPT DEBUG

使用`EVAL`可以开启对脚本的调试。Redis包含完整Lua Debugger和codename LDB，这大大降低了复杂脚本编写的难度。 在调试模式下，Redis 既做调试服务器又做客户端，像`redis-cli` 可以单步执行，设置断点，观察变量等等

```bash
#格式
SCRIPT DEBUG YES|SYNC|NO
```

- `YES`. 打开非阻塞异步调试模式，调试Lua脚本(回退数据修改)
- `SYNC`. 打开阻塞同步调试模式，调试Lua脚本(保留数据修改稿)
- `NO`. 关闭脚本调试模式

#### SCRIPT EXISTS

检查脚本是否存在脚本缓存里面。这个命令可以接受一个或者多个脚本SHA1信息，返回一个1或者0的列表，如果脚本存在或不存在。

```bash
#格式
SCRIPT EXISTS script [script ...]
```

#### SCRIPT FLUSH 

清空Lua脚本缓存 Flush the Lua scripts cache。

```bash
#格式
SCRIPT FLUSH 
```

#### SCRIPT KILL 

杀死当前正在运行的 Lua 脚本，当且仅当这个脚本没有执行过任何写操作时，这个命令才生效。

`SCRIPT KILL` 执行之后，当前正在运行的脚本会被杀死，执行这个脚本的客户端会从`EVAL`中退出，并收到一个错误作为返回值。

```bash
#格式
SCRIPT KILL 
```

#### SCRIPT LOAD

将脚本 script 添加到脚本缓存中，但不会立即执行。

```bash
#格式
SCRIPT LOAD script
```

