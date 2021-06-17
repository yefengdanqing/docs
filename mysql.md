# 背景

由于大禹系统用来支持当前单一的渠道系统较重，框架不易维护，整体实现也比较复杂，工作量比较大；对于大禹的分拣也能通过已经有的flink分拣实现了，因此为了节省资源，维护方便，下掉大禹系统，进行重构。

# 名词解释

点击库：保存渠道的点击数据，imei、oaid、androidid对应同一条点击数据，3天，最好只用这一个点击库。

判新库：保存一年的mbsdkinstall的数据【appkey+hdid+ts]

# 设计目标

1. 好 数据准确性：不丢 / exactly-once

2. 快 数据时效性：分钟级延迟 / freshness，高时效性伴随着**低准确度**，二者不可兼得

3. 稳 数据可追溯和回溯：支持延迟数据的实时追溯和异常数据的实时回溯，能抗住追溯压力

## 目标框架

![重构目标](/Users/skt/Documents/大禹重构/重构目标.jpg)

从etl-kafka里面读取数据到flink中，flink 处理多个并行流

## 性能指标

## 其他系统联系

Kafka：分布式消息队列，衔接数据采集子系统和数据处理子系统。提供可靠的数据pub/sub服务，在流量高峰时，可起到数据缓冲池作用。

Redis：存储DAU、HAU的中间状态数据。

Mysql：存储DAU、HAU最终结果，供业务方查询使用。

判新库:（一年37G,yylive申请128G)，joyy新表(50G), mysql 

点击库:（每天30G * 3), 都用redis

总结：joyy数据量比较少，判新和点击库申请一个128G的redis；yylive判新库和点击库都用mysql；

## 相关软件及硬件

## 系统限制

## 数据规模估计

| act          | 天级分区大小                | ***\*条数\****          | ***\*百度\*******\*(\*******\*手y\*******\*)\**** | ***\*JOYY\**** |
| ------------ | --------------------------- | ----------------------- | ------------------------------------------------- | -------------- |
| hiidopush    | 12G(分区大小，select * 40G) | 31042333(单条527Byte)   |                                                   |                |
| mbsdkinstall | 2G                          | 7296544（单条601 Byte） |                                                   |                |
| mbsdkdo      | 70-300G                     | 194718309               |                                                   |                |
| mbskdrun     | 50G                         | 92226868                |                                                   |                |

| 库名   | 条数     | 大小             | 备注                                                         |
| ------ | -------- | ---------------- | ------------------------------------------------------------ |
| 判新库 | 6445422  | 590M/day（32*3） | yy总hdid个数：207898838;yy每天去重数据1153727（18%,约105M)；其他：10M * 3MB |
| 点击库 | 31042333 | 15G/day          |                                                              |

分布式流计算所有数据会进行Shuffle，怎么才能保障左右两边流的要JOIN的数据会在相同的节点进行处理呢？在双流JOIN的场景，我们会利用JOIN中ON的联接key进行partition，确保两个流相同的联接key会在同一个节点处理。

# 设计思路
判新.jpg![判新](https://user-images.githubusercontent.com/13494606/122421900-4a77cb00-cfbf-11eb-8f8a-0e1569320ff7.jpg)

## 判新模块

![判新](/Users/skt/Documents/大禹重构/判新.jpg)


## 处理模块

![处理模块](/Users/skt/Documents/大禹重构/处理模块.jpg)

激活和拉活用同一个逻辑，用同一个job

## 统计模块

uv、pv通过Flink-sql进行统计

## 回调模块

从join-kafka中读取数据，支持激活和次留，根据channel_index的逻辑

通过模块进行回调，用链接池实现



# 详细设计

## 判新模块

输入1：python导入离线用户数据到redis或者通过调度导入离线老用户到mysql

<img src="/Users/skt/Documents/大禹重构/image2019-11-5_16-47-17.png" alt="image2019-11-5_16-47-17" style="zoom:80%;" />

输入2：读取分拣kafka的mbsdkinstall实时数据

输入3：读取数据库中imei和oaid的白名单表

Redis的key: joyy:channel:mbsdkinstall:judgenew:appkey.hdid 

redis的value: ts:1615802729;imei:863690040465571;oaid:863690040465571;android:863690040465571;idfa:863690040465571;caid:863690040465571

[from 、ori_from、redis和mysql]

![详细设计-判新](/Users/skt/Documents/大禹重构/详细设计-判新.jpg)

<img src="/Users/skt/Documents/大禹重构/splitter.png" alt="splitter" style="zoom: 50%;" />

手y:hdid、imei、oaid;和splitter 保持一致

输出：处理后输出到kafka

## 处理模块

### 输入

输入1：mbsdkinstall或者mbsdkdo kafka数据

输入2：hiidopush kafka数据

点击数据字段：**redis_key**:imei_md5、oaid/oaid_md5、android_md5、idfa、idfa_md5、caid

​					**redis_value**:adrom、hiidoeid、material_id、click_imei、click_oaid、click_androidid、click_idfa、callbackurl、 	               					click_clickid

mbsdkinstall：**redis_key**:joyy:channel:mbsdkinstall:redo:appkey.hdid

​							**redis_value**:start-ts、end-ts、imei、oaid、androidid、idfa、caid



### 处理流程:

渠道修正

```sql
SELECT
DISTINCT a.channel, b.product_key
FROM
hiidosdk.sync_m_channel_status_yy_info a
left join
hiidosdk.m_product b
on a.product_id = b.id
WHERE
a.is_info=1 AND a.dt="${hiveconf:DATE}" AND a.product_id in (171, 25677)
```

<img src="/Users/skt/Documents/大禹重构/渠道修正第一步.png" alt="image-20210317155020500" style="zoom:50%;" />

上面的数据写入dayu_joiner_feed_channel

<img src="/Users/skt/Library/Application Support/typora-user-images/image-20210317151358424.png" alt="image-20210317151358424" style="zoom: 67%;" />

![image-20210317210501296](/Users/skt/Documents/大禹重构/渠道修正3.png)



```sql
SELECT
if(t1.product_id=171, t1.hdid, concat(t2.product_key, '_', t1.hdid)),
t1.channel,
t1.channel_pack,
IF(imei IS null OR imei = "", "null", imei),
IF(arid IS null OR arid = "", "null", arid),
CAST(`time` AS BIGINT) * 1000
FROM
hiidosdk.yy_mbsdkinstall_hdid_total_original_channel_corrected t1
LEFT JOIN
hiidosdk.m_product t2
ON
t1.product_id=t2.id
WHERE
t1.product_id in (25677, 171) AND
t1.dt="${hiveconf:DATE}"
```

![image-20210317161114046](/Users/skt/Documents/大禹重构/渠道修正第二步.png)

![image-20210317161225497](/Users/skt/Documents/大禹重构/渠道修正第三步.png)

DATE=`date "+%Y%m%d" -d "$1 -3 days"`上面的数据写入dayu_hdid_belong



![渠道修正4](/Users/skt/Documents/大禹重构/渠道修正4.jpg)

整体处理模块

![详细-处理模块](/Users/skt/Downloads/详细-处理模块.jpg)



### 输出

具体的数据接口和schema

join后的数据写入kafka

## 统计模块

<img src="/Users/skt/Documents/大禹重构/pvuv.png" alt="企业微信截图_c83aed7c-e600-42e9-9cda-fd0e80693c4a" style="zoom:150%;" />

一个json一条数据

## 回调模块

定义joyy的业务指标映射渠道方的的业务指标

![企业微信截图_3e466ae2-1223-4835-a9bc-3b8634ce8cf9](/Users/skt/Documents/大禹重构/回调1.png)



![未命名文件 (1)](/Users/skt/Documents/大禹重构/callback.png)

微服务---k8s 单独的服务，

# 系统监控

# 稳定性

# 部署

# 未解决问题

回溯:kafka偏移量，支持配置

flink没有最大化使用

redo拼接时间确定，间隔性的缓解压力，限速的topic

所有的拼接用一个存储介质

redo 用kafka还是redis保存

redis用同一个



|      | kafka                                     | redis                                                        |
| ---- | ----------------------------------------- | ------------------------------------------------------------ |
| 优点 | 1、生产与消费是异步的，互不干扰           | 1、实现hdid去重                                              |
|      | 2、方便获取日志的全部信息                 | 2、方便灵活的修改数据，不利于数据的回溯                      |
|      | 3、方便重启定位问题                       | 3、定时scan，260万10s左右，不包括拼接时间                    |
| 缺点 | 1、hdid无法去重，数据量过大，导致性能问题 | 1、存储部分信息                                              |
|      | 2、回溯时手动重启offset                   | 2、存储量过大时，成本增加                                    |
|      |                                           | 3、同一个hdid存在拼接错误素材id，回调渠道错误，总的diff数不会错，可能存在分渠道较大的diff |



https://blog.csdn.net/qq_36039236/article/details/108325477



```c++
if (!rejoin && !write_attribution(*log_detail, nullptr)) {
        return false;
    }
```







