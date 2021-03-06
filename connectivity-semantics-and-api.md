gRPC连接语义和API
================================
> 原文：[gRPC Connectivity Semantics and API](https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md)

> 译者：[0x5010](https://github.com/0x5010)

本文档描述了gRPC通道的连接语义以及对RPC的相应影响。然后我们浅谈下API。

## 连接状态

gRPC抽象了客户端与服务器进行通信的方式。客户端通道对象可以使用多于一个DNS名称来创建。通道封装了一系列功能，包括名称解析，建立TCP连接(带有retry和backoff)以及TLS握手。通道还可以处理已建立的连接上的错误并重新连接，或者在HTTP/2 `GO_AWAY`的情况下，重新解析并重新连接。

为了对使用者隐藏gRPC API(即程序代码)的这些活动的细节，同时暴露有关信道状态的有意义的信息，用具有五种状态的状态机表示，定义如下：

CONNECTING: 该通道正在尝试建立连接，正在等待名称解析，TCP连接建立或TLS握手所涉及的其中一个步骤。这可以被用作创建时的通道的初始状态。

READY: 通道已经通过TLS握手(或相当的操作)后一直成功地建立连接，并且所有后续的通信尝试都成功(或者正在等待而没有任何已知的故障)。

TRANSIENT_FAILURE: 出现了一些暂时的故障(如TCP三次握手超时或socket错误)。此状态下的通道最终将切换到`CONNECTING`状态，并尝试再次建立连接。由于重试是以指数backoff的方式完成的，所以不能连接的信道将在这个状态下花费很少的时间，但是由于尝试重复失败，信道将花费越来越多的时间在这个状态。对于许多非致命故障(例如，由于服务器尚不可用而导致TCP连接尝试超时)，信道可能在此状态下花费越来越多的时间。

IDLE: 这是由于缺乏新的或待处理的RPC，通道甚至不尝试创建连接的状态。新的RPC可以在这个状态下创建。任何尝试在通道上启动RPC都会将通道的状态变更为CONNECTING。当一个指定`IDLE_TIMEOUT`的通道上没有RPC活动时，即在此期间没有新的或挂起的（活动）RPC时，`READY`或`CONNECTING`通道状态变更为`IDLE`。另外，当没有活动或待处理的RPC时，接收`GOAWAY`的通道也应变更到IDLE状态，以避免试图断开连接的服务器的连接超载。我们将使用300秒(5分钟)的默认`IDLE_TIMEOUT`。

SHUTDOWN: 这个通道已经开始关闭了。任何新的RPC应该立即失败。待处理的RPC可能会继续运行，直到程序取消它们。通道可能会进入此状态，因为程序明确要求关闭或在尝试连接通信期间发生了不可恢复的错误(截至2015年12月6日，没有已知的错误(连接或通信中)被归类为不可恢复)。 进入此状态的通道永远不会改变这个状态。

下表列出了从一个状态到另一个状态的转换规则以及相应的原因。`-`单元格表示不允许的转换。

|From/To|CONNECTING|READY|TRANSIENT_FAILURE|IDLE|SHUTDOWN|
|-|:-:|:-:|:-:|:-:|:-:|
|CONNECTING       |在连接建立期间增量|建立连接所需的所有步骤都成功了|在建立连接所需的任何步骤中出现任何故障|通道上没有RPC活动直到`IDLE_TIMEOUT`|程序触发shutdown|
|READY            |\-|在已建立的通道上增加成功的通话|预期在已建立的通道上成功通信时遇到任何故障|没有活动或待处理的RPC时接收`GOAWAY`或没有待处理的RPC直到`IDLE_TIMEOUT`|程序触发shutdown|
|TRANSIENT_FAILURE|指数backoff重试等待时间结束|\-|\-|\-|程序触发shutdown|
|IDLE             |频道上的任何新的RPC活动|\-|\-|\-|程序触发shutdown|
|SHUTDOWN         |\-|\-|\-|\-|\-|

## 通道状态API

所有的gRPC库都会公开一个通道级别的API方法来轮询当前的通道状态。在C++中，这种方法称为`GetState`，并返回五个合法状态之一的枚举。如果通道当前是IDLE的，它也接受布尔`try_to_connect`转换到CONNECTING，他的行为像一个RPC发生，所以它也应该重置`IDLE_TIMEOUT`。
```c++
grpc_connectivity_state GetState(bool try_to_connect);
```
所有的库都应该公开一个API，使得程序(gRPC API的使用者)在通道状态改变时得到通知。由于状态变化可以很快并且与任何这样的通知竞争，所以通知应该只是通知使用者已经发生了一些状态改变，留给使用者轮询当前状态。

这个API的同步版本是:
```c++
bool WaitForStateChange(grpc_connectivity_state source_state, gpr_timespec deadline);
```
当状态是`source_state`以外的状态时返回true，如果截止时间到期则返回false。基于异步和期货的API应该有一个相应的方法，允许在通道状态改变时通知程序。

请注意，每次从任何状态转换到其他任何状态时都会发送通知。另一方面，合法状态转换的规则，即使相应的指数回退在重试之前不需要等待，也需要从连接转换到`TRANSIENT_FAILURE`，并返回连接到每个可恢复故障。综合的影响是应用程序可能会收到虚假的状态更改通知。例如，在`CONNECTING`状态的通道上等待状态改变的应用程序可以接收状态改变通知，但是在轮询当前状态时找到处于还是`CONNECTING`状态的通道，因为该通道可能在`TRANSIENT_FAILURE`状态中花费了无限小的时间量。



