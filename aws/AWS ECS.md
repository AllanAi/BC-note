## 什么是 Amazon ECS？
https://docs.aws.amazon.com/zh_cn/AmazonECS/latest/developerguide/Welcome.html
Amazon Elastic Container Service (Amazon ECS) 是一项高度可扩展的快速容器管理服务，它可轻松运行、停止和管理集群上的 Docker 容器。您可以通过使用 Fargate 启动类型启动服务或任务，将集群托管在由 Amazon ECS 管理的无服务器基础设施上。若要进行更多控制，您可以在使用 EC2 启动类型进行管理的 Amazon Elastic Compute Cloud (Amazon EC2) 实例集群上托管您的任务。

## 设置
https://docs.aws.amazon.com/zh_cn/AmazonECS/latest/developerguide/get-set-up-for-amazon-ecs.html
创建密钥对
**对于 Amazon ECS，密钥对只有在您打算使用 EC2 启动类型时才需要。**
AWS 使用公有密钥密码术来保护实例的登录信息。 Linux 实例（例如 Amazon ECS 容器实例）没有用于 SSH 访问的密码。您使用密钥对安全地登录到实例。您可以在启动容器实例时指定密钥对的名称，然后提供私有密钥（使用 SSH 登录时）。

创建安全组

安全组用作相关容器实例的防火墙，可在容器实例级别控制入站和出站流量。

您可能需要添加 SSH 规则，以便登录容器实例并使用 Docker 命令检查任务。如果您希望容器实例托管运行 Web 服务器的任务，也可以添加适用于 HTTP 和 HTTPS 的规则。完成以下步骤可添加这些可选的安全组规则。

在 **Inbound** 选项卡上，创建以下规则 (为每个新规则选择 **Add Rule**)，然后选择 **Create**：

- 从 **Type** 列表中选择 **HTTP**，确保 **Source** 设置为 **Anywhere** (`0.0.0.0/0`)。
- 从 **Type** 列表中选择 **HTTPS**，确保 **Source** 设置为 **Anywhere** (`0.0.0.0/0`)。
- 从 **Type** 列表中选择 **SSH**。在 **Source** 字段中，确保选中 **Custom IP**，然后采用 CIDR 表示法指定您计算机或网络的公有 IP 地址。要采用 CIDR 表示法指定单个 IP 地址，请添加路由前缀 `/32`。例如，如果您的 IP 地址是 `203.0.113.25`，请指定 `203.0.113.25/32`。如果您的公司要分配同一范围内的地址，请指定整个范围，例如 `203.0.113.0/24`。

## AWS Fargate

https://docs.aws.amazon.com/zh_cn/AmazonECS/latest/developerguide/AWS_Fargate.html

## 任务定义参数

https://docs.aws.amazon.com/zh_cn/AmazonECS/latest/developerguide/task_definition_parameters.html

logConfiguration

容器的日志配置规范。

此参数将映射到 [Docker Remote API](https://docs.docker.com/engine/api/v1.38/) 的[创建容器](https://docs.aws.amazon.com/zh_cn/AmazonECS/latest/developerguide/task_definition_parameters.html#operation/ContainerCreate)部分中的 `LogConfig` 以及 [`docker run`](https://docs.docker.com/engine/reference/commandline/run/) 的 `--log-driver` 选项。默认情况下，容器使用 Docker 守护程序所用的同一日志记录驱动程序；但容器可能通过在容器定义中使用此参数指定日志驱动程序来使用不同于 Docker 守护程序的日志记录驱动程序。要对容器使用不同的日志记录驱动程序，必须在容器实例上正确配置日志系统（或者在不同的日志服务器上使用远程日志记录选项）。有关其他支持的日志驱动程序选项的更多信息，请参阅 Docker 文档中的[配置日志记录驱动程序](https://docs.docker.com/engine/admin/logging/overview/)。

指定容器的日志配置时，应注意以下事项：

- 对于使用 EC2 启动类型的任务，在容器实例上运行的 Amazon ECS 容器代理必须先将该实例上的可用日志记录驱动程序注册到 `ECS_AVAILABLE_LOGGING_DRIVERS` 环境变量，然后该实例上的容器才能使用这些日志配置选项。有关更多信息，请参阅 [Amazon ECS 容器代理配置](https://docs.aws.amazon.com/zh_cn/AmazonECS/latest/developerguide/ecs-agent-config.html)。
- 对于使用 Fargate 启动类型的任务，因为您无权访问托管您的任务的底层基础设施，所以必须在此任务之外安装任何所需的其他软件。例如，Fluentd 输出聚合函数或运行 Logstash 以将 Gelf 日志发送到的远程主机。

logDriver

类型：字符串

对于使用 Fargate 启动类型的任务，受支持的日志驱动程序为 `awslogs`、`splunk` 和 `awsfirelens`。

对于使用 EC2 启动类型的任务，受支持的日志驱动程序为 `awslogs`、`fluentd`、`gelf`、`json-file`、`journald`、`logentries`、`syslog`、`splunk` 和 `awsfirelens`。

有关在任务定义中使用 `awslogs` 日志驱动程序以将容器日志发送到 CloudWatch Logs 的更多信息，请参阅[使用 awslogs 日志驱动程序](https://docs.aws.amazon.com/zh_cn/AmazonECS/latest/developerguide/using_awslogs.html)。

有关使用 `awsfirelens` 日志驱动程序的更多信息，请参阅[自定义日志路由](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/using_firelens.html)。

