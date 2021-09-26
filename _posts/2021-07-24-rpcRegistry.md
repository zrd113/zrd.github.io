---
layout: post
title: RPC框架添加服务注册功能
date: 2021-07-24
Author: zrd
tags: [RPC, Netty, ZooKeeper]
toc: true
---

## RPC注册中心

在上一篇文章中，我们实现了一个RPC框架的基本原理，但是还有很多功能还没有完善，比如说注册中心功能，注册中心负责服务地址的注册与查找，相对于字典的目录，通过索引我们可以快速查找指定服务的地址，然后客户端就可以根据该地址连接服务端进行消费。

首先，我们分别定义了两个接口 ServiceDiscovery.java 和 ServiceRegistry.java，这两个接口分别定义了服务注册和服务发现行为。

```
/**
 * @Description 服务注册接口
 * @Author ZRD
 * @Date 2021/7/13
 */
public interface ServiceRegistry {
    /**
     * @Description: 提供rpc服务注册
     * @Param rpcServiceName rpc服务名称
     * @Param inetSocketAddress rpc服务提供地址
     * @Return: void
     * @Date: 2021/7/13
     */
    void registerService(String rpcServiceName, InetSocketAddress inetSocketAddress);
}

/**
 * @Description 服务发现接口
 * @Author zrd
 * @Date 2021/7/10
 */
public interface ServiceDiscovery {
    /**
     * @Description: 通过请求参数找到对应的服务
     * @Param request
     * @Return: java.net.InetSocketAddress
     * @Date: 2021/7/10
     */
    InetSocketAddress findService(RpcRequest request);
}
```

接下来我们使用 ZooKeeper 作为注册中心的实现方式实现这两个接口。

```
/**
 * @Description Zookeeper服务注册实现类
 * @Author ZRD
 * @Date 2021/7/13
 */
public class ZkServiceRegistry implements ServiceRegistry {
    @Override
    public void registerService(String rpcServiceName, InetSocketAddress inetSocketAddress) {
        String servicePath = CuratorUtils.ZK_ROOT_PATH + "/" + rpcServiceName + inetSocketAddress.toString();
        CuratorFramework zkClient = CuratorUtils.getZkClient();
        CuratorUtils.createPersistentNode(zkClient, servicePath);
    }
}
```

当我们将服务注册进ZooKeeper的时候，会将服务的全限定类名作为根节点，服务地址(ip+port)作为子节点，一个根节点也可能会有多个子节点。

```
/**
 * @Description Zookeeper服务发现实现类
 * @Author zrd
 * @Date 2021/7/10
 */
@Slf4j
public class ZkServiceDiscovery implements ServiceDiscovery {
    @Override
    public InetSocketAddress findService(RpcRequest request) {
        String rpcServiceName = request.getClassName();
        CuratorFramework zkClient = CuratorUtils.getZkClient();
        List<String> serviceUrlList = CuratorUtils.getChildrenNodes(zkClient, rpcServiceName);
        if (serviceUrlList == null || serviceUrlList.size() == 0) {
            log.error("没有注册该服务[{}]", rpcServiceName);
        }
        String targetAddress = loadBalance.selectServiceAddress(serviceUrlList, request);
        String[] addrArray = targetAddress.split(":");
        String host = addrArray[0];
        int port = Integer.parseInt(addrArray[1]);
        return new InetSocketAddress(host, port);
    }
}
```

我们可以根据请求对象中的全限定类名来查询对应的服务地址，地址可能不止一个，所以我们需要通过对应的负载均衡策略来选择出一个服务地址。

## 负载均衡算法

负载均衡包括很多算法，这里主要讲一致性哈希算法。

### 普通哈希

假设有三台缓存服务器s0、s1、s2，对缓存下的键进行hash计算，哈希后的值是个整数，再用缓存服务器的数量对这个值进行取模计算，余数决定数据应该缓存到哪台服务器上。hash(名称) % 机器数 = 余数。缺陷是当服务器数量发生变化时，会导致不能正常访问缓存数据。

### 一致性哈希

普通哈希会出问题本质上是因为除余的数是一个变化的值，如果除以一个很大的数，影响的数据就会有限，一致性哈希将整个哈希值空间组织成一个虚拟的圆环，如假设某哈希函数H的值空间为0-2^32-1。接下来定位数据时主要分为两步，第一步 将各个服务器使用Hash进行一个哈希，具体可以选择服务器的ip或主机名作为关键字进行哈希，这样每台机器就能确定其在哈希环上的位置。第二步将数据key使用相同的函数Hash计算出哈希值，并确定此数据在环上的位置，从此位置沿环顺时针“行走”，第一台遇到的服务器就是其应该定位到的服务器。




