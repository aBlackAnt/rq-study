# 一、remoting功能

基于netty的底层通信实现，把通信功能和消息处理功能分离，不同类型的通信内容被抽象成发送带有对应类型代码的Command，同时根据类型代码查找对应的Processor和Executor来执行，所有服务间的交互都基于此模块。（关于netty可参阅 Netty实战）

参考：http://www.iocoder.cn/RocketMQ/huzhongtang/rpc-1/

​	    http://www.iocoder.cn/RocketMQ/huzhongtang/rpc-2/

# 二、代码结构

## 2.1 源码结构

![](https://github.com/aBlackAnt/rq-study/blob/master/remoting/images/code_all.png?raw=true)

## 2.2 类关系图

![](https://github.com/aBlackAnt/rq-study/blob/master/remoting/images/class_pic.png?raw=true)

***RemotingServer/RemotingClient***：都继承了```RemotingService```，分别为Server和Client提供所必须的方法；

***NettyRemotingServer/NettyRemotingClient***：分别实现了```RemotingServer```和```RemotingClient```，都继承了```NettyRemotingAbstract```抽象类。RocketMQ中其他的组件（如nameServer、broker在进行消息发送和接收时均使用这两个组件）

# 三、源码解析

思路：Client端发送请求消息、Server端接收消息并处理的流程以及回调过程

## 3.1 Client发送消息

支持三种通信方式：同步（sync）、异步（async）、单向（oneway）

***以异步为例***：

调用```NettyRemotingClient```的 invokeAsync 方法，如果根据addr获取的channel可用，则调用```NettyRemotingAbstract```的 invokeAsyncImpl 方法。

```java
public void invokeAsyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis, final InvokeCallback invokeCallback)
        throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
        long beginStartTime = System.currentTimeMillis();
        final int opaque = request.getOpaque();
        boolean acquired = this.semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
        if (acquired) {
            final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreAsync);
            long costTime = System.currentTimeMillis() - beginStartTime;
            if (timeoutMillis < costTime) {
                once.release();
                throw new RemotingTimeoutException("invokeAsyncImpl call timeout");
            }

            final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis - costTime, invokeCallback, once);
            this.responseTable.put(opaque, responseFuture);
            try {
                channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture f) throws Exception {
                        if (f.isSuccess()) {
                            responseFuture.setSendRequestOK(true);
                            return;
                        }
                        requestFail(opaque);
                        log.warn("send a request command to channel <{}> failed.", RemotingHelper.parseChannelRemoteAddr(channel));
                    }
                });
            } catch (Exception e) {
                responseFuture.release();
                log.warn("send a request command to channel <" + RemotingHelper.parseChannelRemoteAddr(channel) + "> Exception", e);
                throw new RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e);
            }
        } else {
            if (timeoutMillis <= 0) {
                throw new RemotingTooMuchRequestException("invokeAsyncImpl invoke too fast");
            } else {
                String info =
                    String.format("invokeAsyncImpl tryAcquire semaphore timeout, %dms, waiting thread nums: %d semaphoreAsyncValue: %d",
                        timeoutMillis,
                        this.semaphoreAsync.getQueueLength(),
                        this.semaphoreAsync.availablePermits()
                    );
                log.warn(info);
                throw new RemotingTimeoutException(info);
            }
        }
    }
```

- responseTable—保存请求码与响应关联映射：opaque表示请求发起方在同个连接上不同的请求标识代码，每次发送一个消息的时候，可以选择同步阻塞/异步非阻塞的方式。无论是哪种通信方式，都会保存请求操作码至ResponseFuture的Map映射—responseTable中。
- ResponseFuture—保存返回响应（包括回调执行方法和信号量）：invokeCallback是在收到消息响应的时候能够根据responseTable找到请求码对应的回调执行方法，semaphore参数用作流控，当多个线程同时往一个连接写数据时可以通过信号量控制permit同时写许可的数量。
- 异常发送流程处理—定时扫描responseTable本地缓存：在发送消息时候，如果遇到异常情况（比如服务端没有response返回给客户端或者response因网络而丢失），上面所述的responseTable的本地缓存Map将会出现堆积情况。这个时候需要一个定时任务来专门做responseTable的清理回收。在RocketMQ的客户端/服务端启动时候会产生一个频率为1s调用一次来的定时任务检查所有的responseTable缓存中的responseFuture变量，判断是否已经得到返回, 并进行相应的处理。

## 3.2 Server接收并处理

```NettyServerHandler```中的方法 channelRead0 是Server接收消息的入口

```java
class NettyServerHandler extends SimpleChannelInboundHandler<RemotingCommand> {
        @Override
     	protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
            processMessageReceived(ctx, msg);
        }
    }
```

具体处理过程如下：

```java
public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
        final Pair<NettyRequestProcessor, ExecutorService> matched = this.processorTable.get(cmd.getCode());
        final Pair<NettyRequestProcessor, ExecutorService> pair = null == matched ? this.defaultRequestProcessor : matched;
        final int opaque = cmd.getOpaque();

        if (pair != null) {
            Runnable run = new Runnable() {
                @Override
                public void run() {
                    try {
                        RPCHook rpcHook = NettyRemotingAbstract.this.getRPCHook();
                        if (rpcHook != null) {
                            rpcHook.doBeforeRequest(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd);
                        }

                        final RemotingCommand response = pair.getObject1().processRequest(ctx, cmd);
                        if (rpcHook != null) {
                            rpcHook.doAfterResponse(RemotingHelper.parseChannelRemoteAddr(ctx.channel()), cmd, response);
                        }

                        if (!cmd.isOnewayRPC()) {
                            if (response != null) {
                                response.setOpaque(opaque);
                                response.markResponseType();
                                try {
                                    ctx.writeAndFlush(response);
                                } catch (Throwable e) {
                                    log.error("process request over, but response failed", e);
                                    log.error(cmd.toString());
                                    log.error(response.toString());
                                }
                            } else {

                            }
                        }
                    } catch (Throwable e) {
                        log.error("process request exception", e);
                        log.error(cmd.toString());

                        if (!cmd.isOnewayRPC()) {
                            final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_ERROR,
                                RemotingHelper.exceptionSimpleDesc(e));
                            response.setOpaque(opaque);
                            ctx.writeAndFlush(response);
                        }
                    }
                }
            };

            if (pair.getObject1().rejectRequest()) {
                final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                    "[REJECTREQUEST]system busy, start flow control for a while");
                response.setOpaque(opaque);
                ctx.writeAndFlush(response);
                return;
            }

            try {
                final RequestTask requestTask = new RequestTask(run, ctx.channel(), cmd);
                pair.getObject2().submit(requestTask);
            } catch (RejectedExecutionException e) {
                if ((System.currentTimeMillis() % 10000) == 0) {
                    log.warn(RemotingHelper.parseChannelRemoteAddr(ctx.channel())
                        + ", too many requests and system thread pool busy, RejectedExecutionException "
                        + pair.getObject2().toString()
                        + " request code: " + cmd.getCode());
                }

                if (!cmd.isOnewayRPC()) {
                    final RemotingCommand response = RemotingCommand.createResponseCommand(RemotingSysResponseCode.SYSTEM_BUSY,
                        "[OVERLOAD]system busy, start flow control for a while");
                    response.setOpaque(opaque);
                    ctx.writeAndFlush(response);
                }
            }
        } else {
            String error = " request type " + cmd.getCode() + " not supported";
            final RemotingCommand response =
                RemotingCommand.createResponseCommand(RemotingSysResponseCode.REQUEST_CODE_NOT_SUPPORTED, error);
            response.setOpaque(opaque);
            ctx.writeAndFlush(response);
            log.error(RemotingHelper.parseChannelRemoteAddr(ctx.channel()) + error);
        }
    }

```

- 根据```RemotingCommand```的code获取对应的业务处理器（```RequestCode```类中存储了业务码）和线程池；
- 创建一个新的线程，用于处理code对应的业务，并使用code对应的线程池提交执行。

## 3.3 Client端异步回调

在client发起请求处理时会初始化一个```InvokeCallback```，在Server端的异步线程由上面所讲的业务线程池真正执行后，返回response给Client端时候才会去触发执行。

例如：

异步发送消息时注入```InvokeCallback```：

```java
 private void sendMessageAsync(
        final String addr,
        final String brokerName,
        final Message msg,
        final long timeoutMillis,
        final RemotingCommand request,
        final SendCallback sendCallback,
        final TopicPublishInfo topicPublishInfo,
        final MQClientInstance instance,
        final int retryTimesWhenSendFailed,
        final AtomicInteger times,
        final SendMessageContext context,
        final DefaultMQProducerImpl producer
    ) throws InterruptedException, RemotingException {
        this.remotingClient.invokeAsync(addr, request, timeoutMillis, new InvokeCallback() {
            @Override
            public void operationComplete(ResponseFuture responseFuture) {
                RemotingCommand response = responseFuture.getResponseCommand();
                if (null == sendCallback && response != null) {

                    try {
                        SendResult sendResult = MQClientAPIImpl.this.processSendResponse(brokerName, msg, response);
                        if (context != null && sendResult != null) {
                            context.setSendResult(sendResult);
                            context.getProducer().executeSendMessageHookAfter(context);
                        }
                    } catch (Throwable e) {
                    }

                    producer.updateFaultItem(brokerName, System.currentTimeMillis() - responseFuture.getBeginTimestamp(), false);
                    return;
                }
 //省略其他代码..
```

返回response给Client端时执行：

```java
 public void processResponseCommand(ChannelHandlerContext ctx, RemotingCommand cmd) {
        final int opaque = cmd.getOpaque();
        final ResponseFuture responseFuture = responseTable.get(opaque);
        if (responseFuture != null) {
            responseFuture.setResponseCommand(cmd);

            responseTable.remove(opaque);

            if (responseFuture.getInvokeCallback() != null) {
                executeInvokeCallback(responseFuture);
            } else {
                responseFuture.putResponse(cmd);
                responseFuture.release();
            }
        } else {
            log.warn("receive response, but not matched any request, " + RemotingHelper.parseChannelRemoteAddr(ctx.channel()));
            log.warn(cmd.toString());
        }
    }
```













































