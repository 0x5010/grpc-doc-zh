gRPC健康检查协议
================================
> 原文：[gRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md)

> 译者：[0x5010](https://github.com/0x5010)

健康检查用于探测服务器是否能够处理rpc。client到server的健康状况检查可能会发生在点对点或通过某个控制系统。server可能会选择回复“不健康”，因为它没有准备好接受请求，正在关闭或其他原因。如果在一段时间内没有收到回复，或者回复内容不健康，client可以采取相应措施。

使用gRPC service作为对client到server简单场景和其他控制系统(负载均衡)的健康检查机制。作为一个高层次的service带来一些好处。首先，由于它本身是一个gRPC service，所以进行健康检查的形式与正常的rpc相同。其次，它具有丰富的语义，如每个service的健康状况。第三，作为gRPC服务，它可以重用所有现有的如计费和限制基础设施等，因此server可以完全控制健康检查service的访问。

## Service定义
server由下面的proto中service定义导出：
```protobuf
syntax = "proto3";

package grpc.health.v1;

message HealthCheckRequest {
  string service = 1;
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
  }
  ServingStatus status = 1;
}

service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
}

```
client可以通过调用`Check`方法来查询service的健康状态，并且应该在rpc上设置超时。client可以选择设置要查询健康状态的服务名称。建议的服务名称格式是`package_names.ServiceName`，比如`grpc.health.v1.Health`。

这个server应手动注册所有的service并设置各自的状态，包括空服务名称及其状态。对于收到的每个请求，如果可以在注册表中找到服务名称，则必须以`OK`状态发回响应，并且相应地将状态字段设置为`SERVING`或`NOT_SERVING`。如果服务名称未注册，则服务器返回`NOT_FOUND` GRPC状态。

server应该使用一个空字符串作为server的整体运行状况的关键字，以便对特定service不感兴趣的client可以用空请求查询服务器的状态。server可以在不支持任何通配符匹配的情况下完成对service名称的精确匹配。但是，service所有者可以自由地实现对client和server一致更复杂的匹配语义。

client如果rpc一段时间后没有完成可以宣布server不健康。client应该能够处理server没有`Health` service的情况。


