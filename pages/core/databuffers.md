<a id="databuffers"></a>

[](#databuffers)8\. 数据缓冲区和编解码器
------------------------------

Java NIO虽然提供了`ByteBuffer`，但许多库在顶层构建自己的字节缓冲区API，尤其是对于重用缓冲区和/或使用直接缓冲区有利于性能的网络操作。 例如， Netty具有`ByteBuf`层次结构，Undertow使用XNIO，Jetty使用带有回调的池化字节缓冲区，等等。 `spring-core` 模块提供了一组抽象来处理各种字节缓冲API，如下所示：

*   [`DataBufferFactory`](#databuffers-factory)创建抽象数据缓冲区。.

*   [`DataBuffer`](#databuffers-buffer) DataBuffer表示可以[pooled](#databuffers-buffer-pooled)的字节缓冲区。 .

*   [`DataBufferUtils`](#databuffers-utils)为数据缓冲区提供实用程序方法。

*   [Codecs](#codecs) 将数据缓冲流解码或编码为更高级别的对象。


<a id="databuffers-factory"></a>

### [](#databuffers-factory)8.1. `DataBufferFactory`

`DataBufferFactory` 以两种方式之一创建数据缓冲区:

1.  分配新的数据缓冲区，可选择预先指定容量（如果已知），即使`DataBuffer` 的实现可以按需增长和缩小，这也更有效。

2.  包装现有的 `byte[]` 或`java.nio.ByteBuffer`，并使用`DataBuffer`实现来修饰给定的数据，且不涉及分配。


请注意，WebFlux应用程序不直接创建`DataBufferFactory`，而是通过`ServerHttpResponse`或客户端的`ClientHttpRequest`访问它。 工厂类型取决于底层客户端或服务器，例如 Reactor Netty的`NettyDataBufferFactory` ，其他的`DefaultDataBufferFactory`。

<a id="databuffers-buffer"></a>

### [](#databuffers-buffer)8.2. `DataBuffer`

`DataBuffer` 接口提供与`java.nio.ByteBuffer`类似的操作，但也带来了一些额外的好处，其中一些受Netty `ByteBuf`的启发。 以下是部分好处清单:

*   可以独立的读写，即不需要调用`flip()` 来在读写之间交替。

*   与`java.lang.StringBuilder`一样，按需扩展容量。.

*   通过 [`PooledDataBuffer`](#databuffers-buffer-pooled)Pooled 缓冲区和引用计数。

*   以`java.nio.ByteBuffer`， `InputStream`或`OutputStream`的形式查看缓冲区。

*   确定给定字节的索引或最后一个索引。


<a id="databuffers-buffer-pooled"></a>

### [](#databuffers-buffer-pooled)8.3. `PooledDataBuffer`

正如Javadoc for [ByteBuffer](https://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html)中所解释的，字节缓冲区可以是直接缓冲区，也可以是非直接缓冲区。 直接缓冲区可以驻留在Java堆之外，这样就无需复制本机I/O操作。 这使得直接缓冲区对于通过套接字接收和发送数据特别有用，但是创建和释放它们也更加昂贵，这导致了池化缓冲区的想法。

`PooledDataBuffer`是`DataBuffer`的扩展，它有助于引用计数，这对于字节缓冲池是必不可少的。它是如何工作的？ 当分配`PooledDataBuffer` 时，引用计数为1\. 调用 `retain()`递增计数，而对`release()`的调用则递减计数。只要计数大于0，就保证缓冲区不被释放。 当计数减少到0时，可以释放池化缓冲区，这实际上可能意味着缓冲区的保留内存返回到内存池。

请注意，不是直接对`PooledDataBuffer`进行操作，在大多数情况下，最好使用`DataBufferUtils`中的方法， 只有当它是`PooledDataBuffer`的实例时才应用release或retain到`DataBuffer`。

<a id="databuffers-utils"></a>

### [](#databuffers-utils)8.4. `DataBufferUtils`

`DataBufferUtils` 提供了许多用于操作数据缓冲区的实用方法:

*   将数据缓冲区流加入单个缓冲区中，可能只有零拷贝，例如 通过复合缓冲区，如果底层字节缓冲区API支持。 Join a stream of data buffers into a single buffer possibly with zero copy, e.g. via composite buffers, if that’s supported by the underlying byte buffer API.

*   将 `InputStream` or NIO `Channel` 转化为 `Flux<DataBuffer>`, 反之亦然， 将`Publisher<DataBuffer>` 转化为 `OutputStream` 或 NIO `Channel`.

*   如果缓冲区是`PooledDataBuffer`的实例，则释放或保留`DataBuffer` 的方法。

*   从字节流中跳过或取出，直到特定的字节数。


<a id="codecs"></a>

### [](#codecs)8.5. Codecs

`org.springframework.core.codec` 包提供以下策略接口:

*   `Encoder` 将`Publisher<T>` 编码为数据缓冲区流。

*   `Decoder` 将 `Publisher<DataBuffer>`为更高级别的对象流。


`spring-core`模块提供`byte[]`, `ByteBuffer`, `DataBuffer`, `Resource`, 和 `String`编码器和解码器实现。 The `spring-web`模块增加了 Jackson JSON, Jackson Smile, JAXB2, Protocol Buffers 和其他编码器和解码器。请参阅WebFlux部分中的[Codecs](web-reactive.html#webflux-codecs) 。

<a id="databuffers-using"></a>

### [](#databuffers-using)8.6. 使用 `DataBuffer`

使用数据缓冲区时，必须特别注意确保缓冲区被释放，因为它们可能被[pooled](#databuffers-buffer-pooled)。我们将使用编解码器来说明它是如何工作的，但概念更普遍适用。 让我们看看内部编解码器必须在内部管理数据缓冲区。

A `Decoder` 是在创建更高级别对象之前读取输入数据缓冲区的最后一个，因此必须按如下方式释放它们：:

1.  如果`Decoder`只是读取每个输入缓冲区并准备立即释放它，它可以通过 `DataBufferUtils.release(dataBuffer)`来实现。

2.  如果`Decoder`使用`Flux` 或`Mono` 运算符（如`flatMap`，`reduce`等）在内部预取和缓存数据项，或者正在使用诸如`filter`, `skip`和其他省略项的运算符， 则 `doOnDiscard(PooledDataBuffer.class, DataBufferUtils::release)` 必须将:: release）添加到组合链中，以确保在丢弃之前释放这些缓冲区，可能还会导致错误或取消信号。

3.  如果 `Decoder` 以任何其他方式保持一个或多个数据缓冲区，则必须确保在完全读取时释放它们，或者在读取和释放高速缓存数据缓冲区之前发生错误或取消信号。


请注意， `DataBufferUtils#join`提供了一种安全有效的方法，可将数据缓冲区流聚合到单个数据缓冲区中。 同样，`skipUntilByteCount`和 `takeUntilByteCount`是解码器使用的其他安全方法。

`Encoder`分配其他人必须读取（和释放）的数据缓冲区。 所以`Encoder`没什么可做的。 但是，如果在使用数据填充缓冲区时发生序列化错误，则`Encoder`必须注意释放数据缓冲区。 例如：

    DataBuffer buffer = factory.allocateBuffer();
    boolean release = true;
    try {
        // serialize and populate buffer..
        release = false;
    }
    finally {
        if (release) {
            DataBufferUtils.release(buffer);
        }
    }
    return buffer;

`Encoder` 的使用者负责释放它接收的数据缓冲区。 在WebFlux应用程序中，Encoder的输出用于写入HTTP服务器响应或客户端HTTP请求， 在这种情况下，释放数据缓冲区是代码写入服务器响应或客户端的责任。 请求。

请注意，在Netty上运行时，可以使用调试选项来 [排除缓冲区泄漏](https://github.com/netty/netty/wiki/Reference-counted-objects#troubleshooting-buffer-leaks)。