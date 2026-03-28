---
title: minikube 使用 Virtual Kubelet 秒级快速扩展到 Azure 容器实例 (ACI)
date: 2020-02-27
categories: [Azure, Kubernetes]
tags: [minikube, virtual-kubelet, aci, azure, k8s, containers]
---

本文介绍如何在 minikube 环境中使用 Virtual Kubelet，将 Pod 秒级快速扩展到 Azure 容器实例 (ACI)，实现本地 K8S 集群与云端容器的无缝对接。

## 概述

通过 Virtual Kubelet，我们可以让 minikube 集群无缝对接 Azure Container Instances (ACI)，从而在需要时快速扩展工作负载到云端，无需管理额外的虚拟机。

## 详细内容

📖 完整文章请查看：[minikube 使用 Virtual Kubelet 秒级快速扩展到 Azure 容器实例 (ACI)](https://www.cyberflying.cn/minikube-aci/)

> 源代码仓库：[github.com/cyberflying/minikube-aci](https://github.com/cyberflying/minikube-aci)
{: .prompt-info }
