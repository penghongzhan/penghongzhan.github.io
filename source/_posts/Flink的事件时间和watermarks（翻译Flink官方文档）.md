---
title: Flink的事件时间和watermarks（翻译Flink官方文档）
date: 2018-05-02 22:14:29
tags:
- flink
- watermark
categories:
- flink
---

---

翻译：[Event Time](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/event_time.html)

---
# Event Time / Processing Time / Ingestion Time（事件时间/处理时间/摄入时间）
---

flink支持不同的时间概念：

- 处理时间：当前机器处理该条事件的时间

流处理程序使用该时间进行处理的时候，所有的操作（类似于时间窗口）都会使用当前机器的时间，例如按照小时时间窗进行处理，程序将处理该机器一个小时内接收到的数据。

```
However, in distributed and asynchronous environments processing time does not provide determinism, because it is susceptible to the speed at which records arrive in the system (for example from the message queue), and to the speed at which the records flow between operators inside the system.
```

处理时间是最简单的概念，不需要协调机器时间和流中事件相关的时间。他提供了最小的延时和最佳的性能。但是在分布式和异步环境中，处理时间不能提供确定性，因为它对事件到达系统的速度和数据流在系统的各个operator之间处理的速度很敏感。

- 事件时间：事件时间是每条事件在它产生的时候记录的时间，该时间记录在事件中，在处理的时候可以被提取出来。小时的时间窗处理将会包含事件时间在该小时内的所有事件，而忽略事件到达的时间和到达的顺序

事件时间对于乱序、延时、或者数据重放等情况，都能给出正确的结果。事件时间依赖于事件本身，而跟物理时钟没有关系。利用事件时间编程必须指定如何生成事件时间的watermark，这是使用事件时间处理事件的机制。机制是这样描述的：

事件时间处理通常存在一定的延时，因此自然的需要为延时和无序的事件等待一段时间。因此，使用事件时间编程通常需要与处理时间相结合。

- 摄入时间：摄入时间是事件进入flink的时间，在source operator中，每个事件拿到当前时间作为时间戳，后续的时间窗口基于该时间。

摄入时间在概念上处于事件时间和处理时间之间，与处理时间相比稍微昂贵一点，但是能过够给出更多可预测的结果。因为摄入时间使用的是source operator产生的不变的时间，后续不同的operator都将基于这个不变的时间进行处理，但是处理时间使用的是处理消息当时的机器系统时钟的时间。

与事件时间相比，摄入时间无法处理延时和无序的情况，但是不需要明确执行如何生成watermark。

在系统内部，摄入时间采用更类似于事件时间的处理方式进行处理，但是有自动生成的时间戳和自动的watermark。

![三种时间](https://ci.apache.org/projects/flink/flink-docs-release-1.4/fig/times_clocks.svg)

## Setting a Time Characteristic（设置时间特性）

flink程序的第一部分工作通常是设置时间特性，该设置用于定义数据源使用什么时间，在时间窗口处理中使用什么时间。

下面的例子展示了一个flink程序在一个小时的时间窗口中的聚合操作。窗口的操作取决于时间特征。

```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.addSource(new FlinkKafkaConsumer09<MyEvent>(topic, schema, props));

stream
    .keyBy( (event) -> event.getUser() )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) -> a.add(b) )
    .addSink(...);
```

需要注意的是，如果使用事件时间来运行该程序，程序不仅需要直接定义事件的事件时间，还需要发送一个watermark，或者在数据源之后需要注入一个`时间戳分配方法和watermark生成方法`。这些方法描述怎么获取事件时间以及时间流展示的乱序程度。

---
# Event Time and Watermarks（事件时间和watermark）
---

注意：flink的时间流模型有很多的实现技术，推荐先看一下下面两篇文章

- [Streaming 101](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101) by Tyler Akidau
- The [Dataflow Model paper](https://research.google.com/pubs/archive/43864.pdf)

支持事件时间的流处理需要能够获取事件时间的处理进度。对于一个小时时间窗，当事件时间处理超过了这个小时的时候需要被通知，以便operator可以关闭该小时的窗口处理。

```
On the other hand, another streaming program might progress through weeks of event time with only a few seconds of processing, by fast-forwarding through some historic data already buffered in a Kafka topic (or another message queue).
```

事件时间能够独立于处理时间进行处理。例如，在一个程序中，当前operator处理的事件时间可能轻微落后于当前的处理时间，然而两者都已相同的速度进行处理。在另一方面，一个程序可能在数秒之内处理完几周的事件时间，通过快速转发缓存在kafka等消息队列中的消息。（其实主要考虑的是实时系统中，由于各种原因造成的延时，造成某些消息发到flink的时间延时于事件产生的时间，如果基于事件时间的时间窗，可能该时间窗采集了一小时之后还需要等待几分钟，才能接收到这条延时的事件，因此需要检测当前时间窗口的处理进度，可能等待一段时间是必要的，但是不可能无限等待某些延时的时间。）

flink中检测事件时间处理进度的机制是watermark，watermark跟事件一样在流中进行传输并且携带一个时间戳`t`。一个watermark(t)声明了在流中的事件时间有一个到达时间t，意味着流中应该不再有时间比t小的事件（例如某个事件的时间戳比watermark的时间戳老）。

下图展示了包含时间戳和watermark的事件流。在该例子中，时间是有顺序的，意味着该流中的watermark值是一个简单的标记。

![image](https://ci.apache.org/projects/flink/flink-docs-release-1.4/fig/stream_watermark_in_order.svg)

watermarks对于乱序的流至关重要，例如下面时间戳乱序的事件流。一般来说，watermark是用来声明，在这一点上，某个特定时间戳之前的所有事件都已经到达。一旦一个watermark到达了operator，operator可以将内部事件时间提前到watermark的时间戳。

![image](https://ci.apache.org/projects/flink/flink-docs-release-1.4/fig/stream_watermark_out_of_order.svg)

---
# Watermarks in Parallel Streams
---

watermark在source处或者之后生成。source的每个并行的子任务都会独立生成自己的watermark。这些watermarks定义了特定并行source的事件时间。

watermark流经operator的时候，会将该operator的事件时间向前推进。当一个operator的事件时间被推前，它会为后续的operator生成一个新的watermark。

一些operator有多个输入流，例如一个union操作和keyby操作等。这些operator的当前事件时间是各个流的最小的事件时间。随着输入流中事件时间的更新，该operator的事件时间也会更新。

下图展示了并行流中的事件和watermark，operator会跟踪事件时间。

![image](https://ci.apache.org/projects/flink/flink-docs-release-1.4/fig/parallel_streams_watermarks.svg)

---
# Late Elements
---

某些事件(t')可能违背watermark的条件，就是说晚于watermark(t)到达，但是事件时间在watermark之前(t'小于t)。事实上，实际运行过程中，事件可能被延迟任意的时间，所以不可能指定一个时间，保证该时间之前的所有事件都被处理。而且，即使延时时间是有限的，过多的延时watermark的时间也是不理想的，因为这样会造成时间窗口处理的太多延时。

---
# 总结
---

基于时间窗口的处理程序，需要依赖一个时间标识。因为我们进行聚合等操作，最常用的就是根据事件发生的时间进行，比如统计一小时之内的订单数量。如果压根不需要这种以时间作为限制的聚合操作，随便聚合都可以，那么不需要在意上面讲到的事件时间和watermark什么的，随便怎么聚合都可以。

- 事件时间：事件发生的事件，例如一个用户在手机上下单之后生成一条订单事件，下单的时间就是事件时间。
- 处理时间：根据当前operator的本地时间，无法解决事件延时进入flink，也无法解决本来有序进入flink的事件，经过各个节点处理之后，被延时，造成无序。
- 注入时间：优于处理时间，表示事件进入flink的时间，能够解决事件在flink的各个operator之间处理过程中造成的延时，但是无法解决事件是延时到达flink的。

使用事件时间作为窗口操作的条件是最合适的，但是由于某些事件会延时，不可能无限等待这些事件的到来，所以需要制定一个watermark来告诉operator当前的事件时间，时间窗口的处理也会根据这个时间来进行。延时的事件过来之后，如果已经落后于watermark的时间，那么该消息会被丢弃或者采用什么自定义的方案。

---

下面是翻译官方文档中的：[Generating Timestamps / Watermarks](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/event_timestamps_watermarks.html)

---
# watermark的生成方案
---

使用事件时间作为流处理的时间，需要在代码上进行如下的设置：

```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```

---
# 分配时间戳
---

使用事件时间需要每个事件都有一个事件时间戳，通常能够从事件的某个字段中提取出来。

时间戳分配与生成watermark相结合，watermark告诉系统事件时间的处理进度。

有两种分配事件时间和产生watermark的方式：

- 直接从数据源中提取
- 通过一个时间戳分配器和watermark生成器：在flink的时间戳分配器中需要定义要发送的watermark

注意：时间戳和watermark都是采用毫秒，从java的1970-01-01T00:00:00Z时间作为起始。

---
# 具有时间戳和watermark的source函数
---

source函数可以为他生成的时间分配时间戳，并可以向后发送watermark（我理解这个应该是flink自带的，这个时间更像摄入时间）。如果使用这种生成方案，就不需要再定义时间戳分配器了。需要注意的是，如果使用了时间戳分配器，source函数生成的所有的时间戳和watermark都会被重写。

直接在source中将时间戳指定为某个元素，source函数中必须使用`collectWithTimestamp(...)`方法，要想生成watermark，source必须调用`emitWatermark(Watermark)`函数。

下面是source指定时间戳和产生watermark的一个例子：

```
@Override
public void run(SourceContext<MyType> ctx) throws Exception {
	while (/* condition */) {
		MyType next = getNext();
		ctx.collectWithTimestamp(next, next.getEventTimestamp());

		if (next.hasWatermarkTime()) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime()));
		}
	}
}
```

---
# 指定时间戳/生成watermark
---

时间戳分配器接收流，并生成具有时间戳和watermark的新流。如果原始的流中具有时间戳或者watermark，那么将被替换。

时间戳通常在数据源生成之后立即被分配，但是也不是严格要求必须这样。一个常见的情况例如，map和filter函数可以在分配时间戳之前执行。在任何情况下，在第一次基于事件时间的操作进行之前，时间戳必须被分配。例如，使用kafka作为数据源，flink允许在数据源（或consumer）内部指定时间戳分配器和watermark发送器。关于如何操作参见[Kafka Connector documentation](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/connectors/kafka.html)

注意：本章余下的部分介绍程序员必须实现的主要接口，以创建自己的时间戳提取器和watermark发送器。要查看flink预先实现的提取器，请参阅[预定义的时间戳提取器和watermark发送器页面](https://ci.apache.org/projects/flink/flink-docs-release-1.4/dev/event_timestamp_extractors.html)

```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.readFile(
        myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
        FilePathFilter.createDefaultFilter(), typeInfo);

DataStream<MyEvent> withTimestampsAndWatermarks = stream
        .filter( event -> event.severity() == WARNING )
        .assignTimestampsAndWatermarks(new MyTimestampsAndWatermarks());

withTimestampsAndWatermarks
        .keyBy( (event) -> event.getGroup() )
        .timeWindow(Time.seconds(10))
        .reduce( (a, b) -> a.add(b) )
        .addSink(...);
```

---
# 周期生成的watermark
---

`AssignerWithPeriodicWatermarks`定期的分配时间戳和生成watermark（可能依赖于流中的元素或者单纯的依赖于处理时间）。

watermark生成的时间间隔通过`ExecutionConfig.setAutoWatermarkInterval(...)`方法来定义。分配器的`getCurrentWatermark()`方法每次都会被触发，如果返回的结果不为空或者大于上一个的watermark，那么新的watermark将会被发送。

两个分配时间戳并且生成watermark的例子如下：

```
/**
 * This generator generates watermarks assuming that elements arrive out of order,
 * but only to a certain degree. The latest elements for a certain timestamp t will arrive
 * at most n milliseconds after the earliest elements for timestamp t.
 */
/**
 * 该发生器生成watermark，假定元素按顺序到达，但仅在一定程度上到达。某个时间戳t的最新元素将在时间戳t的最早元素之后最多n毫秒。
 */
public class BoundedOutOfOrdernessGenerator extends AssignerWithPeriodicWatermarks<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    @Override
    public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
        long timestamp = element.getCreationTime();
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
        return timestamp;
    }

    @Override
    public Watermark getCurrentWatermark() {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}

/**
 * This generator generates watermarks that are lagging behind processing time by a fixed amount.
 * It assumes that elements arrive in Flink after a bounded delay.
 */
public class TimeLagWatermarkGenerator extends AssignerWithPeriodicWatermarks<MyEvent> {

	private final long maxTimeLag = 5000; // 5 seconds

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark getCurrentWatermark() {
		// return the watermark as current time minus the maximum time lag
		return new Watermark(System.currentTimeMillis() - maxTimeLag);
	}
}
```

上面两种生成watermark的方案为：

- 当前事件的事件时间和当前最大时间（定义的一个变量，初始化为0）两个时间取最大值，这个最大值减去一个允许的延时时间作为watermark的时间
- 系统时间减去允许的延时时间作为watermark的时间

方案二比较容易理解，多介绍一下方案1：

个人理解，但是感觉比方案二对于延时的事件更加宽松一些，如果大批事件发生延时，那么对应的watermark的时间就会往后推。但是方案二的时间只跟当前系统时间有关系，所以方案二对于大批事件出现延时的情况，可能很多被卡在watermark的时间之后出现了，有可能被丢弃。

---
# 带有标记的watermark
---

如果想要在某个事件指定生成新的watermark的时候生成watermark，请调用`AssignerWithPunctuatedWatermarks`。这种情况下，flink首先会调用`extractTimestamp(...)`方法分配时间戳，然后立即调用`checkAndGetNextWatermark(...)`。

`checkAndGetNextWatermark（...）`方法传递在`extractTimestamp（...）`方法中分配的时间戳，并可以决定是否要生成watermark。只要`checkAndGetNextWatermark（...）`方法返回了一个非空的watermark，并且这个watermark的时间大于上一个watermark的时间，那么该watermark将向后发送。

```
public class PunctuatedAssigner extends AssignerWithPunctuatedWatermarks<MyEvent> {

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark checkAndGetNextWatermark(MyEvent lastElement, long extractedTimestamp) {
		return lastElement.hasWatermarkMarker() ? new Watermark(extractedTimestamp) : null;
	}
}
```

注意：每个事件都可以生成watermark。但是，由于每个watermark都会导致一些后续的计算，因此过多的watermark会降低性能。

---
# 每个kafka的partition一个时间戳
---

当使用kafka作为数据源的时候，每个partition都会有一个简单的事件时间模式（时间升序或者有序的无序。。。（这个翻译有点怪））。然而，在消费来自Kafka的流时，多个分区通常会并发消费，交错分区中的事件并销毁每个分区模式（这与Kafka的消费者客户端工作方式有关）。

在这种情况下，可以使用flink的`Kafka-partition-aware`watermark生成器。使用这个特性的时候，watermark会在kafka的消费者中为每个partition生成，并且partition上的watermark的合并方式将会与流shuffle过程中watermark的合并方式相同。

例如，如果partition中的时间戳是严格升序的，那么使用升序时间戳的watermark生成器为每个partition生成watermark，将带来完美的整体的watermark。

下图显示了如何为每个Kafka分区生成watermark以及在这种情况下watermark如何通过流式数据流传播。

**简单的理解（自己的理解）：**

就是直接使用kafka中的事件时间作为watermark的时间，利用的是kafka的每个分区内部的事件时间是有序的。

```
FlinkKafkaConsumer09<MyType> kafkaSource = new FlinkKafkaConsumer09<>("myTopic", schema, props);
kafkaSource.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyType>() {

    @Override
    public long extractAscendingTimestamp(MyType element) {
        return element.eventTimestamp();
    }
});

DataStream<MyType> stream = env.addSource(kafkaSource);
```

![ascending timestamps watermark generator](https://ci.apache.org/projects/flink/flink-docs-release-1.4/fig/parallel_kafka_watermarks.svg)


