---
title: 在 Azure VM 上部署 OpenClaw，并通过 Managed Identity + Azure CLI 实现最小权限资源操作
date: 2026-03-31
categories: [Azure, AI]
tags: [azure, vm, managed-identity, azure-cli, rbac, openclaw, agent]
description: 在 Azure Linux VM 上安装 OpenClaw 和 Azure CLI，使用 Managed Identity 与 Azure RBAC 让 OpenClaw 仅在受限范围内访问和操作 Azure 资源。
---

当我们把 OpenClaw 部署在一台长期在线的 Azure VM 上时，一个很自然的需求是：让它能够访问 Azure 资源，但又不能拿到一个权限过大的账号，更不能把长期有效的 Client Secret、证书或者账号密码直接放进机器里。

比较稳妥的一种做法，是把 **Azure VM 自己的 Managed Identity** 作为 OpenClaw 访问 Azure 的身份，再在这台 VM 上安装 `az` CLI。这样一来，OpenClaw 不需要保存任何静态凭据，只要在本机执行 `az login --identity`，就可以基于这台 VM 的托管身份访问 Azure 资源。

更重要的是，这种方式天然适合和 **Azure RBAC** 配合：我们可以把权限授予这个 Managed Identity，而不是授予某个人类用户；也可以把权限范围收敛到订阅、资源组，甚至单个资源；如果内置角色不够贴合需求，还可以创建自定义角色，只开放必要的 `Actions`。这样，OpenClaw 能做什么、不能做什么，都可以通过 RBAC 明确约束。

本文记录一套我认为比较适合生产环境的方式：

- OpenClaw 运行在 Azure Linux VM 上
- VM 启用 Managed Identity
- VM 安装 Azure CLI
- OpenClaw 通过宿主机上的 `az` 命令访问 Azure
- 通过最小权限 RBAC 控制它的操作边界

## 整体思路

可以把这套方案理解成下面这条链路：

```text
OpenClaw -> 宿主机 az CLI -> Managed Identity -> Azure RBAC -> Azure Resources
```

这里的关键点不是“让 Agent 拿到 Azure 权限”，而是“让 Agent 只能拿到被明确授权的 Azure 权限”。

与传统的 Service Principal + Client Secret 方案相比，这种做法有几个实际优势：

1. 不需要在 VM 上保存长期凭据。
2. 身份直接绑定到 Azure 资源，生命周期更清晰。
3. 权限授予和回收都走 Azure RBAC，审计路径更统一。
4. 可以把授权范围压缩到非常小，例如只允许读取某个资源组，或者只允许启动和停止某几个虚拟机。

如果只是给单台 VM 使用，**System-assigned Managed Identity** 往往是最简单的选择；如果未来需要多台 VM 共享同一组权限，或者希望身份生命周期独立于 VM，则可以考虑 **User-assigned Managed Identity**。

> 截图位 1：整套架构示意图，建议画成 “OpenClaw / Azure CLI / Managed Identity / RBAC / Target Resources” 五层关系图。

## 前提条件

开始之前，我默认你已经具备以下条件：

- 已有一台 Azure Linux VM
- 可以登录到这台 VM
- 有权限为 VM 启用 Managed Identity
- 有权限在目标作用域上分配 Azure RBAC 角色
- 准备好了 OpenClaw 所需的模型提供商 API Key

如果你还没有部署 VM，可以直接参考 OpenClaw 官方提供的 Azure Linux VM 指南，它包含了 Azure CLI、NSG、Bastion 和 OpenClaw 安装的完整流程。

## 在 Azure VM 上安装 OpenClaw

OpenClaw 官方已经提供了安装脚本，Linux 上可以直接执行：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

安装完成后，可以先做几个基本检查：

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

如果是第一次安装，建议继续执行引导流程：

```bash
openclaw onboard --install-daemon
```

这一步会完成网关、认证和基础配置。对于长期运行在云上的实例，我更建议把 OpenClaw 当成一个持续运行的服务来管理，而不是只在交互式 shell 中临时启动。

> 截图位 2：OpenClaw 安装完成后的 `openclaw --version` / `openclaw gateway status` 输出。

## 在 VM 上安装 Azure CLI

接下来要做的是在这台 VM 上安装 Azure CLI。因为 OpenClaw 最终是通过宿主机命令来访问 Azure，所以 `az` 就是它和 Azure Resource Manager 之间最直接的桥梁。

不同 Linux 发行版的安装方式略有不同。最稳妥的做法，是按照官方文档选择对应的包管理器安装。下面给一个 Ubuntu / Debian 上比较常见的示例：

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

安装完成后先确认版本：

```bash
az version
```

如果你的 VM 不是 Ubuntu / Debian，例如 Azure Linux、RHEL、CentOS Stream 或 SLES，建议直接按官方文档切换到对应的 `tdnf`、`dnf` 或 `zypper` 安装方式。

> 截图位 3：VM 上 `az version` 的输出。

## 为 Azure VM 启用 Managed Identity

接下来进入整篇文章最关键的一步：给这台 VM 启用 Managed Identity。

如果这台 VM 只是单独运行一个 OpenClaw 实例，我建议优先使用 **System-assigned Managed Identity**。它和 VM 一一绑定，配置简单，也比较符合“一台机器一个身份”的思路。

在 Azure Portal 中操作很直接：

1. 打开目标虚拟机。
2. 进入 `Security -> Identity`。
3. 在 `System assigned` 中把状态切换为 `On`。
4. 点击保存。

启用之后，Azure 会自动为这台 VM 在 Microsoft Entra ID 中创建并维护对应的服务主体身份。后续我们要做的，就是把 Azure RBAC 权限授予这个身份。

> 截图位 4：Azure Portal 中 VM 的 `Identity` 页面，`System assigned` 为 `On`。

如果你更偏好命令行，也可以使用 Azure CLI：

```bash
az vm identity assign \
  --resource-group <vm-resource-group> \
  --name <vm-name>
```

启用完成后，可以查看这台 VM 的托管身份信息：

```bash
az vm identity show \
  --resource-group <vm-resource-group> \
  --name <vm-name>
```

## 在 VM 内使用 Managed Identity 登录 Azure CLI

Managed Identity 启用后，就可以回到这台 VM 内部，用它自己的身份登录 Azure CLI：

```bash
az login --identity
```

如果你使用的是 **User-assigned Managed Identity**，则可以显式指定它的客户端 ID：

```bash
az login --identity --client-id <managed-identity-client-id>
```

登录成功后，可以先做几个最基础的验证：

```bash
az account show
az group list -o table
```

如果此时已经给这个 Managed Identity 分配了某个资源组的读取权限，那么这里应该能看到对应范围内允许访问的资源；如果还没有授权，或者授权范围太小，则命令会失败或者返回受限结果。

这里要强调一点：**`az login --identity` 只是完成“身份认证”，真正决定它能做什么的是后面的 RBAC 授权。**

> 截图位 5：VM 里执行 `az login --identity` 和 `az account show` 的输出。

## 用 Azure RBAC 限制 OpenClaw 的权限边界

这一部分是整套方案里最有价值的地方。

很多时候，问题不是“能不能让 OpenClaw 操作 Azure”，而是“怎样让它只操作一小部分 Azure 资源，并且只能做少数几个动作”。Azure RBAC 正是用来解决这个问题的。

一个完整的角色分配，至少包含三个元素：

- 谁需要权限：这里就是 VM 的 Managed Identity
- 分配什么角色：内置角色或自定义角色
- 在什么范围上分配：管理组、订阅、资源组或具体资源

### 先决定作用域，再决定角色

一个非常重要的原则是：**先把作用域收小，再考虑角色本身。**

例如，如果 OpenClaw 只需要管理某个实验环境资源组里的几台 VM，那就没有必要把权限分配到整个订阅。更合理的方式是：

- 只在目标资源组上授权
- 如果还可以继续缩小，就授权到单个资源

常见的几个作用域层级如下：

- 管理组
- 订阅
- 资源组
- 单个资源

从安全角度看，我通常会优先采用“资源组级”或“资源级”的授权方式。

### 使用内置角色的做法

如果你的需求只是让 OpenClaw 读取资源信息，那么可以先考虑 `Reader` 这类内置角色。

如果需要执行较少量的管理动作，也可以先找一个最接近的内置角色作为起点，然后再评估它是否过宽。这里不建议一上来就给 `Contributor`，因为它的范围通常会大于真实需求。

在 Portal 中给 Managed Identity 分配角色的大致过程如下：

1. 打开目标作用域，例如某个资源组。
2. 进入 `Access control (IAM)`。
3. 点击 `Add -> Add role assignment`。
4. 选择角色。
5. 在成员类型中选择 `Managed identity`。
6. 选择对应的 VM 托管身份并完成分配。

> 截图位 6：资源组的 `Access control (IAM)` 页面。
>
> 截图位 7：给 Managed Identity 分配角色的向导页面。

### 使用 Azure CLI 分配角色

如果你更喜欢用命令行，也可以直接查询这台 VM 的 Managed Identity 主体 ID，再把角色授给它。

先拿到主体 ID：

```bash
PRINCIPAL_ID=$(az vm identity show \
  --resource-group <vm-resource-group> \
  --name <vm-name> \
  --query principalId \
  -o tsv)
```

然后在指定作用域上授予角色：

```bash
az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role Reader \
  --scope /subscriptions/<subscription-id>/resourceGroups/<target-resource-group>
```

如果你只是希望 OpenClaw 能读取某个资源组中的资源信息，这样的授权通常已经够用了。

## 使用自定义角色实现更细的最小权限

当内置角色仍然太宽时，就该上自定义角色了。

自定义角色的核心价值在于：你可以明确列出允许的 `Actions`，从而只开放真正需要的能力。例如，只允许：

- 读取虚拟机信息
- 启动虚拟机
- 重启虚拟机
- 停止或释放虚拟机

而不允许创建、删除、修改其他 Azure 资源。

下面是一个简单的示例角色，它只允许在指定作用域内查看虚拟机和资源组，并对虚拟机执行启动、重启和停止动作：

```json
{
  "Name": "OpenClaw VM Operator",
  "IsCustom": true,
  "Description": "Allow OpenClaw to read VM state and perform limited VM operations.",
  "Actions": [
    "Microsoft.Resources/subscriptions/resourceGroups/read",
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/<subscription-id>"
  ]
}
```

把上面的内容保存为 `openclaw-vm-operator.json`，然后创建角色：

```bash
az role definition create --role-definition @openclaw-vm-operator.json
```

上面的示例把 `AssignableScopes` 写到了订阅级别，主要是为了便于演示和复用。到了生产环境里，如果你的授权边界已经非常明确，也可以进一步把它收紧到某个特定资源组。

再把这个自定义角色授予 VM 的 Managed Identity：

```bash
az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "OpenClaw VM Operator" \
  --scope /subscriptions/<subscription-id>/resourceGroups/<target-resource-group>
```

这样做以后，OpenClaw 即使能执行 `az` 命令，也只能在这个资源组里做这几个动作。超出边界的命令会直接被 Azure 拒绝。

这正是我最看重的一点：**把 Agent 的可执行能力，收敛成一组明确、可审计、可回收的 RBAC 权限。**

> 截图位 8：自定义角色的 JSON 或 Portal 中的自定义角色页面。

## 验证权限是否符合预期

权限配置完成后，不要急着直接投入使用，最好先做一次“正向验证 + 反向验证”。

正向验证，就是测试它应该能做的事是否真的能做：

```bash
az vm list -g <target-resource-group> -o table
az vm start -g <target-resource-group> -n <vm-name>
az vm deallocate -g <target-resource-group> -n <vm-name>
```

反向验证，则是故意执行一个不应该被允许的动作，例如访问授权范围外的资源组，或者读取未授权的资源类型：

```bash
az vm list -g <another-resource-group> -o table
az network vnet list -g <target-resource-group> -o table
```

如果 RBAC 配置正确，这些超出边界的操作应该会被明确拒绝。

这一轮验证非常重要，因为它决定了你是在“理论上最小权限”，还是在“实际上最小权限”。

## 让 OpenClaw 通过 `az` 操作 Azure 资源

当 OpenClaw 运行在这台 VM 上，而且它所在的运行环境允许调用宿主机命令时，它就可以直接复用本机上的 Azure CLI。

这意味着你可以把它当成一个“带受限 Azure 操作能力的 Agent”。例如，你可以让它执行类似这样的任务：

- 列出某个资源组中的虚拟机状态
- 启动或停止指定 VM
- 检查某个资源组中的资源清单
- 按照你授权的范围执行日常运维动作

但如果你没有授予创建、删除、修改网络、改 RBAC 等能力，那么它也就做不到这些事。

也就是说，OpenClaw 的“工具能力”来自宿主机，而它的“权限边界”来自 Azure RBAC。把这两者组合起来，才能既保留自动化能力，又避免权限失控。

## 几点实践建议

最后总结几个我自己更倾向采用的实践方式。

### 1. 默认使用最小作用域

能授权到资源组，就不要授权到订阅；能授权到单个资源，就不要放大到整个资源组。

### 2. 默认不用 Contributor

很多场景里，`Contributor` 看起来省事，但往往比真实需求宽得多。先从 `Reader` 或自定义角色开始，通常更稳妥。

### 3. 把自定义角色做成“面向任务”的角色

不要做一个“万能 OpenClaw 角色”，而是根据任务做拆分，例如：

- `OpenClaw VM Reader`
- `OpenClaw VM Operator`
- `OpenClaw Storage Reader`

这样后续排错、审计和回收权限都会简单很多。

### 4. 先手工验证，再交给 Agent

先在 VM 上手工执行一遍 `az login --identity` 和目标命令，确认行为完全符合预期，再让 OpenClaw 调用它们。这样可以把问题切分清楚：到底是 OpenClaw 配置问题，还是 Azure 身份和 RBAC 配置问题。

### 5. 不要把 OpenClaw 网关直接暴露在公网

如果是长期运行在云上的实例，我更建议把 OpenClaw 控制面保留在本机回环地址，通过 SSH Tunnel、Bastion 或其他受控方式访问，而不是直接把管理入口暴露到公网。

## 小结

把 OpenClaw 部署在 Azure VM 上，并通过 **Managed Identity + Azure CLI + Azure RBAC** 来访问 Azure 资源，是一种很适合长期运行 Agent 的方式。

这套方案真正有价值的地方，不是“让 OpenClaw 能操作 Azure”，而是：

- 不需要在机器里保存静态凭据
- 权限绑定在 Azure 资源身份上
- 权限范围可以压缩到很小
- 可以用自定义角色精确描述允许的动作
- 权限回收、审计和调整都能走 Azure 的标准机制

如果你的目标是把 OpenClaw 作为一个受控的 Azure 运维入口，而不是一个拥有全局权限的自动化账号，那么这种模式会比直接塞一个高权限 Service Principal 更可控，也更适合长期维护。

后续我还会继续补几张截图，包括：

- VM 启用 Managed Identity
- IAM 中给 Managed Identity 分配角色
- 自定义角色配置页面
- VM 内 `az login --identity` 的验证输出
- OpenClaw 调用 `az` 的实际效果

## 参考资料

- [OpenClaw 安装文档](https://docs.openclaw.ai/install)
- [OpenClaw Azure Linux VM 部署指南](https://docs.openclaw.ai/install/azure)
- [Azure CLI: Sign into Azure using a managed identity](https://learn.microsoft.com/en-us/cli/azure/authenticate-azure-cli-managed-identity?view=azure-cli-latest)
- [Managed identities on Azure virtual machines](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-to-configure-managed-identities)
- [Understand scope for Azure RBAC](https://learn.microsoft.com/en-us/azure/role-based-access-control/scope-overview)
- [Assign Azure roles using the Azure portal](https://learn.microsoft.com/en-us/azure/role-based-access-control/role-assignments-portal)
- [Azure custom roles](https://learn.microsoft.com/en-us/azure/role-based-access-control/custom-roles)
- [Create or update Azure custom roles using Azure CLI](https://learn.microsoft.com/en-us/azure/role-based-access-control/custom-roles-cli)
