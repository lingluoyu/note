#### 配置业务队列

```java
@Configuration
public class DeadLetterRabbitMQConfig {

    @Bean
    public DirectExchange ddlDirectExchange() {
        return new DirectExchange("dl_direct_exchange", true, false);
    }

    @Bean
    public Queue ddlQueue() {
        return new Queue("dl.direct.queue", true);
    }

    @Bean
    public Binding ddlBinding() {
        return BindingBuilder.bind(ddlQueue()).to(ddlDirectExchange()).with("dl");
    }
}
```

#### 死信队列配置

```java
@Bean
public DirectExchange ttlDirectExchange() {
    return new DirectExchange("ttl_direct_exchange", true, false);
}

/**
 * 进入死信队列条件（可同时共存）
 *  1. 消息过期
 *  2. 队列消息条数超过最大条数
 */
@Bean
public Queue ttlQueue() {
    HashMap<Object, Object> args = new HashMap<>();
    // 设置队列过期时间
    args.put("x-message-ttl", 5000);
    // 设置队列最大长度
    args.put("x-max-length", 8);
    // 设置业务队列的死信交换机
    args.put("x-dead-letter-exchange", "dl_direct_exchange");
    // 设置死信队列路由 key 若是 fanout 模式则不需要配
    args.put("x-dead-letter-routing-key", "dl");
    return new Queue("ttl.direct.queue", true);
}

@Bean
public Binding ddlBinding() {
    return BindingBuilder.bind(ttlQueue()).to(ttlDirectExchange()).with("ttl");
}
```

