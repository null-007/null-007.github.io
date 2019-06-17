java.nio.channel 包下主要的元素是 ```Channel```, ```Selector```, ```Buffer```。它们三者之间的关系如下：
    
           读写          注册
    Buffer <--> Channel <--> Selector 
    
## 1、Channel
channel 的继承图如下所示：
![](https://null-007.github.io/img/2019_06_16/channel.png)
##### 首先第一层是 ```Channel```

    /*
    Channel代表了实体之间的连接，如硬件、文件、socket、进程等。它只有 open和close两种状态，一旦通道被关闭了，那么它将永远关闭，任何企图进行 IO 操作都会抛出一个ClosedChannelException异常。
    */
    public interface Channel extends Closeable {
        public boolean isOpen();
        /*
        需要注意以下事情：
        1、如果Channel已经关闭了，那么执行任何该通道的 IO 操作都会抛出ClosedChannelException异常
        2、如果Channel已经关闭了，那么再次调用close()没有任何效果和副作用
        3、如果Channel正在被关闭(该操作是阻塞的)，那么其它线程调用close()将被阻塞，直到close()完成并返回，并没有任何作用。
        */
        public void close() throws IOException;
    }
    


##### ```Channel```需要有可中断的能力
即当通道的操作被无限制阻塞时，需要能够被中断(就像线程)。因此产生了 ```InterruptibleChannel```

    /**
        InterruptibleChannel 能够被 异步关闭 和 可中断
        1、如果一个线程在该 channel 上的IO操作被阻塞了，那么其它线程可以调用 该 channel.close()实现异步关闭，阻塞线程会抛出异常  AsynchronousCloseException
        2、如果一个线程在该 channel 上的IO操作被阻塞了，那么其它线程可以调用该阻塞线程的 Thread#interrupt()。阻塞线程会有三个效果：channel 被关闭；抛出异常 ClosedByInterruptException；线程 interrupt 状态被设置；
        3、如果一个线程的interrupt状态已经被设置，那么任何该channel的IO
        阻塞操作会立即返回，线程会有三个效果：channel 被关闭；抛出异常 ClosedByInterruptException；线程 interrupt 状态保持；
    */
    public interface InterruptibleChannel extends Channel
    {
        public void close() throws IOException;
    
    }
    
##### 接下来 ```channel```需要有读写能力
因此就出来了一个```ByteChannel```。该接口其实是对```ReadableByteChannel```和```WritableByteChannel```能力对聚合。
    
                    ByteChannel
                         ^
            -------------|-------------
            |                         |
    ReadableByteChannel     WritableByteChannel
    
至于```ScatteringByteChannel```, ```GatheringByteChannel```就是黑科技 矢量IO 对实现了。

##### ```Channel```具体实现
接下来就是具体对 ```Channel```的实现类了：
```FileChannel```:文件IO，不讨论
```SocketChannel```, ```ServerSocketChannel```, ```DatagramChannel```：网络IO
可以发现```ServerSocketChannel```是不负责数据读写的，只是用来监听并创建连接，因此不实现```ByteChannel```接口，当然也就不具备矢量IO的能力了；而其它几个IO都是双向的（可读写）。

##### ```channel```得具备多路IO复用能力
因此就出现了```SelectableChannel```.

    /**
        支持多路IO复用
        1、SelectableChannel，Selector，optSet 通过#register(Selector,int,Object) 实现绑定（注册）
            返回值 SelectionKey 表示了它们的注册关系
        2、SelectableChannel不能直接解除注册关系。而是需要先将SelectionKey设置为cancelled，
            然后在下一轮的select操作中进行注册解除。
            key被cancelled的方式有：1）直接调用 SelectionKey#cancel()
                                    2）Channel被关闭时，该Channel所属的所有key都会被cancelled（一个cahnnel可以注册到多个Selector上）；
                                       Channel关闭方式可能是同步关闭，异步关闭，或者通过中断关闭；
            如果 Selector被close了，那么注册关系将直接解除；key直接无效，没有任何延时；
         3、SelectableChannel是线程安全的；
         4、SelectableChannel有阻塞模式和非阻塞模式；非阻塞模式需要和 Selector 配合使用，才能达到概念上的非阻塞；
         
     * @see SelectionKey
     * @see Selector
     */
    
    public abstract class SelectableChannel
        extends AbstractInterruptibleChannel
        implements Channel
    {
        protected SelectableChannel() { }
    
        // 返回创建 SelectableChannel 的provider
        public abstract SelectorProvider provider();
        // 不同的channel能够支持的操作是不同的，如 ServerSocketChannel 只支持 ACCEPT
        public abstract int validOps();
        public abstract boolean isRegistered();// 同步保护
        public abstract SelectionKey keyFor(Selector sel);
    
        /**
         多次注册只是对之前的注册关系 key 的更新
         该方法和configureBlocking是同步的
         对于注册已经 closed的channel，返回的key是cancelled的
         */
        public abstract SelectionKey register(Selector sel, int ops, Object att)
            throws ClosedChannelException;
            
        public abstract SelectableChannel configureBlocking(boolean block)
            throws IOException;
    
        public abstract boolean isBlocking();
    
        /**
        这把锁在 configureBlocking() 和 register() 起到同步保护
        居然把它暴露出来了
        */
        public abstract Object blockingLock();
    
    }
    
##### ```Channel```和```socket```的关系
```channel``` 和 ```socket``` 其实是代理的关系；
每个 ```channel``` 内会存在一个与之对应的```socket(DatagramSocket, Socekt, ServerSocket)```;
当然现在 ```socket``` 内也会引用一个 ```channel```；

到此，```channel```组织框架就分析完了!




    











