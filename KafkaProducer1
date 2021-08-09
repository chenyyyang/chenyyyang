> Kafka-client 版本 2.2.2

这里用一个demo来解释这个问题的原因和排查思路
```
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;

public class MessageQueueProducer {

    KafkaProducer kafkaProducer;

    public MessageQueueProducer(KafkaProducer kafkaProducer) {
        this.kafkaProducer = kafkaProducer;
    }

    public synchronized void send(final String topic, final String message) {
        ProducerRecord<String, String> kafkaMessage = new ProducerRecord<String, String>(topic, message);

        kafkaProducer.send(kafkaMessage, (recordMetadata, e) -> {
                    if (e == null) {
                        //success
                    } else {
                        //send "retry-topic" to retry
                        send("retry-topic", message);
                    }
                }
        );
    }

    public synchronized void close() {
        kafkaProducer.flush();
        kafkaProducer.close();
    }

}
```
### 原因分析
1. 假设我们对KafkaProducer进行了一个简单的封装（如上）  
2. 假设有两个线程 threadA 和  threadB  
3. threadA 调用 MessageQueueProducer.close方法,close方法中的 flush本意是想，在Producer被close之前把buffer中数据一次性发送到Broker来保障数据的完整。
4. 所有方法都有synchronized修饰，所以threadA拿到了MessageQueueProducer对象锁
5. flush方法要等待所有的数据发送完成并收到Broker的确认

**就在此刻，问题来了**  
假设threadB 调用了 MessageQueueProducer.send时遇到了异常，需要在Kafka的回调函数中向重试队列发送消息，进行异步重试
>Note that callbacks will generally execute in the I/O thread of the producer and so should be reasonably fast or they will delay the sending of messages from other threads. If you want to execute blocking or computationally expensive callbacks it is recommended to use your own java.util.concurrent.Executor in the callback body to parallelize processing.

参看Kafka的文档，回掉函数是由Producer的ioThread进行调用的，所以此刻，ioThread开始请求  MessageQueueProducer 这个对象的对象锁，但是这个对象的锁已经被threadA占有，且threadA在等待ioThread执行所有回调，来保证消息发送完成。  
**所以MessageQueueProducer对象就一直被锁住了，send也send不了，close也close不了**  


### flush方法的源码解析
```
RecordAccumulator.java :690

/**
 * Mark all partitions as ready to send and block until the send is complete
 */
public void awaitFlushCompletion() throws InterruptedException {
    try {
        for (ProducerBatch batch : this.incomplete.copyAll())
        //所有未完成的消息，挨个等待完成确认
            batch.produceFuture.await();
    } finally {
        this.flushesInProgress.decrementAndGet();
    }
}



produceFuture :ProduceRequestResult

    /**
     * Mark this request as complete and unblock any threads waiting on its completion.
     */
    public void done() {
        if (baseOffset == null)
            throw new IllegalStateException("The method `set` must be invoked before this method.");
        this.latch.countDown();
    }

    /**
     * Await the completion of this request
     */
    public void await() throws InterruptedException {
        latch.await();
    }
可以发现CountDownLatch 只有在done方法中会执行countDown，看一下done方法是不是被ioThread调用的

Sender.completeBatch --> ProducerBatch.done --> ProducerBatch.completeFutureAndFireCallbacks  -->
produceFuture.done

 
```


### 问题排查的思路

##### 现象
很明显，就发现调用MessageQueueProducer.send方法的线程都HANG住了，没法继续执行。  
上面例子中就是threadA 和 threadB都处于 BLOCKED状态。

##### 告警
告警对于程序的健壮太重要了，上面的情况如果没有告警，可能难于发现，告警指标可以根据业务需要来配置，比如定时produce消息的线程十分钟还不发一条消息就告警。

##### 排查
1. 到机器上启动Arthas attch当前应用
2. 执行thread -all，查看所有线程的状态
3. 可以看到threadA 和 threadB都是阻塞状态

![Screenshot2021_08_08_085037.jpg](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3256b3a755a04c57af83dfdc3b6eb33a~tplv-k3u1fbpfcp-watermark.image)

4. 执行thread -b
5. Arthas找到 阻塞其他线程的罪魁祸首，也就是threadA。打印threadA的堆栈
6. jstack -l pid 拿到所有线程的堆栈
7. 对比被threadA 阻塞的线程数量和 Arthas输出的信息是否一致。
