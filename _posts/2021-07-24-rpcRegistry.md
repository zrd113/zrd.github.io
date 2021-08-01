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
        String[] addrArray = serviceUrlList.get(0).split(":");
        String host = addrArray[0];
        int port = Integer.parseInt(addrArray[1]);
        return new InetSocketAddress(host, port);
    }
}
```

我们可以根据请求对象中的全限定类名来查询对应的服务地址，地址可能不止一个，所以我们需要通过对应的负载均衡策略来选择出一个服务地址。

