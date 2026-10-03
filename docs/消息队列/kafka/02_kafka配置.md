3.1.x官方文档：[https://kafka.apache.org/31/documentation.html#configuration](https://kafka.apache.org/31/documentation.html#configuration)

常用配置及说明：

| 配置项                   | 默认值            | 作用                                                                 | 建议值                                                           | 备注 |
|-----------------------|----------------|--------------------------------------------------------------------|---------------------------------------------------------------|----|
| session.timeout.ms	   | 1000ms（10s）    | Broker 等待消费者心跳的最大时间，超时则判定消费者离线，触发再平衡（需配合 heartbeat.interval.ms 使用） | 建议 ≥ 最慢消息处理时间 × 1.5                                           |    |
| heartbeat.interval.ms | 3000ms（3s）     | 消费者发送心跳的频率                                                         | 建议 ≤ session.timeout.ms 的 1/3                                 |    |
| max.poll.interval.ms  | 300000ms（5min） | 控制 poll() 调用的最大间隔，超时则踢出组                                           | 单条拉取，建议 ≥ 最慢消息处理时间 × 1.5；多条拉取，建议 >= max.poll.records × 单条处理时间 |    |
| request.timeout.ms    | 30000（30s）     | 客户端等待 Broker 响应的通用超时                                               |                                                               |    |
| max.poll.records      | 500            | 限制每次 poll() 返回的 ConsumerRecords 中包含的最大消息数                          | 注意拉取的条数，以及单条消息处理的时间，防止超时                                      |    |


