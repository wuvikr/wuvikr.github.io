+++
date = '2026-10-12T14:51:37+08:00'
draft = false
title = 'Gitlab 入门和生产部署实践'
categories = ['DevOps']
tags = ['GitLab', 'GitOps', 'CICD']
+++

# 1. 什么是 GitLab
GitLab 是一款开源的 Git 代码托管平台，支持私有化部署，被众多企业用于搭建内部代码仓库。与 GitHub 主打开源协作不同，GitLab 更聚焦于代码托管和研发流程自动化，原生提供 CI/CD、代码评审、权限管控、镜像仓库、日志审计等能力。

GitLab 主要分为两个版本：
- **CE 社区版**：免费开源，基本能满足 95% 中小企业的生产需求（本文案例）；
- **EE 企业版**：付费使用，提供高可用架构、高级安全扫描、集群权限管控以及官方技术支持，比较适合大厂落地。

总之，对企业来说，GitLab 的核心价值主要体现在四个方面：**代码内网私有化存储、研发流程标准化、流水线自动化、操作可审计追溯**。

# 2. 三种部署方式（选型）
1. **Omnibus 包部署（生产首选）**：官方推荐的标准生产部署方案，以一键安装包形式提供，支持主流 Linux 发行版。内置 PostgreSQL、Redis、Gitaly、Web、Sidekiq 等核心组件，部署路径清晰、运维成本较低、稳定性较高，是生产环境中最稳妥的选择。
2. **K8s 混合部署（不推荐新人尝试）**：面向大规模高并发场景时，官方推荐采用混合架构，将 Web、Sidekiq 等无状态组件部署在 Kubernetes 中，便于弹性扩缩；Gitaly、数据库、缓存等有状态服务则保留在虚拟机或独立节点上部署，以保证存储和数据的稳定性。不建议新手直接使用 Kubernetes 全组件部署 GitLab。有状态组件在容器环境中的存储管理、备份恢复和升级复杂度较高，生产风险较大。
3. **Docker 单容器部署（仅测试用）**：Docker 单容器部署启动快、上手简单，但在数据持久化、备份恢复、版本升级和高可用方面能力有限，仅适合本地测试、功能验证或演示环境，不建议用于生产。

# 3. 安装 GitLab
[官方安装文档](https://docs.gitlab.com/omnibus/installation/index.html)

## 3.1 前置硬件&系统要求
- 仅支持 linux，最低配置：4核8G（10人以内小团队），生产推荐 8 核 16G 及以上；
- 关闭 SELinux 或 AppArmor，否则可能影响服务正常运行；
- 存储优先用 SSD；不建议用 NFS、EFS、Azure Files 这类网络文件系统放仓库数据；
- 开放端口：80(http)、443(https)、22(ssh)。

## 3.2 添加 apt 源安装
```bash
# 更新系统依赖
sudo apt update && sudo apt upgrade -y

# 安装必备依赖
sudo apt install -y curl openssh-server ca-certificates tzdata

# 添加 GitLab 官方 APT 源
sudo curl --location "https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh" | sudo bash

# 安装 GitLab 社区版（默认启动初始化）
sudo apt install gitlab-ce -y
```
## 3.3 离线包（无网络环境使用）
[官方离线包下载地址](https://packages.gitlab.com/gitlab/gitlab-ce)

[清华源镜像地址](https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/)

离线安装命令：
```bash
# 下载后进入包目录执行
sudo dpkg -i gitlab-ce_*.deb
sudo apt -f install -y
```

## 3.4 核心配置与启动
编辑主配置文件，配置访问域名/IP：
```bash
# 编辑配置文件
sudo vim /etc/gitlab/gitlab.rb

# 配置域名或者ip，（核心参数）
# 生产环境建议使用 https:// + 域名，GitLab 会自动创建和申请 SSL 证书
external_url 'http://gitlab.wuvikr.top'

# 重新加载配置并启动（核心命令）
sudo gitlab-ctl reconfigure

# 查看运行状态
sudo gitlab-ctl status
```
到这里，gitlab 就算是安装完成了。接下来会介绍一些生产环境中必做的一些其他配置。

# 4. 其他配置
## 4.1 获取初始密码 + 修改密码
GitLab 初始 root 密码自动生成，24小时后自动删除，必须第一时间进行修改：
```bash
# 查看初始密码
cat /etc/gitlab/initial_root_password
```
打开 web 页面并登录，账号：root，密码为上面文件内字符串，登录后立即修改密码。

## 4.2 关闭公开注册
通常新安装的 gitlab，打开设置页面，最上面就会有关闭新用户注册的安全提示，直接点击进去即可，如果没有可以按这个路径操作：
1. 右上角点击 **Admin / 管理区**；
2. 左侧选择 **Settings > General**；
3. 展开 **New user account restrictions**；
4. 取消勾选 **Allow new user accounts**；
5. 拉到最下面点 **Save changes**；

确保所有账号仅管理员手动创建。
> [!TIP] 提示：
> 最新版本的 GitLab 提供了更完善的用户注册控制。如果不想让陌生人随意注册，但希望内部同事可以先注册、再由管理员审批，可以保留 Allow new user accounts，同时勾选 Require admin approval for new user accounts。开启后，新用户注册后会进入“等待批准”状态，管理员审批通过后才能正常使用账号。
>
> 另外，还可以配置邮箱域名限制，只允许指定域名注册，例如只允许 `@公司域名.com` 的用户注册，从而避免被外部人员随意注册的风险。

## 4.3 配置 SMTP 邮件
编辑配置文件`/etc/gitlab/gitlab.rb`，修改邮件相关的配置（以QQ邮箱为例）：
```ruby
### GitLab email server settings
###! Docs: https://docs.gitlab.com/omnibus/settings/smtp.html
###! **Use smtp instead of sendmail/postfix.**

gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.qq.com"
gitlab_rails['smtp_port'] = 465
gitlab_rails['smtp_user_name'] = "wuvikr@qq.com"
gitlab_rails['smtp_password'] = "smtp password"
gitlab_rails['smtp_domain'] = "qq.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = false
gitlab_rails['smtp_tls'] = true
gitlab_rails['smtp_pool'] = false

###! **Can be: 'none', 'peer', 'client_once', 'fail_if_no_peer_cert'**
###! Docs: http://api.rubyonrails.org/classes/ActionMailer/Base.html
# gitlab_rails['smtp_openssl_verify_mode'] = 'none'

# gitlab_rails['smtp_ca_path'] = "/etc/ssl/certs"
# gitlab_rails['smtp_ca_file'] = "/etc/ssl/certs/ca-certificates.crt"

### Email Settings

# gitlab_rails['gitlab_email_enabled'] = true

###! If your SMTP server does not like the default 'From: gitlab@gitlab.example.com'
###! can change the 'From' with this setting.
gitlab_rails['gitlab_email_from'] = 'wuvikr@qq.com'
# gitlab_rails['gitlab_email_display_name'] = 'Example'
# gitlab_rails['gitlab_email_reply_to'] = 'noreply@example.com'
# gitlab_rails['gitlab_email_subject_suffix'] = ''
# gitlab_rails['gitlab_email_smime_enabled'] = false
# gitlab_rails['gitlab_email_smime_key_file'] = '/etc/gitlab/ssl/gitlab_smime.key'
# gitlab_rails['gitlab_email_smime_cert_file'] = '/etc/gitlab/ssl/gitlab_smime.crt'
# gitlab_rails['gitlab_email_smime_ca_certs_file'] = '/etc/gitlab/ssl/gitlab_smime_cas.crt'
```
重新加载配置生效：`gitlab-ctl reconfigure`，后台测试邮件发送。

## 4.4 仓库权限管理和加固
1. **关闭用户创建公开项目**：进入 `Admin → Settings → General → Visibility and access controls`，在 Default project visibility 中设置为 Private。这样新建项目默认就是私有项目，避免用户误建公开仓库。
2. **限制用户修改可见级别**：仅设置私有仓库默认值还不够，项目所有者仍然可以把项目改成公开或内部。建议再把 **Restricted visibility levels** 勾选 Public，必要时再勾选 Internal。设置后，普通用户就不能创建或切换到公开项目，只有管理员可以。
3. **锁定仓库关键权限**：
    - 在 Visibility and access controls 中，把 创建项目所需默认最低角色 **设为 Maintainer 或 管理员**，防止普通用户随意建项目；
    - 在项目或群组中配置 Protected Branches，限制 `main/master` 分支的推送和合并权限；
    - 把普通成员设为 Developer，只有负责人给 Maintainer，避免普通成员修改受保护分支、CI/CD、Webhook 等关键配置。
4. **创建层级分组，统一归类仓库**：
    - 按照公司组织架构建立群组层级：`总公司组 → 部门组 → 项目组`。建议将顶级群组权限设为私有（Private），子群组和项目将自动继承该设置，安全性更高
    - 路径示例：`/company、/company/backend、/company/backend/auth-service`所有仓库统一归入相应的命名空间下，权限可从父群组向下继承，后续人员管理和权限回收都会更方便。


# 5. 生产环境最佳实践
## 5.1 安全加固最佳实践
- **账号安全**：开启两步验证，长期没有使用的账号自动禁用，按时清理离职账号，禁止共用 root 管理员账号；
- **代码安全**：开启 MR 强制评审，禁止开发者直接合并代码；开启敏感文件检测，拦截密钥、密码硬编码提交；
- **访问安全**：服务器配置 IP 白名单，仅办公网、运维网可访问 GitLab，**禁止公网直接暴露**。

## 5.2 自动备份+容灾
GitLab 自带备份工具，配置定时自动备份，防止数据丢失：
1. **开启自动备份**：
编辑`/etc/gitlab/gitlab.rb`：
    ```ruby
    # 备份保留7天
    gitlab_rails['backup_keep_time'] = 604800

    # 备份目录配置
    gitlab_rails['backup_path'] = "/var/opt/gitlab/backups"
    ```
    重载配置生效。

2. **手动备份命令**：`gitlab-backup create`，备份文件默认存放路径：`/var/opt/gitlab/backups`

3. **定时备份脚本（crontab）**：
    ```bash
    # 每日凌晨2点自动备份
    0 2 * * * /usr/bin/gitlab-backup create CRON=1
    ```
    建议没隔一段时间测试一次备份恢复，确保备份文件可用；备份文件同步至异地服务器，不存单机。

以上的备份内容仅包括：Git 仓库、PostgreSQL 数据库、用户、群组、权限、上传附件、CI/CD 产物、LFS 对象等。不包含 gitlab 的关键配置信息，因此，对`/etc/gitlab` 目录还需要单独做备份处理。


## 5.3 日常运维&性能优化
- 日志清理：定期清理流水线日志、过期分支、废弃MR，释放磁盘空间，避免磁盘爆满导致服务宕机
- 版本升级：只做小版本迭代修复漏洞，禁止跨大版本直接升级，极易引发数据兼容问题
- 资源监控：监控服务器CPU、内存、磁盘、GitLab各组件状态，配置钉钉/邮件告警，提前发现负载异常
- 组件维护：定期执行 gitlab-ctl cleanup 清理冗余缓存

## 5.4 企业研发流程规范
采用 Git Flow 分支策略，按下表设置分支：
| 分支 / 标签 | 用途 | 规则 |
|---|---|---|
| `master` | 生产稳定分支 | 禁止直接提交，只能从 `release`、`hotfix` 合并进来 |
| `develop` | 测试/集成分支 | 所有 `feature` 合并到这里 |
| `feature/*` | 功能开发 | 从 `develop` 切出，开发完合回 `develop` |
| `release/*` | 发版候选分支 | 从 `develop` 切出，只修 bug、改版本号，不加新功能 |
| `hotfix/*` | 紧急修复 | 从 `master` 切出，修完合回 `master` 和 `develop` |
| `tag` | 正式版本快照 | 每次合入 `master` 后打 `v1.0.0` 这类标签 |

**具体流程**：
1. **开发阶段**：`feature/*` → 合并到 `develop`;
2. **测试环境准备发版**：从 `develop` 创建 `release/v1.0.0`。这个分支不再加新功能，只做测试、修 bug、改版本号;
3. **发布灰度或者生产环境**：`release/v1.0.0` 测试通过后，合并到 `master`，在 `master` 上打 `tag v1.0.0`，再合回 `develop`，保证修复同步回去;
4. **紧急修复**：线上出问题，从 `master` 创建 `hotfix/v1.0.1`，修完合并到 `master`，打 `tag v1.0.1`，再合回 `develop`。


> [!NOTE] 备注：
> 所有代码必须通过 MR 合并，必须经过至少 1 人评审 + 自动化流水线测试通过，全程留存操作日志，代码提交、合并、权限变更全追溯，满足企业合规要求。
