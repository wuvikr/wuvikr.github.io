+++
date = '2026-10-09T11:34:13+08:00'
draft = false
title = 'Kubespray 部署和维护 K8s 集群指南'
categories = ['Kubernetes']
tags = ['K8s', 'Kubespray','集群部署']
+++

# 1. 简介
Kubespray 是 Kubernetes SIGs 官方维护的部署项目，本质是一套经过大量生产环境验证的 Ansible Playbook、Inventory 模板和角色集合。它不发明新的集群模型，而是把 kubeadm 引导、etcd 集群组建、CNI 安装、证书轮换、节点扩缩容与升级这些容易出错的步骤固化为可重复执行的代码。

## 1.1 Kubespray 解决了什么问题

手动搭一个生产集群的工作量并不仅仅只是简单把 K8s 集群跑起来，而在于：
- etcd 节点集群的证书、成员管理、快照备份；
- 多控制平面节点下 apiserver 的负载均衡与前端证书 SAN；
- kubelet 配置（cgroup 驱动、预留资源、镜像垃圾回收、最大 Pod 数）；
- CNI 插件与内核网络参数、iptables/nftables 后端的匹配；
- 证书到期轮换、跨小版本顺序升级、节点下线清理。

这些工作做一次可以靠文档，做十次就得靠自动化。Kubespray 的价值就在于它把这些工程细节沉淀成了一套**符合生产环境最佳实践的可重复执行代码**，你只需要在默认值的基础上少量改动，而不是自己编写和维护大量的脚本。

## 1.2 和其他工具对比

| 工具           | Kubespray                                   | kubeadm                           | Rancher                     | OpenShift(OKD)            | kops                | KubeKey(kk)                                    | k3s                           |
| ------------ | ------------------------------------------- | --------------------------------- | --------------------------- | ------------------------- | ------------------- | ---------------------------------------------- | ----------------------------- |
| **核心定位**     | Ansible驱动，生产级离线部署K8s集群                      | K8s官方最小化集群引导工具                    | 企业级K8s全生命周期管理平台             | 红帽出品，基于K8s的企业容器平台         | AWS专用K8s集群部署工具      | KubeSphere社区开源部署工具，中文友好                        | Rancher出品轻量K8s发行版             |
| **底层引擎**     | Ansible                                     | Go原生二进制                           | 容器+WebUI                    | 容器/Operator               | Go，调用AWS API        | Go二进制，封装kubeadm                                | 裁剪K8s单二进制                     |
| **适用环境**     | 裸金属、VMware、私有云、离线环境                         | 裸金属/虚拟机/云主机，在线为主                  | 公有云、私有云、边缘，多集群统一管理          | 企业私有/混合云，偏向生产企业           | **仅限AWS**公有云        | 裸机/虚拟机，私有化离线场景                                 | 低配服务器、边缘、开发/小生产               |
| **离线部署**     | ✅ 完美支持，可打包离线介质                              | ⚠️ 支持但麻烦，需要手动下载镜像/二进制             | ✅ 支持离线airgap安装              | ✅ 官方提供离线方案                | ❌ 依赖AWS云API，不适合离线   | ✅原生支持打包离线包                                     | ✅离线包                          |
| **集群维护**     | Ansible playbook升级、扩容、替换节点；版本升级稳定           | 仅引导集群，**不负责后续集群运维**，升级操作繁琐        | Web界面一键升级、扩缩容，多集群统一监控告警     | Operator管理集群，自动化运维能力强     | 集群升级、节点替换由kops命令管理  | 一键HA、扩容升级，可配套KubeSphere控制台                     | 一行命令运维，嵌入式etcd HA             |
| **高可用**      | ✅ 内置etcd三副本、控制面HA方案                         | 需要手动搭建HA（负载均衡+多master）            | ✅ 自动支持HA，内置负载均衡方案           | ✅ 默认HA架构                  | ✅ AWS上自动搭建控制面HA     | ✅一键HA，内置负载均衡                                   | ✅内置HA（嵌入式etcd）                |
| **网络插件**     | 内置Calico/Flannel/Cilium等多种CNI可选             | 仅提供基础CNI对接，需自行部署CNI               | 内置多种CNI，UI可视化配置             | 默认OVN-Kubernetes，也可Calico | 默认Calico，可自定义CNI    | Calico等CNI可选                                   | Flannel默认，支持替换CNI             |
| **存储支持**     | 对接本地卷、Ceph、Longhorn等，ansible角色集成            | 仅提供CSI标准，存储需要自己部署                 | 内置存储管理，支持Longhorn等          | 内置CSI+本地存储方案              | 对接AWS EBS等云存储       | 对接Longhorn、Ceph、本地存储                           | 支持本地存储、CSI                    |
| **版本适配**     | 紧跟K8s社区版本，滞后一小段时间                           | K8s官方同步发布                         | 支持多K8s版本，统一纳管不同版本集群         | OKD版本独立，和上游K8s有差异         | 紧跟AWS支持的K8s版本       | 跟随上游kubeadm                                    | Rancher维护稳定版本                 |
| **学习成本**     | 中等，需要懂Ansible                               | 低，上手简单，适合测试集群                     | 低，WebUI友好，运维门槛低             | 高，体系庞大，有专属概念              | 中等，熟悉AWS即可快速上手      | 中等，中文文档友好                                      | 很低，上手极快                       |
| **典型场景**     | 裸金属机房、私有化项目、需要批量部署大量集群、等保离线环境               | 实验室、POC测试、小规模简单集群                 | 多集群管理、混合云、需要统一权限/审计的企业      | 大型企业生产环境，需要完整容器安全、合规能力    | AWS云上快速搭建销毁K8s集群    | 中小企业裸机/VM离线部署，可选KubeSphere控制台                  | 资源有限、边缘、小生产集群                 |
| **缺点**       | Ansible串行执行，大规模集群部署慢；无WebUI；故障排查需要Ansible基础 | 没有集群生命周期管理，生产HA搭建工作量大；缺少监控日志等附加组件 | 资源开销大；底层K8s被封装，深度定制复杂；商业版收费 | 闭源企业版昂贵；架构重，资源消耗高；学习曲线陡峭  | 只能跑在AWS，无法裸金属/其他云使用 | 依赖kubeadm；底层封装，深度排障需要kubeadm知识；KubeSphere额外耗资源 | 裁剪部分组件，重型大规模生产不推荐；和标准K8s有细微差异 |
| **国内中小企业热度** | ⭐⭐⭐ 更多集成商使用，中小企业自用偏少                        | ⭐⭐⭐⭐⭐ 最高，几乎所有运维都学这个               | ⭐⭐⭐⭐ 多集群场景热门                | ⭐⭐ 大型企业为主，中小企业很少用         | ⭐ 国内几乎不用            | ⭐⭐⭐⭐ 私有化中小企业热门                                 | ⭐⭐⭐⭐ 小集群/边缘场景热门               |


# 2. 底层架构和设计
**关键核心设计**：
- **无 Agent**：底层由 Ansible 实现，因此只依赖 SSH 和目标节点的 Python 3，对目标机侵入极小。
- **分层变量**：`defaults/main.yml < group_vars/all/all.yml < group_vars/k8s_cluster/*.yml < inventory 主机变量`。可以选择性的修改你需要修改的选项，其余保持上游默认值，这样后续升级合并代码时冲突最少。
- **幂等性**：同一个 cluster.yml 可以反复执行（用于修复、追加配置），这是它区别于普通脚本的核心优势。
- **生命周期全覆盖**：除了部署，还提供扩容（scale.yml）、升级（upgrade-cluster.yml，注意只能逐小版本升）、重置（reset.yml）和控制面恢复（recover-control-plane.yml）等功能。

## 2.1 整体架构
Kubespray 架构整体分为三层：**Ansible 控制层（控制节点）** → **目标主机层（被管节点）** → **Kubernetes 集群组件层**
```plaintext
控制节点（Ansible runner）
└── Kubespray：Inventory + Group_vars + Playbooks + Roles
        ↓ SSH/密码
目标主机集群节点
├─ Master 节点（控制面）
└─ Worker 节点（业务面）
        ↓ kubeadm 执行
K8s 组件层：etcd / kube-apiserver / controller-manager / scheduler / kubelet 等
```

### 2.1.1 Ansible 控制层（控制节点）

- **Ansible 主机**：运行 Ansible，执行 kubespray 的 playbook；
- **Inventory（主机清单）**：定义所有节点、分组（kube_control_plane、kube_node、etcd）；
- **group_vars**：全局变量配置，如 K8s 版本、镜像仓库、CNI、离线镜像、组件参数；
- **Roles**：按功能拆分的 Ansible 角色，是 Kubespray 的核心单元。

> [!TIP] 提示：
> 控制节点**可以不加入 K8s 集群**，只需要能 ssh 连通所有目标机器。

### 2.1.2 目标主机层（被管理节点）
通过 Ansible SSH 远程在目标机器执行任务，节点分为 3 组：
1. **kube_control_plane**：控制面节点（master）
2. **kube_node**：业务工作节点（worker）
3. **etcd**：etcd 集群节点，默认和控制面节点复用，也可以独立部署。

### 2.1.3 Kubernetes 组件层
在目标节点部署 K8s 全套组件，核心组件有 etcd、kube-apiserver、controller-manager、scheduler、kube-proxy、kubelet；容器运行时默认 `containerd`，网络插件默认 `Calico`。

## 2.2 核心 Role 执行流程

Kubespray 入口文件：`cluster.yml`，按顺序调用 Role，简化执行链路：

1. **bootstrap-os**：操作系统初始化
   - 关闭防火墙和selinux、设置主机名；
   - 加载内核模块（overlay, br_netfilter）；
   - 配置 sysctl 参数（ip 转发、iptables bridge）；
   - 安装基础依赖包。
2. **container-engine**：安装容器运行时 containerd
   - 二进制安装 containerd、runc；
   - 配置镜像仓库、离线镜像。
3. **kube-prepare**：K8s 前置准备
   - 下载 kubeadm、kubelet、kubectl 二进制；
   - 配置 kubelet 系统服务。
4. **etcd**：部署 etcd 集群
   - 生成 etcd 证书；
   - 二进制部署 etcd（**二进制部署**）。
5. **kube-master**：调用 kubeadm 初始化控制面
   - 生成所有 K8s CA 证书；
   - `kubeadm init` 初始化第一个控制面节点；
   - `kubeadm join` 把其余 master 节点加入控制面；
   - 部署 kube-apiserver、controller-manager、scheduler 静态 Pod。
6. **kube-node**：worker 节点加入集群
   - 执行 `kubeadm join` 将 worker 节点接入集群。
7. **network_plugin**：部署 CNI 网络插件（默认为 Calico ）
8. **addons**：部署附加组件
   - CoreDNS、Metrics Server 等。


# 3. 生产部署
了解了基本知识和原理后，接下来会演示如何使用 kubespray 部署一套生产级高可用 K8s 集群。
> [!NOTE] 备注：
> 本案例基于 Ubuntu24.04，python3.12，K8s v1.35，containerd 容器运行时，3 个 master + 2 个 worker，使用**内置 etcd 集群**（etcd 和 master共置），网络插件选用默认的 Calico；控制平面高可用由额外组件 kube-vip 实现。

**参考文档**：
- [Kubespray GitHub 项目地址](https://github.com/kubernetes-sigs/kubespray)
- [官方文档](https://kubespray.io/#/)
- [Ansible官方文档](https://docs.ansible.com/)


## 3.1 环境架构
| 主机名                 | IP             | 角色                       |
| ------------------- | -------------- | ------------------------ |
| master01.wuvikr.top | 192.168.69.101 | master01、etcd、kube‑vip节点 |
| master02.wuvikr.top | 192.168.69.102 | master02、etcd、kube‑vip节点 |
| master03.wuvikr.top | 192.168.69.103 | master03、etcd、kube‑vip节点 |
| worker01.wuvikr.top | 192.168.69.121 | worker01                 |
| worker02.wuvikr.top | 192.168.69.122 | worker02                 |
| 虚拟VIP               | 192.168.69.149 | kube-vip                 |

> [!NOTE] 备注
> 部署机器可以使用 master01 作为 Ansible 控制节点，控制节点也可以独立一台机器；**控制节点必须能够 ssh 免密登录所有集群节点**。

## 3.2 前置工作
### 3.2.1 配置 SSH 免密登录
仅在 kubespray 控制节点操作即可。
```bash
# 创建密钥对
ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519 -N "" -q

# 循环分发公钥到所有节点
nodes=(192.168.69.101 192.168.69.102 192.168.69.103 192.168.69.121 192.168.69.122)
for ip in ${nodes[@]};do
  ssh-copy-id root@$ip
done

# 测试免密连通
ssh root@192.168.69.102 hostname
```
> [!CAUTION] 注意：
> 所有节点需要允许 root 用户ssh登录；如果生产环境禁止 root 登录，可以使用普通用户 + sudo。

### 3.2.2 准备 kubespray 环境
在开始前，需要先确认 kubespray 和 K8s 的版本对应关系，在 README.md 页面的 [supported-components](https://github.com/kubernetes-sigs/kubespray#supported-components) 部分可以找到当前 release 版本所支持的 K8s 版本和相关组件信息。

本次案例计划部署 K8s v1.35 版本，需要切换到`v2.31.0`分支。

**拉取 Kubespray 源码**：
```bash
git clone https://github.com/kubernetes-sigs/kubespray.git

// 无法访问 github 可以使用下面的地址加速
git clone https://gh-proxy.org/https://github.com/kubernetes-sigs/kubespray.git

# 切换分支
cd kubespray
git checkout release-2.31
```
> [!TIP] 提示
> 操作均基于目录`kubespray`和分支`release-2.31`进行。

**创建虚拟环境和安装 Ansible**：

```bash
# python 虚拟环境管理工具，uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 安装好根据提示加载环境变量，或者重新打开shell
source ~/.local/bin/env
# 验证安装成功
uv --version

# 创建和激活虚拟环境
cd ~/kubespray
uv venv .kubespray-2.31
source .kubespray-2.31/bin/activate

# 安装依赖
uv pip install -U pip
uv pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

# 验证 ansible 版本
ansible --version
```


### 3.2.3 配置 inventory 清单与集群变量
- **inventory 配置**：kubespray 提供 inventory 模板，复制一份用于创建我们自定义的 K8s 集群：
```bash
cp -r inventory/sample inventory/my-cluster
```
接着修改节点清单文件，编辑`inventory/my-cluster/inventory.ini`，填入集群节点信息
```ini
[all]
master01.wuvikr.top ansible_host=192.168.69.101 ip=192.168.69.101 etcd_member_name=etcd1
master02.wuvikr.top ansible_host=192.168.69.102 ip=192.168.69.102 etcd_member_name=etcd2
master03.wuvikr.top ansible_host=192.168.69.103 ip=192.168.69.103 etcd_member_name=etcd3
worker01.wuvikr.top ansible_host=192.168.69.121 ip=192.168.69.121
worker02.wuvikr.top ansible_host=192.168.69.122 ip=192.168.69.122

# 控制面 Master 节点
[kube_control_plane]
master01.wuvikr.top
master02.wuvikr.top
master03.wuvikr.top

# etcd 集群
[etcd]
master01.wuvikr.top
master02.wuvikr.top
master03.wuvikr.top

# 负载均衡（默认使用内置 haproxy）
[kube_lb]
master01.wuvikr.top
master02.wuvikr.top
master03.wuvikr.top

# 工作节点
[kube_node]
master01.wuvikr.top
master02.wuvikr.top
master03.wuvikr.top
worker01.wuvikr.top
worker02.wuvikr.top

# 可选：Calico BGP 路由反射器
[calico_rr]


[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```

- **集群变量配置**： 编辑 `inventory/my‑cluster/group_vars/k8s_cluster/k8s‑cluster.yml` 文件，移动到文件最下面，添加一下内容。
```yaml
...

# ---------- 版本 ----------
kube_version: 1.35.9

# kube-vip要求必须开启严格arp
kube_proxy_strict_arp: true

# ---------- 证书相关配置 ----------
kube_cert_validity_period: 438000h
kube_ca_cert_validity_period: 876000h

kube_kubeadm_controller_extra_args:
  cluster-signing-duration: 175200h

# containerd 镜像源代理
containerd_registries_mirrors:
  - prefix: docker.io
    mirrors:
      - host: https://docker.m.daocloud.io
        capabilities: ["pull", "resolve"]
  - prefix: registry.k8s.io
    mirrors:
      - host: https://k8s.m.daocloud.io
        capabilities: ["pull", "resolve"]
  - prefix: quay.io
    mirrors:
      - host: https://quay.m.daocloud.io
        capabilities: ["pull", "resolve"]
  - prefix: gcr.io
    mirrors:
      - host: https://gcr.m.daocloud.io
        capabilities: ["pull", "resolve"]
  - prefix: ghcr.io
    mirrors:
      - host: https://ghcr.m.daocloud.io
        capabilities: ["pull", "resolve"]



# ---------- 一些默认，但需要关注的参数 ----------
# cluster_name: cluster.local
# dns_mode: coredns
# enable_nodelocaldns: true
# resolvconf_mode: host_resolvconf
# container_manager: containerd

# kube_network_plugin: calico
# kube_service_addresses: 10.233.0.0/18
# kube_pods_subnet: 10.233.64.0/18
# kube_network_plugin_multus: false

# kube_proxy_mode: ipvs
```
另外还有一些重点参数例如：网络插件、pod/service 网段，容器运行时，kubeproxy 模式等，可以按需自定义修改。

> [!NOTE] 备注：
> kubespray 在`roles/kubespray_defaults/vars/main/checksums.yml`文件中定义了当前版本分支能支持的所有 K8s 版本的 checksum 信息。它会在下载安装时对二进制文件做比对和校验，因此，**在设置 kube_version 时必须用 kubespray 内置支持的具体 patch 版本**。比如你设置 1.35.6，kubespray 会去找 kube_checksums["1.35.6"]。如果你写了一个它没有记录的 patch 版本，哪怕 upstream 确实发布了，kubespray 也可能下载失败。

### 3.2.4 附加组件配置
kubespray 还支持集群附加组件的配置和安装，这里我们重点是要部署 kube-vip，其他组件可以按需部署。编辑`inventory/my-cluster/group_vars/k8s_cluster/addons.yml` 文件：
```yaml
# Helm deployment
helm_enabled: true

# Metrics Server
metrics_server_enabled: true

# Kube VIP
kube_vip_enabled: true
kube_vip_arp_enabled: true
kube_vip_controlplane_enabled: true
kube_vip_address: 192.168.69.149
loadbalancer_apiserver:
  address: "{{ kube_vip_address }}"
  port: 6443
kube_vip_interface: enp2s0
# kube_vip_services_enabled: false
# kube_vip_dns_mode: first
# kube_vip_cp_detect: false
# kube_vip_leasename: plndr-cp-lock
# kube_vip_enable_node_labeling: false
# kube_vip_lb_fwdmethod: local
# kube_vip_bgp_sourceip:
# kube_vip_bgp_sourceif:


# argocd
argocd_enabled: true
argocd_namespace: argocd
```

### 3.2.5 其他配置
除了集群配置，还有一些节点级别的全局配置需要设置，例如额外的内核参数配置，节点的 python 版本，镜像源配置等。

编辑 `inventory/my-cluster/group_vars/all/all.yml` 文件，修改或追加一下内容：
```yaml
...

# 加载内核模块（精简内核/云镜像可能默认不加载）
# kernel_modules:
#   - br_netfilter
#   - overlay
#   - nf_conntrack
#   - ip_tables
#   - iptable_nat
#   - iptable_filter

# 内核参数优化
additional_sysctl:
  # K8s 网络必需
  - { name: "net.bridge.bridge-nf-call-iptables", value: "1" }
  - { name: "net.bridge.bridge-nf-call-ip6tables", value: "1" }
  - { name: "net.ipv4.ip_forward", value: "1" }

  # 连接跟踪
  - { name: "net.netfilter.nf_conntrack_max", value: "2097152" }
  - { name: "net.netfilter.nf_conntrack_tcp_timeout_established", value: "3600" }
  - { name: "net.netfilter.nf_conntrack_tcp_timeout_time_wait", value: "60" }
  - { name: "net.netfilter.nf_conntrack_tcp_timeout_close_wait", value: "60" }

  # 网络队列 / 并发
  - { name: "net.core.somaxconn", value: "32768" }
  - { name: "net.core.netdev_max_backlog", value: "16384" }
  - { name: "net.ipv4.tcp_max_syn_backlog", value: "16384" }
  - { name: "net.ipv4.tcp_tw_reuse", value: "1" }
  - { name: "net.ipv4.tcp_fin_timeout", value: "30" }
  - { name: "net.ipv4.ip_local_port_range", value: "1024 65535" }

  # 缓冲区：通用节点先别拉到 128MB
  - { name: "net.core.rmem_max", value: "16777216" }
  - { name: "net.core.wmem_max", value: "16777216" }
  - { name: "net.core.rmem_default", value: "262144" }
  - { name: "net.core.wmem_default", value: "262144" }
  - { name: "net.ipv4.tcp_rmem", value: "4096 87380 16777216" }
  - { name: "net.ipv4.tcp_wmem", value: "4096 65536 16777216" }

  # 系统资源
  - { name: "vm.swappiness", value: "0" }
  - { name: "vm.max_map_count", value: "262144" }
  - { name: "fs.file-max", value: "2097152" }
  - { name: "fs.inotify.max_user_watches", value: "524288" }
  - { name: "fs.inotify.max_user_instances", value: "8192" }
  - { name: "fs.pipe-max-size", value: "4194304" }

# 固定 Python 解释器，关闭自动探测警告
ansible_python_interpreter: /usr/bin/python3.12
# 允许重复创建用户/组
adduser_ignore_existing: true

# 部署机只下载一次，然后分发到各节点（适合节点多、带宽有限的场景）
# download_run_once: true

# 是否在本地下载
# download_localhost: false
```

- **配置 DaoCloud 公共加速源**：kubespray 下载二进制文件和镜像默认都是国外源站，国内网络环下可能无法访问或者超时，编辑 `inventory/my-cluster/group_vars/all/offline.yml`，使用 DaoCloud 源替代。
```yaml
# 1. 二进制文件加速：files.m.daocloud.io 同时代理 dl.k8s.io 和 github.com 的 release 下载
files_repo: "https://files.m.daocloud.io"

# 个别服务硬编码了从github下载，单独处理一下
calico_crds_download_url: "{{ files_repo }}/raw.githubusercontent.com/projectcalico/calico/v{{ calico_version }}/manifests/crds.yaml"
argocd_install_url: "{{ files_repo }}/raw.githubusercontent.com/argoproj/argo-cd/v{{ argocd_version }}/manifests/install.yaml"


# 2. 容器镜像仓库替换
kube_image_repo: "k8s.m.daocloud.io"       # registry.k8s.io 的镜像
gcr_image_repo: "gcr.m.daocloud.io"        # gcr.io
github_image_repo: "ghcr.m.daocloud.io"    # ghcr.io
docker_image_repo: "docker.m.daocloud.io"  # docker.io
quay_image_repo: "quay.m.daocloud.io"      # quay.io（calico 等网络插件在这里）

```

## 3.3 执行 Playbook

### 3.3.1 预检查，校验连通性
执行 playbook 前先测试一下所有节点是否都能正常连通。
```bash
ansible -i inventory/my-cluster/inventory.ini all -m ping
```
输出 `SUCCESS` 代表所有节点通信正常。

### 3.3.2 正式执行集群安装
```bash
ansible-playbook -i inventory/my-cluster/inventory.ini cluster.yml -b
```
- `-b`：使用sudo提权，如果使用root账号也必须带上
> [!TIP] 提示：
> 部署时间取决于机器性能与网络；如果中间网络中断，可以重复执行 playbook，幂等设计不会重复创建资源。

### 3.3.3 部署报错解决和重置集群
如果部署报错，可以询问 AI 解决，一般由于 Ansible 幂等性特点，解决问题后重新跑 playbook 即可。

如果是测试集群功能，验证完后，需要清空集群重新部署可以执行重置集群的 playbook。
```bash
ansible-playbook -i inventory/my-cluster/inventory.ini reset.yml -b
```
此操作会清空 k8s 组件、容器、数据，etcd数据也会清空；重置完成后，再重新执行 cluster.yml 部署。

### 3.3.4 部署完成后配置 kubeconfig
部署完成后，**控制平面节点会自动生成 admin.conf**。
```bash
# 复制 admin.conf 到默认目录
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

## 3.5. 集群可用性验证

### 3.5.1 基础验证
```bash
# 1.查看全部节点状态
kubectl get nodes -o wide

# 2.查看系统 pod 状态
kubectl get pods -n kube-system -o wide

# 3.apiserver 健康检查
curl -sk https://192.168.69.149:6443/livez;echo
curl -sk https://192.168.69.149:6443/healthz;echo

# 4.简单调度测试
kubectl create deployment nginx-test --image=nginx:alpine --replicas=3
kubectl get pods -l app=nginx-test -o wide
```
> [!TIP] 提示：
刚部署完节点状态可能会短暂 NotReady，等待 calico 全部 pod 启动完成后，节点变为Ready。

### 3.5.2 验证 kube-vip
```bash
# 1. kube-vip 静态 Pod 是否在三台 master 上都 Running
kubectl -n kube-system get pods -o wide | grep kube-vip

# 2. VIP 149 是否真的漂到了其中一台 master
# 在三台 master 上分别执行，应只有一台能看到 149
ip addr show | grep 192.168.69.149

# 3. kubeconfig 入口是否已指向 VIP
kubectl config view | grep server

# 4. 通过 VIP 直接访问 apiserver
curl -k --max-time 5 https://192.168.69.149:6443/healthz
```

### 3.5.3 调度 + dns + 网络测试
起一个最简单的 Pod，看能不能调度、有 IP、能 ping集群内网：
```bash
# 1. 部署测试 Deployment
# 期望：3 个 Pod Running，分布在不同节点，每个有自己的 Pod IP
kubectl create deployment nginx-test --image=nginx:alpine --replicas=3
kubectl get pods -l app=nginx-test -o wide

# 2. dns 解析测试
# 期望，正确解析出其 svc 的 ip 地址
POD=$(kubectl get pod -l app=nginx-test -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- getent hosts kubernetes.default
kubectl exec $POD -- nslookup coredns.kube-system.svc.cluster.local

# 3. Pod 内部连通性测试
# 期望：能看到 eth0 IP、能 ping 通外网、能拿到 apiserver 返回的 version JSON。
kubectl exec $POD -- ip addr show eth0
kubectl exec $POD -- ping -c2 223.5.5.5
kubectl exec $POD -- curl -sk https://kubernetes.default/version
```

### 3.5.4 故障切换演练
先找到当前 VIP 落在哪台 master，假设是 master01，然后故意让它失联，看集群是否继续可用：
```bash
# 1. 在 VIP 所在的 master 上，临时关机，模拟该节点宕机
poweroff

# 2. 在另外两台机器上观察 VIP，应在几秒到几十秒内漂到 master02 或 master03
ip addr show | grep 192.168.69.149

# 3. kubectl 仍然能用（访问入口没断）
kubectl get nodes

# 4. 恢复：把关机的节点重新开机
```
这一步验证能过，才说明 **3 控制面 + kube-vip VIP 高可用架构是真正成立的，是生产级可用的，不是仅纸面上配置正确的**。

# 4. 集群常用运维命令
## 4.1 新增节点
修改`inventory.ini`，添加新节点，执行 scale‑nodes.yml：
```bash
ansible-playbook -i inventory/my-cluster/inventory.ini scale-nodes.yml -b
```

## 4.2 K8s 版本升级
kubespray 每个版本都带一份锁死的组件版本矩阵，文件位于`roles/kubespray_defaults/vars/main/checksums.yml`。进行版本升级，**首先要确认目标版本是否存在于该文件中**。里面的版本都是经过社区兼容性验证过的，**不推荐手动在文件中添加不存在的版本 checksum**，可能会出现未知问题。

> [!TIP] 提示：
> 进行版本升级前，务必仔细阅读官方撰写的升级文档，文件路径：`docs/operations/upgrades.md`

### 4.2.1 小版本升级

小版本升级，直接修改`k8s‑cluster.yml`的`kube_version`，然后执行 upgrade.yml 即可。
```bash
ansible-playbook -i inventory/my-cluster/inventory.ini upgrade-cluster.yml -b
```
### 4.2.2 大版本升级
大版本升级需要考虑的点就比较多了，首先 K8s 官方明确不支持跨多个大版本升级，kubespray 也一样，只能一个一个大版本进行升级。

一般一个 release 版本对应一个 K8S 版本。因此升级 K8s 前，首先要切换到新的 release 版本。不同 release 使用的 Ansible 版本和依赖也可能不一样，对条件表达式和模板语法也可能有 breaking changes，官方明确要求升级前先阅读 [Urgent Upgrade Notes](https://github.com/kubernetes-sigs/kubespray/releases)。

```bash
# 切换 release tag
git checkout v2.32.0
git describe --tags

# 重新创建虚拟 python 环境和安装ansible 依赖
uv venv .kubespray-2.32
source .kubespray-2.32/bin/activate
uv pip install -U pip
uv pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

# 验证 ansible 版本
ansible --version

# 配置好 inventory 和集群配置
# 确认无误后，在测试环境进行升级和验证

# 最后生产环境执行升级操作
ansible-playbook -i inventory/my-cluster/inventory.ini upgrade-cluster.yml -b
```

# 5. 参考链接
- [HA 模式（多 master + 负载均衡）](https://kubespray.io/#/docs/operations/ha-mode)：控制面/etcd/LB 高可用，官方定义和方案；
- [安全加固 hardeing](https://kubespray.io/#/docs/operations/hardening)：CIS 基准、审计日志、TLS、RBAC 等生产加固配置；
- [containerd 配置](https://kubespray.io/#/docs/CRI/containerd)：containerd 相关配置；
- [kube-vip](https://kubespray.io/#/docs/ingress/kube-vip)：kube-vip 高可用 VIP 方案官方文档；
- [下载机制](https://kubespray.io/#/docs/advanced/downloads)：download_run_once、离线、mirror 等下载相关变量；
- [集群升级](https://github.com/kubernetes-sigs/kubespray/blob/v2.32.0/docs/operations/upgrades.md)：升级 K8s 版本和组件需要注意的事项和方案。

