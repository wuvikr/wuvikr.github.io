+++
date = '2026-09-03T12:37:52+08:00'
draft = false
title = 'Terrafrom 基础篇'
+++

# 1. 简介

[Terraform](https://developer.hashicorp.com/terraform/intro) 是一款**开源的“基础设施即代码（Infrastructure as Code，IaC）”工具**，由 HashiCorp 公司开发（创始人为 Mitchell Hashimoto）。它的核心作用是用人类可读的配置文件来安全、高效地构建、变更和版本化管理云上和本地的各类基础设施。

它的主要特点包括：

- **基础设施即代码**：用高阶配置语法描述基础设施，便于版本控制、复用和共享。
- **声明式语言**：用户只需描述期望的基础设施最终状态，工具自动处理依赖关系和部署顺序；配置语言使用 HCL（也支持 JSON）。
- **执行计划**：在真正执行前会生成操作计划，避免意外变更。
- **资源图**：构建资源依赖图并并行处理相互独立的资源。
- **多云支持**：可与 AWS、Azure、Google Cloud、阿里云等主流云平台集成。

典型的工作流程包括初始化（`terraform init`）、生成计划（`terraform plan`）和应用（`terraform apply`）三个阶段。Terraform 侧重于定义和管理基础设施结构，常与 Ansible 等侧重于配置管理的工具搭配使用。

# 2. 核心概念

## 2.1. 基础架构即代码 (IaC)

Terraform 允许使用配置文件（而非手动操作或脚本）来定义和管理基础设施。这使基础设施可以像应用程序代码一样进行版本控制、测试和协作。

## 2.2  HCL

HCL 全称 HashiCorp Configuration Language，是 Terraform 使用的一种特定于域的专用语言，专为基础设施管理设计：

- 语法简单易学，比 JSON 和 ARM 模板更直观
- 支持变量、条件表达式和函数
- 提供类型安全和 IntelliSense 支持（在 VS Code 等编辑器中）

## 2.3. Providers

[Provider](https://registry.terraform.io/browse/providers) 是云厂商提供并维护的一个能与云厂商 API 进行交互的插件，其中包含相关资源和数据源对象。Terraform 可以通过 Provider 来管理基础设施。常用的 Providers 有：

- AzureRM：管理 Azure 资源
- AWS：管理 AWS 服务
- Kind：使用 Kind 管理 kubernetes 集群
- Docker：管理容器资源
- 其他支持的云平台和 SaaS 服务

## 2.4. 状态文件 (State File)

Terraform 会跟踪基础设施的当前状态，包括，组件配置，资源之间的依赖关系等，通过比对配置文件与状态文件，确定需要实施哪些变更。

# 3. 安装 Terraform

详细安装步骤参考[官方文档](https://developer.hashicorp.com/terraform/install)

# 4. 目录结构

Terrafrom 的配置文件都是以`.tf`为后缀，支持 HCL 和 JSON 两种格式。当前目录下，所有以`tf`结尾的文件最终都会被识别并加载。一个最简单的 Terraform 目录结构如下：

```bash
project/
├── main.tf            # 主要资源定义
├── variables.tf       # 变量声明
├── outputs.tf         # 输出变量
├── providers.tf       # 提供程序配置
├── .gitignore
├── README.md
└── terraform.tfstate  # 状态文件（实际部署中应使用远程状态）
```

**文件说明**：

- `main.tf`：定义核心资源（如AWS EC2实例、S3存储桶等）
- `variables.tf`：定义所有输入变量（如region、环境等）
- `outputs.tf`：定义需要暴露给外部的资源属性
- `providers.tf`：指定提供程序版本和配置
- `terraform.tfstate`：描述 Terraform 所管理的资源状态的文件**（重要！生产使用中需要保密存储和备份）**

## 3.1. Provider

用户想要操作某个云厂商的资源，必须先在 terraform 模块中声明，并配置 provider 的相应参数：

```hcl
terraform {
  required_version = "1.5.6"

  required_providers {
    alicloud = {
      source  = "aliyun/alicloud"
      version = "1.209.1"
    }
  }
}

provider "alicloud" {
  access_key = var.alicloud_access_key
  secret_key = var.alicloud_secret_key
  region     = var.region
}
```

## 3.2. Resource

- 资源由 Provider 提供，每个资源语法块描述了一个或多个基础设施对象，例如网络，计算实例，安全组等等。
- 定义格式为：`resource "RESOURCE TYPE" "NAME" { content... }`
- 资源名称必须以字母和下划线开头，只能包含数字，字母，下划线和破折号。
- 资源参数之间可以相互引用参数，格式：`<RESOURCE TYPE>.<NAME>.<ATTRIBUTE>`

```hcl
resource "alicloud_vpc" "vpc01" {
  vpc_name   = "terraform-example"
  cidr_block = "172.16.0.0/12"
}

resource "alicloud_vswitch" "vsw01" {
  vswitch_name = "terraform-example"
  cidr_block   = "172.16.0.0/24"
  vpc_id       = alicloud_vpc.vpc01.id
  zone_id      = "cn-shanghai-e"
}
```

## 3.3. 变量

terraform 支持自定义环境变量：

```hcl
variable "alicloud_region" {
  description = "阿里云 region"
  type        = string
  default     = "cn-hangzhou"
}

variable "vpc_cidr" {
  description = "VPC 网段"
  type        = string
  default     = "192.168.0.0/16"
}
```

除了直接在文件中定义默认值，terraform 支持从系统环境变量中来读取变量值，所有以 TF_VAR 开头的环境变量都会被 terraform 当做变量的值来使用，例如：

```bash
export TF_VAR_vpc_cidr="172.16.0.0/16"
export TF_VAR_alicloud_region="cn-shanghai"
```

# 4. providers 凭证

选用不同云厂商的 providers 就可以调用云上所有的 api 资源，那么云厂商如何识别身份呢？

Terraform 默认会按以下优先级顺序查找凭证，一旦找到就会停止，这里以阿里云为例：

1. **环境变量**：检查对应 providers 的环境变量， `ALICLOUD_ACCESS_KEY` 和 `ALICLOUD_SECRET_KEY`。
2. **Provider 配置块**：检查 `provider aliyun` 代码块中是否直接配置了 `access_key` 和 `secret_key`。
3. **共享凭证文件**：检查 `~/.aliyun/config.json` 文件，可以使用`aliyun configure`命令进行配置。
4. **实例 RAM 角色**：如果在阿里云 ECS 实例上运行，会尝试获取实例绑定的 RAM 角色凭证。

在生产环境中，推荐使用环境变量和配置共享凭证的方式来运行 terraform。

# 5. 常用命令

| 命令                     | 功能说明    | 常见使用场景与备注                                           |
|:----------------------:|:-------:|:---------------------------------------------------:|
| `terraform init`       | 初始化工作目录 | 项目启动时执行，下载并安装所需的 Provider 插件和模块，配置后端存储              |
| `terraform fmt`        | 格式化代码   | 扫描并重写配置文件为统一的规范格式，提升代码可读性与一致性。                      |
| `terraform validate`   | 校验配置语法  | 检查配置文件语法是否正确及内部逻辑是否一致，帮助在早期发现错误。                    |
| `terraform plan`       | 预览执行计划  | 分析配置并对比当前状态，展示将要新增(+)、修改(~)或删除(-)的资源，不会实际改变基础设施。    |
| `terraform apply`      | 应用配置变更  | 实际创建、修改或删除资源以匹配期望状态。执行时会提示确认，可通过`--auto-approve`跳过。 |
| `terraform destroy`    | 销毁基础设施  | 删除当前配置文件中定义的所有资源。操作不可逆，需谨慎使用，可通过`-target`销毁特定资源。    |
| `terraform show`       | 展示当前状态  | 详细展示当前 State 中所有被管理的资源及其属性值，支持`--json`格式输出。         |
| `terraform output`     | 打印输出变量  | 获取并展示配置文件中定义的 output 值（如生成的公网IP、数据库连接串等）。           |
| `terraform import`     | 导入存量资源  | 将云平台上已存在的资源导入到 Terraform State 中，纳入代码化管理体系。         |
| `terraform state list` | 列出状态资源  |                                                     |
| `terraform state show` | 查看特定资源  |                                                     |
| `terraform workspace`  | 管理工作区   | 用于在不同环境（如 `dev/staging/prod`）之间切换和管理独立的状态文件。        |
| `terraform refresh`    | 刷新状态信息  | 将 State 文件与真实世界的基础设施进行同步，捕获手动修改等外部变更，不改变实际资源。       |
| `terraform taint`      | 标记资源被污染 |                                                     |
| `terraform untaint`    | 取消污染标记  |                                                     |
