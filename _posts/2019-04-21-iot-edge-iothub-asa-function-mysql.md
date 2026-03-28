---
title: 使用 Azure IoT Edge 将设备信息实时处理并存储到 MySQL
date: 2019-04-21
categories: [Azure, IoT]
tags: [iot-edge, iot-hub, stream-analytics, azure-functions, mysql, java]
---

本文介绍如何使用 Azure IoT Edge、IoT Hub、流分析 (Stream Analytics)、Azure Functions 等服务，构建一个从边缘设备到云端的完整数据处理管道，将设备遥测数据实时分析并存储到 Azure MySQL 数据库。

## 架构概览

整体架构如下：

```
IoT Edge Device → IoT Hub → Stream Analytics → Azure Functions (Java) → Azure MySQL
```

- **IoT Edge**：边缘设备采集和预处理数据
- **IoT Hub**：云端消息中心，接收设备数据
- **Stream Analytics**：实时流分析，过滤和转换数据
- **Azure Functions**：使用 Java 编写的无服务器函数，处理数据写入
- **Azure MySQL**：持久化存储

## 详细内容

📖 完整文章请查看：[使用 Azure IoT Edge 将设备信息实时处理并存储到 MySQL](https://www.cyberflying.cn/iotedge-iothub-asa-function-mysql/)

> 源代码仓库：[github.com/cyberflying/iotedge-iothub-asa-function-mysql](https://github.com/cyberflying/iotedge-iothub-asa-function-mysql)
{: .prompt-info }
