# In-Depth Look: Watermarking in Apache Spark Structured Streaming


## Key Concepts
- **Watermarks** Watermarks in Spark are crucial for managing event time in streaming data, allowing the system to track processing progress, decide when to emit windowed aggregates, and when to discard old state data.
- When joining streams of data, Spark employs a global watermark by default, which manages state eviction based on the earliest event time across all input streams.
- **RocksDB** RocksDB can be utilized to alleviate memory pressure and reduce garbage collection (GC) pauses within the cluster.
- **StreamingQueryProgress** and **StateOperatorProgress** StreamingQueryProgress and StateOperatorProgress objects provide essential insights into how watermarks are impacting the performance and state of your streaming queries.

## Introduction
In real-time data pipelines, dealing with unordered data due to the distributed nature of data ingestion is a common challenge. This unordered nature, combined with the need for stateful streaming operations, requires accurate tracking of event time to correctly perform time-window aggregations and other state-dependent operations. **Structured Streaming** in Apache Spark addresses these challenges effectively.

Consider a scenario where your team is responsible for building a pipeline to monitor mining equipment leased to customers, ensuring they operate at peak efficiency. Real-time monitoring is crucial, and stateful aggregations on streaming data are necessary to detect and diagnose potential issues in the equipment. This is where **Structured Streaming** and **Watermarking** come into play, enabling the generation of stateful aggregates that support predictive maintenance and other critical decisions.

This is where we need to leverage **Structured Streaming** and **Watermarking** to produce the necessary stateful aggregations that will help inform decisions around predictive maintenance and more for these machines.

## What Is Watermarking?
In real-time streaming, the delay between event time and processing time is inevitable due to the nature of data ingestion and potential application downtimes. To manage these delays, a processing engine like Spark needs a mechanism to determine when to finalize and emit aggregate results for time windows. **Watermarking** serves this purpose, helping the system decide when to close aggregation windows and produce results based on the event time of incoming data.

While the natural inclination to remedy these issues might be to use a fixed delay based on the wall clock time, we will show in this upcoming example why this is not the best solution.

![image](https://github.com/user-attachments/assets/da109148-8871-45ea-b4a4-badbe464b852)


### Example: Watermarking in Action
Let’s take a scenario where we are receiving data at various times from around 10:50 AM to 11:20 AM. We are creating 10-minute tumbling windows that calculate the average of the temperature and pressure readings that came in during the windowed period.

In this first scenario, the tumbling windows trigger at 11:00 AM, 11:10 AM, and 11:20 AM, leading to the result tables shown at the respective times. When the second batch of data comes around 11:10 AM with data that has an event time of 10:53 AM, this gets incorporated into the temperature and pressure averages calculated for the 11:00 AM to 11:10 AM window that closes at 11:10 AM, which does not give the correct result.

To ensure we get the correct results for the aggregates we want to produce, we need to define a watermark that will allow Spark to understand when to close the aggregate window and produce the correct aggregate result.

![image](https://github.com/user-attachments/assets/05a399f6-fbad-4565-992c-9a9b644cb5c8)


### How Watermarking Works
In **Structured Streaming** applications, we can ensure that all relevant data for the aggregations we want to compute is accounted for by using a feature called watermarking. By defining a watermark, Spark Structured Streaming can determine when it has ingested all necessary data up to a specific time, T, based on an expected delay or lateness. This allows Spark to finalize and produce windowed aggregates up to the timestamp T.

With watermarking, instead of closing and outputting the windowed aggregation immediately, Spark waits until the maximum event time observed minus the specified watermark exceeds the upper bound of the window. This approach ensures that the aggregation incorporates all relevant data, even if some data arrives late within the defined watermark period. Once the results are generated, the corresponding state is purged from the state store.

## Incorporating Watermarking into Your Pipelines
To effectively integrate watermarks into your Structured Streaming pipelines, consider a scenario where you're ingesting sensor data from a Kafka cluster in the cloud. Suppose you need to calculate average temperature and pressure readings every ten minutes, with an allowance for a time skew of up to ten minutes. In this case, watermarking ensures that all relevant sensor data is included in the aggregation, even if some data arrives late within the defined time skew.

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
