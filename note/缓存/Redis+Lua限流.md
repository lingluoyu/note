### 计数器

**Lua脚本**：

```bash
-- 计数器限流
-- 此处支持的最小单位时间是秒, 若将 expire 改成 pexpire 则可支持毫秒粒度.
-- KEYS[1]  string  限流的key
-- ARGV[1]  int     限流数
-- ARGV[2]  int     单位时间(秒)

local cnt = tonumber(redis.call("incr", KEYS[1]))

if (cnt == 1) then
    -- cnt 值为1说明之前不存在该值, 因此需要设置其过期时间
    redis.call("expire", KEYS[1], tonumber(ARGV[2]))
elseif (cnt > tonumber(ARGV[1])) then
    return -1
end 

return cnt
```

> 返回 -1 表示超过限流, 否则返回当前单位时间已通过的请求数

key 可以但不限于以下的情况

- ip + 接口
- user_id + 接口

**优点**

- 实现简单

**缺点**

- 粒度不够细的情况下, 会出现在同一个窗口时间内出现双倍请求数

**注意**

- 尽量保持时间粒度精细

### 漏桶

**Lua脚本**：

```bash
local key = KEYS[1] --限流KEY（一秒一个）
local limit = tonumber(ARGV[1]) --限流大小
local current = tonumber(redis.call('get', key) or "0")
if current + 1 > limit then --如果超出限流大小
    return 0
else --请求数+1，并设置2秒过期
    redis.call("INCRBY", key,"1")
    redis.call("expire", key,"2")
end
return 1
```
**IP限流Lua脚本**：
```bash
local key = "rate.limit:" .. KEYS[1]
local limit = tonumber(ARGV[1])
local expire_time = ARGV[2]

local is_exists = redis.call("EXISTS", key)
if is_exists == 1 then
    if redis.call("INCR", key) > limit then
        return 0
    else
        return 1
    end
else
    redis.call("SET", key, 1)
    redis.call("EXPIRE", key, expire_time)
    return 1
end
```

### 令牌桶

**Lua脚本**：

```bash
-- 令牌桶限流: 不支持预消费, 初始桶是满的
-- KEYS[1]  string  限流的key

-- ARGV[1]  int     桶最大容量
-- ARGV[2]  int     每次添加令牌数
-- ARGV[3]  int     令牌添加间隔(秒)
-- ARGV[4]  int     当前时间戳

local bucket_capacity = tonumber(ARGV[1])
local add_token = tonumber(ARGV[2])
local add_interval = tonumber(ARGV[3])
local now = tonumber(ARGV[4])

-- 保存上一次更新桶的时间的key
local LAST_TIME_KEY = KEYS[1].."_time";         
-- 获取当前桶中令牌数
local token_cnt = redis.call("get", KEYS[1])    
-- 桶完全恢复需要的最大时长
local reset_time = math.ceil(bucket_capacity / add_token) * add_interval;

if token_cnt then   -- 令牌桶存在
    -- 上一次更新桶的时间
    local last_time = redis.call('get', LAST_TIME_KEY)
    -- 恢复倍数
    local multiple = math.floor((now - last_time) / add_interval)
    -- 恢复令牌数
    local recovery_cnt = multiple * add_token
    -- 确保不超过桶容量
    local token_cnt = math.min(bucket_capacity, token_cnt + recovery_cnt) - 1
    
    if token_cnt < 0 then
        return -1;
    end
    
    -- 重新设置过期时间, 避免key过期
    redis.call('set', KEYS[1], token_cnt, 'EX', reset_time)                     
    redis.call('set', LAST_TIME_KEY, last_time + multiple * add_interval, 'EX', reset_time)
    return token_cnt
    
else    -- 令牌桶不存在
    token_cnt = bucket_capacity - 1
    -- 设置过期时间避免key一直存在
    redis.call('set', KEYS[1], token_cnt, 'EX', reset_time);
    redis.call('set', LAST_TIME_KEY, now, 'EX', reset_time + 1);    
    return token_cnt    
end
```

