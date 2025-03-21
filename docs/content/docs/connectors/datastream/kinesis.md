---
title: Kinesis
weight: 5
type: docs
aliases:
  - /dev/connectors/kinesis.html
  - /apis/streaming/connectors/kinesis.html
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# Amazon Kinesis Data Streams Connector

The Kinesis connector provides access to [Amazon Kinesis Data Streams](http://aws.amazon.com/kinesis/streams/).

To use this connector, add one or more of the following dependencies to your project, depending on whether you are reading from and/or writing to Kinesis Data Streams:

<table class="table table-bordered">
  <thead>
    <tr>
      <th class="text-left">KDS Connectivity</th>
      <th class="text-left">Maven Dependency</th>
    </tr>
  </thead>
  <tbody>
    <tr>
        <td>Source</td>
        <td>{{< artifact flink-connector-kinesis >}}</td>
    </tr>
    <tr>
        <td>Sink</td>
        <td>{{< artifact flink-connector-aws-kinesis-streams >}}</td>
    </tr>
  </tbody>
</table>

Due to the licensing issue, the `flink-connector-kinesis` artifact is not deployed to Maven central for the prior versions. Please see the version specific documentation for further information.

{{< py_download_link "kinesis" >}}

## Using the Amazon Kinesis Streams Service
Follow the instructions from the [Amazon Kinesis Streams Developer Guide](https://docs.aws.amazon.com/streams/latest/dev/learning-kinesis-module-one-create-stream.html)
to setup Kinesis streams.

## Configuring Access to Kinesis with IAM
Make sure to create the appropriate IAM policy to allow reading / writing to / from the Kinesis streams. See examples [here](https://docs.aws.amazon.com/streams/latest/dev/controlling-access.html).

Depending on your deployment you would choose a different Credentials Provider to allow access to Kinesis.
By default, the `AUTO` Credentials Provider is used.
If the access key ID and secret key are set in the configuration, the `BASIC` provider is used.  

A specific Credentials Provider can **optionally** be set by using the `AWSConfigConstants.AWS_CREDENTIALS_PROVIDER` setting.
 
Supported Credential Providers are:
* `AUTO` - Using the default AWS Credentials Provider chain that searches for credentials in the following order: `ENV_VARS`, `SYS_PROPS`, `WEB_IDENTITY_TOKEN`, `PROFILE` and EC2/ECS credentials provider.
* `BASIC` - Using access key ID and secret key supplied as configuration. 
* `ENV_VAR` - Using `AWS_ACCESS_KEY_ID` & `AWS_SECRET_ACCESS_KEY` environment variables.
* `SYS_PROP` - Using Java system properties aws.accessKeyId and aws.secretKey.
* `PROFILE` - Use AWS credentials profile file to create the AWS credentials.
* `ASSUME_ROLE` - Create AWS credentials by assuming a role. The credentials for assuming the role must be supplied.
* `WEB_IDENTITY_TOKEN` - Create AWS credentials by assuming a role using Web Identity Token. 

## Kinesis Consumer

The `FlinkKinesisConsumer` is an exactly-once parallel streaming data source that subscribes to multiple AWS Kinesis
streams within the same AWS service region, and can transparently handle resharding of streams while the job is running. Each subtask of the consumer is
responsible for fetching data records from multiple Kinesis shards. The number of shards fetched by each subtask will
change as shards are closed and created by Kinesis.

Before consuming data from Kinesis streams, make sure that all streams are created with the status "ACTIVE" in the Amazon Kinesis Data Stream console.

{{< tabs "58b6c235-48ee-4cf7-aabc-41e0679a3370" >}}
{{< tab "Java" >}}
```java
Properties consumerConfig = new Properties();
consumerConfig.put(AWSConfigConstants.AWS_REGION, "us-east-1");
consumerConfig.put(AWSConfigConstants.AWS_ACCESS_KEY_ID, "aws_access_key_id");
consumerConfig.put(AWSConfigConstants.AWS_SECRET_ACCESS_KEY, "aws_secret_access_key");
consumerConfig.put(ConsumerConfigConstants.STREAM_INITIAL_POSITION, "LATEST");

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> kinesis = env.addSource(new FlinkKinesisConsumer<>(
    "kinesis_stream_name", new SimpleStringSchema(), consumerConfig));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
val consumerConfig = new Properties()
consumerConfig.put(AWSConfigConstants.AWS_REGION, "us-east-1")
consumerConfig.put(AWSConfigConstants.AWS_ACCESS_KEY_ID, "aws_access_key_id")
consumerConfig.put(AWSConfigConstants.AWS_SECRET_ACCESS_KEY, "aws_secret_access_key")
consumerConfig.put(ConsumerConfigConstants.STREAM_INITIAL_POSITION, "LATEST")

val env = StreamExecutionEnvironment.getExecutionEnvironment

val kinesis = env.addSource(new FlinkKinesisConsumer[String](
    "kinesis_stream_name", new SimpleStringSchema, consumerConfig))
```
{{< /tab >}}
{{< tab "Python" >}}
```python
consumer_config = {
    'aws.region': 'us-east-1',
    'aws.credentials.provider.basic.accesskeyid': 'aws_access_key_id',
    'aws.credentials.provider.basic.secretkey': 'aws_secret_access_key',
    'flink.stream.initpos': 'LATEST'
}

env = StreamExecutionEnvironment.get_execution_environment()

kinesis = env.add_source(FlinkKinesisConsumer("stream-1", SimpleStringSchema(), consumer_config))
```
{{< /tab >}}
{{< /tabs >}}

The above is a simple example of using the consumer. Configuration for the consumer is supplied with a `java.util.Properties`
instance, the configuration keys for which can be found in `AWSConfigConstants` (AWS-specific parameters) and 
`ConsumerConfigConstants` (Kinesis consumer parameters). The example
demonstrates consuming a single Kinesis stream in the AWS region "us-east-1". The AWS credentials are supplied using the basic method in which
the AWS access key ID and secret access key are directly supplied in the configuration. Also, data is being consumed
from the newest position in the Kinesis stream (the other option will be setting `ConsumerConfigConstants.STREAM_INITIAL_POSITION`
to `TRIM_HORIZON`, which lets the consumer start reading the Kinesis stream from the earliest record possible).

Other optional configuration keys for the consumer can be found in `ConsumerConfigConstants`.

Note that the configured parallelism of the Flink Kinesis Consumer source
can be completely independent of the total number of shards in the Kinesis streams.
When the number of shards is larger than the parallelism of the consumer,
then each consumer subtask can subscribe to multiple shards; otherwise
if the number of shards is smaller than the parallelism of the consumer,
then some consumer subtasks will simply be idle and wait until it gets assigned
new shards (i.e., when the streams are resharded to increase the
number of shards for higher provisioned Kinesis service throughput).

Also note that the default assignment of shards to subtasks is based on the hashes of the shard and stream names,
which will more-or-less balance the shards across the subtasks.
However, assuming the default Kinesis shard management is used on the stream (UpdateShardCount with `UNIFORM_SCALING`),
setting `UniformShardAssigner` as the shard assigner on the consumer will much more evenly distribute shards to subtasks.
Assuming the incoming Kinesis records are assigned random Kinesis `PartitionKey` or `ExplicitHashKey` values,
the result is consistent subtask loading.
If neither the default assigner nor the `UniformShardAssigner` suffice, a custom implementation of `KinesisShardAssigner` can be set.

### The `DeserializationSchema`

Flink Kinesis Consumer also needs a schema to know how to turn the binary data in a Kinesis Data Stream into Java objects.
The `KinesisDeserializationSchema` allows users to specify such a schema. The `T deserialize(byte[] recordValue, String partitionKey, String seqNum, long approxArrivalTimestamp, String stream, String shardId)` 
method gets called for each Kinesis record.

For convenience, Flink provides the following schemas out of the box:
  
1. `TypeInformationSerializationSchema` which creates a schema based on a Flink's `TypeInformation`. 
    This is useful if the data is both written and read by Flink.
    This schema is a performant Flink-specific alternative to other generic serialization approaches.
    
2. `GlueSchemaRegistryJsonDeserializationSchema` offers the ability to lookup the writer's schema (schema which was used to write the record)
   in [AWS Glue Schema Registry](https://docs.aws.amazon.com/glue/latest/dg/schema-registry.html). Using this, deserialization schema record will be
   read with the schema retrieved from AWS Glue Schema Registry and transformed to either `com.amazonaws.services.schemaregistry.serializers.json.JsonDataWithSchema`
   that represents generic record with a manually provided schema or a JAVA POJO generated by [mbknor-jackson-jsonSchema](https://github.com/mbknor/mbknor-jackson-jsonSchema).  
   
   <br>To use this deserialization schema one has to add the following additional dependency:
       
{{< tabs "8c6721c7-4a48-496e-b0fe-6522cf6a5e13" >}}
{{< tab "GlueSchemaRegistryJsonDeserializationSchema" >}}
{{< artifact flink-jsonschema-glue-schema-registry >}}
{{< /tab >}}
{{< /tabs >}}
    
3. `AvroDeserializationSchema` which reads data serialized with Avro format using a statically provided schema. It can
    infer the schema from Avro generated classes (`AvroDeserializationSchema.forSpecific(...)`) or it can work with `GenericRecords`
    with a manually provided schema (with `AvroDeserializationSchema.forGeneric(...)`). This deserialization schema expects that
    the serialized records DO NOT contain the embedded schema.

    - You can use [AWS Glue Schema Registry](https://docs.aws.amazon.com/glue/latest/dg/schema-registry.html)
      to retrieve the writer’s schema. Similarly, the deserialization record will be read with the schema from AWS Glue Schema Registry and transformed
      (either through `GlueSchemaRegistryAvroDeserializationSchema.forGeneric(...)` or `GlueSchemaRegistryAvroDeserializationSchema.forSpecific(...)`).
      For more information on integrating the AWS Glue Schema Registry with Apache Flink see
      [Use Case: Amazon Kinesis Data Analytics for Apache Flink](https://docs.aws.amazon.com/glue/latest/dg/schema-registry-integrations.html#schema-registry-integrations-kinesis-data-analytics-apache-flink).

    <br>To use this deserialization schema one has to add the following additional dependency:
    
{{< tabs "8c6721c7-4a48-496e-b0fe-6522cf6a5e13" >}}
{{< tab "AvroDeserializationSchema" >}}
{{< artifact flink-avro >}}
{{< /tab >}}
{{< tab "GlueSchemaRegistryAvroDeserializationSchema" >}}
{{< artifact flink-avro-glue-schema-registry >}}
{{< /tab >}}
{{< /tabs >}}

### Configuring Starting Position

The Flink Kinesis Consumer currently provides the following options to configure where to start reading Kinesis streams, simply by setting `ConsumerConfigConstants.STREAM_INITIAL_POSITION` to
one of the following values in the provided configuration properties (the naming of the options identically follows [the namings used by the AWS Kinesis Streams service](http://docs.aws.amazon.com/kinesis/latest/APIReference/API_GetShardIterator.html#API_GetShardIterator_RequestSyntax)):

- `LATEST`: read all shards of all streams starting from the latest record.
- `TRIM_HORIZON`: read all shards of all streams starting from the earliest record possible (data may be trimmed by Kinesis depending on the retention settings).
- `AT_TIMESTAMP`: read all shards of all streams starting from a specified timestamp. The timestamp must also be specified in the configuration
properties by providing a value for `ConsumerConfigConstants.STREAM_INITIAL_TIMESTAMP`, in one of the following date pattern :
    - a non-negative double value representing the number of seconds that has elapsed since the Unix epoch (for example, `1459799926.480`).
    - a user defined pattern, which is a valid pattern for `SimpleDateFormat` provided by `ConsumerConfigConstants.STREAM_TIMESTAMP_DATE_FORMAT`.
    If `ConsumerConfigConstants.STREAM_TIMESTAMP_DATE_FORMAT` is not defined then the default pattern will be `yyyy-MM-dd'T'HH:mm:ss.SSSXXX`
    (for example, timestamp value is `2016-04-04` and pattern is `yyyy-MM-dd` given by user or timestamp value is `2016-04-04T19:58:46.480-00:00` without given a pattern).

### Fault Tolerance for Exactly-Once User-Defined State Update Semantics

With Flink's checkpointing enabled, the Flink Kinesis Consumer will consume records from shards in Kinesis streams and
periodically checkpoint each shard's progress. In case of a job failure, Flink will restore the streaming program to the
state of the latest complete checkpoint and re-consume the records from Kinesis shards, starting from the progress that
was stored in the checkpoint.

The interval of drawing checkpoints therefore defines how much the program may have to go back at most, in case of a failure.

To use fault tolerant Kinesis Consumers, checkpointing of the topology needs to be enabled at the execution environment:

{{< tabs "b1399ed7-5855-446d-9684-7a49de9b4c97" >}}
{{< tab "Java" >}}
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(5000); // checkpoint every 5000 msecs
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment()
env.enableCheckpointing(5000) // checkpoint every 5000 msecs
```
{{< /tab >}}
{{< tab "Python" >}}
```python
env = StreamExecutionEnvironment.get_execution_environment()
env.enable_checkpointing(5000) # checkpoint every 5000 msecs
```
{{< /tab >}}
{{< /tabs >}}

Also note that Flink can only restart the topology if enough processing slots are available to restart the topology.
Therefore, if the topology fails due to loss of a TaskManager, there must still be enough slots available afterwards.
Flink on YARN supports automatic restart of lost YARN containers.

### Using Enhanced Fan-Out

[Enhanced Fan-Out (EFO)](https://aws.amazon.com/blogs/aws/kds-enhanced-fanout/) increases the maximum 
number of concurrent consumers per Kinesis stream.
Without EFO, all concurrent consumers share a single read quota per shard. 
Using EFO, each consumer gets a distinct dedicated read quota per shard, allowing read throughput to scale with the number of consumers. 
Using EFO will [incur additional cost](https://aws.amazon.com/kinesis/data-streams/pricing/).
 
In order to enable EFO two additional configuration parameters are required:

- `RECORD_PUBLISHER_TYPE`: Determines whether to use `EFO` or `POLLING`. The default `RecordPublisher` is `POLLING`.
- `EFO_CONSUMER_NAME`: A name to identify the consumer. 
For a given Kinesis data stream, each consumer must have a unique name. 
However, consumer names do not have to be unique across data streams. 
Reusing a consumer name will result in existing subscriptions being terminated.

The code snippet below shows a simple example configurating an EFO consumer.

{{< tabs "42345893-70c3-4678-a348-4c419b337eb1" >}}
{{< tab "Java" >}}
```java
Properties consumerConfig = new Properties();
consumerConfig.put(AWSConfigConstants.AWS_REGION, "us-east-1");
consumerConfig.put(ConsumerConfigConstants.STREAM_INITIAL_POSITION, "LATEST");

consumerConfig.put(ConsumerConfigConstants.RECORD_PUBLISHER_TYPE, 
    ConsumerConfigConstants.RecordPublisherType.EFO.name());
consumerConfig.put(ConsumerConfigConstants.EFO_CONSUMER_NAME, "my-flink-efo-consumer");

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> kinesis = env.addSource(new FlinkKinesisConsumer<>(
    "kinesis_stream_name", new SimpleStringSchema(), consumerConfig));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
val consumerConfig = new Properties()
consumerConfig.put(AWSConfigConstants.AWS_REGION, "us-east-1")
consumerConfig.put(ConsumerConfigConstants.STREAM_INITIAL_POSITION, "LATEST")

consumerConfig.put(ConsumerConfigConstants.RECORD_PUBLISHER_TYPE, 
    ConsumerConfigConstants.RecordPublisherType.EFO.name());
consumerConfig.put(ConsumerConfigConstants.EFO_CONSUMER_NAME, "my-flink-efo-consumer");

val env = StreamExecutionEnvironment.getExecutionEnvironment()

val kinesis = env.addSource(new FlinkKinesisConsumer[String](
    "kinesis_stream_name", new SimpleStringSchema, consumerConfig))
```
{{< /tab >}}
{{< tab "Python" >}}
```python
consumer_config = {
    'aws.region': 'us-east-1',
    'flink.stream.initpos': 'LATEST',
    'flink.stream.recordpublisher':  'EFO',
    'flink.stream.efo.consumername': 'my-flink-efo-consumer'
}

env = StreamExecutionEnvironment.get_execution_environment()

kinesis = env.add_source(FlinkKinesisConsumer(
    "kinesis_stream_name", SimpleStringSchema(), consumer_config))
```
{{< /tab >}}
{{< /tabs >}}

#### EFO Stream Consumer Registration/Deregistration

In order to use EFO, a stream consumer must be registered against each stream you wish to consume.
By default, the `FlinkKinesisConsumer` will register the stream consumer automatically when the Flink job starts.
The stream consumer will be registered using the name provided by the `EFO_CONSUMER_NAME` configuration.
`FlinkKinesisConsumer` provides three registration strategies:

- Registration
  - `LAZY` (default): Stream consumers are registered when the Flink job starts running.
    If the stream consumer already exists, it will be reused.
    This is the preferred strategy for the majority of applications.
    However, jobs with parallelism greater than 1 will result in tasks competing to register and acquire the stream consumer ARN.
    For jobs with very large parallelism this can result in an increased start-up time.
    The `DescribeStreamConsumer` operation has a limit of 20 [transactions per second](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_DescribeStreamConsumer.html),
    this means application startup time will increase by roughly `parallelism/20 seconds`.
  - `EAGER`: Stream consumers are registered in the `FlinkKinesisConsumer` constructor.
    If the stream consumer already exists, it will be reused. 
    This will result in registration occurring when the job is constructed, 
    either on the Flink Job Manager or client environment submitting the job.
    Using this strategy results in a single thread registering and retrieving the stream consumer ARN, 
    reducing startup time over `LAZY` (with large parallelism).
    However, consider that the client environment will require access to the AWS services.
  - `NONE`: Stream consumer registration is not performed by `FlinkKinesisConsumer`.
    Registration must be performed externally using the [AWS CLI or SDK](https://aws.amazon.com/tools/)
    to invoke [RegisterStreamConsumer](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_RegisterStreamConsumer.html).
    Stream consumer ARNs should be provided to the job via the consumer configuration.
- Deregistration
  - `LAZY|EAGER` (default): Stream consumers are deregistered when the job is shutdown gracefully.
    In the event that a job terminates without executing the shutdown hooks, stream consumers will remain active.
    In this situation the stream consumers will be gracefully reused when the application restarts. 
  - `NONE`: Stream consumer deregistration is not performed by `FlinkKinesisConsumer`.

Below is an example configuration to use the `EAGER` registration strategy:

{{< tabs "a85d716b-6c1c-46d8-9ee4-12d8380a0c06" >}}
{{< tab "Java" >}}
```java
Properties consumerConfig = new Properties();
consumerConfig.put(AWSConfigConstants.AWS_REGION, "us-east-1");
consumerConfig.put(ConsumerConfigConstants.STREAM_INITIAL_POSITION, "LATEST");

consumerConfig.put(ConsumerConfigConstants.RECORD_PUBLISHER_TYPE, 
    ConsumerConfigConstants.RecordPublisherType.EFO.name());
consumerConfig.put(ConsumerConfigConstants.EFO_CONSUMER_NAME, "my-flink-efo-consumer");

consumerConfig.put(ConsumerConfigConstants.EFO_REGISTRATION_TYPE, 
    ConsumerConfigConstants.EFORegistrationType.EAGER.name());

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> kinesis = env.addSource(new FlinkKinesisConsumer<>(
    "kinesis_stream_name", new SimpleStringSchema(), consumerConfig));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
val consumerConfig = new Properties()
consumerConfig.put(AWSConfigConstants.AWS_REGION, "us-east-1")
consumerConfig.put(ConsumerConfigConstants.STREAM_INITIAL_POSITION, "LATEST")

consumerConfig.put(ConsumerConfigConstants.RECORD_PUBLISHER_TYPE, 
    ConsumerConfigConstants.RecordPublisherType.EFO.name());
consumerConfig.put(ConsumerConfigConstants.EFO_CONSUMER_NAME, "my-flink-efo-consumer");

consumerConfig.put(ConsumerConfigConstants.EFO_REGISTRATION_TYPE, 
    ConsumerConfigConstants.EFORegistrationType.EAGER.name());

val env = StreamExecutionEnvironment.getExecutionEnvironment()

val kinesis = env.addSource(new FlinkKinesisConsumer[String](
    "kinesis_stream_name", new SimpleStringSchema, consumerConfig))
```
{{< /tab >}}
{{< tab "Python" >}}
```python
consumer_config = {
    'aws.region': 'us-east-1',
    'flink.stream.initpos': 'LATEST',
    'flink.stream.recordpublisher':  'EFO',
    'flink.stream.efo.consumername': 'my-flink-efo-consumer',
    'flink.stream.efo.registration': 'EAGER'
}

env = StreamExecutionEnvironment.get_execution_environment()

kinesis = env.add_source(FlinkKinesisConsumer(
    "kinesis_stream_name", SimpleStringSchema(), consumer_config))
```
{{< /tab >}}
{{< /tabs >}}

Below is an example configuration to use the `NONE` registration strategy:

{{< tabs "00b46c87-7740-4263-8040-2aa7e2960513" >}}
{{< tab "Java" >}}
```java
Properties consumerConfig = new Properties();
consumerConfig.put(AWSConfigConstants.AWS_REGION, "us-east-1");
consumerConfig.put(ConsumerConfigConstants.STREAM_INITIAL_POSITION, "LATEST");

consumerConfig.put(ConsumerConfigConstants.RECORD_PUBLISHER_TYPE, 
    ConsumerConfigConstants.RecordPublisherType.EFO.name());
consumerConfig.put(ConsumerConfigConstants.EFO_CONSUMER_NAME, "my-flink-efo-consumer");

consumerConfig.put(ConsumerConfigConstants.EFO_REGISTRATION_TYPE, 
    ConsumerConfigConstants.EFORegistrationType.NONE.name());
consumerConfig.put(ConsumerConfigConstants.efoConsumerArn("stream-name"), 
    "arn:aws:kinesis:<region>:<account>>:stream/<stream-name>/consumer/<consumer-name>:<create-timestamp>");

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> kinesis = env.addSource(new FlinkKinesisConsumer<>(
    "kinesis_stream_name", new SimpleStringSchema(), consumerConfig));
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
val consumerConfig = new Properties()
consumerConfig.put(AWSConfigConstants.AWS_REGION, "us-east-1")
consumerConfig.put(ConsumerConfigConstants.STREAM_INITIAL_POSITION, "LATEST")

consumerConfig.put(ConsumerConfigConstants.RECORD_PUBLISHER_TYPE, 
    ConsumerConfigConstants.RecordPublisherType.EFO.name());
consumerConfig.put(ConsumerConfigConstants.EFO_CONSUMER_NAME, "my-flink-efo-consumer");

consumerConfig.put(ConsumerConfigConstants.EFO_REGISTRATION_TYPE, 
    ConsumerConfigConstants.EFORegistrationType.NONE.name());
consumerConfig.put(ConsumerConfigConstants.efoConsumerArn("stream-name"), 
    "arn:aws:kinesis:<region>:<account>>:stream/<stream-name>/consumer/<consumer-name>:<create-timestamp>");

val env = StreamExecutionEnvironment.getExecutionEnvironment()

val kinesis = env.addSource(new FlinkKinesisConsumer[String](
    "kinesis_stream_name", new SimpleStringSchema, consumerConfig))
```
{{< /tab >}}
{{< tab "Python" >}}
```python
consumer_config = {
    'aws.region': 'us-east-1',
    'flink.stream.initpos': 'LATEST',
    'flink.stream.recordpublisher':  'EFO',
    'flink.stream.efo.consumername': 'my-flink-efo-consumer',
    'flink.stream.efo.consumerarn.stream-name':
        'arn:aws:kinesis:<region>:<account>>:stream/<stream-name>/consumer/<consumer-name>:<create-timestamp>'
}

env = StreamExecutionEnvironment.get_execution_environment()

kinesis = env.add_source(FlinkKinesisConsumer(
    "kinesis_stream_name", SimpleStringSchema(), consumer_config))
```
{{< /tab >}}
{{< /tabs >}}

### Event Time for Consumed Records

If streaming topologies choose to use the [event time notion]({{< ref "docs/concepts/time" >}}) for record
timestamps, an *approximate arrival timestamp* will be used by default. This timestamp is attached to records by Kinesis once they
were successfully received and stored by streams. Note that this timestamp is typically referred to as a Kinesis server-side
timestamp, and there are no guarantees about the accuracy or order correctness (i.e., the timestamps may not always be
ascending).

Users can choose to override this default with a custom timestamp, as described [here]({{< ref "docs/dev/datastream/event-time/generating_watermarks" >}}),
or use one from the [predefined ones]({{< ref "docs/dev/datastream/event-time/built_in" >}}). After doing so,
it can be passed to the consumer in the following way:

{{< tabs "8fbaf5cb-3b76-4c62-a74e-db51b60f6600" >}}
{{< tab "Java" >}}
```java
FlinkKinesisConsumer<String> consumer = new FlinkKinesisConsumer<>(
    "kinesis_stream_name",
    new SimpleStringSchema(),
    kinesisConsumerConfig);
consumer.setPeriodicWatermarkAssigner(new CustomAssignerWithPeriodicWatermarks());
DataStream<String> stream = env
	.addSource(consumer)
	.print();
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
val consumer = new FlinkKinesisConsumer[String](
    "kinesis_stream_name",
    new SimpleStringSchema(),
    kinesisConsumerConfig);
consumer.setPeriodicWatermarkAssigner(new CustomAssignerWithPeriodicWatermarks());
val stream = env
	.addSource(consumer)
	.print();
```
{{< /tab >}}
{{< tab "Python" >}}
```python
consumer = FlinkKinesisConsumer(
    "kinesis_stream_name",
    SimpleStringSchema(),
    consumer_config)
stream = env.add_source(consumer).print()
```
{{< /tab >}}
{{< /tabs >}}

Internally, an instance of the assigner is executed per shard / consumer thread (see threading model below).
When an assigner is specified, for each record read from Kinesis, the extractTimestamp(T element, long previousElementTimestamp)
is called to assign a timestamp to the record and getCurrentWatermark() to determine the new watermark for the shard.
The watermark of the consumer subtask is then determined as the minimum watermark of all its shards and emitted periodically.
The per shard watermark is essential to deal with varying consumption speed between shards, that otherwise could lead
to issues with downstream logic that relies on the watermark, such as incorrect late data dropping.

By default, the watermark is going to stall if shards do not deliver new records.
The property `ConsumerConfigConstants.SHARD_IDLE_INTERVAL_MILLIS` can be used to avoid this potential issue through a
timeout that will allow the watermark to progress despite of idle shards.

### Event Time Alignment for Shard Consumers

The Flink Kinesis Consumer optionally supports synchronization between parallel consumer subtasks (and their threads)
to avoid the event time skew related problems described in [Event time synchronization across sources](https://issues.apache.org/jira/browse/FLINK-10886).

To enable synchronization, set the watermark tracker on the consumer:

{{< tabs "8fbaf5cb-3b76-4c62-a74e-db51b60f6601" >}}
{{< tab "Java" >}}
```java
JobManagerWatermarkTracker watermarkTracker =
    new JobManagerWatermarkTracker("myKinesisSource");
consumer.setWatermarkTracker(watermarkTracker);
```
{{< /tab >}}
{{< tab "Python" >}}
```python
watermark_tracker = WatermarkTracker.job_manager_watermark_tracker("myKinesisSource")
consumer.set_watermark_tracker(watermark_tracker)
```
{{< /tab >}}
{{< /tabs >}}

The `JobManagerWatermarkTracker` will use a global aggregate to synchronize the per subtask watermarks. Each subtask
uses a per shard queue to control the rate at which records are emitted downstream based on how far ahead of the global
watermark the next record in the queue is.

The "emit ahead" limit is configured via `ConsumerConfigConstants.WATERMARK_LOOKAHEAD_MILLIS`. Smaller values reduce
the skew but also the throughput. Larger values will allow the subtask to proceed further before waiting for the global
watermark to advance.

Another variable in the throughput equation is how frequently the watermark is propagated by the tracker.
The interval can be configured via `ConsumerConfigConstants.WATERMARK_SYNC_MILLIS`.
Smaller values reduce emitter waits and come at the cost of increased communication with the job manager.

Since records accumulate in the queues when skew occurs, increased memory consumption needs to be expected.
How much depends on the average record size. With larger sizes, it may be necessary to adjust the emitter queue capacity via
`ConsumerConfigConstants.WATERMARK_SYNC_QUEUE_CAPACITY`.

### Threading Model

The Flink Kinesis Consumer uses multiple threads for shard discovery and data consumption.

#### Shard Discovery

For shard discovery, each parallel consumer subtask will have a single thread that constantly queries Kinesis for shard
information even if the subtask initially did not have shards to read from when the consumer was started. In other words, if
the consumer is run with a parallelism of 10, there will be a total of 10 threads constantly querying Kinesis regardless
of the total amount of shards in the subscribed streams.

#### Polling (default) Record Publisher

For `POLLING` data consumption, a single thread will be created to consume each discovered shard. Threads will terminate when the
shard it is responsible of consuming is closed as a result of stream resharding. In other words, there will always be
one thread per open shard.

#### Enhanced Fan-Out Record Publisher

For `EFO` data consumption the threading model is the same as `POLLING`, with additional thread pools to handle 
asynchronous communication with Kinesis. AWS SDK v2.x `KinesisAsyncClient` uses additional threads for 
Netty to handle IO and asynchronous response. Each parallel consumer subtask will have their own instance of the `KinesisAsyncClient`.
In other words, if the consumer is run with a parallelism of 10, there will be a total of 10 `KinesisAsyncClient` instances.
A separate client will be created and subsequently destroyed when registering and deregistering stream consumers.

### Internally Used Kinesis APIs

The Flink Kinesis Consumer uses the [AWS Java SDK](http://aws.amazon.com/sdk-for-java/) internally to call Kinesis APIs
for shard discovery and data consumption. Due to Amazon's [service limits for Kinesis Streams](http://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html)
on the APIs, the consumer will be competing with other non-Flink consuming applications that the user may be running.
Below is a list of APIs called by the consumer with description of how the consumer uses the API, as well as information
on how to deal with any errors or warnings that the Flink Kinesis Consumer may have due to these service limits.

#### Shard Discovery

- *[ListShards](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_ListShards.html)*: this is constantly called
by a single thread in each parallel consumer subtask to discover any new shards as a result of stream resharding. By default,
the consumer performs the shard discovery at an interval of 10 seconds, and will retry indefinitely until it gets a result
from Kinesis. If this interferes with other non-Flink consuming applications, users can slow down the consumer of
calling this API by setting a value for `ConsumerConfigConstants.SHARD_DISCOVERY_INTERVAL_MILLIS` in the supplied
configuration properties. This sets the discovery interval to a different value. Note that this setting directly impacts
the maximum delay of discovering a new shard and starting to consume it, as shards will not be discovered during the interval.

#### Polling (default) Record Publisher

- *[GetShardIterator](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_GetShardIterator.html)*: this is called
only once when per shard consuming threads are started, and will retry if Kinesis complains that the transaction limit for the
API has exceeded, up to a default of 3 attempts. Note that since the rate limit for this API is per shard (not per stream),
the consumer itself should not exceed the limit. Usually, if this happens, users can either try to slow down any other
non-Flink consuming applications of calling this API, or modify the retry behaviour of this API call in the consumer by
setting keys prefixed by `ConsumerConfigConstants.SHARD_GETITERATOR_*` in the supplied configuration properties.

- *[GetRecords](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_GetRecords.html)*: this is constantly called
by per shard consuming threads to fetch records from Kinesis. When a shard has multiple concurrent consumers (when there
are any other non-Flink consuming applications running), the per shard rate limit may be exceeded. By default, on each call
of this API, the consumer will retry if Kinesis complains that the data size / transaction limit for the API has exceeded,
up to a default of 3 attempts. Users can either try to slow down other non-Flink consuming applications, or adjust the throughput
of the consumer by setting the `ConsumerConfigConstants.SHARD_GETRECORDS_MAX` and
`ConsumerConfigConstants.SHARD_GETRECORDS_INTERVAL_MILLIS` keys in the supplied configuration properties. Setting the former
adjusts the maximum number of records each consuming thread tries to fetch from shards on each call (default is 10,000), while
the latter modifies the sleep interval between each fetch (default is 200). The retry behaviour of the
consumer when calling this API can also be modified by using the other keys prefixed by `ConsumerConfigConstants.SHARD_GETRECORDS_*`.

#### Enhanced Fan-Out Record Publisher

- *[SubscribeToShard](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_SubscribeToShard.html)*: this is called
by per shard consuming threads to obtain shard subscriptions. A shard subscription is typically active for 5 minutes, 
but subscriptions will be reaquired if any recoverable errors are thrown. Once a subscription is acquired, the consumer
will receive a stream of [SubscribeToShardEvents](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_SubscribeToShardEvent.html)s.
Retry and backoff parameters can be configured using the `ConsumerConfigConstants.SUBSCRIBE_TO_SHARD_*` keys.

- *[DescribeStreamSummary](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_DescribeStreamSummary.html)*: this is called 
once per stream, during stream consumer registration. By default, the `LAZY` registration strategy will scale the
number of calls by the job parallelism. `EAGER` will invoke this once per stream and `NONE` will not invoke this API. 
Retry and backoff parameters can be configured using the 
`ConsumerConfigConstants.STREAM_DESCRIBE_*` keys.

- *[DescribeStreamConsumer](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_DescribeStreamConsumer.html)*:
this is called during stream consumer registration and deregistration. For each stream this service will be invoked 
periodically until the stream consumer is reported `ACTIVE`/`not found` for registration/deregistration. By default,
the `LAZY` registration strategy will scale the number of calls by the job parallelism. `EAGER` will call the service 
once per stream for registration only. `NONE` will not invoke this service. Retry and backoff parameters can be configured using the 
`ConsumerConfigConstants.DESCRIBE_STREAM_CONSUMER_*` keys.  

- *[RegisterStreamConsumer](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_RegisterStreamConsumer.html)*: 
this is called once per stream during stream consumer registration, unless the `NONE` registration strategy is configured.
Retry and backoff parameters can be configured using the `ConsumerConfigConstants.REGISTER_STREAM_*` keys.

- *[DeregisterStreamConsumer](https://docs.aws.amazon.com/kinesis/latest/APIReference/API_DeregisterStreamConsumer.html)*: 
this is called once per stream during stream consumer deregistration, unless the `NONE` or `EAGER` registration strategy is configured.
Retry and backoff parameters can be configured using the `ConsumerConfigConstants.DEREGISTER_STREAM_*` keys.  

## Kinesis Streams Sink

The Kinesis Streams sink (hereafter "Kinesis sink") uses the [AWS v2 SDK for Java](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/home.html) to write data from a Flink stream into a Kinesis stream.

To write data into a Kinesis stream, make sure the stream is marked as "ACTIVE" in the Amazon Kinesis Data Stream console.

For the monitoring to work, the user accessing the stream needs access to the CloudWatch service.

{{< tabs "6df3b696-c2ca-4f44-bea0-96cf8275d61c" >}}
{{< tab "Java" >}}
```java
Properties sinkProperties = new Properties();
// Required
sinkProperties.put(AWSConfigConstants.AWS_REGION, "us-east-1");

// Optional, provide via alternative routes e.g. environment variables
sinkProperties.put(AWSConfigConstants.AWS_ACCESS_KEY_ID, "aws_access_key_id");
sinkProperties.put(AWSConfigConstants.AWS_SECRET_ACCESS_KEY, "aws_secret_access_key");

KinesisStreamsSink<String> kdsSink =
    KinesisStreamsSink.<String>builder()
        .setKinesisClientProperties(sinkProperties)                               // Required
        .setSerializationSchema(new SimpleStringSchema())                         // Required
        .setPartitionKeyGenerator(element -> String.valueOf(element.hashCode()))  // Required
        .setStreamName("your-stream-name")                                        // Required
        .setFailOnError(false)                                                    // Optional
        .setMaxBatchSize(500)                                                     // Optional
        .setMaxInFlightRequests(50)                                               // Optional
        .setMaxBufferedRequests(10_000)                                           // Optional
        .setMaxBatchSizeInBytes(5 * 1024 * 1024)                                  // Optional
        .setMaxTimeInBufferMS(5000)                                               // Optional
        .setMaxRecordSizeInBytes(1 * 1024 * 1024)                                 // Optional
        .build();

DataStream<String> simpleStringStream = ...;
simpleStringStream.sinkTo(kdsSink);
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
val sinkProperties = new Properties()
// Required
sinkProperties.put(AWSConfigConstants.AWS_REGION, "us-east-1")

// Optional, provide via alternative routes e.g. environment variables
sinkProperties.put(AWSConfigConstants.AWS_ACCESS_KEY_ID, "aws_access_key_id")
sinkProperties.put(AWSConfigConstants.AWS_SECRET_ACCESS_KEY, "aws_secret_access_key")

val kdsSink = KinesisStreamsSink.<String>builder()
    .setKinesisClientProperties(sinkProperties)                               // Required
    .setSerializationSchema(new SimpleStringSchema())                         // Required
    .setPartitionKeyGenerator(element -> String.valueOf(element.hashCode()))  // Required
    .setStreamName("your-stream-name")                                        // Required
    .setFailOnError(false)                                                    // Optional
    .setMaxBatchSize(500)                                                     // Optional
    .setMaxInFlightRequests(50)                                               // Optional
    .setMaxBufferedRequests(10000)                                            // Optional
    .setMaxBatchSizeInBytes(5 * 1024 * 1024)                                  // Optional
    .setMaxTimeInBufferMS(5000)                                               // Optional
    .setMaxRecordSizeInBytes(1 * 1024 * 1024)                                 // Optional
    .build()

val simpleStringStream = ...
simpleStringStream.sinkTo(kdsSink)
```
{{< /tab >}}
{{< tab "Python" >}}
```python
# Required
sink_properties = {
    # Required
    'aws.region': 'us-east-1',
    # Optional, provide via alternative routes e.g. environment variables
    'aws.credentials.provider.basic.accesskeyid': 'aws_access_key_id',
    'aws.credentials.provider.basic.secretkey': 'aws_secret_access_key',
    'aws.endpoint': 'http://localhost:4567'
}

kds_sink = KinesisStreamsSink.builder() \
    .set_kinesis_client_properties(sink_properties) \                      # Required
    .set_serialization_schema(SimpleStringSchema()) \                      # Required
    .set_partition_key_generator(PartitionKeyGenerator.fixed()) \          # Required
    .set_stream_name("your-stream-name") \                                 # Required
    .set_fail_on_error(False) \                                            # Optional
    .set_max_batch_size(500) \                                             # Optional
    .set_max_in_flight_requests(50) \                                      # Optional
    .set_max_buffered_requests(10000) \                                    # Optional
    .set_max_batch_size_in_bytes(5 * 1024 * 1024) \                        # Optional
    .set_max_time_in_buffer_ms(5000) \                                     # Optional
    .set_max_record_size_in_bytes(1 * 1024 * 1024) \                       # Optional
    .build()

simple_string_stream = ...
simple_string_stream.sink_to(kds_sink)
```
{{< /tab >}}
{{< /tabs >}}

The above is a simple example of using the Kinesis sink. Begin by creating a `java.util.Properties` instance with the `AWS_REGION`, `AWS_ACCESS_KEY_ID`, and `AWS_SECRET_ACCESS_KEY` configured. You can then construct the sink with the builder. The default values for the optional configurations are shown above. Some of these values have been set as a result of [configuration on KDS](https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html). 

You will always need to specify your serialization schema and logic for generating a [partition key](https://docs.aws.amazon.com/streams/latest/dev/key-concepts.html#partition-key) from a record.

Some or all of the records in a request may fail to be persisted by Kinesis Data Streams for a number of reasons. If `failOnError` is on, then a runtime exception will be raised. Otherwise those records will be requeued in the buffer for retry.

The Kinesis Sink provides some metrics through Flink's [metrics system]({{< ref "docs/ops/metrics" >}}) to analyze the behavior of the connector. A list of all exposed metrics may be found [here]({{<ref "docs/ops/metrics#kinesis-sink">}}).

The sink default maximum record size is 1MB and maximum batch size is 5MB in line with the Kinesis Data Streams maximums. The AWS documentation detailing these maximums may be found [here](https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html).

### Kinesis Sinks and Fault Tolerance

The sink is designed to participate in Flink's checkpointing to provide at-least-once processing guarantees. It does this by completing any in-flight requests while taking a checkpoint. This effectively assures all requests that were triggered before the checkpoint have been successfully delivered to Kinesis Data Streams, before proceeding to process more records.

If Flink needs to restore from a checkpoint (or savepoint), data that has been written since that checkpoint will be written to Kinesis again, leading to duplicates in the stream. Moreover, the sink uses the `PutRecords` API call internally, which does not guarantee to maintain the order of events.

### Backpressure

Backpressure in the sink arises as the sink buffer fills up and writes to the sink 
begins to exhibit blocking behaviour. More information on the rate restrictions of Kinesis Data Streams may be
found at [Quotas and Limits](https://docs.aws.amazon.com/streams/latest/dev/service-sizes-and-limits.html).

You generally reduce backpressure by increasing the size of the internal queue:

{{< tabs "6df3b696-c2ca-4f44-bea0-96cf8275d61d" >}}
{{< tab "Java" >}}
```java
KinesisStreamsSink<String> kdsSink =
    KinesisStreamsSink.<String>builder()
        ...
        .setMaxBufferedRequests(10_000)
        ...
```
{{< /tab >}}
{{< tab "Python" >}}
```python
kds_sink = KinesisStreamsSink.builder() \
    .set_max_buffered_requests(10000) \
    .build()
```
{{< /tab >}}
{{< /tabs >}}

## Kinesis Producer

{{< hint warning >}}
The old Kinesis sink `org.apache.flink.streaming.connectors.kinesis.FlinkKinesisProducer` is deprecated and may be removed with a future release of Flink, please use [Kinesis Sink]({{<ref "docs/connectors/datastream/kinesis#kinesis-streams-sink">}}) instead.
{{< /hint >}}

The new sink uses the [AWS v2 SDK for Java](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/home.html) whereas the old sink uses the Kinesis Producer Library. Because of this, the new Kinesis sink does not support [aggregation](https://docs.aws.amazon.com/streams/latest/dev/kinesis-kpl-concepts.html#kinesis-kpl-concepts-aggretation).

## Using Custom Kinesis Endpoints

It is sometimes desirable to have Flink operate as a source or sink against a Kinesis VPC endpoint or a non-AWS
Kinesis endpoint such as [Kinesalite](https://github.com/mhart/kinesalite); this is especially useful when performing
functional testing of a Flink application. The AWS endpoint that would normally be inferred by the AWS region set in the
Flink configuration must be overridden via a configuration property.

To override the AWS endpoint, set the `AWSConfigConstants.AWS_ENDPOINT` and `AWSConfigConstants.AWS_REGION` properties. The region will be used to sign the endpoint URL.

{{< tabs "bcadd466-8416-4d3c-a6a7-c46eee0cbd4a" >}}
{{< tab "Java" >}}
```java
Properties config = new Properties();
config.put(AWSConfigConstants.AWS_REGION, "us-east-1");
config.put(AWSConfigConstants.AWS_ACCESS_KEY_ID, "aws_access_key_id");
config.put(AWSConfigConstants.AWS_SECRET_ACCESS_KEY, "aws_secret_access_key");
config.put(AWSConfigConstants.AWS_ENDPOINT, "http://localhost:4567");
```
{{< /tab >}}
{{< tab "Scala" >}}
```scala
val config = new Properties()
config.put(AWSConfigConstants.AWS_REGION, "us-east-1")
config.put(AWSConfigConstants.AWS_ACCESS_KEY_ID, "aws_access_key_id")
config.put(AWSConfigConstants.AWS_SECRET_ACCESS_KEY, "aws_secret_access_key")
config.put(AWSConfigConstants.AWS_ENDPOINT, "http://localhost:4567")
```
{{< /tab >}}
{{< tab "Python" >}}
```python
config = {
    'aws.region': 'us-east-1',
    'aws.credentials.provider.basic.accesskeyid': 'aws_access_key_id',
    'aws.credentials.provider.basic.secretkey': 'aws_secret_access_key',
    'aws.endpoint': 'http://localhost:4567'
}
```
{{< /tab >}}
{{< /tabs >}}

{{< top >}}
