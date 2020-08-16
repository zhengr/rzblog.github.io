---
title: 现代工业物联网(IIoT)大数据实战-风力发电机优化
date: 2020-08-05 00:00:00
author: 郑瑞(robin RUI ZHENG)
categories: 大数据
tags:
  - 大数据
  - 数据湖
  - IIOT
---

工业物联网(IIoT)在过去的几年里，从一个主要在石油和天然气行业进行试点的草根技术栈，如今发展到在制造、化工、公用事业、运输和能源行业广泛采用和生产使用。传统的物联网系统，如Scada、Historians甚至Hadoop，由于以下因素，无法提供大多数组织所需的大数据分析能力，无法对工业资产进行预测性优化。

| **挑战**                   | 所需能力                                                     |
| -------------------------- | ------------------------------------------------------------ |
| 数据量显著增大且更加频繁   | 从物联网设备中可靠地采集和存储亚秒级颗粒读数的能力，且成本效益要高，每天的数据流为TB级。 |
| 数据处理需求更加复杂       | 符合ACID标准的数据处理--基于时间的窗口、聚合、枢轴、回填、移动，并能轻松地重新处理旧数据。 |
| 更多的用户角色希望访问数据 | 数据是一种开放的格式，易于与运营工程师、数据分析师、数据工程师和数据科学家共享，而不会形成孤岛。 |
| 决策需要可扩展的ML         | 能够在细化的历史数据上快速、协作地训练预测模型，以做出智能的资产优化决策。 |
| 降低成本的要求比以往更高   | 低成本的按需管理平台，可随数据和工作负载独立扩展，不需要大量的前期资金。 |

企业正在转向类似Microsoft Azure这样的云计算平台，以利用它们所提供的可扩展的、支持IIoT的技术，使获取、处理、分析和服务时间序列数据源（如Historians和SCADA系统）变得简单。

- 在第1部分中，我们将讨论端到端技术堆栈以及Azure Databricks在现代物联网分析的工业应用的架构和设计中所扮演的角色。
- 在第2部分中，我们将深入探讨部署现代IIoT分析技术，将现场设备的实时IIoT机对机数据摄入数据湖存储中，并直接在数据湖上进行复杂的时间序列处理。
- 在第3部分中，我们将探讨利用工业物联网数据进行机器学习和分析。


## 第一部分

### 案例 - 风力发电机优化

大多数IIoT分析项目都是为了最大限度地提高工业资产的短期利用率，同时最大限度地降低其长期维护成本。在本文中，我们重点关注一个假设的能源供应商，试图优化其风力涡轮机。最终目标是确定一组最佳的涡轮机运行参数，以最大限度地提高每个涡轮机的功率输出，同时最大限度地减少其故障时间。

![](https://i.loli.net/2020/08/16/GWpl38mcxF2y4dR.png)

该项目的最终交付物是：

1. 一个自动的数据摄取和处理管道，将数据传送给所有终端用户。

2. 预测模型，在当前天气和运行条件下，估计每个涡轮机的功率输出。

3. 预测模型，在当前天气和运行条件下，估计每个涡轮机的剩余寿命。

4. 确定最佳运行条件的优化模型，以最大限度地提高功率输出，最大限度地降低维护成本，从而实现总利润的最大化。

5. 为高管们提供了一个实时分析仪表盘，以可视化地显示其风电场的当前和未来状态，如下图所示。

![高管看板](https://i.loli.net/2020/08/16/LwglJ5TfroHVzAY.png)

### 架构 - 获取、存储、准备、培训、服务、可视化。

下面的架构说明了许多组织使用的现代最佳平台，该平台利用了Azure为IIoT分析提供的所有功能。如果企业有独立的自研技术能力，按照类似的架构通过增强集成开源项目也可以低成本实现。

![](https://i.loli.net/2020/08/16/FgYRmkV2NjMdcUr.png)

该架构的一个关键组件是 Azure 数据湖存储 (ADLS)，它在 Azure 中实现了一次写入、多次访问的分析模式。然而，仅靠数据湖并不能解决时间序列流数据带来的现实世界挑战。砖厂的Delta存储格式在ADLS中存储的所有数据源上提供了一层弹性和性能。具体到时间序列数据，与ADLS上的其他存储格式相比，Delta具有以下优势。

| 所需能力       | **ADLS Gen 2的其他格式**                                     | **ADLS Gen 2的Delta格式**                                    |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 流批一体化     | 数据湖经常与像CosmosDB这样的流式存储一起使用，从而形成一个复杂的架构。 | 符合ACID标准的事务使数据工程师能够在ADLS上执行流式摄取和历史上的批量加载到相同的位置。 |
| 模式执行和演变 | 数据湖不强制执行模式，要求所有数据都推送到关系型数据库中以保证可靠性。 | 模式是默认执行的。随着新的物联网设备被添加到数据流中，模式可以安全地演进，因此下游应用不会失败。 |
| 高效 Upserts   | 数据湖不支持在线更新和合并，需要删除和插入整个分区才能执行更新。 | MERGE命令对于处理延迟的物联网读数、用于实时丰富的修改维度表或需要重新处理数据的情况非常有效。 |
| 文件压缩       | 将时间序列数据串流到数据湖中，会产生数百甚至数千个微小的文件。 | Delta中的自动压缩功能可以优化文件大小，以提高吞吐量和并行性。 |
| 多维聚类       | 数据湖仅在分区上提供下推式过滤功能。                         | 在时间戳或传感器ID等字段上对时间序列进行ZORDER，可以对这些列进行过滤和连接，比简单的分区技术快100倍。 |

### 总结

在第一部分，回顾了传统IIoT系统面临的一些不同挑战。并介绍了现代IIoT分析的用例和目标，分享了组织已经在大规模部署的可重复架构，并探讨了Delta格式对每个所需功能的好处。

在第二部分中，将把现场设备的实时IIoT数据摄入Azure，并直接在数据湖上进行复杂的时间序列处理。将一切联系在一起的关键技术是砖厂的Delta Lake。ADLS上的Delta提供了可靠的流式数据管道和对海量时间序列数据的高性能数据科学和分析查询。最后，它使组织能够真正采用**Lakehouse模式**，将最佳的Azure工具引入到一次写入、多次访问的数据存储中。

## 第二部分

第一部分中，我们介绍了大数据用例和现代IIoT分析的目标，分享了企业用于大规模部署IIoT的实际可重复架构，并探讨了Delta格式对现代IIoT分析所需的每个数据湖功能的好处。我们再回顾一下架构图：

![](https://i.loli.net/2020/08/16/xzydpaCbU56ofsE.png)

### 部署

我们使用Azure的Raspberry PI IoT模拟器来模拟实时的机器对机器传感器读数，并将其发送到Azure IoT Hub。

#### 数据获取：Azure IoT Hub到数据湖

我们的部署将天气（风速和风向、温度、湿度）和风力涡轮机远程信息（角度和转速）的传感器读数发送到 IoT 云计算中心。Azure Databricks 可以原生地将来自 IoT 集线器的数据直接流到 ADLS 上的 Delta 表中，并显示数据的输入与处理率。

```scala
# Read directly from IoT Hubs using the EventHubs library for Azure Databricks
iot_stream = (
    spark.readStream.format("eventhubs")                                        # Read from IoT Hubs directly
    .options(**ehConf)                                                        # Use the Event-Hub-enabled connect string
    .load()                                                                   # Load the data
    .withColumn('reading', F.from_json(F.col('body').cast('string'), schema)) # Extract the payload from the messages
    .select('reading.*', F.to_date('reading.timestamp').alias('date'))        # Create a "date" field for partitioning
)

# Split our IoT Hubs stream into separate streams and write them both into their own Delta locations
write_turbine_to_delta = (
    iot_stream.filter('temperature is null')                          # Filter out turbine telemetry from other streams
    .select('date','timestamp','deviceId','rpm','angle')            # Extract the fields of interest
    .writeStream.format('delta')                                    # Write our stream to the Delta format
    .partitionBy('date')                                            # Partition our data by Date for performance
    .option("checkpointLocation", ROOT_PATH + "/bronze/cp/turbine") # Checkpoint so we can restart streams gracefully
    .start(ROOT_PATH + "/bronze/data/turbine_raw")                  # Stream the data into an ADLS Path
)
```

Delta允许我们的IoT数据在IoT Hub采集后的几秒钟内被查询。

```sql
%sql 
-- We can query the data directly from storage immediately as it streams into Delta 
SELECT * FROM delta.`/tmp/iiot/bronze/data/turbine_raw` WHERE deviceid = 'WindTurbine-1'
```

<img src="https://i.loli.net/2020/08/16/JbG1p492CQek58I.png"  />

我们现在可以建立一个下游管道，丰富和聚合我们的IIoT应用数据，进行数据分析。

#### 数据存储和处理。Azure Databricks和Delta Lake

Delta 支持采用多跳流水线方法进行数据工程，数据质量和聚合度随着流水线的流动而提高。我们的时间序列数据将流经以下青铜级、白银级和黄金级数据。（青铜-白银-黄金 可以理解为数据成熟度的不同阶段， Bronze (raw) to Silver (aggregated) to Gold (enriched)）

![](https://i.loli.net/2020/08/16/8sgJKbticdAEpSH.png)

我们从青铜到白银的管道将简单地把我们的涡轮机传感器数据汇总到1小时间隔。我们将执行一个流式MERGE命令，将汇总的记录上传到银色的Delta表中。

![](https://i.loli.net/2020/08/16/E1oZAqWFVh5fxcp.png)

```scala
# Create functions to merge turbine and weather data into their target Delta tables
def merge_records(incremental, target_path): 
    incremental.createOrReplaceTempView("incremental")
    
# MERGE consists of a target table, a source table (incremental),
# a join key to identify matches (deviceid, time_interval), and operations to perform 
# (UPDATE, INSERT, DELETE) when a match occurs or not
    incremental._jdf.sparkSession().sql(f"""
        MERGE INTO turbine_hourly t
        USING incremental i
        ON i.date=t.date AND i.deviceId = t.deviceid AND i.time_interval = t.time_interval
        WHEN MATCHED THEN UPDATE SET *
        WHEN NOT MATCHED THEN INSERT *
    """)


# Perform the streaming merge into our  data stream
turbine_stream = (
    spark.readStream.format('delta').table('turbine_raw')        # Read data as a stream from our source Delta table
    .groupBy('deviceId','date',F.window('timestamp','1 hour')) # Aggregate readings to hourly intervals
    .agg({"rpm":"avg","angle":"avg"})
    .writeStream                                                                                         
    .foreachBatch(merge_records)                              # Pass each micro-batch to a function
    .outputMode("update")                                     # Merge works with update mod
    .start()
)
```

我们从白银到黄金的管道将把这两条管道连接在一起，形成一个单一的表，用于每小时的天气和涡轮机测量。

```Scala
# Read streams from Delta Silver tables
turbine_hourly = spark.readStream.format('delta').option("ignoreChanges", True).table("turbine_hourly")
weather_hourly = spark.readStream.format('delta').option("ignoreChanges", True).table("weather_hourly")

# Perform a streaming join to enrich the data
turbine_enriched = turbine_hourly.join(weather_hourly, ['date','time_interval'])

# Perform a streaming merge into our Gold data stream
merge_gold_stream = (
    turbine_enriched.writeStream 
    .foreachBatch(merge_records)
    .start()
)
```

我们可以立即查询我们的黄金级Delta表。

![](https://i.loli.net/2020/08/16/Ny9HaQ2go4sdqkc.png)

数据分析Notebook里还包含一个单元格，它将生成历史每小时功率读数和日常维护日志，用于模型训练。运行该单元格将：

1. 在涡轮机丰富表中回填一年的历史读数。

2. 为power_output表中的每台涡轮机生成历史功率读数。

3. 在turbine_maintenance表中为每个风机生成历史维护日志。


现在，我们在Azure Data Lake上拥有丰富的就绪的人工智能(AI)数据，这些数据以高性能、可靠的格式提供给我们的数据科学建模，以优化资产利用率。

```sql
%sql
-- Query all 3 tables together
CREATE OR REPLACE VIEW gold_readings AS
SELECT r.*, 
    p.power, 
    m.maintenance as maintenance
FROM turbine_enriched r 
    JOIN turbine_power p ON (r.date=p.date AND r.time_interval=p.time_interval AND r.deviceid=p.deviceid)
    LEFT JOIN turbine_maintenance m ON (r.date=m.date AND r.deviceid=m.deviceid);
    
SELECT * FROM gold_readings
```

![](https://i.loli.net/2020/08/16/FBgumeayqjNScpv.png)

我们的数据工程管道已经完成! 数据现在正从IoT Hubs流向青铜级（原始）到白银级（聚合）再到黄金级（丰富）。现在是时候对我们的数据进行一些分析了。

### 总结

综上所述，我们已经成功完成：

- 将现场设备的实时IIoT数据输入Azure中。

- 直接在Data Lake上进行复杂的时间序列处理。


其中关键技术是我们采用的数据湖技术（Delta Lake）。ADLS上的Delta提供了可靠的流式数据管道和对海量时间序列数据的高性能数据科学和分析查询。最后，它使组织能够真正采用Lakehouse模式，将最佳的Azure工具带入一次写入、多次访问的数据存储中。

在第三部分中，我们将探讨如何使用机器学习来最大限度地提高风力发电机的收益，同时最大限度地降低停机的机会成本。

## 第三部分

#### （待更新）