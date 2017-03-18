启动之前：

    ./bin/zookeeper-server-start.sh config/zookeeper.properties
    
上面是Kafka自带的ZK，一般都是使用自己安装的ZK。

## 启动：

    ./bin/kafka-server-start.sh config/server.properties &
    
控制台创建topic：

    ./bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
    
控制台查看topic：

    ./bin/kafka-topics.sh --list --zookeeper localhost:2181
    
配置broker，发布一个不存在的topic时，自动创建topic。

控制台发送msg：

    ./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test 
    
控制台消费msg：

    ./bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning
    
设置多个broker集群：

首先为每个broker创建一个配置文件：

    cp config/server.properties config/server-1.properties 
    cp config/server.properties config/server-2.properties
    
编辑新建的文件，设置以下属性：

    config/server-1.properties: 
        broker.id=1 
        listeners=PLAINTEXT://:9093 
        log.dir=/tmp/kafka-logs-1

    config/server-2.properties: 
        broker.id=2 
        listeners=PLAINTEXT://:9094 
        log.dir=/tmp/kafka-logs-2
        
broker.id 是集群中每个节点的唯一且永久的名称，修改端口和日志分区是因为在同一台机器上运行，需要防止broker在同一端口上注册和覆盖对方的数据。

启动新的Kafka节点：

    ./bin/kafka-server-start.sh config/server-1.properties &
    
    ./bin/kafka-server-start.sh config/server-2.properties &
    
查看topic详情：

    ./bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
    
## 生产者API：

鼓励所有开发者使用新的JAVA生产者。新的java客户端比老版本的scala的客户端更快，更全面。

    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>0.10.1.0</version>
    </dependency>
    
kafka客户端发布record（消息）到kafka集群。
新的生产者是线程安全的，在线程之间共享单个生产者实例，通常比多个实例要快。

废话少说，直接来一个例子：

    package com.chuyun.kafkaConsumer;
    import org.apache.kafka.clients.producer.KafkaProducer;
    import org.apache.kafka.clients.producer.ProducerRecord;
    import org.apache.kafka.clients.producer.RecordMetadata;
    
    import java.util.Properties;
    import java.util.concurrent.ExecutionException;
    import java.util.concurrent.Future;
    
    /**
     * Created by DELL on 2017/3/17.
     */
    public class KafkaProducerDemo {
        public static void main(String[] args) throws ExecutionException, InterruptedException {
            Properties props = new Properties();
            props.put("bootstrap.servers", "spark-dev:9092");
            props.put("acks", "all");//判别请求是否为完整的条件，all将会阻塞消息，这种设置性能最低，但是是最可靠的。
            props.put("retries", 0);//如果请求失败，重试次数，如果启用重试，则会有重复消息的可能性。
            props.put("batch.size", 16384);//producer缓冲每个分区未发送消息，缓冲大小的设置，越大需要更多内存。
            props.put("linger.ms", 1);//0的话，不满缓冲直接发送，大于0，等待一段时间，1毫秒延迟，换取更有效的请求。
            props.put("buffer.memory", 33554432); //控制生产者可用的缓存总量，如果消息生产快与传输，将慢慢消耗这个缓存总量，消耗完了，其他
            //发送将被阻塞，超过设定阻塞时间，跑出TimeoutException。
            props.put("max.brock.ms",1024);//设定消息阻塞时间。
            props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");//key，value生产者是ProducerRecord对象，需要转换
            props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");//ByteArraySerializer和StringSerializer处理byte和String类型
    
            KafkaProducer<String, String> producer = new KafkaProducer<String, String>(props);
            Future<RecordMetadata> rs; //发送的结果是RecordMetadata，指定了消息发送的分区，分配的offset和消息的时间戳。send调用时异步的，为分配消息的RecordMetadata返回一个
            //Future，如果future调用get()，则将阻塞，知道相关请求完成并返回该消息的metadata，或抛出发送异常。
            for(int i = 0; i < 100; i++){
                //send()方法是异步的，添加消息到缓冲区等待发送，并立即返回，生产者将单个的消息批量在一起发送来提高效率。
                RecordMetadata result = producer.send(new ProducerRecord<String, String>("my-topic", Integer.toString(i), Integer.toString(2 * i))).get();
                System.out.println("topic:"+result.topic() +  " partition:" + result.partition() + "   offset:"+result.offset());
            }
           
            producer.close(); //生产者的缓冲空间池保留尚未发送到服务器的消息，后台IO线程负责将这些消息转换成请求发送到集群，如果使用后不关闭
            //生产者，则会泄露这些资源。
        }
    }
    
 想要完全无阻塞的话，可以利用回调参数提供的请求完成时将调用的回调通知。
 
        ProducerRecord<byte[],byte[]> record = new ProducerRecord<byte[],byte[]>("the-topic", key, value);
     producer.send(myRecord,
                   new Callback() {
                       public void onCompletion(RecordMetadata metadata, Exception e) {
                           if(e != null)
                               e.printStackTrace();
                           System.out.println("The offset of the record we just sent is: " + metadata.offset());
                       }
                   });


发送到同一个分区的消息回调保证按照一定的顺序执行，例子：

    producer.send(new ProducerRecord<byte[],byte[]>(topic, partition, key1, value1), callback1);
    producer.send(new ProducerRecord<byte[],byte[]>(topic, partition, key2, value2), callback2);
    
callback一般在生产者的IO线程中执行，是很快的，否则将延迟其他的线程的消息发送，如果需要执行阻塞或消耗的回调，建议在callcak主体中使用自己的executor来并行处理。

InterruptException - 如果线程在阻塞中断。

KafkaException - Kafka有关的错误，不属于API的异常。

## 消费者API
0.9版本以后，增加一个新的Java消费者替代现有的基于Zookeeper的高级和低级消费者。新的消费API，清除了0.8版本的高版本和低版本之间的区别。

    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>0.10.1.0</version>
    </dependency>
    
消费者的位置给出了下一条记录的偏移量，在每次消费者调用poll(long)中接收消息时自动增长。

“已提交”的位置是已安全保存的最后偏移量，如果进程失败或重新启动时，消费者将恢复到这个偏移量，消费者可以徐娜则定期自动提交偏移量，也可以选择通过调用commit API来手动的控制（如commitSync和commitAsync）。

之所以搞这么复杂，直观原因就是如果定义什么时候才算消费完数据呢。

group.id是一组消费者组，组内的消费者通过subscribe API动态的订阅一个topic列表。Kafka将已订阅topic的消息发送到每个消费者组中。并通过平衡分区在消费者组所有成员之间平均。比如一个topic有4分分区，一个消费者组有2个消费者，每个消费者消费2个分区。

消费者需要持续的调用poll,消费者将一直保持可用，并继续从分配的分区中接收消息。消费者向服务器定时发送心跳，如果消费者奔溃或无法再session.timeout.ms配置的时间内发送心跳，则消费者将被视为死亡，并且其分区将被重新分配。

但是持续发送心跳，并没有处理也不行，kafka服务器可能认为消费端出现了“活锁”情况，一直持有分区，max.poll.interval.ms活跃检测机制：调用poll的频率大于最大间隔，则客户端将主动离开组，以便其他消费者接管该分区。出现offset提交失败。

消费者提供两个配置设置来控制poll循环：
    
- max.poll.nterval.ms:增大poll的间隔，为消费者提供更多的时间去处理返回的消息，但会延迟组重新平衡。
- max.poll.records：每次调用poll返回的消息数。

对于那些消息处理不可预测的问题，这些选项是不够的，推荐的方法是将消息移到另一个线程中，让消费者继续调用poll。但是要确保已提交的offset不超过实际位置。必须要禁用自动提交，并在线程完成处理后才为记手动提交偏移量。还需要pause分区，让线程处理完之前返回的消息，不会从poll接收到新消息。

#### 上例子：自动提交偏移量

    Properties props = new Properties();
    props.put("bootstrap.servers", "localhost:9092");
    props.put("group.id", "test");
    props.put("enable.auto.commit", "true");
    props.put("auto.commit.interval.ms", "1000");
    props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
    consumer.subscribe(Arrays.asList("foo", "bar"));
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records)
             System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
    }

设置enable.auto.commit，偏移量由auto.commit.interval.ms控制自动提交的频率。
上面的例子中，客户端订阅了topic主题foo和bar，消费者组叫test。

#### 例子：手动控制偏移量

    Properties props = new Properties();
     props.put("bootstrap.servers", "localhost:9092");
     props.put("group.id", "test");
     props.put("enable.auto.commit", "false");
     props.put("auto.commit.interval.ms", "1000");
     props.put("session.timeout.ms", "30000");
     props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
     consumer.subscribe(Arrays.asList("foo", "bar"));
     final int minBatchSize = 200;
     List<ConsumerRecord<String, String>> buffer = new ArrayList<>();
     while (true) {
         ConsumerRecords<String, String> records = consumer.poll(100);
         for (ConsumerRecord<String, String> record : records) {
             buffer.add(record);
         }
         if (buffer.size() >= minBatchSize) {
             insertIntoDb(buffer);
             consumer.commitSync();
             buffer.clear();
         }
     }
     
这个例子的特点是所有接收到消息为“已提交”，如果想要“至少一次”的保证，就要下次调用poll(long)之前或关闭消费者之前，处理完所有返回的数据。如果处理操作失败，有手动提交offset，那么实际上有些数据没有真正意义上消费到，也就是消息丢失了。

commitSync表示所欲的消息为“已提交”。需要更精细控制，在处理完每个分区中的消息后，提交偏移量。

    try {
         while(running) {
             ConsumerRecords<String, String> records = consumer.poll(Long.MAX_VALUE);
             for (TopicPartition partition : records.partitions()) {
                 List<ConsumerRecord<String, String>> partitionRecords = records.records(partition);
                 for (ConsumerRecord<String, String> record : partitionRecords) {
                     System.out.println(record.offset() + ": " + record.value());
                 }
                 long lastOffset = partitionRecords.get(partitionRecords.size() - 1).offset();
                 consumer.commitSync(Collections.singletonMap(partition, new OffsetAndMetadata(lastOffset + 1)));
             }
         }
     } finally {
       consumer.close();
     }

#### 消费者流量控制
比如流处理，从2个topic中获取消息并把两个topic消息合并，当其中一个topic长时间落后另一个，则暂停消费，以便落后的赶上来。

Kafka支持动态控制消费流量，分别在future的poll(long)中使用pause(Collection)和resume(Collection)来暂停消费指定分配的分区，重新开始消费指定暂停的分区。

#### 多线程处理
Kafka消费者不是线程安全的，所有的网络IO都发生在进行调用应用程序的线程中，非同步访问将导致ConcurrentModificationException。

此规则唯一的例外是wakeup(),它可以安全地从外部线程来中断活动操作。在这种情况下，将从操作的线程阻塞并抛出一个WakeupException。这可用于从其他线程来关闭消费者。

上例子：

	public class KafkaConsumerRunner implements Runnable {
	     private final AtomicBoolean closed = new AtomicBoolean(false);
	     private final KafkaConsumer consumer;
	
	     public void run() {
	         try {
	             consumer.subscribe(Arrays.asList("topic"));
	             while (!closed.get()) {
	                 ConsumerRecords records = consumer.poll(10000);
	                 // Handle new records
	             }
	         } catch (WakeupException e) {
	             // Ignore exception if closing
	             if (!closed.get()) throw e;
	         } finally {
	             consumer.close();
	         }
	     }
	
	     // Shutdown hook which can be called from a separate thread
	     public void shutdown() {
	         closed.set(true);
	         consumer.wakeup();
	     }
	 }

在另外一个单独的线程中，可以通过设置标志和唤醒消费者来关闭消费者。
	
	closed.set(true);
	consumer.wakeup();

#### 线程多少合适？
一般情况下，一个线程作为消费者，一个线程一个消费者实例。其优缺点如下：
- PRO：单线程单消费者实例是最容易实现的
- PRO：不需要县城之间协调，通常是最快的
- PRO：按照顺序处理每个分区，每个线程只处理它接受的消息
- CON：多消费者更多的TCP连接到集群，每个线程一个连接，Kafka处理连接非常快
- CON：更多消费者，意味更多请求发送到服务器，数据低谷时导致IO吞吐量下降
- CON：所有进程中的线程总数受到分区总数的限制

#### 解耦消费和处理
一个或多个消费者线程，来消费所有数据，其消费所有数据并将COnsumerRecords实例切换到由实际处理记录的处理器线程池来消费的阻塞队列。
-PRO：可扩展消费者和处理进程的数量，单个消费者的数据可以分发给多个处理器线程来执行，避免对分区的任何限制。
-CON：线程是独立执行的，后来的消息可能优先处理，顺序是没有可能了。
-CON：手动提交变得更困难了。

还可以搞得更加复杂，每个线程可以有自己的队列。


## Kafka Stream API
在0.10.0增加了一个新的客户端库，Kafka Stream。

	 <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-streams</artifactId>
        <version>0.10.0.0</version>
    </dependency>

这个还是放弃吧，直接使用Spark Streaming消费Kafka数据来做实时处理吧！








