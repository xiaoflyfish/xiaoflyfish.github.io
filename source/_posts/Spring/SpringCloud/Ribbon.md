---
title: Ribbon
categories: 
- Spring
- SpringCloud
---

**负载均衡策略**

| 规则名称                    | 特点                                                         |
| :-------------------------- | :----------------------------------------------------------- |
| `AvailabilityFilteringRule` | 过滤掉一直连接失败的被标记为circuit tripped（电路跳闸）的后端Service，并过滤掉那些高并发的后端Server或者使用一个AvailabilityPredicate来包含过滤Server的逻辑，其实就是检查status的记录的各个Server的运行状态 |
| `BestAvailableRule`         | 选择一个最小的并发请求的Server，逐个考察Server，如果Server被tripped了，则跳过 |
| `RandomRule`                | 随机选择一个Server                                           |
| `ResponseTimeWeightedRule`  | 已废弃，作用同WeightedResponseTimeRule                       |
| `RetryRule`                 | 对选定的负责均衡策略机上重试机制，在一个配置时间段内当选择Server不成功，则一直尝试使用subRule的方式选择一个可用的Server |
| `RoundRobinRule`            |（默认）轮询选择，轮询index，选择index对应位置Server                 |
| `WeightedResponseTimeRule`  | 根据相应时间加权，相应时间越长，权重越小，被选中的可能性越低 |
| `ZoneAvoidanceRule`         | 负责判断Server所Zone的性能和Server的可用性选择Server，在没有Zone的环境下，类似于轮询（`RoundRobinRule`） |

> WeightedResponseTimeRule策略

该策略与请求的响应时间有关，显然，如果响应时间越长，就代表这个服务的响应能力越有限，那么分配给该服务的权重就应该越小。

而响应时间的计算就依赖于 ILoadBalancer 接口中的 LoadBalancerStats。

WeightedResponseTimeRule会定时从 LoadBalancerStats 读取平均响应时间，为每个服务更新权重。权重的计算也比较简单，即每次请求的响应时间减去每个服务自己平均的响应时间就是该服务的权重。

> AvailabilityFilteringRule策略

通过检查 LoadBalancerStats 中记录的各个服务器的运行状态，过滤掉那些处于一直连接失败或处于高并发状态的后端服务器。