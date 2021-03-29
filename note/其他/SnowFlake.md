### 原理

snowflake生产的ID是一个18位的long型数字，二进制结构表示如下(每部分用-分开):

0 - 00000000 00000000 00000000 00000000 00000000 0 - 00000 - 00000 - 00000000 0000

**未使用：**1bit，因为最高位是符号位，0表示正，1表示负，所以这里固定为0
 **时间戳：**41bit，服务上线的时间毫秒级的时间戳（为当前时间-服务第一次上线时间），这里为（2^41-1）/1000/60/60/24/365  = 69年
 **工作机器id：**10bit，表示工作机器id，用于处理分布式部署id不重复问题，可支持2^10 = 1024个节点（5位datacenterId：最大支持2^5 ＝32个，二进制表示从00000-11111，也即是十进制0-31，5位workerId：最大支持2^5 ＝32个，原理同datacenterId）
 **序列号：**12bit，用于离散同一机器同一毫秒级别生成多条Id时，可允许同一毫秒生成2^12 = 4096个Id，则一秒就可生成4096*1000 = 400w个Id

```java
/**
 * @author lius
 * @date 2021/3/29
 */
public class SnowFlake {
    // 起始的时间戳
    private final static long START_STMP = 1480166465631L;
    // 每一部分占用的位数，就三个
    private final static long SEQUENCE_BIT = 12;// 序列号占用的位数
    private final static long MACHINE_BIT = 5; // 机器标识占用的位数
    private final static long DATACENTER_BIT = 5;// 数据中心占用的位数
    // 每一部分最大值
    private final static long MAX_DATACENTER_NUM = -1L ^ (-1L << DATACENTER_BIT);
    private final static long MAX_MACHINE_NUM = -1L ^ (-1L << MACHINE_BIT);
    private final static long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);
    // 每一部分向左的位移
    private final static long MACHINE_LEFT = SEQUENCE_BIT;
    private final static long DATACENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;
    private final static long TIMESTMP_LEFT = DATACENTER_LEFT + DATACENTER_BIT;
    // 数据中心
    private long dataCenterId;
    // 机器标识
    private long machineId;
    // 序列号
    private long sequence = 0L;
    // 上一次时间戳
    private long lastTimestamp = -1L;

    private static SnowFlake snowFlake;

    static {
        snowFlake = new SnowFlake(1L, 1L);
    }

    public SnowFlake(long dataCenterId, long machineId) {
        if (dataCenterId > MAX_DATACENTER_NUM || dataCenterId < 0) {
            throw new IllegalArgumentException("dataCenterId can't be greater than MAX_DATACENTER_NUM or less than 0");
        }
        if (machineId > MAX_MACHINE_NUM || machineId < 0) {
            throw new IllegalArgumentException("machineId can't be greater than MAX_MACHINE_NUM or less than 0");
        }
        this.dataCenterId = dataCenterId;
        this.machineId = machineId;
    }
    //产生下一个ID
    public synchronized long nextId() {
        long currTimestamp = getNewTimestamp();
        if (currTimestamp < lastTimestamp) {
            throw new RuntimeException("Clock moved backwards.  Refusing to generate id");
        }

        if (currTimestamp == lastTimestamp) {
            //if条件里表示当前调用和上一次调用落在了相同毫秒内，只能通过第三部分，序列号自增来判断为唯一，所以+1.
            sequence = (sequence + 1) & MAX_SEQUENCE;
            //同一毫秒的序列数已经达到最大，只能等待下一个毫秒
            if (sequence == 0L) {
                currTimestamp = getNextMill();
            }
        } else {
            //不同毫秒内，序列号置为0
            //执行到这个分支的前提是currTimestamp > lastTimestamp，说明本次调用跟上次调用对比，已经不再同一个毫秒内了，这个时候序号可以重新回置0了。
            sequence = 0L;
        }

        lastTimestamp = currTimestamp;
        //就是用相对毫秒数、机器ID和自增序号拼接
        return (currTimestamp - START_STMP) << TIMESTMP_LEFT //时间戳部分
                | dataCenterId << DATACENTER_LEFT      //数据中心部分
                | machineId << MACHINE_LEFT            //机器标识部分
                | sequence;                            //序列号部分
    }

    private long getNextMill() {
        long mill = getNewTimestamp();
        while (mill <= lastTimestamp) {
            mill = getNewTimestamp();
        }
        return mill;
    }

    private long getNewTimestamp() {
        return System.currentTimeMillis();
    }

    public static synchronized Long generateId() {
        return snowFlake.nextId();
    }
}

class Test {
    public static void main(String[] args) {
        for (int i = 0; i < 1 << 12; i ++) {
            System.out.println(SnowFlake.generateId());
        }
    }
}
```

