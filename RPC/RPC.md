### RPC

---

#### RPC概念

RPC 的全称是 Remote Procedure Call 是一种进程间通信方式。它允许程序调用另一个地址空间（通常是共享网络的另一台机器上）的过程或函数，而不用程序员显式编码这个远程调用的细节。即程序员无论是调用本地的还是远程的，本质上编写的调用代码基本相同。

#### RPC结构

Nelson 的论文中指出实现 RPC 的程序包括 5 个部分：

> 1. User
> 2. User-stub
> 3. RPCRuntime
> 4. Server-stub
> 5. Server

这几个部分的关系如下图所示：

![截屏2020-12-03 下午2.51.28](https://tva1.sinaimg.cn/large/0081Kckwgy1glao6634ddj313a0fmtcg.jpg)

这里 user 就是 client 端，当 user 想发起一个远程调用时，它实际是通过本地调用 user-stub。user-stub 负责将调用的接口、方法和参数通过约定的协议规范进行编码并通过本地的 RPCRuntime 实例传输到远端的实例。远端 RPCRuntime 实例收到请求后交给 server-stub 进行解码后发起本地端调用，调用结果再返回给 user 端。

#### 选择RFC框架要考虑的因素

1. 性能指标
2.  是否需要跨语言平台
3.  内网开放还是公网开放
4.  开源 RPC 框架本身的质量、社区活跃度

#### RPC结构拆解

![截屏2020-12-03 下午3.16.07](https://tva1.sinaimg.cn/large/0081Kckwgy1glaovsnuntj313w0jcdkc.jpg)

RPC 服务方通过 `RpcServer` 去导出（export）远程接口方法，而客户方通过 `RpcClient` 去引入（import）远程接口方法。客户方像调用本地方法一样去调用远程接口方法，RPC 框架提供接口的代理实现，实际的调用将委托给代理`RpcProxy` 。代理封装调用信息并将调用转交给`RpcInvoker` 去实际执行。在客户端的`RpcInvoker` 通过连接器`RpcConnector` 去维持与服务端的通道`RpcChannel`，并使用`RpcProtocol` 执行协议编码（encode）并将编码后的请求消息通过通道发送给服务方。

RPC 服务端接收器 `RpcAcceptor` 接收客户端的调用请求，同样使用`RpcProtocol` 执行协议解码（decode）。解码后的调用信息传递给`RpcProcessor` 去控制处理调用过程，最后再委托调用给`RpcInvoker` 去实际执行并返回调用结果。

#### RPC组件职责

上面我们进一步拆解了 RPC 实现结构的各个组件组成部分，下面我们详细说明下每个组件的职责划分。

```markdown
1. RpcServer
   负责导出（export）远程接口
2. RpcClient
   负责导入（import）远程接口的代理实现
3. RpcProxy
   远程接口的代理实现
4. RpcInvoker
   客户方实现：负责编码调用信息和发送调用请求到服务方并等待调用结果返回
   服务方实现：负责调用服务端接口的具体实现并返回调用结果
5. RpcProtocol
   负责协议编/解码
6. RpcConnector
   负责维持客户方和服务方的连接通道和发送数据到服务方
7. RpcAcceptor
   负责接收客户方请求并返回请求结果
8. RpcProcessor
   负责在服务方控制调用过程，包括管理调用线程池、超时时间等
9. RpcChannel
   数据传输通道
```

#### RPC和HTTP的区别

1. HTTP 与 RPC 协议在实现上是不同的，大家都了解到 HTTP 原理就是 客户端请求服务端，服务端去响应并返回结果，但是 RPC 协议设计的时候采用的方式就是服务端给客户端提供 TCP 长连接服务，Client 端去调用 Server 提供的接口，实现特定的功能；
2. RPC 可以同时提供同步调用及异步调用，而 HTTP 提供的方式就是同步调用，客户端会等待并接受服务端的请求处理的结果；
3. RPC 服务设计可以提高代码编写过程中的解耦操作，提高代码的可移植性，每一个 服务可以设计成提供特定功能的小服务，客户端去调取远程的服务，而不用去关心远程是怎么实现的。

#### RPC 应用领域

- 大型网站的内部子系统设计；
- 为系统提供降级功能；
- 并发设计场景；

#### RPC实现分析

在进一步拆解了组件并划分了职责之后，这里以在 java 平台实现该 RPC 框架概念模型为例，详细分析下实现中需要考虑的因素。

**导出远程接口**

导出远程接口的意思是指只有导出的接口可以供远程调用，而未导出的接口则不能。在 java 中导出接口的代码片段可能如下：

```java
DemoService demo   = new ...;



RpcServer   server = new ...;



server.export(DemoService.class, demo, options);
```

我们可以导出整个接口，也可以更细粒度一点只导出接口中的某些方法，如：

```java
// 只导出 DemoService 中签名为 hi(String s) 的方法



server.export(DemoService.class, demo, "hi", new Class<?>[] { String.class }, options);
```

java 中还有一种比较特殊的调用就是多态，也就是一个接口可能有多个实现，那么远程调用时到底调用哪个？这个本地调用的语义是通过 jvm 提供的引用多态性隐式实现的，那么对于 RPC 来说跨进程的调用就没法隐式实现了。如果前面`DemoService` 接口有 2 个实现，那么在导出接口时就需要特殊标记不同的实现，如：

```java
DemoService demo   = new ...;



DemoService demo2  = new ...;



RpcServer   server = new ...;



server.export(DemoService.class, demo, options);



server.export("demo2", DemoService.class, demo2, options);
```

上面 demo2 是另一个实现，我们标记为 "demo2" 来导出，那么远程调用时也需要传递该标记才能调用到正确的实现类，这样就解决了多态调用的语义。

**导入远程接口与客户端代理**

导入相对于导出远程接口，客户端代码为了能够发起调用必须要获得远程接口的方法或过程定义。目前，大部分跨语言平台 RPC 框架采用根据 IDL 定义通过 code generator 去生成 stub 代码，这种方式下实际导入的过程就是通过代码生成器在编译期完成的。我所使用过的一些跨语言平台 RPC 框架如 CORBAR、WebService、ICE、Thrift 均是此类方式。

代码生成的方式对跨语言平台 RPC 框架而言是必然的选择，而对于同一语言平台的 RPC 则可以通过共享接口定义来实现。在 java 中导入接口的代码片段可能如下：

```java
RpcClient client = new ...;



DemoService demo = client.refer(DemoService.class);



demo.hi("how are you?");
```

在 java 中 'import' 是关键字，所以代码片段中我们用 refer 来表达导入接口的意思。这里的导入方式本质也是一种代码生成技术，只不过是在运行时生成，比静态编译期的代码生成看起来更简洁些。java 里至少提供了两种技术来提供动态代码生成，一种是 jdk 动态代理，另外一种是字节码生成。动态代理相比字节码生成使用起来更方便，但动态代理方式在性能上是要逊色于直接的字节码生成的，而字节码生成在代码可读性上要差很多。两者权衡起来，个人认为牺牲一些性能来获得代码可读性和可维护性显得更重要。

**协议编解码**

客户端代理在发起调用前需要对调用信息进行编码，这就要考虑需要编码些什么信息并以什么格式传输到服务端才能让服务端完成调用。出于效率考虑，编码的信息越少越好（传输数据少），编码的规则越简单越好（执行效率高）。我们先看下需要编码些什么信息：

```markdown
-- 调用编码 --
1. 接口方法
   包括接口名、方法名
2. 方法参数
   包括参数类型、参数值
3. 调用属性
   包括调用属性信息，例如调用附件隐式参数、调用超时时间等

-- 返回编码 --
1. 返回结果
   接口方法中定义的返回值
2. 返回码
   异常返回码
3. 返回异常信息
   调用异常信息
```

除了以上这些必须的调用信息，我们可能还需要一些元信息以方便程序编解码以及未来可能的扩展。这样我们的编码消息里面就分成了两部分，一部分是元信息、另一部分是调用的必要信息。如果设计一种 RPC 协议消息的话，元信息我们把它放在协议消息头中，而必要信息放在协议消息体中。下面给出一种概念上的 RPC 协议消息设计格式：

![img](https://img-blog.csdn.net/20150108170315663?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbWluZGZsb2F0aW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

```groovy
-- 消息头 --
magic      : 协议魔数，为解码设计
header size: 协议头长度，为扩展设计
version    : 协议版本，为兼容设计
st         : 消息体序列化类型
hb         : 心跳消息标记，为长连接传输层心跳设计
ow         : 单向消息标记，
rp         : 响应消息标记，不置位默认是请求消息
status code: 响应消息状态码
reserved   : 为字节对齐保留
message id : 消息 id
body size  : 消息体长度

-- 消息体 --
采用序列化编码，常见有以下格式
xml   : 如 webservie soap
json  : 如 JSON-RPC
binary: 如 thrift; hession; kryo 等
```

格式确定后编解码就简单了，由于头长度一定所以我们比较关心的就是消息体的序列化方式。序列化我们关心三个方面：

1. 序列化和反序列化的效率，越快越好。
2.  序列化后的字节长度，越小越好。
3.  序列化和反序列化的兼容性，接口参数对象若增加了字段，是否兼容。

上面这三点有时是鱼与熊掌不可兼得，这里面涉及到具体的序列化库实现细节，就不在本文进一步展开分析了。

**传输服务**

协议编码之后，自然就是需要将编码后的 RPC 请求消息传输到服务方，服务方执行后返回结果消息或确认消息给客户方。RPC 的应用场景实质是一种可靠的请求应答消息流，和 HTTP 类似。因此选择长连接方式的 TCP 协议会更高效，与 HTTP 不同的是在协议层面我们定义了每个消息的唯一 id，因此可以更容易的复用连接。

既然使用长连接，那么第一个问题是到底 client 和 server 之间需要多少根连接？实际上单连接和多连接在使用上没有区别，对于数据传输量较小的应用类型，单连接基本足够。单连接和多连接最大的区别在于，每根连接都有自己私有的发送和接收缓冲区，因此大数据量传输时分散在不同的连接缓冲区会得到更好的吞吐效率。所以，如果你的数据传输量不足以让单连接的缓冲区一直处于饱和状态的话，那么使用多连接并不会产生任何明显的提升，反而会增加连接管理的开销。

连接是由 client 端发起建立并维持。如果 client 和 server 之间是直连的，那么连接一般不会中断（当然物理链路故障除外）。如果 client 和 server 连接经过一些负载中转设备，有可能连接一段时间不活跃时会被这些中间设备中断。为了保持连接有必要定时为每个连接发送心跳数据以维持连接不中断。心跳消息是 RPC 框架库使用的内部消息，在前文协议头结构中也有一个专门的心跳位，就是用来标记心跳消息的，它对业务应用透明。

**执行调用**

client stub 所做的事情仅仅是编码消息并传输给服务方，而真正调用过程发生在服务方。server stub 从前文的结构拆解中我们细分了 `RpcProcessor` 和 `RpcInvoker` 两个组件，一个负责控制调用过程，一个负责真正调用。这里我们还是以 java 中实现这两个组件为例来分析下它们到底需要做什么？

java 中实现代码的动态接口调用目前一般通过反射调用。除了原生的 jdk 自带的反射，一些第三方库也提供了性能更优的反射调用，因此 `RpcInvoker` 就是封装了反射调用的实现细节。

调用过程的控制需要考虑哪些因素，`RpcProcessor` 需要提供什么样地调用控制服务呢？下面提出几点以启发思考：

```markdown
1. 效率提升
   每个请求应该尽快被执行，因此我们不能每请求来再创建线程去执行，需要提供线程池服务。
2. 资源隔离
   当我们导出多个远程接口时，如何避免单一接口调用占据所有线程资源，而引发其他接口执行阻塞。
3. 超时控制
   当某个接口执行缓慢，而 client 端已经超时放弃等待后，server 端的线程继续执行此时显得毫无意义。
```



**RPC 异常处理**

无论 RPC 怎样努力把远程调用伪装的像本地调用，但它们依然有很大的不同点，而且有一些异常情况是在本地调用时绝对不会碰到的。在说异常处理之前，我们先比较下本地调用和 RPC 调用的一些差异：

\1. 本地调用一定会执行，而远程调用则不一定，调用消息可能因为网络原因并未发送到服务方。
\2. 本地调用只会抛出接口声明的异常，而远程调用还会跑出 RPC 框架运行时的其他异常。
\3. 本地调用和远程调用的性能可能差距很大，这取决于 RPC 固有消耗所占的比重。

正是这些区别决定了使用 RPC 时需要更多考量。当调用远程接口抛出异常时，异常可能是一个业务异常，也可能是 RPC 框架抛出的运行时异常（如：网络中断等）。业务异常表明服务方已经执行了调用，可能因为某些原因导致未能正常执行，而 RPC 运行时异常则有可能服务方根本没有执行，对调用方而言的异常处理策略自然需要区分。

由于 RPC 固有的消耗相对本地调用高出几个数量级，本地调用的固有消耗是纳秒级，而 RPC 的固有消耗是在毫秒级。那么对于过于轻量的计算任务就并不合适导出远程接口由独立的进程提供服务，只有花在计算任务上时间远远高于 RPC 的固有消耗才值得导出为远程接口提供服务。

#### GRPC

gRPC 是 Google 基于 HTTP/2 以及 protobuf 的，要了解 gRPC 协议，只需要知道 gRPC 是如何在 HTTP/2 上面传输就可以了。

gRPC 通常有四种模式，**unary，client streaming，server streaming** 以及 bidirectional streaming，对于底层 HTTP/2 来说，它们都是 stream，并且仍然是一套 request + response 模型。

gRPC 的基石就是 HTTP/2，然后在上面使用 protobuf 协议定义好 service RPC。虽然看起来很简单，但如果一门语言没有 HTTP/2，protobuf 等支持，要支持 gRPC 就是一件非常困难的事情了。



---

参考资料：

https://www.jianshu.com/p/48ad37e8b4ed

https://blog.csdn.net/mindfloating/article/details/39473807

https://blog.csdn.net/iteye_2022/article/details/82605112?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.control