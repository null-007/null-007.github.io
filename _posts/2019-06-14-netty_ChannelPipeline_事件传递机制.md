### 0、目录
	
	1、目标读者
	2、前言
	3、```ChannelPipeline```
	4、事件传递模仿：```EventDelivery```
	5、```EventDelivery``` 测试
	
### 1、目标读者
简单使用过```netty```即可。
### 2、前言
本文主要介绍```netty```中关于```ChannelPipeline```的事件传递机制。首先我们会通过源码分析```ChannelPipeline```中的事件传递机制（netty 3.7）；然后我们有根据分析提炼出来的规则，模仿实现了该事件传递机制，我们称其为```EventDelivery```框架。该框架只关注
事件传递机制，剔除了netty中其他业务相关的内容，所以通过阅读该部分代码，能够更快更清楚地了解该事件传递机制。
[EventDelivery的github地址](https://github.com/null-007/netty-learning/tree/master/netty-3.7/src/test/java/org/jboss/netty/null007/EventDelivery)
### 3、```ChannelPipeline``` 事件传递源码分析
```ChannelPipeline```的整个设计思路是如下图所示。通过```ChannelPipeline```将```Channel```和一组 ```ChannelHandler``` 联系在一起：


    +----------------------------------------+---------------+
    *  |                  ChannelPipeline       |               |
    *  |                                       \|/              |
    *  |  +----------------------+  +-----------+------------+  |
    *  |  | Upstream Handler  N  |  | Downstream Handler  1  |  |
    *  |  +----------+-----------+  +-----------+------------+  |
    *  |            /|\                         |               |
    *  |             |                         \|/              |
    *  |  +----------+-----------+  +-----------+------------+  |
    *  |  | Upstream Handler N-1 |  | Downstream Handler  2  |  |
    *  |  +----------+-----------+  +-----------+------------+  |
    *  |            /|\                         .               |
    *  |             .                          .               |
    *  |     [ sendUpstream() ]        [ sendDownstream() ]     |
    *  |     [ + INBOUND data ]        [ + OUTBOUND data  ]     |
    *  |             .                          .               |
    *  |             .                         \|/              |
    *  |  +----------+-----------+  +-----------+------------+  |
    *  |  | Upstream Handler  2  |  | Downstream Handler M-1 |  |
    *  |  +----------+-----------+  +-----------+------------+  |
    *  |            /|\                         |               |
    *  |             |                         \|/              |
    *  |  +----------+-----------+  +-----------+------------+  |
    *  |  | Upstream Handler  1  |  | Downstream Handler  M  |  |
    *  |  +----------+-----------+  +-----------+------------+  |
    *  |            /|\                         |               |
    *  +-------------+--------------------------+---------------+
    *                |                         \|/
    *  +-------------+--------------------------+---------------+
    *  |             |                          |               |
    *  |     [ Socket.read() ]          [ Socket.write() ]      |
    *  |                                                        |
    *  |  Netty Internal I/O Threads (Transport Implementation) |
    *  +--------------------------------------------------------+

 
```ChannelPipeline```的继承关系如下：


           ChannelPipeline
                  |
        DefaultChannelPipeline   // 核心
                  |
        EmbeddedChannelPipeline  // AbstractCodecEmbedder 的 内部类，仅仅将 notifyHandlerException()方法重写了


可以发现```ChannelPipeline```的核心实现类是```DefaultChannelPipeline```。观察该类的成员：

    public class DefaultChannelPipeline implements ChannelPipeline {
        // 暂时不关心
        static final ChannelSink discardingSink = new DiscardingChannelSink();
        // 管理的 Channel
        private volatile Channel channel;
        // 暂时不关心
        private volatile ChannelSink sink;
        // ChannelHandler被封装在 DefaultChannelHandlerContext里了，而且从 head，tail命名方式可以看出 ChannelHandler是以链表形式被组织了
        private volatile DefaultChannelHandlerContext head;
        private volatile DefaultChannelHandlerContext tail;
        // 链表的查询时O(n)，该Map用于快速查询
        private final Map<String, DefaultChannelHandlerContext> name2ctx =
            new HashMap<String, DefaultChannelHandlerContext>(4);
        ...
    }
	
可以看出```netty```通过```ChannelPipeline```将```Channel```和一组 ```ChannelHandler``` 联系在一起。

在深入看一下 ```DefaultChannelHandlerContext```。它的继承关系如下：

        ChannelHandlerContext       
                 |
    DefaultChannelHandlerContext
	
其成员变量如下：

    private final class DefaultChannelHandlerContext implements ChannelHandlerContext {
        // 就是一个链表节点
        volatile DefaultChannelHandlerContext next;
        volatile DefaultChannelHandlerContext prev;
        // channelHandler的名字
        private final String name;
        // 封装了一个 ChannelHandler
        private final ChannelHandler handler;
        // 判断handler是处理上行事件，还是下行事件
        private final boolean canHandleUpstream;
        private final boolean canHandleDownstream;
        // 暂时不关心
        private volatile Object attachment;
        ...
    }
判断一个```handler```是上行还下行的方法是依据handler实现的接口。观察其构造方法就能看出：

    DefaultChannelHandlerContext(
                DefaultChannelHandlerContext prev, DefaultChannelHandlerContext next,
                String name, ChannelHandler handler) {
            ...
            canHandleUpstream = handler instanceof ChannelUpstreamHandler;
            canHandleDownstream = handler instanceof ChannelDownstreamHandler;
            ...
        }
		
除此外，该类只有下面两个方法有分析的必要了：
    
    public void sendDownstream(ChannelEvent e); // 下行事件传递中，向下一个handler传递事件
    public void sendUpstream(ChannelEvent e);   // 上行事件传递中，向下一个handler传递事件
	
这两个方法的具体分析会在下文中结合 ```DefaultChannelPipeline```的相关方法一起分析：

小结一下，```netty```通过```ChannelPipeline```将```Channel```和一组 ```ChannelHandler``` 联系在一起；而一组 ```ChannelHandler```以链表的形式存在。 

再回到 ```DefaultChannelPipeline```，分析其方法会发现存在大量的```get,add,remove,replace```方法，这不就是再对一个链表进行管理吗！

我们都知道```netty```的一个特点是**异步事件**，也就是事件```Event```会在```handler```链表中传递。接下来我们就分析这个传递过程。
这个传递过程主要涉及6个方法，先简单列一下：
    
    // 上行
    (1) DefaultChannelPipeline#sendUpstream(ChannelEvent);
    (2) DefaultChannelPipeline#sendUpstream(DefaultChannelHandlerContext, ChannelEvent);
    (3) DefaultChannelHandlerContext#sendUpstream(ChannelEvent)
    // 下行
    (4) DefaultChannelPipeline#sendDownstream(ChannelEvent);
    (5) DefaultChannelPipeline#sendDownstream(DefaultChannelHandlerContext, ChannelEvent);
    (6) DefaultChannelHandlerContext#sendDownstream(ChannelEvent)
	
先来看上行事件传递过程。该过程的起始点是方法(1):
    
    public void sendUpstream(ChannelEvent e) {
        // this.head 可能不支持 upStream
        DefaultChannelHandlerContext head = getActualUpstreamContext(this.head);
        ...
        sendUpstream(head, e);
    }
```getActualUpstreamContext()```方法的目的找到上行链的起始节点。```ChannelPipeline```中物理链表只有一条，为了区分上行链和下行链，通过```handler```实现的接口将其组织成两条链：

    DefaultChannelHandlerContext(
                DefaultChannelHandlerContext prev, DefaultChannelHandlerContext next,
                String name, ChannelHandler handler) {
            ...
            canHandleUpstream = handler instanceof ChannelUpstreamHandler;
            canHandleDownstream = handler instanceof ChannelDownstreamHandler;
            ...
        }
		
所以需要找到上行链真正的```head```。
```sendUpstream(head, e)```方法即方法(2)，表示的是以当前节点为起始节点传递事件：

    void sendUpstream(DefaultChannelHandlerContext ctx, ChannelEvent e) {
        try {
            // 触发 handler.handlerUpstream
            ((ChannelUpstreamHandler) ctx.getHandler()).handleUpstream(ctx, e);
        } catch (Throwable t) {
            notifyHandlerException(e, t);
        }
    }

    
```handler.handlerUpstream```其目的主要有两个：一是根据事件类型将事件分发（调用）给相应的方法；二是根据需要将事件传递给下一个```handler```。我们找了一个```SimpleChannelUpstreamHandler#handleUpstream```来分析一下，发现下面这一堆代码就是为了上述的两个目的：

    public void handleUpstream(
            ChannelHandlerContext ctx, ChannelEvent e) throws Exception {

        if (e instanceof MessageEvent) {
            // 如果是 MessageEvent，就调用messageReceived()方法。下同
            messageReceived(ctx, (MessageEvent) e);
        } else if (e instanceof WriteCompletionEvent) {
            WriteCompletionEvent evt = (WriteCompletionEvent) e;
            writeComplete(ctx, evt);
        } else if (e instanceof ChildChannelStateEvent) {
            ChildChannelStateEvent evt = (ChildChannelStateEvent) e;
            if (evt.getChildChannel().isOpen()) {
                childChannelOpen(ctx, evt);
            } else {
                childChannelClosed(ctx, evt);
            }
        } else if (e instanceof ChannelStateEvent) {
            ChannelStateEvent evt = (ChannelStateEvent) e;
            switch (evt.getState()) {
            case OPEN:
                if (Boolean.TRUE.equals(evt.getValue())) {
                    channelOpen(ctx, evt);
                } else {
                    channelClosed(ctx, evt);
                }
                break;
            case BOUND:
                if (evt.getValue() != null) {
                    channelBound(ctx, evt);
                } else {
                    channelUnbound(ctx, evt);
                }
                break;
            case CONNECTED:
                if (evt.getValue() != null) {
                    channelConnected(ctx, evt);
                } else {
                    channelDisconnected(ctx, evt);
                }
                break;
            case INTEREST_OPS:
                channelInterestChanged(ctx, evt);
                break;
            default:
                ctx.sendUpstream(e);
            }
        } else if (e instanceof ExceptionEvent) {
            exceptionCaught(ctx, (ExceptionEvent) e);
        } else {
            // 事件传递给下一个handler 
            ctx.sendUpstream(e);
        }
    }
	
有一点特别重要，根据该方法的条件分支形式可以发现，事件是否传递给下一个```handler```完全由处理事件的方法决定。只有当所有事件类型都不满足时才会自动传递给下一个```handler```。可以看一下```messageReceived()```方法:

    public void messageReceived(
            ChannelHandlerContext ctx, MessageEvent e) throws Exception {
        // 处理任务
        ...
        // 传递给下一handler
        ctx.sendUpstream(e);
    }
	
到了这一步了，我们来看看到底是如何传递给下一个```handler```的。记住，上文的```ctx```可是```ChannelPipeline```中的一个节点哦。```ctx.sendUpstream(e)```就是上文的方法(3):

    public void sendUpstream(ChannelEvent e) {
            // 寻找下一个合法上行节点
            DefaultChannelHandlerContext next = getActualUpstreamContext(this.next);
            if (next != null) {
                // 触发下一个节点的 handler.upStream()方法
                DefaultChannelPipeline.this.sendUpstream(next, e);
            }
        }
		
总结一下上行中整个事件传递的调用关系：

    (1) DefaultChannelPipeline#sendUpstream(ChannelEvent); // 事件刚刚产生时会调用，之后不会再调用
    
    (2) DefaultChannelPipeline#sendUpstream(DefaultChannelHandlerContext, ChannelEvent);

    (3) SimpleChannelUpstreamHandler#handleUpstream(ChannelHandlerContext, ChannelEvent);
    
    (4) SimpleChannelUpstreamHandler#messageReceived(ChannelHandlerContext ctx, ChannelEvent e);
    
    (5) DefaultChannelHandlerContext#sendUpstream(ChannelEvent);
    
    (1) -----> (2) -----> (3) -----> (4)
                ^                     |
                |                     |
                |---------(5)<--------|
                

### 4、事件传递模仿：```EventDelivery```
为了加深映像，我们将上述框架中的关于事件传递的核心提取出来，自己模仿实现了该事件传递框架```EventDelivery```，完整的代码在
[EventDelivery的github地址](https://github.com/null-007/netty-learning/tree/master/netty-3.7/src/test/java/org/jboss/netty/null007/EventDelivery)
代码目录如下：
![](https://null-007.github.io/img/2019_06_14/EventDelivery目录.png)

### 5、```EventDelivery```测试
	

	public class PipelineTest {

		public static void main(String[] args) {
			// 创建 ChannelHandler
			ChannelHandler upHandlerFirst = new UpstreamChannelHandlerFirst();
			ChannelHandler upHandlerSecond = new UpstreamChannelHandlerSecond();
			ChannelHandler downHandlerFirst = new DownStreamChannelHandlerFirst();

			// 创建 ChannelPipeline
			ChannelPipeline channelPipeline = new ChannelPipeline(new Channel());
			channelPipeline.add(upHandlerFirst);
			channelPipeline.add(upHandlerSecond);
			channelPipeline.add(downHandlerFirst);
			// 创建 Connect 事件
			ChannelEvent connectEvent = new ConnectEvent("connect to the world, hello!");
			// 传递事件
			channelPipeline.upstreamSendEvent(connectEvent);
		}
	}



测试结果：
	

	连接事件被UpstreamChannelHandlerFirst处理了: 事件内容是: connect to the world, hello!
	连接事件被UpstreamChannelHandlerSecond处理了: 事件内容是: connect to the world, hello!

    
    
    

    















