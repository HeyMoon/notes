我们知道，ChannelPipeline初始化之后，其中包含两个Context:HeadContext,TailContext。
我们看看这两个类的声明：

````
final class HeadContext extends AbstractChannelHandlerContext
            implements ChannelOutboundHandler, ChannelInboundHandler

final class TailContext extends AbstractChannelHandlerContext implements ChannelInboundHandler
````

我们可以看到`HeadContext`即实现了`ChannelOutboundHandler`,也实现了`ChannelInboundHandler`.而
`TailContext`只实现了`ChannelInboundHandler`.

这有什么区别呢？这关系到Netty的事件传递。在Netty中有两种事件，inBound和outBound事件。
