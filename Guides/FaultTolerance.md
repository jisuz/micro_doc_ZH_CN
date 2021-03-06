# 容错
在分布式系统中，世界各地的故障一直在发生。Micro试图用一些容错的最佳实践来解决这个问题。本文档介绍了一些可以配置的方法。

## 心跳
心跳是服务发现中刷新注册的机制。

### 基本原理
服务在启动时注册服务发现，并在关闭时取消注册。有时这些服务可能会意外死亡，被强行杀死或面临暂时的网络问题。在这些情况下，遗留的节点将存在服务发现中。如果服务被自动删除，这将是理想的。

### 解决办法
由于这个确切原因，Micro支持注册的TTL选项和间隔。TTL指定在发现之后注册的信息存在多长时间，然后过期并被删除。时间间隔是服务应该重新注册的时间，以保留其在服务发现中的注册信息。

这些选项可以用micro工具包中提供的标志。

### 用法
对于micro工具包，只需使用内置标志来设置ttl和间隔。

```
micro --register_ttl=30 --register_interval=15 api
```

以上这个例子，我们设置了一个30秒的ttl，重新注册间隔为15秒。

对于go-micro，当声明一个微服务时，你可以通过time.duration传递选项。

```
service := micro.NewService(
        micro.Name("com.example.srv.foo"),
        micro.RegisterTTL(time.Second*30),
        micro.RegisterInterval(time.Second*15),
)
```

## 负载均衡
负载均衡是传输请求负载或维持高可用性的一种方式

### 基本原理
任何单个流程应用程序的可用性和扩展都存在限制。如果应用程序因任何原因死亡，您将无法再处理请求。如果有足够的请求负载发送到应用程序，它可能会开始缓慢响应或根本不响应。通过发送请求至多个应用程序副本可以解决这些问题。

### 解决办法
Micro通过[选择器](https://godoc.org/github.com/micro/go-micro/selector#Selector)接口进行客户端负载平衡，以在任意数量的服务节点上传播请求。当服务启动时，它将服务发现注册为具有唯一地址和ID的服务节点。在发出请求时，Micro客户端使用选择器来决定向哪个节点发出请求。选择器使用服务注册表查找服务的节点，然后使用负载均衡策略（例如：随机哈希或循环法）选择要发送请求的节点。

### 用法
客户端负载平衡内置于go-micro客户端。这是自动完成的。

## 重试
重试是一种在不成功时重试请求的方法

### 基本原理
由于许多原因，请求可能会失败; 网络错误，请求加载，应用程序死亡。在这些情况下，如果我们可以针对不同的应用程序副本重试请求以获得成功的响应，那将是理想的。

### 解决办法
微客户端包括一个重试请求的机制。选择器（如上所述）返回一个Next函数，该函数在执行时使用负载平衡策略从列表中返回一个节点。Next函数可以执行多次，根据负载均衡策略返回一个新节点。如果设置了重试次数，如果请求失败，则会执行Next函数，并且将针对新节点重试请求。

### 用法
重试可以设置为客户端的标志或选项。它默认为1，意味着1次尝试请求。

通过标志进行更改。

```
micro --client_retries=3
```

通过选项进行设置。

```
client.Init(
	client.Retries(3),
)
```

## 缓存发现
发现缓存是服务发现信息的客户端缓存

### 基本原理
服务发现是微服务的核心依赖，但如果架构不正确，也可能成为单点故障。每个发现系统都有自己的扩展性和高可用性。在事件发生下降的情况下，由于服务不知道如何解析名称到地址，系统的其余部分将无法使用。如果为系统中的每个请求执行查找，发现也可能成为瓶颈。

### 解决办法
客户端缓存是一种消除服务发现瓶颈和单点故障的方法。Micro包含一个选择器（客户端负载平衡器），它在服务发现信息的内存缓存中维护与之相关的信息。如果发生缓存未命中，选择器将使用服务注册表进行查找并缓存数据。缓存也会周期性地被TTL掉，以确保陈旧的数据不会存在。

### 用法
高速缓存选择器可以通过标志或在创建新服务时进行设置。

作为工具包的标志执行以下操作

```
micro --selector=cache api
```

如果调用Init方法，Go-micro服务也可以使用相同的标志。或者，选择器可以用代码设置。

```
import (
	"github.com/micro/go-micro/client"
	"github.com/micro/go-micro/selector/cache"
)

service := micro.NewService(
	micro.Name("com.example.srv.foo"),
)

service.Client().Init(cache.NewSelector())
```
