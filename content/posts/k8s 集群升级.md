+++
date = '2026-09-09T14:26:14+08:00'
draft = false
title = 'K8s 集群升级'
categories = ['Kubernetes']
tags = ['K8S', '集群升级']
+++

Kubernetes 集群的大版本（跨次要版本）升级是一项复杂且高风险的运维操作。为了确保业务连续性，必须遵循严格的规范。以下是升级的核心原则、标准流程以及可能遇到的关键问题。

# 1. 大版本升级的核心原则

1. **逐级递进，严禁跨版本**：Kubernetes 不支持跨多个次要版本直接升级（例如不能从 1.24 直接升级到 1.26），必须按照`1.24 → 1.25 → 1.26`的顺序逐版本进行。每一跳都需要完成控制面和节点的升级与验证。
2. **组件版本兼容策略**：
   - **控制面**：`kube-apiserver` 版本最高且需最先升级，`controller-manager` 和 `scheduler` 版本应小于或等于 `apiserver`。
   - **kubelet**：节点上的 `kubelet` 版本必须小于或等于 `apiserver`，且不能低于 `apiserver` 的前一个次要版本（例如 apiserver 为 v1.28 时，kubelet 可以是 v1.28 或 v1.27）。
3. **先升级控制面，再升级节点**：必须先完成 Master 节点（控制平面）的升级，确认稳定后再分批升级 Worker 节点。
4. **第三方组件兼容性检查**：升级前必须确认 CNI 插件（如 Calico/Cilium）、CoreDNS、kube-proxy 以及 Ingress 控制器等是否支持目标版本。通常建议先升级 CNI，CoreDNS 则与 K8s 版本强绑定，需查阅官方对应表。

# 2. 标准升级流程

1. **升级前准备**：
   - 检查集群健康状态，确保所有节点为 Ready，无 CrashLoopBackOff 的 Pod。
   - **备份关键数据**：必须备份 etcd 数据、证书文件及核心配置文件，以防升级失败导致数据丢失。
   - 检查并处理废弃 API（Deprecated APIs），高版本可能会移除旧版 API，导致 HPA 或 Ingress 等资源失效。
2. **升级控制面**：升级 `kubeadm`，执行 `kubeadm upgrade apply`，依次更新 `kube-apiserver`、`kube-controller-manager` 和 `kube-scheduler`。
3. **升级工作节点**：
   - 腾空节点（Cordon & Drain），将 Pod 驱逐到其他节点。
   - 升级节点的 `kubeadm`、`kubelet` 和 `kubectl`。
   - 解除节点保护（Uncordon），使其重新加入调度。
   - 建议在生产环境中逐个节点离线升级后再上线，避免全部同时升级。

# 3. 升级过程中可能遇到的问题

1. **废弃 API 导致业务失效**：高版本 K8s 会废弃旧版 API。例如升级到 1.22+ 时，`networking.k8s.io/v1beta1` 的 Ingress 将不再可用；升级到 1.18+ 后，HPA 的 `apiVersion` 必须切换为 `apps/v1`，否则功能会失效。
2. **容器运行时变更**：从 Kubernetes 1.24 开始，不再支持将 Docker 作为内置容器运行时。若从 1.22/1.23 升级，必须提前将节点运行时从 Docker 迁移到 containerd，否则升级会失败。
3. **节点排水（Drain）失败与 PDB 冲突**：在驱逐 Pod 时，如果配置了 Pod Disruption Budget (PDB) 且当前可用副本数达到下限，Pod 将无法被驱逐，导致节点升级超时或失败。
4. **网络连通性中断**：如果 Pod 通过 `LoadBalancer` 类型的 Service 访问同节点上的另一个 Pod，且 Service 的 `externalTrafficPolicy` 设置为 `Local`，在节点轮转升级后，两个 Pod 可能不再位于同一节点，导致网络不通。
5. **自定义配置被覆盖**：如果曾通过非产品化方式（如直接登录节点修改 kubelet 配置、打开 SWAP 分区等）更改过节点配置，升级过程中这些自定义配置可能会被重置或覆盖，甚至导致升级失败。
6. **磁盘空间不足**：升级过程需要下载软件包并可能产生临时文件。如果节点磁盘水位过高（建议预留至少 20% 空间），可能导致 Pod 被驱逐或升级失败。
7. **Cgroup v1 兼容性问题**：在较新的版本（如 1.36）中，kubelet 默认仅支持 Cgroup v2。如果节点操作系统仍在使用 Cgroup v1，可能会导致 kubelet 启动失败或节点 NotReady。

