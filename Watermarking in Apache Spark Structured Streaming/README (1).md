# Feature Deep Dive: Watermarking in Apache Spark Structured Streaming

## Key Takeaways
- **Watermarks** help Spark understand the processing progress based on event time, determine when to produce windowed aggregates, and when to trim the aggregation state.
- When **joining streams of data**, Spark, by default, uses a single, global watermark that evicts state based on the minimum event time seen across the input streams.
- **RocksDB** can be leveraged to reduce pressure on cluster memory and garbage collection (GC) pauses.
- **StreamingQueryProgress** and **StateOperatorProgress** objects contain key information about how watermarks affect your stream.

## Introduction
When building real-time pipelines, one of the realities that teams have to work with is that distributed data ingestion is inherently unordered. Additionally, in the context of stateful streaming operations, teams need to be able to properly track event time progress in the stream of data they are ingesting for the proper calculation of time-window aggregations and other stateful operations. We can solve this using **Structured Streaming**.

For example, let’s say we are a team working on building a pipeline to help our company do proactive maintenance on our mining machines that we lease to our customers. These machines always need to be running in top condition, so we monitor them in real-time. We need to perform stateful aggregations on the streaming data to understand and identify problems in the machines.

This is where we need to leverage **Structured Streaming** and **Watermarking** to produce the necessary stateful aggregations that will help inform decisions around predictive maintenance and more for these machines.

## What Is Watermarking?
Generally speaking, when working with real-time streaming data, there will be delays between event time and processing time due to how data is ingested and whether the overall application experiences issues like downtime. Due to these potential variable delays, the engine that you use to process this data needs to have some mechanism to decide when to close the aggregate windows and produce the aggregate result.

While the natural inclination to remedy these issues might be to use a fixed delay based on the wall clock time, we will show in this upcoming example why this is not the best solution.

![image](https://github.com/user-attachments/assets/da109148-8871-45ea-b4a4-badbe464b852)


### Example: Watermarking in Action
Let’s take a scenario where we are receiving data at various times from around 10:50 AM to 11:20 AM. We are creating 10-minute tumbling windows that calculate the average of the temperature and pressure readings that came in during the windowed period.

In this first scenario, the tumbling windows trigger at 11:00 AM, 11:10 AM, and 11:20 AM, leading to the result tables shown at the respective times. When the second batch of data comes around 11:10 AM with data that has an event time of 10:53 AM, this gets incorporated into the temperature and pressure averages calculated for the 11:00 AM to 11:10 AM window that closes at 11:10 AM, which does not give the correct result.

To ensure we get the correct results for the aggregates we want to produce, we need to define a watermark that will allow Spark to understand when to close the aggregate window and produce the correct aggregate result.

![image](https://github.com/user-attachments/assets/05a399f6-fbad-4565-992c-9a9b644cb5c8)


### How Watermarking Works
In **Structured Streaming** applications, we can ensure that all relevant data for the aggregations we want to calculate is collected by using a feature called watermarking. By defining a watermark, Spark Structured Streaming then knows when it has ingested all data up to some time, T, based on a set lateness expectation, so that it can close and produce windowed aggregates up to timestamp T.

In contrast to the first scenario, with watermarking, Spark waits to close and output the windowed aggregation once the max event time seen minus the specified watermark is greater than the upper bound of the window.

This produces the correct result by properly incorporating the data based on the expected lateness defined by the watermark. Once the results are emitted, the corresponding state is removed from the state store.

## Incorporating Watermarking into Your Pipelines
To understand how to incorporate these watermarks into our Structured Streaming pipelines, let’s explore a scenario where we are ingesting all our sensor data from a Kafka cluster in the cloud and want to calculate temperature and pressure averages every ten minutes with an expected time skew of ten minutes.

Here’s how the Structured Streaming pipeline with watermarking would look:

```python
sensorStreamDF = spark \
  .readStream \
  .format("kafka") \
  .option("kafka.bootstrap.servers", "host1:port1,host2:port2") \
  .option("subscribe", "tempAndPressureReadings") \
  .load()

sensorStreamDF = sensorStreamDF \
.withWatermark("eventTimestamp", "10 minutes") \
.groupBy(window(sensorStreamDF.eventTimestamp, "10 minutes")) \
.avg(sensorStreamDF.temperature,
     sensorStreamDF.pressure)

sensorStreamDF.writeStream
  .format("delta")
  .outputMode("append")
  .option("checkpointLocation", "/delta/events/_checkpoints/temp_pressure_job/")
  .start("/delta/temperatureAndPressureAverages")
```

## Example Output
The output written to the table for a particular sample of data would look like this:

```json
[
  {
    "window": {
      "start": "2021-08-29T05:50:00.000+0000",
      "end": "2021-08-29T06:00:00.000+0000"
    },
    "temperature": 33.03,
    "pressure": 289,
    "avg(temperature)": 33.03,
    "avg(pressure)": 289
  },
  ...
]
```

## Key Considerations
When implementing watermarking, you need to identify two items:

1. The column that represents the event time of the sensor reading.

2. The estimated expected time skew of the data.

Here’s how to implement watermarking in your Structured Streaming pipeline:

```
sensorStreamDF = sensorStreamDF \
.withWatermark("eventTimestamp", "10 minutes") \
.groupBy(window(sensorStreamDF.eventTimestamp, "10 minutes")) \
.avg(sensorStreamDF.temperature,
     sensorStreamDF.pressure)
```

## Watermarks in Different Output Modes
Before diving deeper, it's important to understand how your choice of output mode affects the behavior of the watermarks you set.

- **Append Mode**: An aggregate can be produced only once and cannot be updated. Therefore, late records are dropped after the watermark delay period.
- **Update Mode**: The aggregate can be produced repeatedly, starting from the first record, so a watermark is optional. It trims the state once the engine heuristically knows that no more records for that aggregate can be received.

### Implications of Watermarks on State and Latency
- **Longer Watermark Delay**: Higher precision, but also higher latency.
- **Shorter Watermark Delay**: Lower precision, but lower latency.

## Deeper Dive into Watermarking

### Joins and Watermarking
When doing join operations in your streaming applications, particularly when joining two streams, it is important to use watermarks to handle late records and trim the state. An example of a streaming join with watermarks is shown below:

```

sensorStreamDF = spark.readStream.format("delta").table("sensorData")
tempAndPressStreamDF = spark.readStream.format("delta").table("tempPressData")

sensorStreamDF_wtmrk = sensorStreamDF.withWatermark("timestamp", "5 minutes")
tempAndPressStreamDF_wtmrk = tempAndPressStreamDF.withWatermark("timestamp", "5 minutes")

joinedDF = tempAndPressStreamDF_wtmrk.alias("t").join(
 sensorStreamDF_wtmrk.alias("s"),
 expr("""
   s.sensor_id == t.sensor_id AND
   s.timestamp >= t.timestamp AND
   s.timestamp <= t.timestamp + interval 5 minutes
   """),
 joinType="inner"
).withColumn("sensorMeasure", col("Sensor1")+col("Sensor2")) \
.groupBy(window(col("t.timestamp"), "10 minutes")) \
.agg(avg(col("sensorMeasure")).alias("avg_sensor_measure"), avg(col("temperature")).alias("avg_temperature"), avg(col("pressure")).alias("avg_pressure")) \
.select("window", "avg_sensor_measure", "avg_temperature", "avg_pressure")

joinedDF.writeStream.format("delta") \
       .outputMode("append") \
       .option("checkpointLocation", "/checkpoint/files/") \
       .toTable("output_table")
```

## Monitoring and Managing Streams with Watermarks
When managing a streaming query, where Spark may need to manage millions of keys and keep state for each of them, the default state store may not be effective. To mitigate this, RocksDB can be leveraged in Databricks to efficiently manage state.

```
spark.conf.set(
  "spark.sql.streaming.stateStore.providerClass",
  "com.databricks.sql.streaming.state.RocksDBStateStoreProvider")
```

## Monitoring Metrics
Key metrics to track include those provided by **StreamingQueryProgress** and **StateOperatorProgress** objects:

- **eventTime**: Max, min, avg, and watermark timestamps.
- **numRowsDroppedByWatermark**: Indicates rows considered too late to be included in stateful aggregation.

These metrics are essential for reconciling data in result tables and ensuring that watermarks are configured correctly.

## Multiple Aggregations, Streaming, and Watermarks
A current limitation of Structured Streaming queries is the inability to chain multiple stateful operators (e.g., aggregations, streaming joins) in a single streaming query. This limitation is being addressed by Databricks as part of their ongoing improvements to the platform.
