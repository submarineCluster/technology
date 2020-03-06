# gRPC + JSON

##### 发表于2018年8月15日星期三，[卡尔·马斯特兰奇洛](https://carlmastrangelo.com/)（ [Carl Mastrangelo）](https://carlmastrangelo.com/)



因此，您已经购买了整个RPC，并想尝试一下，但对协议缓冲区不太确定。您现有的代码对您自己的对象进行编码，或者也许您的代码需要特定的编码。该怎么办？

幸运的是，gRPC编码不可知！如果不使用Protobuf，您仍然可以获得gRPC的很多好处。在本文中，我们将介绍如何使gRPC与其他编码和类型一起使用。让我们尝试使用JSON。

gRPC实际上是具有高度凝聚力的技术的集合，而不是单一的整体框架。这意味着可以交换gRPC的某些部分，并且仍然可以利用gRPC的优势。[Gson](https://github.com/google/gson)是Java的流行库，用于执行JSON编码。让我们删除所有与protobuf相关的内容，并用Gson替换它们：

```diff
- Protobuf wire encoding
- Protobuf generated message types
- gRPC generated stub types
+ JSON wire encoding
+ Gson message types
```

以前，Protobuf和gRPC为我们生成代码，但是我们想使用我们自己的类型。另外，我们也将使用我们自己的编码。Gson允许我们在代码中引入自己的类型，但是提供了一种将这些类型序列化为字节的方法。

让我们继续[键值](https://github.com/carl-mastrangelo/kvstore/tree/04-gson-marshaller)存储服务。我们将修改我之前的[So You Want to Optimize gRPC](https://grpc.io/blog/optimizing-grpc-part-2)帖子中使用的代码。

## 反正什么是服务？

从gRPC的角度来看，*服务*是*方法*的集合。在Java中，方法表示为[`MethodDescriptor`](https://grpc.io/grpc-java/javadoc/io/grpc/MethodDescriptor.html)。每个都`MethodDescriptor`包含方法的名称，`Marshaller`用于编码请求的a和`Marshaller`用于编码响应的a。它们还包括其他详细信息，例如呼叫是否在流中。为简单起见，我们将坚持使用具有单个请求和单个响应的一元RPC。

由于我们不会生成任何代码，因此我们需要自己编写消息类。有四种方法，每种方法都有一个请求和一个响应类型。这意味着我们需要发出八条消息：

```java
  static final class CreateRequest {
    byte[] key;
    byte[] value;
  }

  static final class CreateResponse {
  }

  static final class RetrieveRequest {
    byte[] key;
  }

  static final class RetrieveResponse {
    byte[] value;
  }

  static final class UpdateRequest {
    byte[] key;
    byte[] value;
  }

  static final class UpdateResponse {
  }

  static final class DeleteRequest {
    byte[] key;
  }

  static final class DeleteResponse {
  }
```

因为GSON使用反射来确定类中的字段如何映射到序列化的JSON，所以我们不需要注释消息。

我们的客户端和服务器逻辑将使用请求和响应类型，但是gRPC需要知道如何产生和使用这些消息。为此，我们需要实现一个[`Marshaller`](https://grpc.io/grpc-java/javadoc/io/grpc/MethodDescriptor.Marshaller.html)。编组知道如何将任意类型转换为`InputStream`，然后将其传递到gRPC核心库中。当解码来自网络的数据时，它也能够进行反向转换。对于GSON，以下是编组器的外观：

```java
  static <T> Marshaller<T> marshallerFor(Class<T> clz) {
    Gson gson = new Gson();
    return new Marshaller<T>() {
      @Override
      public InputStream stream(T value) {
        return new ByteArrayInputStream(gson.toJson(value, clz).getBytes(StandardCharsets.UTF_8));
      }

      @Override
      public T parse(InputStream stream) {
        return gson.fromJson(new InputStreamReader(stream, StandardCharsets.UTF_8), clz);
      }
    };
  }
```

给定`Class`某个请求或响应的对象，此函数将产生编组器。使用编组器，我们可以`MethodDescriptor`为四个CRUD方法的每一个编写一个完整的内容。这是*Create*的方法描述符的示例：

```java
  static final MethodDescriptor<CreateRequest, CreateResponse> CREATE_METHOD =
      MethodDescriptor.newBuilder(
          marshallerFor(CreateRequest.class),
          marshallerFor(CreateResponse.class))
          .setFullMethodName(
              MethodDescriptor.generateFullMethodName(SERVICE_NAME, "Create"))
          .setType(MethodType.UNARY)
          .build();
```

请注意，如果使用Protobuf，将使用现有的Protobuf编组器，并且 [方法描述符](https://github.com/carl-mastrangelo/kvstore/blob/03-nonblocking-server/build/generated/source/proto/main/grpc/io/grpc/examples/proto/KeyValueServiceGrpc.java#L44) 将自动生成。

## 发送RPC

现在我们可以[`KvClient`](https://github.com/carl-mastrangelo/kvstore/blob/b225d28c7c2f3c356b0f3753384b3329f2ab5911/src/main/java/io/grpc/examples/KvClient.java#L98)封送JSON请求和响应，我们需要更新 上一篇文章中使用的gRPC客户端，以使用MethodDescriptors。此外，由于我们不会使用任何Protobuf类型，因此代码需要使用`ByteBuffer`而不是`ByteString`。也就是说，我们仍然可以使用`grpc-stub`Maven上的软件包来发行RPC。使用*创建*再次方法为例，这里是如何使一个RPC：

```java
    ByteBuffer key = createRandomKey();
    ClientCall<CreateRequest, CreateResponse> call =
        chan.newCall(KvGson.CREATE_METHOD, CallOptions.DEFAULT);
    KvGson.CreateRequest req = new KvGson.CreateRequest();
    req.key = key.array();
    req.value = randomBytes(MEAN_VALUE_SIZE).array();

    ListenableFuture<CreateResponse> res = ClientCalls.futureUnaryCall(call, req);
    // ...
```

如您所见，我们`ClientCall`从中创建了一个新对象`MethodDescriptor`，创建了请求，然后`ClientCalls.futureUnaryCall`在存根库中使用发送该对象。gRPC负责其余的工作。您也可以使阻塞存根或异步存根代替将来的存根。

## 接收RPC

要更新服务器，我们需要创建一个键值服务和实现。回想一下，在gRPC中，*服务器*可以处理一个或多个*服务*。同样，Protobuf通常会为我们产生什么，我们需要自己编写。基本服务如下所示：

```java
  static abstract class KeyValueServiceImplBase implements BindableService {
    public abstract void create(
        KvGson.CreateRequest request, StreamObserver<CreateResponse> responseObserver);

    public abstract void retrieve(/*...*/);

    public abstract void update(/*...*/);

    public abstract void delete(/*...*/);

    /* Called by the Server to wire up methods to the handlers */
    @Override
    public final ServerServiceDefinition bindService() {
      ServerServiceDefinition.Builder ssd = ServerServiceDefinition.builder(SERVICE_NAME);
      ssd.addMethod(CREATE_METHOD, ServerCalls.asyncUnaryCall(
          (request, responseObserver) -> create(request, responseObserver)));

      ssd.addMethod(RETRIEVE_METHOD, /*...*/);
      ssd.addMethod(UPDATE_METHOD, /*...*/);
      ssd.addMethod(DELETE_METHOD, /*...*/);
      return ssd.build();
    }
  }
```

`KeyValueServiceImplBase`将同时用作服务定义（描述服务器可以处理的方法）和实现（描述每个方法的操作）。它充当gRPC与我们的应用程序逻辑之间的粘合剂。从服务器代码中的Proto交换到GSON几乎不需要更改：

```java
final class KvService extends KvGson.KeyValueServiceImplBase {

  @Override
  public void create(
      KvGson.CreateRequest request, StreamObserver<KvGson.CreateResponse> responseObserver) {
    ByteBuffer key = ByteBuffer.wrap(request.key);
    ByteBuffer value = ByteBuffer.wrap(request.value);
    // ...
  }
```

在服务器上实现所有方法之后，我们现在有了一个功能齐全的gRPC Java，JSON编码RPC系统。并向您展示我没有袖手旁观：

```sh
$ ./gradlew :dependencies | grep -i proto
$ # no proto deps!
```

## 优化代码

尽管Gson的速度不如Protobuf，但没有选择不成熟的水果是没有道理的。运行代码，我们看到性能很慢：

```sh
./gradlew installDist
time ./build/install/kvstore/bin/kvstore

INFO: Did 215.883 RPCs/s
```

发生了什么？在之前的[优化](https://grpc.io/blog/optimizing-grpc-part-2)文章中，我们看到Protobuf版本执行了近*2500 RPC / s的速度*。JSON是缓慢的，但不能*说*慢。我们可以通过打印出经过编组器的JSON数据来了解问题所在：

```json
{"key":[4,-100,-48,22,-128,85,115,5,56,34,-48,-1,-119,60,17,-13,-118]}
```

那是不对的！看一下`RetrieveRequest`，我们看到关键字节被编码为数组，而不是字节字符串。电线尺寸比需要的尺寸大得多，并且可能与其他JSON代码不兼容。为了解决这个问题，让我们告诉GSON将这些数据编码和解码为base64编码的字节：

```java
  private static final Gson gson =
      new GsonBuilder().registerTypeAdapter(byte[].class, new TypeAdapter<byte[]>() {
    @Override
    public void write(JsonWriter out, byte[] value) throws IOException {
      out.value(Base64.getEncoder().encodeToString(value));
    }

    @Override
    public byte[] read(JsonReader in) throws IOException {
      return Base64.getDecoder().decode(in.nextString());
    }
  }).create();
```

在我们的编组中使用它，我们可以看到巨大的性能差异：

```sh
./gradlew installDist
time ./build/install/kvstore/bin/kvstore

INFO: Did 2,202.2 RPCs/s
```

几乎**10倍的**速度比以前快！我们仍然可以利用gRPC的效率，同时带来我们自己的编码器和消息。

## 结论

gRPC允许您使用Protobuf以外的编码器。它不依赖Protobuf，并且是专门为与各种环境一起使用而设计的。我们可以看到，只需增加一点样板，就可以使用所需的任何编码器。虽然本文仅介绍JSON，但gRPC与Thrift，Avro，Flatbuffers，Cap'n Proto甚至原始字节兼容！gRPC使您可以控制数据的处理方式。（尽管由于强大的向后兼容性，类型检查和性能，我们仍然推荐Protobuf。）

如果您希望看到一个完整的实现，所有代码都可以在[GitHub](https://github.com/carl-mastrangelo/kvstore/tree/04-gson-marshaller)上[找到](https://github.com/carl-mastrangelo/kvstore/tree/04-gson-marshaller)。