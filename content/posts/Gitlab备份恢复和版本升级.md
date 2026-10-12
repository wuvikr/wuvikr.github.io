+++
date = '2026-10-12T19:32:50+08:00'
draft = false
title = 'Gitlab 备份恢复和版本升级'
categories = ['DevOps']
tags = ['GitLab', '备份恢复', '版本升级']
+++

# 1. gitlab 数据备份

## 1.1 数据备份

GitLab 提供一条命令 `gitlab-backup create`，运行命令后会产出一个压缩 tar 归档，包含数据库、Git 仓库与 LFS、CI/CD artifacts、Wiki、附件上传、Pages、注册表等。但它**不包含**配置文件、密钥、Redis 队列和对象存储，这些要另行处理。

官方提示至少必须单独备份 `/etc/gitlab/gitlab-secrets.json`（内含数据库加密密钥，丢失后无法解密两步验证和 CI 安全变量）与 `/etc/gitlab/gitlab.rb`。建议连同 `/etc/gitlab/trusted-certs`一起备份，避免整机恢复时出现中间人警告。

## 1.2 执行备份
Omnibus 包安装的环境下，可以直接调用官方封装命令：
```bash
gitlab-backup create
```
归档默认生成在 `/var/opt/gitlab/backups/`，文件名一般是这个样子： `1730000000_2026_10_12_17.5_gitlab_backup.tar`。

可以在 `/etc/gitlab/gitlab.rb` 中调整文件保存路径和时间:
```bash
# 备份保留7天
gitlab_rails['backup_keep_time'] = 604800

# 备份目录配置
gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"
```
修改后执行 `gitlab-ctl reconfigure`。

需要排除某类数据时用 `SKIP`，例如跳过 CI 输出日志：
```bash
gitlab-backup create SKIP=builds,artifacts
```

## 1.3 生产备份策略

| 策略 | 做法 | 为什么 |
| --- | --- | --- |
| **定时执行** | 用 cron 每天/每周触发 `gitlab-backup create` | 备份频率决定最多可丢失多少数据（RPO） |
| **异地/对象存储** | 把 tar 归档拷贝到另一台机器或对象存储（支持 S3 等 remote 配置） | 单机磁盘故障时本地归档同样会丢 |
| **配置密钥单独备** | 定期打包 `/etc/gitlab` 到加密存储 | 密钥丢失=数据库无法解密，归档也无意义 |
| **定期恢复演练** | 在测试机用备份实际恢复一次 | 没验证能恢复的备份等于没有备份 |

> [!TIP] 提示：
多节点部署时，应在装有 Rails 的节点上执行备份，保证有足够磁盘和内存。

# 2. 数据恢复
## 2.1 恢复的硬约束与前提

官方规定：**只能把备份恢复到创建它的完全相同版本和类型（CE/EE）** 的 GitLab 中。恢复前先确认：目标实例版本与备份一致、已备份好 secrets 与配置、磁盘空间充足、有测试环境可先演练。

## 2.2 恢复步骤
1.  停止连接到数据库的服务，避免写入冲突：
    ```bash
    gitlab-ctl stop puma
    gitlab-ctl stop sidekiq
    ```

2.  确认待恢复的备份文件已放到 `/var/opt/gitlab/backups/`目录下，然后执行恢复命令：

    ```bash
    # 填文件名去掉 `_gitlab_backup.tar` 的部分
    gitlab-backup restore BACKUP=1730000000_2026_10_12_17.5
    ```

3.  恢复脚本会先清空数据库表并提示确认，可设 `GITLAB_ASSUME_YES=1` 跳过交互提示。

4.  恢复完成后重启并重建相关服务：

    ```bash
    gitlab-ctl restart
    gitlab-rake gitlab:check
    ```

5.  重新配置受影响的组件（如 CI/CD、外部服务），并核对用户能登录、仓库可克隆推送。


# 3. 版本升级

## 3.1 升级须知
GitLab 不支持跨多个大版本直接跳级。官方维护一张升级路径表，规定若干**必经停靠版本（required upgrade stops）**，例如 GitLab 17 需停靠 17.5、17.8、17.11；16 需停靠 16.3、16.7、16.11 等。每升到一级，都要**等后台迁移（background migrations）全部完成**，才能继续下一级。

以 17.10 → 19.4 为例，升级路径如下所示：
`17.10.8 => 17.11.7 => 18.2.8 => 18.5.7 => 18.8.11 => 18.11.12 => 19.2.7 => 19.4.1`

为什么要逐级进行升级？官方给出的解释是，某些 minor 版本包含升级流程相关的修复，必须停靠；**每升一级都须等后台迁移（background migrations）全部完成后再升下一级**。跨多 major 直接跳级会失败。

好在官方提供了 **[Upgrade Path 工具](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/)** ，这是一个由 GitLab Support 团队维护的在线网页，打开链接即可使用，输入当前版本和目标版本后会自动算出需停靠的版本，升级前务必用它核对。

## 3.2 升级前的准备

- **先备份**：按上文方法执行 `gitlab-backup create`，并备份 secrets 与配置，作为回滚依据。
- **运行健康检查**：`gitlab-rake gitlab:check`、`gitlab-rake gitlab:doctor:secrets`（确认数据库加密值可解密），并确认所有后台迁移已完成。
- **确认升级路径与系统兼容性**：使用[Upgrade Path 工具](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/) 确认整个升级流程需要经过哪些版本，目标版本是否支持当前操作系统。
- **在测试环境演练**：先在测试环境，创建一台克隆副本，完整跑一遍升级与回滚，测量迁移流程和耗时。

## 3.3 升级过程中的做法

- **开启维护模式 / 暂停 CI**：升级期间暂停 Runner 接收新任务，避免作业追踪失败；可配 `nginx['custom_gitlab_server_config']` 拒绝新的 jobs/request。

- **执行升级**：升级 Omnibus 包（若已配置官方 GitLab 源）：

    ```bash
    apt update
    apt install gitlab-ce=18.8.11-ce.0
    ```

- **升级 Runner**：GitLab Runner 应升级到与目标 GitLab 匹配的版本，避免兼容问题。

## 3.4 升级后验证

- 再次运行 `gitlab-rake gitlab:check`，确认无红色告警。
- 核对关键后台迁移是否继续跑完：可通过管理界面后台迁移页或对应命令确认。
- 在 UI 中验证登录、项目列表、issue/PR、仓库克隆与推送、Runner 能接任务。
- 确认新版本无误后再解除维护模式、恢复 Runner，并留意是否需同步升级 Elasticsearch 等可选组件。


# 参考链接

- [GitLab 备份文档](https://docs.gitlab.com/administration/backup_restore/backup_gitlab/)

- [GitLab 数据恢复](https://docs.gitlab.com/ee/administration/backup_restore/restore_gitlab.html)

- [GitLab 升级路径](https://docs.gitlab.com/update/upgrade_paths/)

- [GitLab 升级前备份与检查](https://docs.gitlab.com/update/plan_your_upgrade)

