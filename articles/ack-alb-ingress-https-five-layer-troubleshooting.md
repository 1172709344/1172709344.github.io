---
layout: article
title: "ACK ALB Ingress 开启 HTTPS 仍不可用：从 503 到 HTTPS 恢复的五层排障法"
description: "从 AlbConfig listener、Ingress 关联、Service 网络模式、TLS 证书到应用路径，系统拆解 ACK ALB Ingress HTTPS 故障，并给出可复用的验证命令和决策表。"
date: 2026-08-21
category: DevOps / Kubernetes
reading_time: 15
canonical: /articles/ack-alb-ingress-https-five-layer-troubleshooting/
permalink: /articles/ack-alb-ingress-https-five-layer-troubleshooting/
---

在一次已经脱敏的生产故障中，我们遇到了一个很容易误判的现象：

- Deployment 正常；
- Service 正常；
- ArgoCD 已经 `Synced`；
- 域名 DNS 也指向目标 ALB；
- 但 HTTP 返回 503，HTTPS 仍不可用。

如果只看 Kubernetes 工作负载，很容易把注意力放在 Pod 日志、应用端口或健康检查上。但实际问题连续跨越了三个层次：

1. Ingress 没有显式关联 HTTP 80 和 HTTPS 443 listener；
2. 后端 Service 类型与集群网络模式不兼容；
3. TLS Secret 中的证书不覆盖业务域名。

每修复一层，下一层错误才真正出现。这类故障最值得沉淀的，不是某一行 YAML，而是一个稳定的排障顺序。

## 先说结论：HTTPS 可用是五层契约，不是一个开关

我现在会用下面五层检查 ACK ALB Ingress：

| 层次      | 核心对象        | 必须成立的契约                          | 常见症状                       |
| --------- | --------------- | --------------------------------------- | ------------------------------ |
| 1. 入口层 | AlbConfig       | 存在所需的 HTTP/HTTPS listener          | `listener is not exist in alb` |
| 2. 路由层 | Ingress         | 关联 listener，配置 host、path、backend | listener 存在但规则不生效      |
| 3. 后端层 | Service         | 类型与 Flannel/Terway、服务器组模式兼容 | `just support eni mode`        |
| 4. 加密层 | TLS Secret/证书 | Secret 完整，证书 SAN 覆盖访问域名      | `curl (60)`、主机名不匹配      |
| 5. 应用层 | Pod/API         | 测试路径真实存在并返回预期状态          | 网络已通但 `/` 返回 404        |

这五层中，任何一层失败，都可能表现为“HTTPS 不可用”。因此不要从一个状态码直接跳到根因。

## 第一层：AlbConfig 负责创建 listener

ACK ALB Ingress 中，AlbConfig 管理 ALB 实例与 listener。Ingress 本身不能凭空创建一个 ALB listener。

一个同时提供 HTTP 和 HTTPS 的 AlbConfig 至少需要：

```yaml
apiVersion: alibabacloud.com/v1
kind: AlbConfig
metadata:
  name: alb
spec:
  listeners:
    - port: 80
      protocol: HTTP
    - port: 443
      protocol: HTTPS
```

如果只有 HTTP 80，即使 Ingress 中已经写了 `spec.tls`，HTTPS 443 也不会自动变得可用。

常见 Event 是：

```text
listener is not exist in alb, port: 443, protocol: HTTPS
listener not found for (443/HTTPS)
```

这里要注意一个职责边界：

- AlbConfig 定义 ALB 有哪些入口；
- Ingress 的 `spec.rules` 定义 host、path、backend 等规则内容；
- `listen-ports` 决定这些规则被下发到哪些入口。

只完成前者还不够。

## 第二层：Ingress 必须关联多个 listener

### 这是必须的吗？

**是。** 当一个 Ingress 需要同时承载 HTTP 80 和 HTTPS 443 时，必须显式声明：

```yaml
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
```

原因很简单：AlbConfig 只负责创建 80 和 443 listener，Ingress 的 `spec.rules` 只定义 host、path 和 backend；标准 Ingress API 并没有说明这些规则应该下发到哪个 ALB listener。`listen-ports` 就是 Controller 用来建立“路由规则 -> listener”映射的配置。`spec.tls` 只声明证书，也不能替代这层绑定。

省略该注解时，443 listener 即使已经存在，也可能没有当前 Ingress 的转发规则，导致 HTTPS 无法命中后端。阿里云官方 FAQ 因此明确要求：使用多个监听时，需要通过该 annotation 进行关联。

如果只开放 HTTPS，可以仅配置 `'[{"HTTPS": 443}]'`，不要依赖省略注解后的默认行为。

如果业务要求所有 HTTP 请求强制切到 HTTPS，再增加：

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/ssl-redirect: "true"
```

完整 annotation 是：

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "true"
```

`ssl-redirect` 的前提是 AlbConfig 已配置 HTTPS 443 listener，并且 Controller 能通过 Secret、自动发现或 AlbConfig 指定证书的方式获得有效证书。否则 Controller 没有可供跳转的 HTTPS 入口，还可能出现 `empty https listener default certs`。

配置生效后，HTTP 验证应看到：

```bash
curl -I --max-time 20 http://api.example.com/
```

预期结果：

```text
HTTP/1.1 308 Permanent Redirect
Location: https://api.example.com/
```

这里的验收重点不是“收到一个 3xx”，而是：

- 状态码是预期的永久重定向；
- `Location` 指向正确域名；
- scheme 已变为 `https`；
- 没有错误地跳到 ALB 默认域名或其他 host。

## 第三层：Service 类型必须匹配网络插件

这是本次故障中最有迷惑性的一层。

Ingress listener 修好后，Controller 开始构建后端模型，随后出现：

```text
FailedApplyModel
cluster service type just support eni mode for alb ingress
```

这种错误通常意味着：

- Service 是 `ClusterIP`；
- 当前集群不满足 ENI/IP 后端模式；
- Controller 无法把这个 Service 转换为可用的 ALB 服务器组。

### Flannel 与 Terway 的差异

简化理解：

| 网络模式      | 常见后端方式                            | Service 类型                                 |
| ------------- | --------------------------------------- | -------------------------------------------- |
| Flannel       | ALB 挂载 ECS/Node，再经 NodePort 到 Pod | `NodePort` 或 `LoadBalancer`                 |
| Terway ENI/IP | ALB 直接挂载 Pod IP                     | 可使用 `ClusterIP`，并按要求配置 IP 服务器组 |

ACK 官方文档明确指出：使用 Flannel 时，ALB Ingress 后端 Service 仅支持 `NodePort` 和 `LoadBalancer`。

因此，Flannel 场景下的最小改动通常是：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: production
spec:
  selector:
    app: api
  ports:
    - port: 443
      targetPort: 8080
  type: NodePort
```

不需要手工指定 `nodePort`。让 Kubernetes 自动分配，可以减少端口冲突风险。

### 不要把 Service port 当成后端协议

下面这段配置：

```yaml
ports:
  - port: 443
    targetPort: 8080
```

只表示：

- Service 对内暴露端口 443；
- 流量转发到 Pod 端口 8080。

它不自动表示 ALB 到 Pod 使用 HTTPS。后端协议由 Ingress Controller 配置决定。不要仅根据端口号推断协议，也不要在没有证据时增加：

```yaml
alb.ingress.kubernetes.io/backend-protocol: https
```

如果 Pod 的 8080 实际只提供 HTTP，而你把后端协议设成 HTTPS，ALB 会因为协议不匹配返回 502。

### 一个重要风险：不要随意切换存量 Service 类型

看到 `ClusterIP` 不兼容后，直接改成 `NodePort` 看起来很自然，但并非所有生产场景都可以原地切换。

ACK ALB Ingress FAQ 提醒：服务器组类型一经创建无法更改。在 Flannel 网络模式下，如果修改 Service 类型导致后端从 IP 类型切换为 ECS 类型，原服务器组可能无法接收新的后端。

因此要分两种情况：

#### 新建或尚未成功建模的 Ingress

只有确认对应服务器组尚未创建时，才可以评估原地修正 Service 类型。`FailedApplyModel` 本身不能证明服务器组不存在；如果服务器组已经创建，即使 Ingress 尚未承载生产流量，也应优先新建 Service，避免服务器组类型不一致导致持续调谐失败。

#### 已经承载生产流量的存量 Ingress

不要直接切换。优先评估：

1. 新建一个符合目标模式的 Service；
2. 等待新 Service Endpoints/NodePort 准备完成；
3. 在变更窗口内把 Ingress 指向新 Service；
4. 验证新服务器组健康；
5. 保留可快速切回旧 Service 的回滚路径；
6. 稳定后再清理旧 Service。

这是一个典型例子：同样一行 `type: NodePort`，在“尚未成功创建”与“已经承载流量”两种状态下，风险完全不同。

## 第四层：TLS Secret 存在，不代表证书一定正确

当 listener、Ingress 和 Service 都修好以后，HTTPS 端口可能已经可以连接，但标准 curl 仍然失败：

```text
curl: (60) SSL certificate problem
```

Windows Schannel 常见提示是：

```text
SEC_E_WRONG_PRINCIPAL
The target principal name is incorrect
```

这通常不是网络问题，而是 ALB 返回的证书 SAN/CN 不覆盖请求域名。

Ingress 的 TLS 配置应类似：

```yaml
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
```

Secret 需要满足：

- 与 Ingress 位于同一 namespace；
- 类型为 ACK ALB Ingress 支持的 `kubernetes.io/tls` 或 `IngressTLS`；
- 包含匹配的 `tls.crt` 与 `tls.key`；
- 证书 SAN 覆盖 `api.example.com`；
- 证书链完整且未过期。

本案例使用 Secret 证书。ACK ALB Ingress 也支持自动发现证书和在 AlbConfig 中指定证书 ID，使用时应按所选方式分别检查证书来源和调谐状态。

### `curl -k` 应该怎么用

`-k` 会跳过证书校验。它不是生产验收方式，但在排障中非常有用。

如果下面的命令失败：

```bash
curl -I -v --max-time 20 https://api.example.com/
```

而这个命令能收到应用响应：

```bash
curl -k -I -v --max-time 20 https://api.example.com/
```

可以得到一个很有价值的分层结论：

- 443 listener 已建立；
- TLS 握手至少能进行到服务端证书阶段；
- Ingress 路由和后端可能已经可达；
- 剩余问题集中在证书信任、主机名或证书链。

修复证书后，最终验收必须回到不带 `-k` 的命令。

## 第五层：404 可能说明网络已经通了

很多人看到 HTTPS 返回 404，会继续判断“ALB 还没修好”。这不一定正确。

假设请求：

```bash
curl -v --max-time 20 https://api.example.com/
```

返回：

```text
HTTP/1.1 404
Content-Type: application/json
```

如果能根据响应体、应用特征响应头或应用日志确认 404 来自目标应用，可以判断请求已经穿过 DNS、TLS、listener 和 Ingress 路由，并到达应用处理链路。仅凭 `Content-Type: application/json` 不能完成这一归因。

因此要区分两类 404：

| 404 来源             | 含义             | 下一步                               |
| -------------------- | ---------------- | ------------------------------------ |
| ALB/Ingress 默认响应 | 规则可能未命中   | 检查 host、path、rule priority       |
| 应用 JSON/业务响应   | 网络链路通常已通 | 改用真实 API 或 health endpoint 验证 |

链路验证可以接受应用层 404 作为“请求已到达应用”的证据，但业务验收不能停在这里。最终应该请求真实接口，例如：

```bash
curl -v --max-time 20 https://api.example.com/actuator/health
```

具体路径以应用实际提供的健康检查或业务 API 为准。

## 为什么 ArgoCD Synced 仍然可能不可用

ArgoCD 的 `Synced` 表示 Git 中的 desired state 与 Kubernetes live state 一致。它没有承诺：

- 云 Controller 已经成功创建所有外部资源；
- ALB listener 已经完成配置；
- 服务器组后端全部健康；
- TLS Secret 中的证书匹配域名；
- 应用路径返回 200。

因此需要至少同时看三类状态：

```text
Git revision / Sync status
          +
Kubernetes resource health / Events
          +
Public data-plane probe
```

一个实用的 ArgoCD 检查命令是：

```bash
APP_NAME=my-app
argocd app get "$APP_NAME" --hard-refresh
```

如果 Argo CD Server 位于需要 gRPC-Web 或子路径访问的代理之后，再按实际部署追加 `--grpc-web` 和 `--grpc-web-root-path`。

重点关注：

- revision 是否已经更新到目标提交；
- Application 是否 `Synced`；
- Ingress 是否 `Healthy` 或长期 `Progressing`；
- operation 是否 `Succeeded`；
- Ingress Events 是否仍在增长。

`Progressing` 持续时间过长时，不要反复点击 Sync。优先读取 Events，因为 Controller 通常已经给出了下一步最直接的证据。

## 一套可以直接复用的排障顺序

### Step 1：确认 DNS 与目标 ALB

```bash
dig api.example.com CNAME +short
dig api.example.com A +short
```

检查 CNAME 和 A 记录是否指向预期 ALB。DNS 正确只能证明入口寻址正确，不能证明 listener 与路由正常。

### Step 2：确认 AlbConfig listener

```bash
ALBCONFIG_NAME=alb
kubectl get albconfig "$ALBCONFIG_NAME" -o yaml
```

检查是否同时存在：

```yaml
listeners:
  - port: 80
    protocol: HTTP
  - port: 443
    protocol: HTTPS
```

### Step 3：确认 Ingress 关联

先检查 Ingress 使用的 IngressClass：

```yaml
ingressClassName: alb
```

再读取 IngressClass：

```bash
INGRESS_CLASS=alb
kubectl get ingressclass "$INGRESS_CLASS" -o yaml
```

确认 `spec.controller` 为 `ingress.k8s.alibabacloud/alb`，且 `spec.parameters` 中的 `apiGroup`、`kind` 和 `name` 指向预期 AlbConfig。完整关联链应为：

```text
Ingress.spec.ingressClassName
  -> IngressClass.spec.parameters
     -> AlbConfig
```

最后检查 listener 关联和重定向注解：

```yaml
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
alb.ingress.kubernetes.io/ssl-redirect: "true"
```

### Step 4：读取 Ingress Events

```bash
NAMESPACE=default
INGRESS_NAME=example-ingress
kubectl describe ingress "$INGRESS_NAME" -n "$NAMESPACE"
```

优先识别这些错误：

```text
listener is not exist in alb
cluster service type just support eni mode
empty https listener default certs
```

每种错误对应不同层次，不要用同一个修复动作处理。

### Step 5：确认 Service 与网络插件

```bash
NAMESPACE=default
SERVICE_NAME=backend-service
kubectl get service "$SERVICE_NAME" -n "$NAMESPACE" -o yaml
kubectl get endpoints "$SERVICE_NAME" -n "$NAMESPACE"
```

检查：

- Service 是 `ClusterIP`、`NodePort` 还是 `LoadBalancer`；
- 集群使用 Flannel 还是 Terway；
- 是否显式配置 IP 类型服务器组；
- Service selector 是否有 Ready Endpoints；
- 服务器组是否已经存在，类型切换是否安全。

### Step 6：验证 HTTP redirect

```bash
curl -I --max-time 20 http://api.example.com/
```

预期 `308` 和正确的 `Location`。

### Step 7：验证 HTTPS 与证书

```bash
curl -I -v --max-time 20 https://api.example.com/
```

最终验收不能使用 `-k`。

### Step 8：验证真实应用路径

```bash
HEALTH_OR_API_PATH=actuator/health
curl -v --max-time 20 "https://api.example.com/${HEALTH_OR_API_PATH}"
```

记录状态码、关键响应头和应用响应。

## 错误到检查项的速查表

| 现象或错误                     | 最可能的层次      | 首要检查                                |
| ------------------------------ | ----------------- | --------------------------------------- |
| 443 timeout                    | listener/网络入口 | AlbConfig 443、ALB 状态、ACL/安全策略   |
| `listener is not exist in alb` | AlbConfig         | 对应 port/protocol 是否存在             |
| listener 存在但规则不生效      | Ingress           | `listen-ports`、IngressClass、host/path |
| `just support eni mode`        | Service/网络插件  | ClusterIP、Flannel/Terway、服务器组模式 |
| HTTP 503                       | 路由或后端        | host/path、Endpoints、服务器组健康      |
| HTTP 308                       | redirect 已生效   | 继续验证 Location 与 HTTPS              |
| `curl (60)`                    | TLS 证书          | SAN/CN、证书链、Secret、SNI             |
| `curl -k` 成功、标准 curl 失败 | TLS 证书          | 不要继续改 Service，先修证书            |
| HTTPS 返回应用 JSON 404        | 应用路径          | 改用真实 health/API 路径                |
| ArgoCD `Synced / Progressing`  | 外部 Controller   | 读取 Ingress Events                     |

## 修复时我会坚持的四个原则

### 1. 一次只修一层

先修 listener 关联，再观察 Events；不要同时改 listener、Service、证书和应用端口。否则即使恢复，也很难证明哪个改动真正有效。

### 2. GitOps 状态与公网探测必须成对出现

每次变更都要保留两类证据：

```text
ArgoCD revision + Sync/Health
curl status + Location/TLS/application response
```

只有控制面证据，没有数据面证据，结论不完整。

### 3. 不手工修改 ACK 托管 ALB

AlbConfig 创建的 ALB 属于 ACK 托管资源，Controller 会周期性调谐。直接在 ALB 控制台改 listener、规则或证书，可能被覆盖，还会制造 desired/live 之外的隐性漂移。

### 4. `-k` 只用于诊断，不能用于验收

跳过证书校验可以帮助定位层次，但它把最重要的 HTTPS 身份校验关掉了。最终结果必须由标准 TLS 校验确认。

## 可复用的 TCA 模式

| 字段    | 内容                                                                                         |
| ------- | -------------------------------------------------------------------------------------------- |
| Trigger | ACK ALB Ingress 长期 `Progressing`，HTTP 503 或 HTTPS 不可用                                 |
| Check   | AlbConfig listener、Ingress listener 关联、Service 类型与网络插件、TLS SAN/SNI、真实应用路径 |
| Action  | 从控制面到数据面逐层检查；每修复一层，重新读取 Events 并执行标准 curl                        |
| 反事实  | 只看 ArgoCD `Synced` 或只看 443 连通，会遗漏 Controller、证书或应用层问题                    |

## 结语

ACK ALB Ingress 的 HTTPS 故障很少只属于“证书问题”或“Ingress 问题”。它更像一组连续契约：

```text
AlbConfig 创建入口
  -> Ingress 关联入口
     -> Service 提供兼容后端
        -> TLS 证明服务身份
           -> 应用响应真实路径
```

排障效率最高的做法，不是一次性修改更多 YAML，而是让每个验证结果回答一个明确问题：

- 443 listener 创建了吗？
- Ingress 真的使用它了吗？
- Controller 能把 Service 建模成服务器组吗？
- 客户端拿到的是正确证书吗？
- 请求最终到达应用了吗？

当这五个问题都有可执行证据时，“HTTPS 已恢复”才是一个可以被复核的工程结论。

## 参考资料

- [创建并使用 ALB Ingress 对外暴露服务](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/create-and-use-alb-ingress-to-expose-services-to-the-public)
- [ALB Ingress 配置词典](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/alb-ingress-configuration-dictionary)
- [通过 AlbConfig 配置 ALB 实例](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/use-albconfigs-to-configure-alb-instances)
- [配置 HTTPS 证书以实现加密通信](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/use-an-alb-ingress-to-configure-certificates-for-an-https-listener)
- [ALB Ingress 服务高级用法](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/advanced-alb-ingress-configurations)
- [ALB Ingress FAQ](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/alb-ingress-faq)
- [Argo CD Application Health](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/)
- [`argocd app get` 命令参考](https://argo-cd.readthedocs.io/en/stable/user-guide/commands/argocd_app_get/)
