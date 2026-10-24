+++
date = '2026-10-24T11:28:31+08:00'
draft = false
title = 'Kaniko 从入门到实践'
categories = ['DevOps', 'Kubernetes']
tags = ['Kaniko', 'CICD', '容器镜像', 'K8s', 'GitLab']
+++

# 1. Kaniko 是什么

**Kaniko** 是一个由 Google 开源的，在**容器或 Kubernetes 集群内部**，直接根据 Dockerfile 构建容器镜像的工具。

它最核心的特点只有一个：**不依赖 Docker daemon（守护进程）**，所有构建动作都在用户态（userspace）完成。这意味着你不需要在集群里跑一个 Docker 服务，也不需要挂载 `/var/run/docker.sock`，就可以构建出标准镜像。


# 2. Kaniko 解决了什么问题

在 Kaniko 出现之前，CI 里构建镜像的主流做法是 **Docker-in-Docker（DinD）**。即在 CI Job 里再起一个 Docker 守护进程，用它去 build。这套方案在 Kubernetes 集群里有两个致命痛点：

1. **需要 privileged 特权模式**：dind 服务容器要操控内核特性，必须给 Pod 开 `privileged: true`。这是很严重的安全隐患——容器可能逃逸到宿主机，很多安全策略严格的集群直接禁止。
2. **性能差**：每个 Job 都额外拉起一个完整 Docker daemon，启动慢、占用高，多任务并发时更是灾难。

Kaniko 就是为了解决这个问题而生的：**不用特权、不用 daemon，在标准 K8s 集群里就能安全构建镜像。**


| 问题 | DinD（旧方案） | Kaniko |
| --- | --- | --- |
| 需要 privileged 特权 | ✅ 必须 | ❌ 不需要 |
| 依赖 Docker daemon | ✅ 必须 | ❌ 完全不需要 |
| 挂载 docker.sock | ✅ 需要 | ❌ 不需要 |
| 安全性 | 有容器逃逸风险 | 纯用户态，安全 |
| 性能开销 | 每个 Job 起 daemon，慢 | 轻量，快 |

Kaniko 的核心价值用一句话来说就是：**在不安全、不方便跑 Docker daemon 的环境（尤其是 K8s 集群）里，仍然能安全、快速地构建镜像。**

# 3. 工作原理

Kaniko 的 executor 镜像负责构建：

1. 提取 Dockerfile 里 `FROM` 基础镜像的文件系统；
2. 在**自己的容器内**逐条执行 Dockerfile 的指令；
3. 每条指令执行后，在用户态对文件系统做快照（snapshot），把变更追加为一层；
4. 更新镜像元数据，最终推送标准镜像到仓库。

由于它没有用任何容器化技术去隔离构建过程，所以**无需任何特权**，代价是偶尔会出现一些反直觉的结果（比如用 Kaniko 构建 Kaniko 镜像时需要特殊处理 `/kaniko` 目录）。

# 4. 使用方案

[kaniko · 使用教程](https://github.com/GoogleContainerTools/kaniko/blob/main/docs/tutorial.md)

## 4.1 本地 Docker 直接运行

Kaniko 以镜像形式发布，本身不会拉起 Docker daemon，因此使用前需要有一台装有 Docker 的机器，且能访问目标镜像仓库。

容器启动后，`/kaniko/executor` 会读取 Dockerfile，在用户空间里逐条执行指令，每执行一条就对文件系统做一次快照、生成一层，最后把成品镜像直接推送到 registry。

所以整个命令的准备工作只有两件事：把"构建上下文"挂进去，把"仓库凭证"挂进去。

```bash
docker run --rm \
  -v "$HOME/.docker/config.json:/kaniko/.docker/config.json:ro" \
  -v "$(pwd):/workspace" \
  ghcr.io/osscontainertools/kaniko:latest \
  --dockerfile /workspace/Dockerfile \
  --context dir:///workspace/ \
  --destination "registry.example.com/myimage:v1.0" \
  --destination "registry.example.com/myimage:latest" \
  --cache=true \
  --cache-repo registry.example.com/myimage/cache
```

**参数解读**：
- `--context dir:///workspace/` 里的路径是"容器内路径"，`dir://` 后面接 `///workspace` 这种三斜杠写法表示绝对路径，写成本机路径会直接报错。`--dockerfile` 同理，必须是容器内可见的路径。
- `--destination` 可以重复多次，一次就把多个 tag 打全，比构建两遍省时得多。
- `--cache=true` 开启层缓存，配合 `--cache-repo` 指向一个专门的缓存仓库，重复构建时未变更的层直接复用。

**操作步骤**：
1. 先在宿主机上执行 `docker login registry.example.com`，让凭证写入 `~/.docker/config.json`；
2. 确认构建目录里同时有 `Dockerfile` 和所有 `COPY`、`ADD` 需要的文件。
3. 修改上面命令中的镜像仓库和 tag
4. 执行命令

> [!TIP] 提示
> - `COPY` 只能引用上下文目录内的文件，引用 `../` 之类的外部路径会失败；
> - 需要排除的大文件（`.git`、`node_modules`）写进 `.dockerignore`，否则上下文会被整体打包，拖慢启动。
> - 想先验证 Dockerfile 语法而不推送，把 `--destination` 换成 `--no-push` 即可，构建完会有提示信息告诉你产物在哪。


## 4.2 在 Kubernetes 中临时构建

这种方式定位是**临时验证**：起一个一次性 Pod，把打好的上下文通过标准输入喂进去，构建完成后 Pod 进入 `Completed` 状态，看日志、删 Pod，集群里不留痕迹。

它的价值在于快速确认"**某个 Dockerfile 在集群网络与权限下能否跑通**"，而不是承担正式发布。

Kaniko 支持的上下文类型包括本地目录、本地 tar、标准输入、Git 仓库，以及对象存储（S3、GCS、Azure Blob）上的 tar.gz。其中标准输入只接受 `.tar.gz` 格式，并且必须让 Pod 保持交互状态（`stdin: true`、`stdinOnce: true`），否则容器读到 EOF 就直接退出了。

```bash
# 1. 先把项目目录打成压缩包
tar -czf context.tar.gz -C /path/to/project .

# 2. 通过管道喂给一次性 Pod
cat context.tar.gz | kubectl run kaniko \
  --image=ghcr.io/osscontainertools/kaniko:latest \
  --restart=Never --rm -i \
  --overrides='{
    "spec": { "containers": [ { "name": "kaniko",
      "image": "ghcr.io/osscontainertools/kaniko:latest",
      "stdin": true, "stdinOnce": true,
      "args": [ "--dockerfile=Dockerfile",
                "--context=tar://stdin",
                "--destination=<registry>/<repo>/my-image" ]
    } ] } }'

# 3. 观察构建过程并清理
kubectl logs -f kaniko
kubectl delete pod kaniko
```

**几个细节必须注意**：
1. `args` 里每个参数都要用 `=` 形式（`--dockerfile=Dockerfile`），这是因为它以 JSON 数组传给容器，不能像 shell 那样拆成两个元素。
2. `--dockerfile` 的相对路径是相对于 tar 包解压后的根目录，所以上下文打包时务必在项目根目录执行 `tar`，让 `Dockerfile` 落在压缩包顶层。
3. 推送私有仓库时，这个 Pod 里没有凭证文件，需要额外挂一个 Secret：

```bash
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username="$USER" \
  --docker-password="$PASS"

# 在 overrides 的容器定义中追加：
# "volumeMounts": [{"name":"regcred","mountPath":"/kaniko/.docker"}],
# "volumes": [{"name":"regcred","secret":{"secretName":"regcred","items":[
#   {"key":".dockerconfigjson","path":"config.json"}]}}]
```
这种写法用完即弃，没有缓存、没有历史记录、参数散落在命令行里，只适合排障；真实发布还是交给 4.3 的流水线。

## 4.3 GitLab CI + K8s Runner（常见场景）

**在 GitLab CI 中必须选用 debug 镜像**，原因是 executor 镜像把入口固定成了 `/kaniko/executor`，镜像里没有 shell，GitLab Runner 拿到容器后没法执行 `script` 里的命令。

**debug 镜像额外打包了 busybox**，让镜像有了执行shell 的能力，配套还要写 `entrypoint: [""]`，清空镜像自带入口，否则 Runner 的脚本会被当成参数传给 executor 而报错。

详见：[GitLab Docs](https://docs.gitlab.com/ee/ci/docker/using_kaniko.html)

```yaml
build-image:
  stage: build-image
  image:
    name: ghcr.io/osscontainertools/kaniko:v1.28.6-debug   # 使用 :debug 标签
    entrypoint: [""]
  variables:
    IMAGE_NAME: "${HARBOR_HOST}/${HARBOR_PROJECT}/my-app"
  before_script:
    # kaniko 读取 Docker 格式的凭证文件，默认位置 /kaniko/.docker/config.json
    - echo "{\"auths\":{\"${HARBOR_REGISTRY}\":{\"username\":\"${ACR_USERNAME}\",\"password\":\"${ACR_PASSWORD}\"}}}" > /kaniko/.docker/config.json
  script:
    - |
      /kaniko/executor \
        --context "${CI_PROJECT_DIR}" \
        --dockerfile "${CI_PROJECT_DIR}/Dockerfile" \
        --destination "${IMAGE_NAME}:${CI_COMMIT_SHORT_SHA}" \
        --destination "${IMAGE_NAME}:latest" \
        --cache=true \
        --cache-repo "${IMAGE_NAME}/cache" \
        --digest-file /kaniko/digest
  tags:
    - k8s
```

**认证部分**要按 kaniko 的实际约定来写：executor 通过 Docker 凭证文件登录 registry（默认读 `/kaniko/.docker/config.json`，也可用 `DOCKER_CONFIG` 环境变量改到别的目录）。

**缓存参数需要区分**：`--cache-dir` 指的是容器内的本地缓存目录（默认 `/cache`），在 Kubernetes 里 Pod 一销毁就没了，写 `--cache-dir=/kaniko/cache` 基本不会带来加速；真正跨流水线生效的是 `--cache-repo`，它把层缓存推到一个远端仓库，下次构建按层摘要命中，实现秒级复用。两个参数可以同时存在，但收益主要来自后者。

**其余常用参数**按需增补：`--build-arg KEY=VALUE` 传入构建参数（值含空格时先 `export IFS=''`）；`--target stage-name` 指定多阶段构建的终点；`--digest-file /dev/termination-log` 可以把镜像 digest 写进 Pod 终止信息，供下游 job 精确引用本次产物；`--label` 打上提交号、流水线 ID 等元数据，便于后续追溯。


## 4.4 常用参数速查

| 参数 | 作用 |
| --- | --- |
| `--context` | 构建上下文（目录 / Git 仓库 / tar / S3 bucket 等） |
| `--dockerfile` | Dockerfile 路径 |
| `--destination` | 目标镜像，可写多个 |
| `--cache=true` | 开启层缓存 |
| `--cache-dir` | 缓存目录 |
| `--insecure-registry` | 使用 HTTP 明文仓库（**仅测试用**） |
| `--registry-mirror` | 使用镜像加速器 |
| `--skip-tls-verify-registry` | 跳过 TLS 校验（慎用） |

# 5. 最佳实践

1. **开启层缓存**：`--cache=true` + `--cache-dir`，配合远端缓存仓库，多分支并发构建能显著提速。
2. **固定镜像版本**：不要用 `:latest` 漂移，锁定具体版本号，保证构建环境可复现。
3. **认证走环境变量**：用户名密码用 `KANIKO_REGISTRY_USER` / `KANIKO_REGISTRY_PASSWORD` 传入，CI 侧把值放在受保护、可掩码的项目变量里，绝不硬编码。
4. **限制构建资源**：给 Kaniko Pod 设置 CPU/内存 limits，防止单个构建打爆节点。
5. **Harbor 自签证书**：给 Kaniko 挂载 CA 证书，或用 `--registry-certificate` 指定证书文件，否则拉取/推送会失败。

# 6. 谷歌原仓库归档了，怎么办？

2025 年 6 月，`GoogleContainerTools/kaniko` 被 Google 标记为 archived（只读），最后一个发布版本停在 v1.24.0（2025-05-21）。此后该仓库不再合并 PR、不再发布新版，也不再主动修复 bug。归档并不意味着工具立刻失效，而是把“维护责任”从上游转移到了使用者身上。

影响主要有三个：
1. 已发布的镜像仍可从 `gcr.io/kaniko-project/executor` 拉取，已有的 tag 不会消失，存量流水线短期内可以照常运行。
2. 不会再有 v1.24.1：Go 依赖升级、基础镜像更新、CVE 修补全部停止；镜像扫描告警会随基础镜像里存在的漏洞数量增加而持续上升，这是最实质的风险。
3. 第三，下游生态已经用“不再收录”表明态度——GitLab 把 `using_kaniko` 页面标注为**已移除**，并说明 kaniko 不再是一个维护中的项目，改用 Docker 构建、Buildah，或在 Kubernetes 上配合 GitLab Runner 使用 Podman；另一份文档则把 rootless BuildKit 列为 Kaniko 的替代路线。[极狐GitLab 文档](https://gitlab.cn/docs/jh/ci/docker/using_kaniko/)

因此长期选型要未雨绸缪，下面是社区接管的现状。


## 6.1 社区继任者：osscontainertools/kaniko

`osscontainertools/kaniko` 是目前接续最完整的社区 fork，README 自述定位为“原仓库的**受支持替代品**”，重点有三件事：保持依赖更新、修复 bug、提升性能。

需要强调，这里的“支持”是社区承诺，不是 Google 背书，也不是商业 SLA。选型时，应当把它当作一个独立开源项目来评估，重点看 release 节奏、issue 响应和赞助方的持续性，而不是把它当成原项目的延续。

从公开信息看，它确实在持续演进：版本线已越过 v1.24.0，进入 v1.25 之后；有独立的发布节奏与 feature flag 毕业策略文档；社区讨论迁移到 Matrix 频道；赞助方包括 Siemens、L3montree 等企业。与 Chainguard 相比，它最大的差异在于**功能仍在增长**——Chainguard 的策略是“保持原样，只打安全补丁”。

- GitHub 仓库：[osscontainertools/kaniko](https://github.com/osscontainertools/kaniko)
- 官方站点：[osscontainertools.org](https://osscontainertools.org/)（发布说明与博客）

## 6.2 官方可用镜像

这个 fork 提供公开免费镜像，这是它与 Chainguard fork 的关键区别——后者不公开发布镜像，只提供给 Chainguard 客户，仅给出从源码自行构建的指引。镜像地址有两处，内容一致：

```bash
# GitHub Container Registry（文档主推）
ghcr.io/osscontainertools/kaniko:latest

# Docker Hub
martizih/kaniko
```

除 `latest` 外，同一仓库还提供几种变体：
- `:debug` 附带 busybox shell（GitLab CI 场景必需）；
- `:warmer` 用于预热基础镜像缓存；
- `:bootstrap` 用于在 kaniko 内构建 kaniko。

镜像自 v1.24.1 起使用 cosign 做无密钥签名，可以用 GitHub Actions 的 OIDC issuer 校验来源；v1.26.0 之后的镜像需要 cosign v3 才能验证。生产环境建议锁定到版本号，甚至锁定 digest，不要跟随 `latest`。

另外，还有一个常被误读的项目：`gitlab-com.gitlab.io/public-sector/kaniko/`。它其实是 GitLab Public Sector 现场团队维护的一条**构建流水线**，不是源码 fork，上游选的是 `chainguard-forks/kaniko`，而不是本节这个 fork。

它的价值在于合规交付：每次发布六个变体（standard、debug、warmer 各自分标准库加密与 FIPS 140-3 两版），全部 `FROM scratch`，镜像经 cosign keyless 签名，附带 CycloneDX SBOM、SLSA v1.0 溯源信息和 OpenVEX 文档，tag 形如 `v1.25.19-1-fips-debug`。同时它也明确声明：这不是 GitLab 产品，不提供支持授权，也没有服务级别协议。因此，它适合对 FIPS 或供应链溯源有硬性要求的场景，而不是“GitLab 官方推荐的 Kaniko”。[GitLab](https://gitlab.com/gitlab-com/public-sector/kaniko)

## 6.3 相比原版的新特性（亮点）

这个 fork 不只是修补，还给 `executor` 增加了三个实用子命令，正好对应流水线中三个高频痛点。

- **`executor login <registry>`**：把仓库凭据写入 `$DOCKER_CONFIG/config.json`（executor 镜像已将该变量设为 `/kaniko/.docker/`），后续调用 executor 时无需再依赖 credential helper。支持 `-u/--username`、`-p/--password`，以及 `--password-stdin` 从标准输入读取密码，避免密码出现在进程列表里。相比在每个 job 里手写一段 `cat > /kaniko/.docker/config.json`，它把登录变成了显式的一步。

- **`executor push <path> --destination <ref>`**：读取一个**已构建好的镜像**直接推送，完全跳过构建阶段。输入可以是 `--tar-path` 产出的 docker-save 格式 tar，也可以是 `--oci-layout-path` 产出的 OCI layout 目录；registry、认证、重试、digest-file 等参数与构建命令完全一致。限制是产物中必须**恰好包含一个镜像**，多镜像 tarball 和 index 不被支持。它带来的是“构建与发布解耦”，完美满足构建，扫描，推送的 CI 流水线：
  ```bash
  # 1. 只构建，不推送
  /kaniko/executor --tar-path=/tmp/image.tar --no-push ...

  # 2. 扫描产物
  trivy image --input /tmp/image.tar --exit-code 1

  # 3. 扫描通过后再推送，同一个二进制，无需额外工具
  /kaniko/executor push /tmp/image.tar --destination registry.example.com/myapp:latest
  ```

- **`executor bake <bakefile> [target]`** 从**一个多阶段 Dockerfile 一次构建多个镜像**。用一个 HCL bakefile 声明每个 target 对应哪个 stage、推到哪个地址，而 context、dockerfile、build-arg、cache、registry 这些设置仍然走常规 flag，并被所有 target 共享；还可以用 `--set` 覆盖某个 target 的 destination。这里有个容易被忽略的点：它是 `docker-bake.hcl` 在 kaniko 命令语义下的对应物，**并不兼容 buildx 的语法**，现成的 `docker-bake.hcl` 直接喂进去不会被解析。

  ```bash
  target "app" {
    destination = ["registry.example.com/app:latest"]
  }
  target "tools" {
    destination = ["registry.example.com/tools:latest"]
  }
  ```

顺带一提，工程化能力也在补：Layer Hints 会在某条指令往层里塞入不该进镜像的缓存目录时给出提示（例如 `/root/.cache/pip` 占了 184MB）；OpenTelemetry 追踪可以把每次构建变成一条 trace，用来定位哪条指令在反复打破缓存；断言失败时可以用 `KANIKO_IGNORE_ASSERTIONS` 临时绕过。这些在排查慢构建时比读日志有效得多。

## 6.4 使用建议

- **不改流水线、直接替换**：直接把 `gcr.io/kaniko-project/executor:vX.Y.Z` 换成 `ghcr.io/osscontainertools/kaniko:<tag>`，GitLab 场景用 `:debug` 变体并保留 `entrypoint: [""]`，原有参数基本兼容。
  > [!TIP]提示
  > 虽然是“drop-in”的社区版本，但社区版本还是做了部分修改，例如缓存键算法调整等，因此建议先在非生产环境用 `--dryrun` 跑一遍，确认 Dockerfile 的构建结果都正常，再切正式流水线。

- **新项目优先选社区版本**：如果确实需要 Kaniko 这种形态，即不依赖 daemon、不需要 privileged，也无需放开 seccomp 与 AppArmor，那么它相对 BuildKit 的真正差异就在于此。BuildKit 虽然也能在容器内以非 root 身份构建，但需要关闭 seccomp 和 AppArmor 才能创建嵌套容器；相比之下，Kaniko 不创建嵌套容器，因此无需调整这些内核开关，在与安全团队沟通时，强调这一差异比强调“社区很活跃”更有说服力。


## 6.5 替代方案对比
如果你最终决定不再使用 Kaniko，以下是两个主流替代路线：：
- **Buildah** 由红帽维护、活跃、无 daemon、不走 Dockerfile 也能构建，是 GitLab 当前文档的主要推荐，适合想彻底脱离 Kaniko 生态的团队；
- **rootless BuildKit** 是 Docker 官方路线，功能最全（尤其是多架构与 `--mount` 系列），但容器内运行仍需评估特权与内核安全配置的放开范围。还有两个硬性限制值得提前说明：Kaniko 本身不支持生成多架构 manifest，需要借助 `manifest-tool` 在多个架构上分别构建再合并；也不支持 Windows 容器。如果这两点在你的路线图里，选型时就应该重新考虑，而不是等到迁移中途才发现。

