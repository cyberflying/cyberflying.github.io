---
title: AKS 使用 Virtual Nodes 快速扩展到 Azure 容器实例 (ACI)
date: 2020-02-17
categories: [Azure, Kubernetes]
tags: [aks, virtual-nodes, aci, azure, k8s, containers]
---

本文介绍如何在 Azure Kubernetes Service (AKS) 中使用 Virtual Nodes 功能，快速将 Pod 扩展到 Azure 容器实例 (ACI)，实现弹性伸缩。

## 概述

AKS 的 Virtual Nodes 功能基于 Virtual Kubelet 实现，允许将 Pod 调度到 Azure Container Instances 上运行。这种方式可以实现秒级扩展，特别适合突发流量场景。

## 详细内容

📖 完整文章请查看：[AKS 使用 Virtual Nodes 快速扩展到 Azure 容器实例 (ACI)](https://www.cyberflying.cn/virtual-kubelet-aci/)

> 源代码仓库：[github.com/cyberflying/virtual-kubelet-aci](https://github.com/cyberflying/virtual-kubelet-aci)
{: .prompt-info }
