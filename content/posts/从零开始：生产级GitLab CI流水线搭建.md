+++
date = '2026-10-30T15:59:39+08:00'
draft = false
title = '从零开始：生产级 GitLab CI 流水线搭建'
categories = ['DevOps']
tags = ['GitLab', 'CICD', 'Runner']
+++

本文以 Go 语言为案例，从「只做编译和测试」的最小流水线起步，逐步加上**构建镜像**、**镜像漏洞扫描**、**推送 Harbor**，最终形成五阶段完整生产级 GitLab CI 流水线。

下面是本文的实践路线：

1. 最小流水线，只做 `build` 和 `test` 阶段，让流水线先简单跑通。
2. 增加 `build-image` 阶段，用 Kaniko 按 Dockerfile 构建镜像。
3. 增加 `scan-image` 阶段，用 Trivy 扫描镜像，有高危漏洞直接退出。
4. 增加 `push` 阶段，扫描通过后把镜像推送并打上标签。
5. **汇总**：给出完整五阶段配置与生产最佳实践。


**为什么从头开始一点点增加步骤**?

直接给出五十行的完整配置的话，出了问题根本不知道是编译、测试、构建、扫描还是推送哪一个阶段的问题。先跑通最小闭环，每次增加一个阶段，有问题能立即验证，排查范围小，人也更有把握。

日常编写构建生产级别的流水线也建议这样一步步验证。

# 1. 最小流水线验证

先只配置两个 job：

- `build` 阶段：负责编译校验，确认代码能否正常打包、是否存在语法错误。
- `test` 阶段：运行单元测试。

这两个 job 是整个 CI 的基础，也是触发最频繁、最需要优先跑稳的部分。

## 1.1 为什么先只做这两步

- **反馈最快**：编译和单元测试在同一个代码环境中运行，几秒内就能拿到结果，不依赖镜像仓库或外部服务。
- **成本最低**：不构建镜像、不推送，也不做扫描，任何分支的提交都可以安全触发，不会产生多余产物。
- **优先保障代码质量**：如果编译或测试失败，就不应继续执行后续的构建、扫描和推送步骤。
## 1.3 写配置文件
在项目根目录新建 `.gitlab-ci.yml`，写入下面内容：
```yaml
stages:
  - build
  - test

variables:
  GO_VERSION: "1.26"

  # 国内须覆盖 GOPROXY，否则 go mod download 走 proxy.golang.org 会超时
  GOPROXY: "https://goproxy.cn,direct"
  GOSUMDB: "sum.golang.google.cn"

  # 镜像仓库地址
  HARBOR_REGISTRY: example.harbor.com
  IMAGE_NAME: ${HARBOR_REGISTRY}/devops/web-demo

  # 缓存目录
  GOMODCACHE: $CI_PROJECT_DIR/.cache/go-mod
  GOCACHE: $CI_PROJECT_DIR/.cache/go-build


# ============================================================================
# Stage 1: 服务构建
# ============================================================================
build:
  stage: build
  image: ${HARBOR_REGISTRY}/devops/golang:1.26-trixie
  variables:
    GOFLAGS: "-mod=mod"
  cache:
    key:
      files:
        - go.sum
    paths:
      - .cache/go-mod/
      - .cache/go-build/
    untracked: true
  before_script:
    - mkdir -p ${GOMODCACHE} ${GOCACHE}
  script:
    - go mod download
    - go build -o /dev/null ./...     # 只验证能否编译，不产出二进制
    - go vet ./...

# ============================================================================
# Stage 2: 单元测试
# ============================================================================
test:
  stage: test
  image: ${HARBOR_REGISTRY}/devops/golang:1.26-trixie
  cache:
    key:
      files:
        - go.sum
    paths:
      - .cache/go-mod/
      - .cache/go-build/
    untracked: true
  before_script:
    - mkdir -p ${GOMODCACHE} ${GOCACHE}
  script:
    - go mod download
    - go test -v -race -coverprofile=coverage.out ./...
  artifacts:
    when: always
    expire_in: 1 week
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.out
    paths:
      - coverage.out
```

**配置说明**：
- `variables`：全局变量定义，在整个流水线中共享，用于统一配置环境参数，避免硬编码。
- `before_script`：在核心脚本执行前，创建缓存目录，防止因目录不存在导致的报错。
- `script`：核心执行命令，真正用来执行构建和单元测试的命令。
- `artifacts`：配置构建产物。
  - `when: always`：无论测试是否通过，都保留产物，便于排查失败原因。
  - `expire_in: 1 week`：产物在 1 周后自动过期清理。
  - `reports: coverage_report`:：将 coverage.out 以 cobertura 格式解析并上报给 GitLab，使其能在合并请求（MR）页面直接展示覆盖率变化。
  - `paths`：将原始覆盖率文件作为附件保留，供人工下载查看。

> [!NOTE] 备注
> `go test -coverprofile` 生成的是 Go 原生格式，GitLab 的 `coverage_report` 只认 **Cobertura XML**。若想流水线页显示覆盖率，需用 `gocover-cobertura` 转换后再上报，否则覆盖率面板是空的（不影响流水线通过）。

## 1.4 验证
1. 把文件提交并推送到仓库（或点击流水线页的「Run pipeline」）。
2. 进入项目的 **CI/CD → Pipelines**，看到一条流水线在跑。
3. 等它结束，`build` 和 `test` 两个 job 都是绿色 **passed**。
4. 故意在代码里加一行语法错误再提交，`build` 应该变红。

**完成标准**：正常提交全绿，错误提交会在 `build` 阶段红掉并停止。

# 2. 增加构建镜像阶段

基础的构建和单元测试完成后，接着添加 `build-image` 阶段，这里使用 Kaniko 来构建容器镜像，并将镜像导出为本地 Tar 归档文件 `image.tar`。该镜像不会推送到镜像仓库，而是通过 GitLab CI Artifacts 传递给下一阶段进行安全扫描。

## 2.2 配套 Dockerfile
先在项目根目录创建一个多阶段构建的 Dockerfile：

```dockerfile
# --- 构建阶段 ---
FROM golang:1.26-trixie AS builder

# kaniko 不支持 buildkit 的 $TARGETARCH 自动变量，用 ARG 手动传
ARG TARGETARCH=arm64
# 关键：镜像内默认 GOPROXY 是 proxy.golang.org，国内必须覆盖
ARG GOPROXY=https://goproxy.cn,direct
ARG GOSUMDB=sum.golang.google.cn
ENV GOPROXY=${GOPROXY} \
    GOSUMDB=${GOSUMDB} \
    CGO_ENABLED=0 \
    GOOS=linux

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .

RUN GOARCH=${TARGETARCH} \
    go build -ldflags="-s -w" -o server .

# --- 运行阶段 ---
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive

RUN groupadd --gid 10001 appgroup && \
    useradd --uid 10001 --gid 10001 --shell /bin/false appuser

WORKDIR /app
COPY --from=builder --chown=10001:10001 /app/server /app/server
USER 10001
EXPOSE 8088
ENTRYPOINT ["/app/server"]
```
CI 的 build 是「快速把关」，这里的构建是为产出最终运行镜像。

## 2.3 配置 `build-image` job

在 `.gitlab-ci.yml` 文件中增加以下内容：
```yaml
stages:
  ...
  - build-image

variables:
  ...

  IMAGE_TAR: $CI_PROJECT_DIR/image.tar
  IMAGE_TAG: ${CI_COMMIT_SHORT_SHA}

# ============================================================================
# Stage 3: 构建镜像（输出 tar，不推送，给下个阶段镜像扫描使用）
# ============================================================================
build-image:
  stage: build-image
  image:
    name: ghcr.io/osscontainertools/kaniko:v1.28.6-debug
    entrypoint: [""]
  variables:
    KANIKO_CACHE: "true"
    KANIKO_CACHE_REPO: ${IMAGE_NAME}-cache
    KANIKO_CACHE_TTL: 24h
    KANIKO_COMPRESSION: gzip      # docker 格式只支持 gzip
    KANIKO_PLATFORM: linux/arm64
  before_script:
    - echo "{\"auths\":{\"${HARBOR_REGISTRY}\":{\"username\":\"${HARBOR_USERNAME}\",\"password\":\"${HARBOR_PASSWORD}\"}}}" > /kaniko/.docker/config.json
  script:
    - |
      /kaniko/executor \
        --context "${CI_PROJECT_DIR}" \
        --dockerfile "${CI_PROJECT_DIR}/Dockerfile" \
        --destination "${IMAGE_NAME}:${IMAGE_TAG}" \
        --build-arg "GOPROXY=${GOPROXY}" \
        --build-arg "GOSUMDB=${GOSUMDB}" \
        --build-arg "TARGETARCH=arm64" \
        --custom-platform "${KANIKO_PLATFORM}" \
        --image-format docker \
        --compression "${KANIKO_COMPRESSION}" \
        --cache="${KANIKO_CACHE}" \
        --cache-repo "${KANIKO_CACHE_REPO}" \
        --cache-ttl "${KANIKO_CACHE_TTL}" \
        --tar-path "${IMAGE_TAR}" \
        --no-push
  artifacts:
    paths:
      - image.tar
    expire_in: 1h
```
**配置说明**:

- **Kaniko 运行环境**：GitLab CI 必须使用 `debug` 镜像。普通 executor 镜像不含 shell，无法执行 `script`；`debug` 镜像内置 `busybox`，并需配置 `entrypoint: [""]` 清除默认入口，避免脚本被当作 executor 参数传入。
- `IMAGE_TAG`：**镜像标签**，使用 Git 提交的短 SHA，保证每次构建的镜像版本唯一且可追溯。
- `IMAGE_TAR`：**产物输出路径**，指定镜像归档的保存位置。
- **仓库认证**：使用 `before_script` 将 Harbor 的认证信息写入 Kaniko 所需的 Docker 配置文件，供访问镜像仓库及缓存仓库时使用。相关账号和密码需要在 GitLab CI/CD 页面上提前配置好。
- **构建参数**：`GOPROXY` 和 `GOSUMDB` 作为构建参数传入 Dockerfile，供 Go 项目配置代理和校验服务；`TARGETARCH=arm64` 与 `KANIKO_PLATFORM=linux/arm64` 用于构建 Linux ARM64 架构的镜像。
- **镜像格式**：使用 Docker 镜像格式，并采用 `gzip` 压缩。
- **缓存机制**：开启 Kaniko 缓存，并使用 `${IMAGE_NAME}-cache` 作为缓存仓库；缓存有效期为 24 小时，可减少重复依赖的下载与构建时间。
- **禁止推送镜像**：`--no-push` 确保构建结果只导出到本地，不会推送到 Harbor 或其他镜像仓库。
- **传递扫描文件**：生成的 `image.tar` 被保存为 Job Artifact，并自动传递给后续阶段。Artifact 的有效期为 1 小时。

## 2.4 验证
1. 提交后看 `build-image` job 是否通过。
2. 日志里应看到 `buildah build` 完成、`buildah push ... oci-archive` 成功，并产出 `image.tar` 产物。
3. 镜像此时只存在于本地归档，Harbor 里还没有任何新镜像，符合预期。

**完成标准**：构建 job 全绿，本地归档成功生成。

# 3. 增加镜像扫描阶段

Trivy 是 Aqua Security 开源的容器镜像漏洞扫描工具，可检测依赖组件及基础镜像中的已知 CVE。将镜像扫描接入 CI 流水线后，若发现高危或严重漏洞，扫描任务将以非零状态码退出并阻断后续流程，从而实现安全风险的提前发现与拦截，也就是所谓的「安全左移」。

## 3.1 配置 `scan-image` job

第 2 步已经生成本地镜像归档 `image.tar`，直接使用 Trivy 扫描该归档文件即可。

在 `.gitlab-ci.yml` 中添加以下配置：

```yaml
stages:
  ...
  - scan-image

# ============================================================================
# Stage 4：镜像安全扫描
# ============================================================================
scan-image:
  stage: scan-image
  image: aquasec/trivy:latest
  needs: ["build-image"]
  variables:
    # 待扫描的本地镜像归档路径
    IMAGE_TAR: ${CI_PROJECT_DIR}/image.tar
    # 使用国内镜像源加速漏洞数据库下载
    TRIVY_DB_REPOSITORY: docker.m.daocloud.io/aquasec/trivy-db:2
    # 不显示扫描进度条
    TRIVY_NO_PROGRESS: "true"
    # 扫描超时时间
    TRIVY_TIMEOUT: 15m

  script:
    # 生成 GitLab 容器扫描报告
    - trivy image --input ${IMAGE_TAR} --scanners vuln --format template --template "@/contrib/gitlab.tpl" -o gl-container-scanning-report.json
    # 生成 CycloneDX 格式的软件物料清单（SBOM）
    - trivy image --input ${IMAGE_TAR} --scanners vuln --format cyclonedx -o sbom.cdx.json
    # 输出 HIGH 和 CRITICAL 级别漏洞的文本报告
    - trivy image --input ${IMAGE_TAR} --scanners vuln --severity HIGH,CRITICAL --no-progress | tee trivy-report.txt
    # 发现 CRITICAL 级别漏洞时返回非零状态码，使流水线失败
    - trivy image --input ${IMAGE_TAR} --scanners vuln --severity CRITICAL --exit-code 1 --no-progress

  artifacts:
    when: always
    # 扫描产物保留一周
    expire_in: 1 week
    reports:
      # 接入 GitLab 的容器安全扫描报告界面
      container_scanning: gl-container-scanning-report.json

    paths:
      - gl-container-scanning-report.json
      - sbom.cdx.json
      - trivy-report.txt
```

**配置说明**：

- **`image: aquasec/trivy:latest`**：使用 Trivy 官方镜像执行扫描，生产环境建议将 `latest` 替换为经过验证的固定版本，以保证构建结果稳定。

- **`TRIVY_DB_REPOSITORY`**：指定 Trivy 漏洞数据库的下载地址。这里使用 DaoCloud 镜像源，可减少数据库下载耗时并提高访问稳定性。
- **`TRIVY_TIMEOUT`**：设置单次扫描操作的超时时间为 15 分钟，避免任务长时间阻塞流水线。
- **`--scanners vuln`**：仅执行漏洞扫描，不额外执行密钥扫描或依赖混淆检查，以缩短扫描时间。

- **GitLab 扫描报告**：使用 GitLab 提供的模板生成 `gl-container-scanning-report.json`，并通过 `container_scanning` 接入 GitLab 的容器安全扫描报告界面。

- **SBOM 报告**：将镜像中的软件包及版本信息输出为 CycloneDX 格式的 `sbom.cdx.json`，便于后续审计和追踪。

- **文本漏洞报告**：`trivy-report.txt` 仅记录 `HIGH` 和 `CRITICAL` 级别漏洞，并通过 `tee` 同时输出到控制台和报告文件。

- **流水线失败策略**：最后一条命令仅在发现 `CRITICAL` 级别漏洞时通过 `--exit-code 1` 返回失败状态，从而使当前任务及后续流水线阶段失败；其他级别的漏洞仅记录在报告中，不会直接阻断流水线。

- **Artifacts 保留策略**：`when: always` 确保扫描成功或失败后都能获取报告文件，`expire_in: 1 week` 则会在一周后自动清理这些产物，避免长期占用存储空间。

## 3.2 验证
1. 正常提交，`scan` 应通过。
2. 可临时给 scan 加 `allow_failure: true`，先看它会报出哪些漏洞，确认无误后去掉该行恢复强制阻断。

**完成标准**：：
- 正常镜像扫描成功，job 全绿；
- 故意使用一个带已知高危漏洞依赖的镜像，scan 应变红并退出流水线。

# 4. 增加镜像推送阶段

镜像扫描通过后，流水线会自动登录 Harbor，把构建阶段生成的本地镜像归档推送过去，并以 **Commit 短哈希**作为镜像标签，例如：

```text
harbor.example.com/project/app:a1b2c3d
```

使用 Commit 短哈希作为镜像标签，可以确保每次提交对应的镜像版本唯一且可追溯。

## 4.1 配置 `push-image` Job

在 `.gitlab-ci.yml` 中追加以下配置：

```yaml
stages:
  ...
  - push-image

...

push-image:
  stage: push-image
  image:
    name: ghcr.io/osscontainertools/kaniko:v1.28.6-debug
    entrypoint: [""]
  needs:
    - build-image
    - scan-image
  variables:
    IMAGE_TAG: ${CI_COMMIT_SHORT_SHA}
    IMAGE_TAR: ${CI_PROJECT_DIR}/image.tar
  before_script:
    - echo "{\"auths\":{\"${HARBOR_REGISTRY}\":{\"username\":\"${HARBOR_USERNAME}\",\"password\":\"${HARBOR_PASSWORD}\"}}}" > /kaniko/.docker/config.json
  script:
    - /kaniko/executor push ${IMAGE_TAR} -d ${IMAGE_NAME}:${IMAGE_TAG}
    - echo "已推送 = ${IMAGE_NAME}:${IMAGE_TAG}"
```

**配置说明**：

- **`needs`**：声明该阶段依赖 `build-image` 和 `scan-image`。只有镜像构建成功且安全扫描通过后，推送任务才会执行，从而避免将未扫描或扫描失败的镜像发布到镜像仓库。

- **`IMAGE_TAG`**：使用 `${CI_COMMIT_SHORT_SHA}` 作为镜像标签，将镜像与当前 Commit 唯一关联，便于版本追踪和问题回溯。

- **`IMAGE_TAR`**：指定本地镜像归档的完整路径。默认指向项目工作目录下的 `image.tar`，文件由 `build-image` 阶段生成。

- **`script`**：调用 Kaniko Executor，将 `image.tar` 推送到镜像仓库。


## 4.2 验证
1. 代码提交后，流水线全绿通过。
2. 打开 Harbor，看到 `myapp/app:短哈希` 镜像。

**完成标准**：扫描通过后镜像成功入库；若 scan 变红，push 不会执行，Harbor 里不会有新镜像。


# 5. 完整五阶段流水线

把上面四步合并，就是一份完整`.gitlab-ci.yml`配置：
```yaml
# ============================================================================
# web-demo CI/CD Pipeline
#
# 设计思路：
#   1. build 阶段  —— 快速验证编译，不产出任何产物（快速失败）
#   2. test  阶段  —— 单元测试 + 覆盖率
#   3. build-image —— 真正用 Dockerfile 构建镜像，输出 image.tar（不推送）
#   4. scan-image —— Trivy 扫描镜像（漏洞报告 + SBOM + CRITICAL 阻断）
#   5. push-image —— 扫描通过后才推送到镜像仓库（先扫描后入库）
# ============================================================================

stages:
  - build
  - test
  - build-image
  - scan-image
  - push-image

variables:
  GO_VERSION: "1.26"

  # 国内须覆盖 GOPROXY，否则 go mod download 走 proxy.golang.org 会超时
  GOPROXY: "https://goproxy.cn,direct"
  GOSUMDB: "sum.golang.google.cn"

  # 镜像仓库地址
  HARBOR_REGISTRY: example.harbor.com
  IMAGE_NAME: ${HARBOR_REGISTRY}/devops/web-demo

  # 缓存目录
  GOMODCACHE: $CI_PROJECT_DIR/.cache/go-mod
  GOCACHE: $CI_PROJECT_DIR/.cache/go-build

  IMAGE_TAG: ${CI_COMMIT_SHORT_SHA}
  IMAGE_TAR: ${CI_PROJECT_DIR}/image.tar


# ============================================================================
# Stage 1: 快速验证（不产出产物）
# ============================================================================
build:
  stage: build
  image: ${HARBOR_REGISTRY}/devops/golang:1.26-trixie
  variables:
    GOFLAGS: "-mod=mod"
  cache:
    key:
      files:
        - go.sum
    paths:
      - .cache/go-mod/
      - .cache/go-build/
    untracked: true
  before_script:
    - mkdir -p ${GOMODCACHE} ${GOCACHE}
  script:
    - go mod download
    - go build -o /dev/null ./...     # 只验证能否编译，不产出二进制
    - go vet ./...

# ============================================================================
# Stage 2: 测试
# ============================================================================
test:
  stage: test
  image: ${HARBOR_REGISTRY}/devops/golang:1.26-trixie
  cache:
    key:
      files:
        - go.sum
    paths:
      - .cache/go-mod/
      - .cache/go-build/
    untracked: true
  before_script:
    - mkdir -p ${GOMODCACHE} ${GOCACHE}
  script:
    - go mod download
    - go test -v -race -coverprofile=coverage.out ./...
  artifacts:
    when: always
    expire_in: 1 week
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.out
    paths:
      - coverage.out

# ============================================================================
# Stage 3: 构建镜像（kaniko，输出 tar，不推送，给下个阶段镜像扫描使用）
# ============================================================================
build-image:
  stage: build-image
  image:
    name: ghcr.io/osscontainertools/kaniko:v1.28.6-debug
    entrypoint: [""]
  variables:
    # kaniko 开关
    KANIKO_CACHE: "true"
    KANIKO_CACHE_REPO: ${IMAGE_NAME}-cache
    KANIKO_CACHE_TTL: 24h
    KANIKO_COMPRESSION: gzip          # docker 格式只支持 gzip
    KANIKO_PLATFORM: linux/arm64
  before_script:
    - echo "{\"auths\":{\"${HARBOR_REGISTRY}\":{\"username\":\"${HARBOR_USERNAME}\",\"password\":\"${HARBOR_PASSWORD}\"}}}" > /kaniko/.docker/config.json
  script:
    - |
      /kaniko/executor \
        --context "${CI_PROJECT_DIR}" \
        --dockerfile "${CI_PROJECT_DIR}/Dockerfile" \
        --destination "${IMAGE_NAME}:${IMAGE_TAG}" \
        --build-arg "GOPROXY=${GOPROXY}" \
        --build-arg "GOSUMDB=${GOSUMDB}" \
        --build-arg "TARGETARCH=arm64" \
        --custom-platform "${KANIKO_PLATFORM}" \
        --image-format docker \
        --compression "${KANIKO_COMPRESSION}" \
        --cache="${KANIKO_CACHE}" \
        --cache-repo "${KANIKO_CACHE_REPO}" \
        --cache-ttl "${KANIKO_CACHE_TTL}" \
        --tar-path "${IMAGE_TAR}" \
        --no-push
  artifacts:
    paths:
      - image.tar
    expire_in: 2h


# ============================================================================
# Stage 4: 镜像扫描
# ============================================================================
scan-image:
  stage: scan-image
  image: aquasec/trivy:latest
  needs: ["build-image"]
  variables:
    IMAGE_TAR: ${CI_PROJECT_DIR}/image.tar
    # 漏洞库国内镜像源加速：docker.m.daocloud.io
    TRIVY_DB_REPOSITORY: docker.m.daocloud.io/aquasec/trivy-db:2
    TRIVY_NO_PROGRESS: "true"
    TRIVY_TIMEOUT: 15m
  script:
    - echo "TRIVY_DB_REPOSITORY=${TRIVY_DB_REPOSITORY}"
    - trivy image --input ${IMAGE_TAR} --scanners vuln --format template --template "@/contrib/gitlab.tpl" -o gl-container-scanning-report.json
    - trivy image --input ${IMAGE_TAR} --scanners vuln --format cyclonedx -o sbom.cdx.json
    - trivy image --input ${IMAGE_TAR} --scanners vuln --severity HIGH,CRITICAL --no-progress | tee trivy-report.txt
    - trivy image --input ${IMAGE_TAR} --scanners vuln --severity CRITICAL --exit-code 1 --no-progress
  artifacts:
    when: always
    expire_in: 1 week
    reports:
      container_scanning: gl-container-scanning-report.json
    paths:
      - gl-container-scanning-report.json
      - sbom.cdx.json
      - trivy-report.txt


# ============================================================================
# Stage 4: 镜像推送
# ============================================================================
push-image:
  stage: push-image
  image:
    name: ghcr.io/osscontainertools/kaniko:v1.28.6-debug
    entrypoint: [""]
  needs:
    - build-image
    - scan-image
  variables:
    IMAGE_TAG: ${CI_COMMIT_SHORT_SHA}
    IMAGE_TAR: ${CI_PROJECT_DIR}/image.tar
  before_script:
    - echo "{\"auths\":{\"${HARBOR_REGISTRY}\":{\"username\":\"${HARBOR_USERNAME}\",\"password\":\"${HARBOR_PASSWORD}\"}}}" > /kaniko/.docker/config.json
  script:
    - /kaniko/executor push ${IMAGE_TAR} -d ${IMAGE_NAME}:${IMAGE_TAG}
    - echo "已推送 = ${IMAGE_NAME}:${IMAGE_TAG}"

```

# 6. 生产最佳实践与建议

## 6.1 流水线设计
- **按职责拆分 Job**：把构建、测试、扫描、推送和部署拆开，既方便定位问题，也能为不同阶段设置独立的权限与规则。
- **用 `needs` 精确控制依赖**：别让 Job 傻等整个阶段，只依赖真正需要的前置任务，可以明显缩短流水线耗时。部署任务要显式依赖安全扫描，避免绕过质量关卡。
- **用 `rules` 管理触发条件**：通过 `workflow: rules` 避免同一分支同时触发重复流水线；再结合 `rules:changes`，跳过与本次改动无关的任务，但关键分支的安全扫描不要随意跳过。
- **只在固定分支推送镜像**：使用 `rules` 将镜像推送限制在 `main` 或指定的发布分支，必要时再配合 Protected Branch，避免开发分支和特性分支污染镜像仓库。
- **镜像标签要唯一且可追溯**：优先使用 commit 短哈希，例如 `app:a1b2c3d`；每个提交对应一个唯一镜像，方便回滚和定位。业务版本号（如 `release-v1.3.0`）可以作为额外标签，供测试环境引用，但不要随意覆盖已经发布的标签。
- **关键扫描设为强制关卡**：扫描任务不要配置 `allow_failure: true`，并配置好漏洞等级和退出码。部署任务必须依赖扫描结果，才能让高危漏洞真正阻断发布。
- **给任务加上超时和重试**：`timeout` 可以避免任务卡死；网络拉取等临时性错误可以有限重试，但安全扫描、发布等关键任务不要无条件重试。
- **Runner 要按用途隔离**：通过 `tags` 为构建、扫描和部署任务选择合适的 Runner，不要让所有任务共用高权限执行环境。

## 6.2 安全与稳定性
- **敏感变量统一放到 CI/CD Variables**：密码、Token 和私钥不要写进 `.gitlab-ci.yml`。勾选 `Masked` 降低日志泄露风险，发布凭据还要勾选 `Protected`。
- **避免在日志中输出敏感信息**：关闭不必要的调试输出，不要主动 `echo` 变量，也不要在失败日志中打印完整凭据或认证文件。
- **控制 Fork 和外部 MR 的权限**：外部贡献者触发的流水线不要拿到生产密钥、部署 Token 或其他受保护变量，必要时直接禁止这些任务运行。
- **镜像仓库和云凭据尽量短期有效**：优先使用短期 Token、OIDC 或自动轮换的凭据，不要长期依赖固定账号密码；`CI_JOB_TOKEN` 也只授予实际需要的权限。
- **基础镜像固定版本**：不要使用 `golang:latest` 这类浮动标签，改为固定版本，例如 `golang:1.26-trixie`；条件允许时，最好进一步锁定镜像 digest，避免同名标签内容发生变化。
- **构建工具和依赖也要锁定版本**：Node、Go、Helm、Trivy 等工具尽量使用明确版本，避免本地能跑、CI 却因版本差异失败。
- **镜像归档及时清理**：为临时产物设置 `expire_in: 1h`，扫描或测试完成后自动删除，避免 GitLab 存储持续膨胀。产物中不要夹带密钥、配置文件等敏感内容。
- **Runner 权限遵循最小化原则**：普通构建任务不要使用特权容器或高权限 Shell；生产部署应放在隔离环境中，并限制 Runner 对宿主机和网络的访问。
- **外部依赖固定来源与版本**：脚本、Helm Chart 和其他下载项要来自可信仓库，并锁定版本或校验值，避免供应链内容被悄悄替换。

## 6.3 进阶方向
- **建立镜像安全基线**：明确高危漏洞的阻断规则、扫描超时时间、允许忽略的漏洞类型和责任人。`.trivyignore` 要按需维护，不能把高危问题一股脑全部忽略。
- **补齐源码和依赖扫描**：在镜像扫描之外，对代码目录执行 `trivy filesystem`，并逐步接入 Secret Detection、SAST 和依赖漏洞检查，形成多层防护。
- **保留完整的质量门禁**：在发布前加入单元测试、代码检查、覆盖率校验、镜像构建和漏洞扫描；可根据团队情况设置覆盖率或漏洞数量阈值。
- **打通 CI/CD 闭环**：镜像推送成功后衔接 Helm 或 Argo CD 部署，通过 Protected Environments 配置人工审批，自动执行发布、健康检查和失败回滚。
- **让部署过程可观测**：记录流水线耗时、失败原因、漏洞趋势和发布结果，并接入告警或通知。这样出了问题能快速定位，也能持续评估流水线质量。
- **定期复盘并清理规则**：定期检查过期的忽略项、失效凭据、长期未更新的依赖和重复 Job。流水线规则不是越多越好，关键是稳定、清晰并且真正能拦住风险。
