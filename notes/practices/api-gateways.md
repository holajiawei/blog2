---
title: API Gateway对比选型
description: Tyk vs Ambassador vs Traefik vs Nginx vs HAProxy vs Gloo
published: true
date: 2021-06-20T04:06:22.101Z
tags: k8s, api gateway
editor: markdown
dateCreated: 2021-06-20T04:06:22.101Z
---

# API Gateway对比选型

对比对象: Tyk vs Ambassador vs Traefik vs Nginx vs HAProxy vs Gloo vs Kong

## 背景

API Gateway价值和优势就不再赘述了，虽然上面👆罗列的诸多API Gateway的功能大多类似，但是在些细节处还是有很多不同的。而**这些不同都是基于不同的发展路径导致的**，不过现在都在往**同一方向**靠拢。

这种情况导致的核心原因就是一点，以Kubernetes为代表的容器编排服务的流行带来的全新的DevOps思潮变更。

### 在K8S🔥之前

Nginx、HAProxy甚至直接拿WebServer来充当外部流量访问入口，这些都是基本操作，常见而且成熟，同时架构层面倾向于代码整体合一的大泥球架构，那时候更多的公用逻辑往Web框架、中间件塞，接口数量也不多，Nginx配置一招鲜胜在简单

### 在K8S🔥之后
不是因为K8S🔥导致API Gateway领域逐渐成长，而是架构思想的变更，由大泥球架构往微服务架构发展带来的开发思路变革继而导致对编排类服务的渴求，包括容器编排、网络编排、数据/存储编排等
