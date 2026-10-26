+++
date = '2026-10-26T10:41:08+08:00'
draft = false
title = 'Trivy 使用笔记：从安装到生产实践'
categories = ['Security']
tags = ['Trivy', 'DevSecOps', '容器镜像']
+++


# 1. Trivy 是什么

- **官方网站：** [https://trivy.dev/](https://trivy.dev/)
- **开源仓库：** [https://github.com/aquasecurity/trivy](https://github.com/aquasecurity/trivy)

**Trivy**（读作 “tri-vee”）是由 **Aqua Security** 主导开发并开源的[统一安全扫描工具](https://trivy.dev/)，名称取自 “Trigger”，寓意快速触发安全检查。它面向整个软件供应链，可用于扫描：

- **容器镜像与文件系统**
- **Git 仓库与依赖项**
- **IaC 配置**，如 Dockerfile、Terraform 和 Kubernetes 清单
- **SBOM**
- **Kubernetes 集群及资源**

借助统一的命令行入口，Trivy 能够发现：

- 操作系统及应用依赖中的已知漏洞（CVE）
- 容器、Kubernetes 和 IaC 配置错误
- 硬编码密码、令牌等泄露密钥
- 依赖项的许可证合规风险

Trivy 的优势在于将多种安全检查整合到一套工具中，并可轻松接入 CI/CD、DevSecOps 和 Kubernetes 等工作流，以结构化报告输出扫描结果。这样无需为不同类型的安全风险维护多套独立工具，就能在代码构建、制品发布和部署运行等阶段持续开展安全检查。

```text
trivy <扫描目标> ─► 镜像 / 文件系统 / Git / IaC / SBOM / K8s
                   ─► 漏洞 + 配置错误 + 密钥 + 许可证风险报告
```

# 2. 为什么我们需要它

## 2.1 供应链安全是真实风险
服务的容器镜像不是凭空造的，它由一层层基础镜像 + 一堆依赖库叠加而成。任何一个基础镜像里的老漏洞（比如某个 glibc 漏洞）、任何一个 `npm install` 下来的依赖带 CVE，都会**原封不动带进生产环境**。

**经典案例**：log4j、Heartbleed、各种容器逃逸漏洞都是基础镜像/依赖里早就躺着一个已知 CVE，直到被利用才被发现。

## 2.2 「左移」是行业共识
以前的做法是：镜像上线被攻击了才排查，或者发版前安全团队手工扫一遍，太慢太被动。

现在的做法叫 **Security as Code / 左移（Shift Left）**：把安全扫描**提前到 CI 流水线里**，构建完镜像顺手扫一遍，高危漏洞直接让流水线失败，坏镜像根本进不了仓库。

Trivy 就是「左移」理念里最常用的一个工具，它能作为一个独立命令嵌进任何 CI 流程中（GitLab、GitHub Actions、Jenkins…），没有平台绑定。

## 2.3 为什么选 Trivy 而不是别的
市面上类似的工具很多（Clair、Anchore、Snyk、Docker Scout…）。Trivy 的核心优势在于：
- **开源免费**：无额外服务依赖，一条命令即可用；
- **扫得快**：适合高频 CI；
- **单一二进制**：无守护进程，部署零成本；
- **覆盖面广**：一个工具解决大部分扫描需求，不用装一堆工具。

# 3. Trivy 能扫哪些东西

很多人只知道它能扫镜像，实际功能远不止于此：

| 扫描对象 | 子命令 | 作用 | 典型场景 |
|---|---|---|---|
| 容器镜像 | `trivy image` | 扫镜像里的 OS 包 + 语言依赖漏洞 | 构建完成后扫镜像 |
| 文件系统 | `trivy fs` | 扫一个目录/工程里的依赖漏洞 | 扫描代码仓库的依赖 |
| Git 仓库 | `trivy repo` | 直接扫 Git 仓库历史/分支 | 扫开源项目、审计 |
| 配置文件 | `trivy config` | 扫 IaC（Dockerfile/K8s yaml/Terraform）配置错误 | 检查镜像/K8s 配置是否安全 |
| SBOM | `trivy sbom` | 生成/比对软件物料清单 | 生成依赖清单供审计 |
| 密钥泄露 | `trivy fs --scanners secret` | 扫硬编码的密钥/Token | 防止密码泄露 |
| K8s 集群 | `trivy k8s` | 扫集群里运行的资源配置 | 集群安全基线 |

我们平时 CI 里最常用的是 `image` 和 `config`。

# 4. 安装 Trivy

[Trivy 官方安装文档](https://trivy.dev/docs/latest/getting-started/installation/)

## 4.1 一键脚本安装
```bash
# 官方推荐，安装对应平台执行版本
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin v0.76.1
trivy --version
```

## 4.2 包管理器安装
```bash
# macOS
brew install aquasecurity/trivy/trivy

# Debian/Ubuntu（apt 源）
sudo apt-get install wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# RHEL/CentOS（rpm 源）
cat << EOF | sudo tee -a /etc/yum.repos.d/trivy.repo
[trivy]
name=Trivy repository
baseurl=https://aquasecurity.github.io/trivy-repo/rpm/releases/\$basearch/
gpgcheck=1
enabled=1
gpgkey=https://aquasecurity.github.io/trivy-repo/rpm/public.key
EOF
sudo yum -y update
sudo yum -y install trivy
```

## 4.3 Docker 方式
用 docker run 时，主机上的容器引擎套接字需要挂载到 Trivy 容器中，工作目录和数据库目录也要挂载出来：
```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/.cache/trivy:/root/.cache/trivy \
  -v $(pwd):/workspace \
  aquasec/trivy:latest image myimage:latest
```

## 4.4 验证安装成功
```bash
trivy --version
# 应输出类似：Version: 0.xx.x
# 顺便看子命令列表
trivy --help
```

# 5. 快速上手


```bash
trivy image golang:1.26-trixie
```
第一次运行会自动下载漏洞数据库（约几百 MB），之后有增量更新，所以第一次会慢一点，属正常现象。

输出会按严重级别分组：
```text
Total: 1 (UNKNOWN: 0, LOW: 0, MEDIUM: 0, HIGH: 1, CRITICAL: 0)

┌─────────┬─────────────────┬──────────┬────────┬───────────────────┬─────────────────┬─────────────────────────────────────────────────────────────┐
│ Library │  Vulnerability  │ Severity │ Status │ Installed Version │  Fixed Version  │                            Title                            │
├─────────┼─────────────────┼──────────┼────────┼───────────────────┼─────────────────┼─────────────────────────────────────────────────────────────┤
│ stdlib  │ CVE-2026-111943 │ HIGH     │ fixed  │ v1.26.9           │ 1.26.10, 1.27.3 │ crypto/x509: golang: crypto/x509: Man-in-the-middle attacks │
│         │                 │          │        │                   │                 │ via untrusted intermediate certificates                     │
│         │                 │          │        │                   │                 │ https://avd.aquasec.com/nvd/cve-2026-111943                 │
└─────────┴─────────────────┴──────────┴────────┴───────────────────┴─────────────────┴─────────────────────────────────────────────────────────────┘
```

> [!NOTE] 备注
> Trivy 扫描镜像会看两个层面，**操作系统层**（apt/yum 装的系统库）和**应用依赖层**（npm、pip、go mod、maven 等语言包）。这也是为什么它能扫出两类不同漏洞。

# 6. 常用参数详解

| 参数 | 作用 | 例子 |
|---|---|---|
| `--severity` | 只显示指定级别 | `--severity HIGH,CRITICAL` |
| `--exit-code` | 发现漏洞时返回非0退出码（CI 用） | `--exit-code 1` |
| `--ignore-unfixed` | 忽略还没有修复补丁的漏洞 | `--ignore-unfixed` |
| `--format` | 输出格式：table/json/sarif/template | `--format json` |
| `--output` | 结果写文件 | `--output trivy.json` |
| `--scanners` | 指定扫描类型 | `--scanners vuln,secret,config` |
| `--skip-dirs` | 跳过某些目录 | `--skip-dirs vendor` |
| `--cache-dir` | 指定漏洞库缓存目录 | `--cache-dir /tmp/trivy` |
| `--offline-scan` | 离线模式（不更新库，用本地库扫） | `--offline-scan` |

## 6.1 组合使用：扫出高危就退出
CI 流程中常用的核心逻辑，有高危漏洞则输出1，流水线会判定失败。
```bash
# 只关注高危和严重漏洞，扫到就返回退出码1
trivy image \
  --exit-code 1 \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  myapp:dev
echo "exit code: $?"
```

## 6.2 输出为 JSON / SARIF
CI 平台一般会有代码扫描的展示页面，输出产物并上传就可以在页面上显示扫描后的结果：
```bash
# JSON 方便二次处理/入库
trivy image --format json --output trivy.json myapp:dev

# SARIF 格式，可直接导入 GitHub/GitLab 的 Code Scanning 页面
trivy image --format sarif --output trivy.sarif myapp:dev
```

## 6.3 扫描文件系统/代码仓库
```bash
# 扫当前目录的依赖漏洞（比如 package-lock.json、go.sum 等）
trivy fs .

# 只扫密钥泄露
trivy fs --scanners secret .

# 直接扫一个 git 仓库
trivy repo "https://github.com/some/repo.git"
```

## 6.4 扫 IaC 配置文件
```bash
# 检查 Dockerfile 和 K8s yaml 里的配置问题
trivy config .
```
能扫出比如：`RUN` 用了 root 用户、镜像没用固定 tag、暴露了特权端口、缺少 resource limits 等。

## 6.5 生成 SBOM（软件物料清单）
```bash
# 导出 CycloneDX 格式的 SBOM
trivy image --format cyclonedx --output sbom.json myapp:dev
```

# 7. 忽略某些漏洞
有时候某个 CVE 不影响你的使用场景（比如只在特定架构/特定库函数触发），你不想每次 CI 都报红。Trivy 支持忽略机制。

## 7.1 项目根目录放 `.trivyignore`
```text
# .trivyignore
CVE-2023-0001
CVE-2023-0002
```

## 7.2 用参数忽略
```bash
# 忽略某个 CVE
trivy image --ignore-unfixed --ignore-pkgs libssl3 myapp:dev
```

> [!CAUTION] 注意：
> 忽略漏洞要有理由、要有期限，最好能在 MR 里说明「为什么可以忽略」，不能养成「一红就 ignore」的习惯，否则扫描就形同虚设。

# 8. 离线环境补充
很多企业集群无法访问公网，Trivy 需要离线漏洞库，可以先提前把漏洞库下载下来再使用离线扫描的方式：

```bash
# 1. 在能上网的机器上先把漏洞数据库拉下来
trivy image --download-db-only --cache-dir /tmp/trivy-db

# 2. 把 /tmp/trivy-db 目录拷贝到内网机器

# 3. 内网机器上指定缓存目录，并用离线模式扫描
trivy image --cache-dir /tmp/trivy-db --offline-scan myapp:dev
```
> [!TIP] 提示
> 建议搭配脚本和定时任务，定期更新漏洞库

# 9. 生产最佳实践

1. **锁版本，不用 latest**：CI 里镜像用固定 tag（如 `aquasec/trivy:0.59`），保证扫描结果可复现。
2. **高危/严重才阻断**：`--severity HIGH,CRITICAL`，忽略中低危噪音，别让流水线天天红。
3. **`--ignore-unfixed` 必加**：只关注有修复方案的漏洞，避免为「无解的 CVE」卡死发版。
4. **`--exit-code 1` 是强制关卡的核心**：扫到高危就失败、流水线停，这才是「扫描有意义」的关键。
5. **密码走变量 + Masked**：不硬编码，GitLab 变量里勾选掩码。
6. **定期更新漏洞库**：Trivy 首次自动下载，CI 场景建议定期更新或设缓存，别用过期库。
7. **离线环境处理**：内网/信创环境需提前下载漏洞库到 `--cache-dir`，再用 `--offline-scan` 扫。
8. **结果归档**：用 `--format json/sarif` 把扫描结果存成产物，供安全团队审计、留痕。
9. **忽略要有记录**：`.trivyignore` 里写明理由和期限，别无脑 ignore。
10. **分层排查**：漏洞可能来自基础镜像（换基础镜像）或语言依赖（升依赖版本），用 `--scanners vuln` 看清来源。
