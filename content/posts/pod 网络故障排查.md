+++
date = '2026-09-12T15:51:54+08:00'
draft = false
title = 'Pod 网络故障排查'
categories = ['Kubernetes']
tags = ['K8S', 'Network', 'Troubleshooting']
+++

Kubernetes 的网络是一个多层虚拟网络体系，流量从客户端到后端 Pod 的路径涉及多个组件。排查网络问题时，如果"东戳一下西戳一下"没有章法，可能需要耗费大量时间。如何最快的定位网络问题的根源？记住常见的具体现象和应对办法，遵循**从内到外**的原则（先确认"自己没问题"，再确认"中间没问题"，最后确认"外面没问题"）或许能够帮你节约大量的时间，。

下面按照从**具体访问现象**出发的思路，由浅入深展开。

---

# 1. 第一层：Pod 自身状态确认

**现象：Pod 完全无法访问，任何请求都无响应**
这是最基础的一层。在排查任何网络问题之前，先确认 Pod 本身是否健康运行。

## 1.1 检查 Pod 状态

```bash
kubectl get pods -o wide
```

关注以下状态：

| 状态 | 含义 | 可能原因 |
|------|------|----------|
| `Running` | 正常运行 | 正常 |
| `Pending` | 等待调度 | 资源不足、节点亲和性不匹配 |
| `CrashLoopBackOff` | 反复崩溃重启 | 应用启动失败、配置错误 |
| `OOMKilled` | 内存超限被杀 | `resources.limits.memory` 设置过低 |
| `ImagePullBackOff` | 镜像拉取失败 | 镜像名错误、仓库认证失败 |
| `ContainerCreating` | 容器创建中 | CNI 插件分配 IP 失败、存储卷挂载失败 |

如果 Pod 不在 `Running` 状态，网络问题只是表象，根因在 Pod 本身：

```bash
# 查看 Pod 详细事件
kubectl describe pod <pod-name> -n <namespace>

# 查看容器日志（如果容器曾启动过）
kubectl logs <pod-name> -n <namespace>

# 查看前一个崩溃容器的日志
kubectl logs <pod-name> -n <namespace> --previous
```



## 1.2 确认 Pod 有 IP 地址

```bash
kubectl get pods -o wide
```

如果 Pod 状态是 `Running` 但 **IP 为空**，说明 CNI 插件没有成功分配 IP。此时需要检查：

```bash
# 确认 CNI 插件 Pod 状态
kubectl get pods -n kube-system | grep -E "calico|flannel|cilium"

# 查看 CNI 插件日志
kubectl logs -n kube-system <cni-pod-name>

# 检查节点上的 CNI 配置
# 在节点上执行
cat /etc/cni/net.d/*.conf
```



## 1.3 检查核心组件是否健康

```bash
kubectl get pods -n kube-system
```

确保以下组件均为 `Running` 状态：

- **kube-apiserver** — API 入口
- **etcd** — 集群状态存储
- **kube-proxy** — Service 流量转发
- **CoreDNS** — 集群 DNS 解析
- **CNI 插件**（calico-node / flannel / cilium）— Pod 网络



---

# 2. 第二层：Pod 内部网络自检

**现象：Pod 状态正常（Running），但从外部访问时连接被拒绝（Connection Refused）或超时**

## 2.1 应用是否真的在监听？

这是最容易被忽略的一步。进入 Pod 内部确认：

```bash
# 进入 Pod
kubectl exec -it <pod-name> -n <namespace> -- sh

# 查看进程监听情况
netstat -tlnp 2>/dev/null || ss -tlnp
```

**重点关注**：

- 如果看不到目标端口在 `LISTEN` → 应用没启动或启动失败，先看应用日志
- 如果端口在监听但绑定的是 `127.0.0.1` → **经典坑！** 只接受本机回环，集群中其他 Pod 无法访问

```bash
# 错误配置示例：只监听 localhost
server.listen('127.0.0.1', 8080)  # ← 其他 Pod 访问 Pod IP:8080 会被拒绝

# 正确配置：监听所有网卡
server.listen('0.0.0.0', 8080)    # ← 接受来自任何 IP 的连接
```



## 2.2 Pod 内能否访问自己？

```bash
# 在 Pod 内 curl 自己
kubectl exec -it <pod-name> -n <namespace> -- curl -s localhost:8080/health
```

- 如果 **localhost 都不通** → 应用层问题，不是网络问题，去查应用日志
- 如果 localhost 通但 Pod IP 不通 → 应用监听了 `127.0.0.1` 而非 `0.0.0.0`

## 2.3 检查 Pod 的网络命名空间

```bash
# 在 Pod 内查看网卡和 IP
kubectl exec -it <pod-name> -n <namespace> -- ip addr show

# 查看路由表
kubectl exec -it <pod-name> -n <namespace> -- ip route show
```

**正常情况**：

- 存在 `eth0` 网卡，IP 属于 CNI 配置的 Pod 网段（如 `10.244.x.x`）
- 默认路由指向 `eth0`，网关指向 CNI 网桥

**异常情况**：

- 没有 `eth0` → CNI 插件未正确创建网络接口
- IP 不在预期网段 → CNI 配置错误
- 没有默认路由 → Pod 无法访问外部网络



---

# 3. 第三层：同节点 Pod 间通信

**现象：Pod A 无法访问同一节点上的 Pod B（通过 Pod IP）**

## 3.1 基础连通性测试

```bash
# 从 Pod A ping Pod B 的 IP
kubectl exec -it <pod-a> -n <namespace> -- ping <pod-b-ip>

# 如果 ping 不通，用更精确的 TCP 端口测试
kubectl exec -it <pod-a> -n <namespace> -- nc -zv <pod-b-ip> <port>
```

> **注意**：有些容器镜像不包含 `ping`、`nc` 等工具，可以用临时调试容器：
> ```bash
> kubectl run debug --rm -it --image=nicolaka/netshoot -- sh
> ```

## 3.2 同节点不通的排查方向

如果同节点 Pod 之间 ping 不通，问题通常在节点本地：

**（1）检查节点上的 veth pair 和网桥**

```bash
# 在节点上执行
ip link | grep veth        # 查看虚拟网卡对
brctl show                 # 查看网桥及挂载的 veth
```

每个 Pod 在宿主机上会有一个 `veth` 对，一端在 Pod 网络命名空间（表现为 `eth0`），另一端挂载到 CNI 网桥上。如果 veth 缺失或网桥未挂载，说明 CNI 插件工作异常。

**（2）检查节点 iptables 规则**

```bash
# 查看是否有规则阻断了 Pod 网段的转发
iptables -L FORWARD -n -v
iptables -L INPUT -n -v
```

确认 `FORWARD` 链的默认策略不是 `DROP`，或者 Pod 网段的流量被明确放行。

**（3）检查内核转发参数**

```bash
sysctl net.ipv4.ip_forward          # 必须为 1
sysctl net.ipv4.conf.all.rp_filter  # 建议为 0 或 2
```



---

# 4. 第四层：跨节点 Pod 间通信

**现象：Pod A（节点 1）无法访问 Pod B（节点 2），但同节点 Pod 通信正常**

这是 K8s 网络排查中**最高频**的问题之一。跨节点通信依赖 CNI 插件建立的隧道或路由。

## 4.1 定位断点

```bash
# 在 Pod A 中 traceroute 到 Pod B
kubectl exec -it <pod-a> -- traceroute <pod-b-ip>

# 如果 traceroute 不可用，用 mtr
kubectl exec -it <pod-a> -- mtr <pod-b-ip>
```

观察在哪一跳丢包，判断问题出在源节点、中间网络还是目标节点。

## 4.2 检查节点间底层网络

```bash
# 在两个节点上互 ping 对方的节点 IP
ping <node-b-ip>
```

如果节点之间都不通，问题在底层物理网络/云网络，与 K8s 无关。

## 4.3 检查安全组 / 防火墙

**这是跨节点不通最常见的原因。** 云环境中的安全组必须放行以下流量：

- **Pod CIDR 网段**的所有流量（入方向和出方向）
- **CNI 封装协议端口**：
  - Flannel VXLAN 模式：UDP 8472
  - Calico IPIP 模式：IP 协议号 4
  - Calico VXLAN 模式：UDP 4789
  - Cilium Geneve：UDP 6081

```bash
# 在节点上抓包，确认封装流量是否发出/收到
# Flannel
tcpdump -i flannel.1 -nn

# Calico
tcpdump -i tunl0 -nn
```



## 4.4 检查 CNI 插件状态

**Calico**：

```bash
calicoctl node status              # 检查 BGP 连接状态
calicoctl get bgppeers             # 检查 BGP Peer 配置
```

**Flannel**：

```bash
cat /var/run/flannel/subnet.env    # 检查子网分配
ip addr show flannel.1             # 检查 flannel 网卡状态
```

**Cilium**：

```bash
cilium status
cilium endpoint list
```

## 4.5 检查 VPC 路由表（非 Overlay 模式）

如果使用路由模式（而非隧道模式）实现跨节点通信，VPC 路由表中必须有每个节点 Pod CIDR 的路由条目：

```bash
# 查看每个节点的 PodCIDR
kubectl describe node <node-name> | grep PodCIDR
```

确认路由表条目数量与节点数一致，目的地址为各节点的 PodCIDR，下一跳指向对应节点。

---

# 5. 第五层：Service 访问排查

**现象：Pod 间通过 Service 名称或 ClusterIP 访问不通**
K8s 中 Pod 间通信通常不直接使用 Pod IP（因为 Pod IP 不稳定），而是通过 Service 进行服务发现。流量路径为：

```
Pod A → ClusterIP:ServicePort → kube-proxy (iptables/IPVS) → EndpointIP:ContainerPort → Pod B
```



## 5.1 Service 是否存在？

```bash
kubectl get svc <service-name> -n <namespace>
```

确认 Service 存在且 ClusterIP 已分配。

## 5.2 Endpoints 是否为空？（最高频问题）

```bash
kubectl get endpoints <service-name> -n <namespace>
```

**如果 Endpoints 为空（`<none>`）**，说明 Service 找不到后端 Pod，常见原因：

### 5.2.1 Selector 标签不匹配

```bash
# 查看 Service 的 selector
kubectl describe svc <service-name> -n <namespace> | grep Selector

# 查看 Pod 的标签
kubectl get pods --show-labels -n <namespace>
```

对比 Service 的 `spec.selector` 和 Pod 的 `metadata.labels` 是否完全一致。注意拼写错误、大小写、命名空间不一致等。

### 5.2.2 Pod 就绪探针（Readiness Probe）失败

即使标签匹配，如果 Pod 的 Readiness Probe 失败，Pod 也会被从 Endpoints 中移除：

```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A10 "Conditions"
```

查看 `Ready` 条件是否为 `True`。如果为 `False`，检查 Readiness Probe 配置是否合理（端口、路径、超时时间等）。

## 5.3 端口映射是否正确？

```bash
kubectl describe svc <service-name> -n <namespace>
```

重点检查：

- `Port` — Service 暴露的端口
- `TargetPort` — 转发到 Pod 的端口，必须与容器实际监听端口一致
- `Protocol` — TCP/UDP 是否正确

```yaml
# 常见错误：targetPort 与容器实际监听端口不一致
spec:
  ports:
    - port: 80
      targetPort: 8080   # ← 必须与容器内进程监听的端口一致
```



## 5.4 kube-proxy 是否正常工作？

kube-proxy 负责在节点上维护 Service 的转发规则。

```bash
# 确认 kube-proxy 模式
kubectl get cm kube-proxy -n kube-system -o yaml | grep mode

# 检查 kube-proxy Pod 状态
kubectl get pods -n kube-system | grep kube-proxy

# 查看 kube-proxy 日志
kubectl logs -n kube-system <kube-proxy-pod>
```

**iptables 模式**下检查规则：

```bash
iptables -t nat -L KUBE-SERVICES -n | grep <service-name>
iptables -t nat -L KUBE-SEP-* -n | grep <pod-ip>
```

**IPVS 模式**下检查规则：

```bash
ipvsadm -Ln
```

如果规则缺失或未更新，尝试重启 kube-proxy：

```bash
kubectl delete pod -n kube-system -l k8s-app=kube-proxy
```



## 5.5 绕过 Service 直连 Pod 验证

为了区分是"应用层问题"还是"Service/网络层问题"，直接 curl Pod IP：

```bash
# 在另一个 Pod 中直连目标 Pod IP
kubectl exec -it <source-pod> -- curl -s <target-pod-ip>:<container-port>
```

- 直连 Pod IP **通** → 应用没问题，问题在 Service/kube-proxy 层
- 直连 Pod IP **也不通** → 问题在 Pod 本身或底层网络



---

# 6. 第六层：DNS 解析排查

**现象：Pod 内通过 Service 名称访问时报 "Name or service not known" 或 "Could not resolve host"**

K8s 内部通过 CoreDNS 实现服务发现，Pod 访问 `service-name.namespace.svc.cluster.local` 时，需要 CoreDNS 解析为 ClusterIP。

## 6.1 确认 DNS 解析是否失败

```bash
# 在 Pod 内测试 DNS 解析
kubectl exec -it <pod-name> -- nslookup <service-name>
kubectl exec -it <pod-name> -- nslookup <service-name>.<namespace>.svc.cluster.local

# 测试外部域名解析
kubectl exec -it <pod-name> -- nslookup www.baidu.com
```

**区分两种情况**：

- **内部域名解析失败，外部域名正常** → CoreDNS 配置问题
- **所有域名都解析失败** → Pod 与 CoreDNS 之间网络不通，或 CoreDNS 本身异常

## 6.2 检查 Pod 的 DNS 配置

```bash
kubectl exec -it <pod-name> -- cat /etc/resolv.conf
```

正常输出应类似：

```
nameserver 10.96.0.10        # kube-dns Service 的 ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- `nameserver` 必须指向 kube-dns Service IP
- `search` 域必须包含 `svc.cluster.local`
- 跨命名空间访问需使用完整域名：`<service>.<namespace>.svc.cluster.local`



## 6.3 检查 CoreDNS 状态

```bash
# 确认 CoreDNS Pod 运行状态
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 查看 CoreDNS 日志（关注 SERVFAIL、timeout、plugin/errors）
kubectl logs -n kube-system -l k8s-app=kube-dns

# 查看 CoreDNS 配置
kubectl get cm coredns -n kube-system -o yaml
```

## 6.4 绕过 Service 直连 CoreDNS Pod

为了区分是"CoreDNS 本身问题"还是"kube-proxy 转发问题"：

```bash
# 获取 CoreDNS Pod IP
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

# 在业务 Pod 中直连 CoreDNS Pod IP（绕过 Service）
kubectl exec -it <pod-name> -- nslookup <service-name> <coredns-pod-ip>
```

- 直连 CoreDNS Pod IP **正常** → CoreDNS 本身没问题，问题在 kube-proxy 或 NetworkPolicy（未放行 UDP 53）
- 直连 CoreDNS Pod IP **也失败** → CoreDNS 本身异常或 Pod 到 CoreDNS 的网络不通



## 6.5 DNS 间歇性超时（5 秒延迟）

如果 DNS 查询经常出现 **5 秒延迟**后返回，这通常是内核 conntrack 冲突导致的丢包：

```bash
# 检查 conntrack 表使用率
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# 如果接近上限，增大 conntrack 表
sysctl -w net.netfilter.nf_conntrack_max=524288
```



---

# 7. 第七层：外部访问排查（NodePort / LoadBalancer / Ingress）

**现象：集群外部无法访问集群内的服务**

## 7.1 NodePort 类型

```bash
kubectl get svc <service-name> -n <namespace>
```

确认：

- Service 类型为 `NodePort`
- `NodePort` 端口已分配（范围默认 30000-32767）
- 在节点上测试：`curl <node-ip>:<node-port>`

如果节点上能通但外部不通：

- 检查云安全组 / 防火墙是否放行 NodePort 端口
- 检查节点 IP 是否对外可达

## 7.2 LoadBalancer 类型

```bash
kubectl get svc <service-name> -n <namespace>
```

确认 `EXTERNAL-IP` 已分配（不是 `<pending>`）。如果长时间 Pending：

- 检查云厂商负载均衡器是否正常创建
- 检查 Cloud Controller Manager 日志

## 7.3 Ingress 类型

```bash
# 检查 Ingress Controller Pod 状态
kubectl get pods -n <ingress-namespace> | grep ingress

# 检查 Ingress 规则
kubectl describe ingress <ingress-name> -n <namespace>

# 测试转发（绕过 DNS）
curl -H "Host: <domain>" <ingress-controller-ip>
```

**Ingress 返回 502** 的常见原因：

- 后端 Service 的 Endpoints 为空
- Ingress Controller 到后端 Pod 网络不通
- `backend.service.port` 配置与实际端口不一致

```bash
# 查看 Ingress Controller 日志
kubectl logs -n <ingress-namespace> <ingress-controller-pod> | grep "upstream timed out\|502"
```



---

# 8. 第八层：NetworkPolicy 排查

**现象：以上所有检查都正常，但特定 Pod 之间的流量就是不通**

这很可能是 **NetworkPolicy（网络策略）** 在起作用。NetworkPolicy 是 K8s 原生的"防火墙"，可以精确控制 Pod 的入站（Ingress）和出站（Egress）流量。

```bash
# 查看目标命名空间下的所有网络策略
kubectl get networkpolicies -n <namespace>

# 查看策略详情
kubectl describe networkpolicy <policy-name> -n <namespace>
```

**关键规则**：

- 如果某个命名空间存在**任何** NetworkPolicy 选中了目标 Pod，则该 Pod 默认**拒绝所有未明确允许的流量**
- 检查 Ingress 规则是否允许源 Pod 的标签和端口
- 检查 Egress 规则是否允许目标 Pod 的标签和端口
- 检查是否放行了 DNS 端口（UDP/TCP 53），否则 Pod 无法进行 DNS 解析



---

# 9. 第九层：高级排查工具

当以上步骤都无法定位问题时，需要使用更底层的工具。

## 9.1 tcpdump 抓包

**在 Pod 内抓包**：

```bash
# 如果容器内有 tcpdump
kubectl exec -it <pod-name> -- tcpdump -i eth0 -nn port <port>

# 如果容器内没有 tcpdump，使用临时调试容器（共享网络命名空间）
kubectl debug -it <pod-name> --image=nicolaka/netshoot -- tcpdump -i eth0 -nn port <port>
```

**在节点上抓包**：

```bash
# 抓取特定 Pod 的流量
tcpdump -i any host <pod-ip> -nn

# 抓取 CNI 隧道接口的流量（跨节点通信）
tcpdump -i flannel.1 -nn          # Flannel
tcpdump -i tunl0 -nn              # Calico IPIP
tcpdump -i vxlan.calico -nn       # Calico VXLAN
```

**通过 nsenter 进入 Pod 网络命名空间**（在宿主机上执行）：

```bash
# 获取 Pod 的 pause 容器 PID
PID=$(crictl inspect $(crictl pods --name <pod-name> -q) | jq .info.pid)

# 进入 Pod 的网络命名空间
nsenter -t $PID -n tcpdump -i eth0 -nn port <port>
```

## 9.2 conntrack 检查

```bash
# 查看连接跟踪表中特定目标的状态
conntrack -L -d <service-clusterip>

# 查看是否有大量 INVALID 状态的连接
conntrack -S | grep -i invalid
```

## 9.3 MTU 问题排查

如果**小包能通但大包丢包**，很可能是 MTU 不匹配：

```bash
# 在 Pod 内测试不同大小的包
kubectl exec -it <pod-name> -- ping -s 1472 -M do <target-ip>   # 1500 MTU
kubectl exec -it <pod-name> -- ping -s 1400 -M do <target-ip>   # 逐步减小

# 查看接口 MTU
kubectl exec -it <pod-name> -- ip link show eth0
```

Overlay 网络（如 VXLAN）会在原始包上额外封装头部（通常 50 字节），因此 Pod 的 MTU 应比物理网卡 MTU 小至少 50 字节。



---

# 10. 排查思路总结

```
┌─────────────────────────────────────────────────────────┐
│                   网络排查黄金路径                         │
│                                                         │
│  Layer 1: Pod 自身 ──→ 状态正常？有 IP？进程在监听？       │
│       │                                                 │
│       ▼                                                 │
│  Layer 2: Pod 内部 ──→ localhost 能通？绑定 0.0.0.0？     │
│       │                                                 │
│       ▼                                                 │
│  Layer 3: 同节点 ──→ 同节点 Pod IP 能 ping？             │
│       │                                                 │
│       ▼                                                 │
│  Layer 4: 跨节点 ──→ 跨节点 Pod IP 能 ping？             │
│       │           安全组？CNI 隧道？路由表？               │
│       ▼                                                 │
│  Layer 5: Service ──→ Endpoints 非空？kube-proxy 规则？  │
│       │                                                 │
│       ▼                                                 │
│  Layer 6: DNS ─────→ 域名能解析？CoreDNS 正常？           │
│       │                                                 │
│       ▼                                                 │
│  Layer 7: 外部访问 ──→ NodePort/LB/Ingress 配置正确？    │
│       │                                                 │
│       ▼                                                 │
│  Layer 8: 网络策略 ──→ NetworkPolicy 是否阻断？          │
│       │                                                 │
│       ▼                                                 │
│  Layer 9: 高级工具 ──→ tcpdump/conntrack/MTU            │
└─────────────────────────────────────────────────────────┘
```

**核心关键**：每次只验证一层，确认这层 OK 了再往下一层走。这样你永远不会在"Pod 本身都起不来"的时候去调 CNI，也不会在"DNS 都没解析"的时候去抓包。

**铁律**：先看后动。牢记不同错误的核心现象，90% 的 K8s 网络问题，`kubectl describe` + `kubectl logs` + 逐层 ping/curl 就能定位。不要上来就 `delete pod` 或重启容器运行时——这会清空最关键的排查现场。

