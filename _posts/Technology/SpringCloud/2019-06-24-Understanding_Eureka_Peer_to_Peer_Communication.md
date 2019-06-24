---

layout: post
title: Eureka 的点对点通信 
category: 技术
tags: eureka
keywords: springCloud eureka

---

## Understanding Eureka Peer to Peer Communication ##

Eureka clients tries to talk to Eureka Server in the same zone. If there are problems talking with the server or if the
 server does not exist in the same zone, the clients fail over to the servers in the other zones.

Once the server starts receiving traffic, all of the operations that is performed on the server is replicated to all of 
the peer nodes that the server knows about. If an operation fails for some reason, the information is reconciled on the
 next heartbeat that also gets replicated between servers.

When the Eureka server comes up, it tries to get all of the instance registry information from a neighboring node. 
If there is a problem getting the information from a node, the server tries all of the peers before it gives up.
If the server is able to successfully get all of the instances, it sets the renewal threshold that it should be 
receiving based on that information. If any time, the renewals falls below the percent configured for that 
value (below 85% within 15 mins), the server stops expiring instances to protect the current instance 
registry information.

In Netflix, the above safeguard is called as self-preservation mode and is primarily used as a protection in scenarios
 where there is a network partition between a group of clients and the Eureka Server. In these scenarios, the server
  tries to protect the information it already has. There may be scenarios in case of a mass outage that this may cause
   the clients to get the instances that do not exist anymore. The clients must make sure they are resilient to eureka
    server returning an instance that is non-existent or un-responsive. The best protection in these scenarios is to 
    timeout quickly and try other servers.

In the case, where the server is not able get the registry information from the neighboring node, it waits for 
a few minutes (5 mins) so that the clients can register their information. The server tries hard not to provide 
partial information to the clients there by skewing traffic only to a group of instances and causing capacity issues.

Eureka servers communicate with one another using the same mechanism used between the Eureka client and the server
 as described here.

Also worth noting is that there are several configurations that can be tuned on the server including the 
communication between the servers if needed.

What happens during network outages between Peers?
In the case of network outages between peers, following things may happen

1. The heartbeat replications between peers may fail and the server detects this situation and enters into a
 self-preservation mode protecting the current state.
2. Registrations may happen in an orphaned server and some clients may reflect new registrations while 
the others may not.
3. The situation autocorrects itself after the network connectivity is restored to a stable state.
 When the peers are able to communicate fine, the registration information is automatically transferred
  to the servers that do not have them.

The bottom line is, during the network outages, the server tries to be as resilient as possible, but there 
is a possibility of clients having different views of the servers during that time.

## 理解Eureka的点对点之间的通信 ##
Eureka客户端尝试与同一区域中的Eureka服务器通信。如果与服务器通信出现问题，或者服务器不存在于同一区域，
则客户端将故障转移到其他区域的服务器。一旦服务器开始接收流量，在服务器上执行的所有操作都将复制到服务器知
道的所有对等节点。如果某个操作由于某种原因失败，信息将在服务器之间复制的下一个心跳上进行协调。当Eureka服务器出现时，
它试图从相邻节点获取所有实例注册表信息。如果从节点获取信息有问题，服务器在放弃之前会尝试所有的对等点。
如果服务器能够成功获取所有实例，它将根据该信息设置应该接收的更新阈值。如果任何时候，更新低于为该值配置的
百分比(15分钟内低于85%)，服务器将停止过期实例以保护当前实例注册表信息。

在Netflix中，上述保护被称为自我保护模式，主要用于一组客户端和Eureka服务器之间存在网络分区的场景。在这些场景中，
服务器试图保护它已经拥有的信息。在大规模停机的情况下，可能会出现这样的情况，这可能导致客户端获得不再存在的实例。
客户端必须确保它们对返回不存在或没有响应的实例的eureka服务器具有弹性。在这些场景中，最好的保护是快速超时并尝试其
他服务器。在这种情况下，如果服务器无法从邻近节点获取注册表信息，它将等待几分钟(5分钟)，以便客户端可以注册其信息。
服务器尽量不向那里的客户端提供部分信息，只向一组实例倾斜流量并导致容量问题。Eureka服务器之间使用与这里描述的
Eureka客户端和服务器之间使用的相同机制进行通信。

同样值得注意的是，有几种配置可以在服务器上进行调优，包括服务器之间的通信(如果需要的话)。

### 在对等点之间的网络中断期间会发生什么? ###
在对等点之间发生网络中断的情况下，可能会发生以下情况：
1. 对等点之间的心跳复制可能会失败，服务器检测到这种情况并进入自我保护模式，保护当前状态。
2. 注册可能发生在孤立的服务器中，一些客户端可能反映新的注册，而其他客户端可能不反映。
3. 当网络连接恢复到稳定状态后，情况会自动纠正。当对等点能够很好地通信时，注册信息将自动传输到没有注册信息的服务器。

底线是，在网络中断期间，服务器尽可能地保持弹性，但是在此期间，客户端可能对服务器有不同的看法。