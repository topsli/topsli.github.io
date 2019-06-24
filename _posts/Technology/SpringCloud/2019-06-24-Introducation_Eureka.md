---

layout: post
title: 介绍Eureka 
category: 技术
tags: eureka
keywords: springCloud eureka

---

## 一 概述（Overview） ##
In this tutorial, we’ll introduce client-side service discovery via “Spring Cloud Netflix Eureka“.
在这一教程中，我们通过使用“Spring Cloud Netfix Eureka”来介绍客户端的服务发现。
Client-side service discovery allows services to find and communicate with each other without hard-coding hostname and port. 
The only ‘fixed point’ in such an architecture consists of a service registry with which each service has to register.
客户端服务发现能够避免使用硬编码的主机名和端口来发现和调用彼此的服务。在这样的架构中，唯一的定点是由一个注册服务中心
组成，每个服务都向该中心注册自己的服务。
A drawback is that all clients must implement a certain logic to interact with this fixed point. 
This assumes an additional network round trip before the actual request.
一个缺点就是所有的客户端都要实现特定的逻辑与和注册中心这个固定点去交互。这样在实际请求前就增加了额外的网络请求。
With Netflix Eureka each client can simultaneously act as a server, to replicate its status to a connected peer. 
In other words, a client retrieves a list of all connected peers of a service registry and makes all further requests 
to any other services through a load-balancing algorithm.
使用Netfix Eureka的每个客户端同时也可以充当一个服务器，将基状态复制到连接的另一端。换句话说，每个客户端从服务注册中心获取
所有的服务注册列表，并通过负载均衡算法向其它的服务做进一步的请求。
To be informed about the presence of a client, they have to send a heartbeat signal to the registry.
所有的客户端通过向注册中心发送心跳，来告诉他们是存活的。
To achieve the goal of this write-up, we’ll implement three microservices:
为了实现本文的目标，我们要实现三个微服务：
a service registry (Eureka Server)
1. 一个注册中心（Eureka Server）
a REST service which registers itself at the registry (Eureka Client) and
2. 一个通过Eureka Client 来注册的REST服务
a web application, which is consuming the REST service as a registry-aware client (Spring Cloud Netflix Feign Client).
3. 一个Web应用程序，它作为一个注册感知客户端(Spring Cloud Netflix佯装客户端)使用REST服务

## 注册中心（Eureka Server） ##
Implementing a Eureka Server for service registry is as easy as:
实现一个Eureka Server 作为注册中心是很容易的事情：
adding spring-cloud-starter-netflix-eureka-server to the dependencies
1. 添加 spring-cloud-starter-netflix-eureka-server  依赖
enable the Eureka Server in a @SpringBootApplication by annotating it with @EnableEurekaServer
2. 添加 @EnableEurekaServer 注解到 @SpringBootApplication 注解的启动类上
configure some properties
3. 配置一些属性

