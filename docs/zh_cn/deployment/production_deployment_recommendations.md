---
sidebar_position: 1
slug: /production_deployment_recommendations
---

# 生产环境部署建议

本文旨在给出一些部署 JuiceFS 至生产环境时的有效建议，请提前并仔细阅读以下内容。

## 监控指标收集与可视化

请务必收集 JuiceFS 客户端的监控指标并通过 Grafana 可视化，具体请参考[文档](../administration/monitoring.md)。

## 元数据自动备份

:::tip 提示
元数据自动备份是自 JuiceFS v1.0.0 版本开始加入的特性
:::

元数据是 JuiceFS 文件系统的核心数据，元数据一旦丢失或损坏将可能产生数据丢失风险。因此必须对元数据进行定期备份。

元数据自动备份特性默认开启，备份间隔为 1 小时，备份的元数据会经过压缩后存储至对应的对象存储中（与文件系统的数据隔离）。备份由 JuiceFS 客户端执行，备份期间会导致其 CPU 和内存使用量上升，默认情况下可认为会在所有客户端中随机选择一个执行备份操作。

特别注意默认情况下当文件系统的**文件数达到一百万**时，元数据自动备份功能将会关闭，需要配置一个更大的备份间隔（`--backup-meta` 选项）才会再次开启。备份间隔每个客户端独立配置，设置 `--backup-meta 0` 则表示关闭元数据自动备份特性。

:::note 注意
备份元数据所需的时间取决于具体的元数据引擎，不同元数据引擎会有不同的性能表现。
:::

有关元数据自动备份的详细介绍请参考[文档](../administration/metadata_dump_load.md#自动备份)，你也可以手动备份元数据。除此之外，也请遵照你所使用的元数据引擎的运维建议对数据进行定期备份。

## 回收站

:::tip 提示
回收站是自 JuiceFS v1.0.0 版本开始加入的特性
:::

回收站默认开启，文件被删除后的保留时间默认配置为 1 天，可以有效防止数据被误删除时造成的数据丢失风险。

不过回收站开启以后也可能带来一些副作用，如果应用需要经常删除文件或者频繁覆盖写文件，会导致对象存储使用量远大于文件系统用量。这本质上是因为 JuiceFS 客户端会将对象存储上被删除的文件或者覆盖写时产生的需要垃圾回收的数据块持续保留一段时间。因此，在部署 JuiceFS 至生产环境时就应该考虑好合适的回收站配置，回收站保留时间可以通过以下方式配置（如果将 `--trash-days` 设置为 `0` 则表示关闭回收站特性）：

- 新建文件系统：通过 `juicefs format` 的 `--trash-days <value>` 选项设置
- 已有文件系统：通过 `juicefs config` 的 `--trash-days <value>` 选项修改

有关回收站的详细介绍请参考[文档](../security/trash.md)。

## 客户端后台任务

同一个 JuiceFS 文件系统的所有客户端在运行过程中共享一个后台任务集，每个任务定时执行，且具体执行的客户端随机选择。具体的后台任务包括：

1. 清理待删除的文件和对象
2. 清理回收站中的过期文件和碎片
3. 清理长时间未响应的客户端会话
4. 自动备份元数据

由于这些任务执行时会占用一定资源，因此可以为业务较繁重的客户端配置 `--no-bgjob` 选项来禁止其参与后台任务。

:::note 注意
请保证至少有一个 JuiceFS 客户端可以执行后台任务
:::

## 客户端日志滚动

当后台运行 JuiceFS 挂载点时，客户端默认会将日志输出到本地文件中。取决于挂载文件系统时的运行用户，本地日志文件的路径稍有区别。root 用户对应的日志文件路径是 `/var/log/juicefs.log`，非 root 用户的日志文件路径是 `$HOME/.juicefs/juicefs.log`。

本地日志文件默认不会滚动，生产环境中为了确保日志文件不占用过多磁盘空间需要手动配置。以下是一个日志滚动的示例配置：

```text title="/etc/logrotate.d/juicefs"
/var/log/juicefs.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
}
```

通过 `logrotate -d` 命令可以验证配置文件的正确性：

```shell
logrotate -d /etc/logrotate.d/juicefs
```

有关日志滚动配置的详细介绍请参考[文档](https://linux.die.net/man/8/logrotate)。

## 命令行自动补全

JuiceFS 为 Bash 和 Zsh 提供了命令行自动补全脚本，方便在命令行中使用 `juicefs` 命令，具体请参考[文档](../reference/command_reference.md#自动补全)。