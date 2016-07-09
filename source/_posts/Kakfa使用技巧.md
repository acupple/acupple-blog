---
title: Kakfa Consumer使用技巧
date: 2016-04-08 11:18:40
tags: 大数据
---
## high-level consumer
一种high-level版本，比较简单不用关心offset, 会自动的读zookeeper中该Consumer group的last offset
不过要注意一些注意事项，对于多个partition和多个consumer 
- 如果consumer比partition多，是浪费，因为kafka的设计是在一个partition上是不允许并发的，所以consumer数不要大于partition数 
- 如果consumer比partition少，一个consumer会对应于多个partitions，这里主要合理分配consumer数和partition数，否则会导致partition里面的数据被取的不均匀 
    最好partiton数目是consumer数目的整数倍，所以partition数目很重要，比如取24，就很容易设定consumer数目 
- 如果consumer从多个partition读到数据，不保证数据间的顺序性，kafka只保证在一个partition上数据是有序的，但多个partition，根据你读的顺序会有不同 
- 增减consumer，broker，partition会导致rebalance，所以rebalance后consumer对应的partition会发生变化 
- High-level接口中获取不到数据的时候是会block的
```java
Properties props = new Properties();
//必须要加，如果要读旧数据， 默认是lastest
props.put("auto.offset.reset", "smallest"); 
props.put("zookeeper.connect", "localhost:2181");
props.put("group.id", "dashcam");
props.put("zookeeper.session.timeout.ms", "400");
props.put("zookeeper.sync.time.ms", "200");
props.put("auto.commit.interval.ms", "1000");

ConsumerConfig conf = new ConsumerConfig(props);
ConsumerConnector consumer = 
		kafka.consumer.Consumer.createJavaConsumerConnector(conf);
String topic = "page_visits";
Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
topicCountMap.put(topic, new Integer(1));
Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = 
		consumer.createMessageStreams(topicCountMap);
List<KafkaStream<byte[], byte[]>> streams = consumerMap.get(topic);

KafkaStream<byte[], byte[]> stream = streams.get(0); 
ConsumerIterator<byte[], byte[]> it = stream.iterator();
while (it.hasNext()){
    System.out.println("message: " + new String(it.next().message()));
}
```
1. 查看消息consume情况
```bash
bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker \
		--group clog-writer --zookeeper xxxxxx:2181
Group         Topic         Pid  Offset      logSize    Lag   Owner
clog-writer   dashcam       0    52009776    52009861   85    writer-14599
clog-writer   dashcam.env   0    10381       10381      0     writer-1459
```
关键就是offset，logSize和Lag 
这里以前读完了，所以offset=logSize，并且Lag=0
1. 重置offset
```bash
bin/kafka-run-class.sh kafka.tools.UpdateOffsetsInZK \
		earliest config/consumer.properties dashcam
## 然后在检查consumer的offset情况
bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker 
		\--group clog-writer --zookeeper xxxxxx:2181
Group        Topic         Pid  Offset   logSize    Lag        Owner
clog-writer  dashcam       0    0        52009861   52009861   writer-14599
clog-writer  dashcam.env   0    10381    10381      0          writer-1459
```
3个参数， 
[earliest | latest]，表示将offset置到哪里 
consumer.properties ，这里是配置文件的路径 
topic，topic名，这里是dashcam

可以看到offset已经被清0，Lag=logSize
## low-level consumer
当然用这个接口是有代价的，即partition,broker,offset对你不再透明，需要自己去管理这些，并且还要handle broker leader的切换
使用SimpleConsumer的步骤：
首先，必须知道读哪个topic的哪个partition
遍历每个broker，取出该topic的metadata，然后再遍历其中的每个partition metadata，如果找到我们要找的partition就返回 
根据返回的PartitionMetadata.leader().host()找到leader broker
```java
private PartitionMetadata findLeader(List<String> a_seedBrokers, 
	int a_port, String a_topic, int a_partition) {
    PartitionMetadata returnMetaData = null;
    //遍历每个broker 
    for (String seed : a_seedBrokers) { 
        SimpleConsumer consumer = null;
        try {
            //创建Simple Consumer，
            consumer = new SimpleConsumer(seed, a_port, 100000, 
            				64 * 1024, "leaderLookup");
            List<String> topics =Collections.singletonList(a_topic);
            TopicMetadataRequest req = new TopicMetadataRequest(topics);
            //发送TopicMetadata Request请求
            kafka.javaapi.TopicMetadataResponse resp = consumer.send(req); 
            //取到Topic的Metadata 
            List<TopicMetadata> metaData = resp.topicsMetadata(); 
            for (TopicMetadata item : metaData) {
            	//遍历每个partition的metadata
                for(PartitionMetadata part : item.partitionsMetadata())
                {
                	//确认是否是我们要找的partition
                    if (part.partitionId() == a_partition) { 
                        returnMetaData = part;
                        break loop; //找到就返回
                    }
                }
            }
        } catch (Exception e) {
            System.out.println("Error communicating with Broker [" +
            			seed + "] to find Leader for [" + a_topic
                    + ", " + a_partition + "] Reason: " + e);
        } finally {
            if (consumer != null) consumer.close();
        }
    }
    return returnMetaData;
}
```
然后，找到负责该partition的broker leader，从而找到存有该partition副本的那个broker
request主要的信息就是Map<TopicAndPartition, PartitionOffsetRequestInfo>

TopicAndPartition就是对topic和partition信息的封装 
PartitionOffsetRequestInfo的定义 
case class PartitionOffsetRequestInfo(time: Long, maxNumOffsets: Int) 
其中参数time，表示where to start reading data，两个取值 
kafka.api.OffsetRequest.EarliestTime()，the beginning of the data in the logs 
kafka.api.OffsetRequest.LatestTime()，will only stream new messages

不要认为起始的offset一定是0，因为messages会过期，被删除
```java
public long getLastOffset(SimpleConsumer consumer, 
	String topic, int partition, long whichTime, String clientName){
        TopicAndPartition topicAndPartition = 
        			new TopicAndPartition(topic, partition);
        Map<TopicAndPartition, PartitionOffsetRequestInfo> requestInfo = 
        new HashMap<TopicAndPartition, PartitionOffsetRequestInfo>();
        //build offset fetch request info
        requestInfo.put(topicAndPartition, 
        		new PartitionOffsetRequestInfo(whichTime, 1)); 
        kafka.javaapi.OffsetRequest request = 
        		new kafka.javaapi.OffsetRequest(requestInfo, 
                	kafka.api.OffsetRequest.CurrentVersion(),
                	clientName);
        //取到offsets
        OffsetResponse response = consumer.getOffsetsBefore(request); 
 
        if (response.hasError()) {
            System.out.println("Error fetching data Offset Data the" + 
            	"Broker. Reason: "+response.errorCode(topic, partition) );
            return 0;
        }
        //取到的一组offset
        long[] offsets = response.offsets(topic, partition); 
         //取第一个开始读
        return offsets[0];
    }
```
再者，自己去写request并fetch数据 
首先在FetchRequest上加上Fetch，指明topic，partition，开始的offset，读取的大小 
如果producer在写入很大的message时，也许这里指定的1000000是不够的，会返回an empty message set，这时需要增加这个值，直到得到一个非空的message set。
```java
FetchRequest req = new FetchRequestBuilder()
        .clientId(clientName)
        .addFetch(a_topic, a_partition, 
        	readOffset, 100000) // 1000000bytes
        .build();
FetchResponse fetchResponse = consumer.fetch(req);
 
if (fetchResponse.hasError()) {
        // See Error Handling
}
numErrors = 0;
 
long numRead = 0;
for (MessageAndOffset messageAndOffset : 
			fetchResponse.messageSet(a_topic, a_partition)) {
    long currentOffset = messageAndOffset.offset();
    // 因为对于compressed message，会返回整个block，所以可能包含old的message
    if (currentOffset < readOffset) {
        System.out.println("Found an old offset: " 
        		+ currentOffset + " Expecting: " + readOffset);
        continue;
    }
    readOffset = messageAndOffset.nextOffset(); // 获取下一个readOffset
    ByteBuffer payload = messageAndOffset.message().payload();
 
    byte[] bytes = new byte[payload.limit()];
    payload.get(bytes);
    System.out.println(String.valueOf(messageAndOffset.offset()) + ": " 
    		+ new String(bytes, "UTF-8"));
    numRead++;
}
 
if (numRead == 0) {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException ie) {
    }
}
```
Error handling
```java
if (fetchResponse.hasError()) {
     numErrors++;
     // Something went wrong!
     short code = fetchResponse.errorCode(a_topic, a_partition);
     System.out.println("Error fetching data from the Broker:" 
     		+ leadBroker + " Reason: " + code);
     if (numErrors > 5) break;
 
 	// 处理offset非法的问题，用最新的offset
     if (code == ErrorMapping.OffsetOutOfRangeCode())  {
         // We asked for an invalid offset. 
         // For simple case ask for the last element to reset
         readOffset = getLastOffset(consumer,a_topic, a_partition, 
         		kafka.api.OffsetRequest.LatestTime(), clientName);
         continue;
     }
     consumer.close();
     consumer = null;
     // 更新leader broker
     leadBroker = findNewLeader(leadBroker, a_topic, a_partition, a_port); 
     continue;
 }
```
find new leader
```java
private String findNewLeader(String a_oldLeader, String a_topic, 
	int a_partition, int a_port) throws Exception {
       for (int i = 0; i < 3; i++) {
           boolean goToSleep = false;
           PartitionMetadata metadata = 
           		findLeader(m_replicaBrokers, 
           			a_port, a_topic, a_partition);
           if (metadata == null) {
               goToSleep = true;
           } else if (metadata.leader() == null) {
               goToSleep = true;
           } else if (a_oldLeader.equalsIgnoreCase(metadata.leader().host()) 
           				&& i == 0) {
               // first time through if the leader hasn't changed
               // give ZooKeeper a second to recover
               // second time, assume the broker did recover before failover, 
               // or it was a non-Broker issue
               goToSleep = true;
           } else {
               return metadata.leader().host();
           }
           if (goToSleep) {
               try {
                   Thread.sleep(1000);
               } catch (InterruptedException ie) {
               }
           }
     }
    System.out.println("Unable to find new leader after Broker failure. " +
       		"Exiting");
    throw new Exception("Unable to find new leader after Broker failure." + 
       		"Exiting");
   }
```