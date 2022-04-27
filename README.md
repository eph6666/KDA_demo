# 流式数据分析

# 创建MSK

详情参考：[第 1 步：创建亚马逊 MSK 集群 - Amazon Managed Streaming for Apache Kafka](https://docs.aws.amazon.com/zh_cn/msk/latest/developerguide/create-cluster.html)

1. 打开MSK[控制台](https://us-east-1.console.aws.amazon.com/msk/home)
2. 点击创建集群
3. 选择Custom create
4. 第一页填写集群名称，集群规模
5. 第二页填写网络信息，选择可用区和ap子网
6. 第三页需要配置权限控制，因为是内网环境，为了测试方便作出以下配置，在生产环境中应自行调整为合适的安全策略
    - 去掉IAM role-based authentication的勾选，保留Unauthenticated access
    - 去掉Between clients and brokers下TLS encryption勾选，改为Plaintext
7. 创建集群
    
    等待集群创建完毕后可以点击集群名称，进入集群详情页面查看详细信息
    
    点击**View client information获取Kafka broker地址和zookeeper地址**
    

# 测试MSK

详情参考：[第 3 步：创建主题 - Amazon Managed Streaming for Apache Kafka](https://docs.aws.amazon.com/zh_cn/msk/latest/developerguide/create-topic.html)

1. 因为是MSK集群仅开放内网，选择一台内网联通的EC2（推荐同VPC内）
    
    将MSK安全组附加给EC2
    
2. 下载Java和Apache Kafka
3. 创建Topic
    
    ```bash
    tar -xzf kafka_2.12-2.6.2.tgz
    cd kafka_2.12-2.6.2
    # create topic
    bin/kafka-topics.sh --create --zookeeper ZOOKEEPER_ENDPOINT --replication-factor 2 --partitions 1 --topic AWSKafkaTutorialTopic
    ```
    
4. List topic
    
    ```bash
    bin/kafka-topics.sh --list --zookeeper ZOOKEEPER_ENDPOINT
    ```
    
5. 生产消息
    
    ```bash
    cd bin
    echo 'security.protocol=PLAINTEXT' > client.properties
    ./kafka-console-producer.sh --broker-list BROKER_LIST --producer.config client.properties --topic AWSKafkaTutorialTopic
    ```
    
6. 消费消息
    
    ```bash
    ./kafka-console-consumer.sh --bootstrap-server BROKER_LIST --consumer.config client.properties --topic AWSKafkaTutorialTopic --from-beginning
    ```
    

# 创建流式分析服务

KDA有两种方式可以进行分析

1. 通过 Java 或 python 代码调研Flink API进行分析，编写java代码，打包上传到S3，部署应用，详情参考[文档](https://docs.aws.amazon.com/zh_cn/kinesisanalytics/latest/java/get-started-exercise.html#get-started-exercise-5)
2. 通过可视化平台，KDA Studio，使用Flink SQL进行分析

这里详细介绍第种

在msk控制台左侧解决方案中心中，选择第四个解决方案

「Amazon MSK, Amazon Kinesis Data Analytics, and Amazon S3」

点击部署，填写MSK Arn(可以在MSK控制台详情找到)

部署完成后在KDA页面中选择studio，点击运行

进入zeppelin notebook后创建新notebook

```bash
# Create source table
%flink.ssql(type=update)
CREATE TABLE IF NOT EXISTS stock_ods (
  `EVENT_TIME` STRING,
  `TICKER` STRING,
  `PRICE` DOUBLE,
) WITH (
  'connector' = 'kafka',
  'topic' = 'AWSKafkaTutorialTopic',
  'properties.bootstrap.servers' = 'BROKER_LIST',
  'scan.startup.mode' = 'earliest-offset',
  'json.ignore-parse-errors' = 'true',
  'json.fail-on-missing-field' = 'false',
  'format' = 'json'
);

# Create sink table to S3
%flink.ssql(type=update)
CREATE TABLE IF NOT EXISTS stock_dwd (
  `EVENT_TIME` STRING,
  `TICKER` STRING,
  `PRICE` DOUBLE
)  WITH (
  'connector' = 'filesystem',          
  'path' = 'S3_PATH',
  'json.ignore-parse-errors' = 'true',
  'json.fail-on-missing-field' = 'false',
  'format' = 'json'                   
)
```

查询数据作为测试，并插入数据到 sink table，以复制数据到S3

```bash
# Select data
%flink.ssql(type=update)
select * from stock_ods

# Set Checkpoint
%flink
senv.enableCheckpointing(3000)

# Insert data
%flink.ssql(type=update,parallelism=2)
INSERT INTO stock_dwd
  SELECT EVENT_TIME, TICKER, PRICE
    FROM stock_ods;
```

# Copy 数据到 Redshift

在 Redshift 中创建库和表：

```bash
CREATE DATABASE stock;

CREATE TABLE IF NOT EXISTS stock.public.stock_dwh (
    EVENT_TIME VARCHAR not null,
    TICKER VARCHAR not null,
    PRICE DOUBLE PRECISION not null
);
```

使用copy命令复制数据到 Redshift, 详情[参考文档](https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/tutorial-loading-run-copy.html)

```bash
copy table from 's3://<your-bucket-name>/load/key_prefix'
```
