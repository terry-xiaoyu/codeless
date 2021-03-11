# codeless

基于 emqx(https://github.com/emqx/emqx) 的物联网数据处理中间件（实验阶段）

提供管理各种数据源、数据库、其他消息中间件等的功能。
外加一套简单、灵活的数据处理 DSL。

数据源：
- mqtt
- gateways: 各种协议

数据库：
- sql
- nosql

其他 Broker 桥接:
- kafka
- rabbitmq

其中数据源、桥接提供 triggers 和 一些消息发送等 functions.
数据库提供一些 query functions.

DSL 需要选择一个或者多个 trigger 作为输入，调用数据库和中间件等提供的 function 做消息转移。

DSL 最好提供类似 switch 语言的 playground 那种 debug 能力。

I. 场景举例

当收到主题为 't/1' 的 MQTT 消息，首先做 Avro 解码。
解码后发送给 lwm2m 设备（指令下发），然后根据消息类型分别发送到 mysql 和 kafka 做消息备份。

playground:

```
## define a function sending data according to the data type
func send_data('write', data):
    ## call the 'produce' API of a kafka resource 'res_4' 
    kafka:res_4.produce('t/1', data)
end

func send_data('read', data):
    mysql:res_3.query('insert into table values (data.x, data.y))
end

trigger 'mqtt:publish' when topic = 't/1':

    ## decode it by avro schema
    data = schema_registry.decode(avro, 'schema-1' payload)

    ## log it out for debugging
    system.logger('info', data)

    ## send to the lwm2m device with endpoint name = 'client-a'
    ## call the 'send' API of the lwm2m resource named 'res_2'
    lwm2m:res_2.send('client-a', '/3/0/1', data)

    ## call the self-defined function
    send_data(data.type)

end
```

II. DSL 功能

1. 语法

基本：

- 数据类型: strings, numbers, maps, lists
- 数据操作: 下标读取，Map 插入等等
- 函数定义:
- 函数调用:
- 计算: 数学计算
- 条件分支: case
- 循环: while, foreach

高级：

- pipe
- list comprehension

2. stdlibs

- strings
- binary
- crypto
- time
- jq (json string processing)
- emqx (getting connection info, publishing messages, etc. All the functions that are provided by HTTP API should be supported here)

3. triggers

- mqtt.client_connected
- mqtt.publish
- mqtt.auth_request
...

- coap.put
- coap.post
- coap.auth_request
...

- lwm2m.registered
- lwm2m.notify
- lwm2m.auth_request
...

- kafka.msg_in
...

- timers.interval
- timers.schedule_at
...

4. function provided by plugins/gateways/sinks

- coap:res_1.send('post', peer_ip_port, '/path').
- lwm2m:res_2.send('client-a', '/3/0/1', payload).
- mysql:res_3.query('SELECT * FROM table', timeout).
- kafka:res_4.produce('t/1', payload)

5. in memory key-value db

- kv.put(key, value)
- kv.get(key, default)
