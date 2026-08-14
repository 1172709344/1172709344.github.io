---
layout: article
title: "把 YAML 放进 Git，就是 GitOps 了吗？我用四个问题判断"
description: "从声明式、版本化与不可变、自动拉取、持续调谐四项原则出发，区分 Git 存储、Git 驱动交付与严格意义上的 Pull-based GitOps。"
date: 2026-08-14
category: Platform Engineering / GitOps
reading_time: 10
canonical: /articles/is-it-gitops-if-it-is-stored-in-git/
permalink: /articles/is-it-gitops-if-it-is-stored-in-git/
---

我在评审一套平台架构时，经常听到一句话：

> 我们的 YAML 已经全部放到 Git 里了，所以现在已经是 GitOps。

这句话不能说完全错，但它把“使用 Git”与“形成 GitOps 闭环”混在了一起。

Git 解决的是版本管理、协作和审计问题。GitOps 还要继续回答：谁把 Git 中的期望状态带到目标环境？谁持续观察实际状态？有人绕过 Git 改了线上资源之后，系统会发生什么？

所以，我不会先问团队“你们是不是 GitOps”，而会先问四个更具体的问题。

## 我的结论：存入 Git 是起点，不是终点

严格意义上的 GitOps，通常应同时具备四项能力：

1. **Declarative**：系统的期望状态以声明式配置表达。
2. **Versioned and Immutable**：期望状态被版本化保存，历史完整且不可随意篡改。
3. **Pulled Automatically**：软件代理从期望状态来源自动拉取声明。
4. **Continuously Reconciled**：软件代理持续观察实际状态，并尝试让它回到期望状态。

这四项原则中，把文件放入 Git，主要覆盖了第二项；如果 Git 中保存的是声明式配置，还可以覆盖第一项。但第三项“自动拉取”和第四项“持续调谐”，不会因为仓库里多了一个 `deployment.yaml` 就自动出现。

我更愿意把常见实践分成三个层级：

| 层级                     | 典型形态                       | Git 的角色             | 变更如何进入环境                      | 漂移如何处理                   |
| ------------------------ | ------------------------------ | ---------------------- | ------------------------------------- | ------------------------------ |
| Git-backed configuration | YAML、脚本或参数保存在 Git     | 版本仓库和审计记录     | 人工执行或从控制台操作                | 通常依赖人工发现               |
| Git-driven delivery      | 合并 PR 后由 CI/CD 主动部署    | 期望状态来源和流程入口 | GitHub Actions、Jenkins 等从外部 Push | 定时扫描、下次流水线或人工修复 |
| Pull-based GitOps        | 集群内 Controller 持续关注 Git | 期望状态的权威来源     | Controller 自动 Pull 并 Reconcile     | 持续告警或自动恢复             |

这三层都可能是合理的工程选择。问题不在于第二层是不是“足够高级”，而在于团队不能用第三层的名字，掩盖第二层尚未具备的能力。

## 同一份 YAML，可能对应三种完全不同的系统

假设 Git 中保存了下面这份声明：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-api
spec:
  replicas: 3
```

仅看文件内容，我们无法判断系统是不是 GitOps。真正决定系统性质的是围绕这份文件建立的控制回路。

### 场景一：Git 只是存储位置

工程师合并配置后，在本地执行：

```text
git pull
kubectl apply -f deployment.yaml
```

这里已经比“只在控制台里点”更好：配置有历史，变更可以 review，也可以找到责任边界。但 Git 仍然只是文件来源，最终执行依赖某个人的电脑、权限和操作时机。

如果有人在线上把副本数改成 5，Git 不会知道，环境也不会自动恢复。

### 场景二：Git 驱动 Push-based delivery

合并 PR 后，GitHub Actions 或 Jenkins 执行：

```text
Git -> CI/CD runner -> kubectl apply -> Kubernetes
```

这时，交付过程已经自动化。我们还可以增加审批、策略检查、manifest render、live diff、审计证据和回滚门禁。它完全可以是一套严谨、可靠的持续交付系统。

但从控制方向看，仍然是集群外的流水线拿着写权限向目标环境 Push。线上副本数被改成 5 以后，如果没有独立的 drift detection，系统可能一直保持 5；即使有定时扫描，恢复也取决于下一次 workflow 何时运行。

### 场景三：Pull-based continuous reconciliation

目标环境中运行 Argo CD、Flux 或其他符合相同控制模型的 Controller：

```text
Git <- Controller 主动拉取
          |
          v
   Compare desired/live
          |
          v
     Reconcile drift
```

Controller 持续看到 Git 期望副本数为 3、线上实际副本数为 5。根据配置的 reconciliation policy，它会发出告警，或自动把副本数恢复为 3。

这一步才是 GitOps 与普通“Git + 自动部署”最关键的分水岭：**Git 不只是触发一次部署，而是持续参与系统控制。**

## 我用四个问题判断，而不是看工具名称

### 一、Git 中保存的是“期望状态”，还是“操作步骤”？

声明式配置描述系统最终应该是什么样子：

```text
checkout-api 应运行 3 个副本
```

命令式脚本描述人或机器应该依次做什么：

```text
先登录服务器
再停止旧进程
复制文件
执行启动命令
```

把一组 Bash 脚本存进 Git，当然能获得版本历史，但脚本本身不一定构成可持续比较的 desired state。Controller 很难仅凭“执行过哪些步骤”判断当前系统是否已经偏离目标。

所以我首先看的是配置模型，不是仓库地址。

### 二、历史真的版本化且不可变吗？

“使用 Git”也有不同质量。

如果团队允许 force push、直接修改默认分支、覆盖 release tag，或者部署时依赖仓库外的一组可变参数，那么 Git 中的记录未必能完整还原当时部署的状态。

我会继续检查：

- 生产变更是否必须通过 PR 和 review。
- 默认分支是否受到保护。
- release tag 和制品是否禁止覆盖。
- 部署是否绑定明确 commit，而不是浮动分支名。
- Git 外部参数是否同样有版本和审计记录。

不可变不是一句口号，而是仓库规则、制品规则和审批规则共同形成的约束。

### 三、到底是谁发起变更？

这是最容易被流程图隐藏的问题。

```text
Push-based：CI/CD -> Cluster API
Pull-based：Controller -> Git / Artifact source
```

在 Push-based 模型里，CI/CD runner 通常需要目标环境写权限。流水线是部署执行者，集群是被操作对象。

在 Pull-based 模型里，集群内 Controller 主动获取已批准的期望状态。CI 可以负责测试、扫描、render 和生成 evidence，但通常不需要直接持有生产集群写权限。

两者的权限模型、网络边界、故障模式和灾备方式都不同。架构文档如果只写“通过 GitOps 部署”，却不画出箭头方向，我会认为这个设计还没有说清楚。

### 四、发生漂移以后，谁负责闭环？

这是我最看重的问题。

设想一次很常见的应急操作：值班人员为了止损，临时把 Deployment 副本数从 3 调到 5。事件结束后，Git 仍然写着 3。

接下来可能有四种结果：

1. 没有人发现，线上长期保持 5。
2. 下次发布时被新的 `kubectl apply` 顺带覆盖。
3. 定时 drift workflow 发现差异，再由人决定是否修复。
4. Controller 持续检测差异，并按策略告警或自动恢复。

前两种没有形成漂移治理闭环；第三种是可以落地的补偿机制；第四种才是 continuous reconciliation。

我不要求所有资源都必须开启自动恢复。生产环境里，有些高风险资源更适合先告警、再人工确认。但无论采用哪种 policy，系统都应该明确回答：检测频率是什么、谁收到告警、谁有权接受或修复漂移、紧急变更如何回写 Git。

## 为什么“广义 GitOps”仍然有价值

在真实组织里，“GitOps”经常被用作一个更宽泛的 umbrella term，泛指：

- Git 作为变更入口。
- Pull Request 作为审批载体。
- 配置即代码。
- 自动化测试和部署。
- 用 Git 历史完成审计与回滚。

我理解这种用法。它帮助团队从控制台手工操作走向可 review、可追踪的工程流程，本身就是重要进步。

但我会给“广义 GitOps”加一个前提：**在架构和风险讨论中，必须把具体实现写出来。**

例如，与其写：

```text
本平台采用 GitOps 部署。
```

不如写：

```text
本阶段采用 Git-as-source、PR-governed、Push-based continuous delivery。
GitHub Actions 在审批后向目标集群应用非敏感资源；
漂移由定时 workflow 检测，尚未引入集群内持续调谐 Controller。
```

第二种表述没有第一种听起来简洁，却让权限、安全、漂移和恢复边界一目了然。对架构师来说，这种准确性比术语显得先进更重要。

## Push-based 不是错误答案

我不赞成把 GitOps 讨论变成工具或流派之争。

迁移初期选择 Push-based delivery，可能有非常现实的理由：

- 现有 CI/CD 和审批体系已经成熟。
- 团队需要先做 shadow diff，再逐步接管在线资源。
- 管理 Controller 自身的 bootstrap 和灾备方案尚未确定。
- 某些目标环境暂时不允许安装额外 Controller。
- 团队希望先建立资源 allowlist、人工门禁和变更证据。

这些都可以是负责任的阶段性决策。我的要求只有两个：

1. 准确命名当前架构，不借 GitOps 标签隐藏能力缺口。
2. 对漂移检测、集群写权限、失败恢复和下一阶段演进作出显式设计。

一个边界清楚的 Push-based 平台，往往比一个只装了 Argo CD、却允许所有人绕过 Git 直接修改生产环境的平台更可靠。

## 装了 Argo CD，也不自动等于做好了 GitOps

GitOps 是操作模型，不是产品安装清单。

即使集群里已经运行 Argo CD，如果 Application 长期关闭自动同步、团队习惯在 UI 里直接改参数、紧急变更从不回写 Git、关键资源被设置为永久忽略差异，那么整个系统仍可能没有建立可信的 desired-state 闭环。

反过来，工具也不只 Argo CD。Flux 或其他 Controller 同样可以实现 Pull 和 Reconcile。判断标准应该回到控制模型：声明在哪里，谁拉取，谁比较，谁调谐。

## 我建议的渐进式落地路径

不是所有团队都要一步跳到最终形态。我更倾向按能力逐层补齐。

### 第一阶段：先让 Git 成为可信变更入口

- 将声明式配置纳入 Git。
- 生产变更通过 PR、review 和受保护分支。
- 禁止 release ref 和不可变制品被覆盖。
- 消除只存在于个人电脑和控制台里的关键参数。
- 建立 Git commit、构建制品与部署记录之间的追踪关系。

### 第二阶段：建立受治理的 Push-based delivery

- 由 CI/CD 统一执行 render、validate、diff 和 deploy。
- 对生产环境设置审批、资源 allowlist 和最小权限身份。
- 将部署失败、部分成功和回滚流程设计成明确状态机。
- 增加定时 drift detection，并定义告警与修复责任。
- Secret 使用专门的 secret manager，不以明文进入 Git 或日志。

### 第三阶段：引入 Pull-based reconciliation

- 在目标环境中部署并保护 GitOps Controller。
- 让 Controller 拉取经过批准的期望状态。
- 按资源风险定义 auto-sync、self-heal、prune 与人工审批策略。
- 将 CI 的职责收敛到测试、策略、安全扫描和变更证据。
- 补齐 Controller bootstrap、自身升级、凭证轮换、监控和灾备。

这条路径的重点不是“终于有资格叫 GitOps”，而是每一步都减少一个明确的风险：不可审计变更、人工部署错误、流水线过度授权、配置漂移或恢复依赖个人经验。

## 一张表完成架构自检

| 检查项                                     | 评估结果 | 否时意味着什么                     |
| ------------------------------------------ | -------- | ---------------------------------- |
| Git 中保存可比较的声明式期望状态           | 待评估   | 可能只是脚本版本管理               |
| 生产变更受 PR、review 和分支保护约束       | 待评估   | 历史可信度与审批边界不足           |
| 部署绑定明确 commit 和不可变制品           | 待评估   | 同一次部署难以精确复现             |
| 目标环境中的 Agent 自动拉取声明            | 待评估   | 当前仍是人工或 Push-based delivery |
| Agent 持续比较 desired state 与 live state | 待评估   | 漂移可能长期不可见                 |
| 漂移有明确的告警或自动恢复策略             | 待评估   | 发现问题后无法形成闭环             |
| 紧急线上变更会回写 Git                     | 待评估   | Git 可能失去权威事实源地位         |
| CI 不需要持有生产集群广泛写权限            | 待评估   | 需要额外控制 runner 与凭证风险     |
| Controller 自身有升级、监控和灾备方案      | 待评估   | GitOps 控制面可能成为新的单点风险  |

如果前两三项已经完成，我会称它为 Git-backed 或 Git-governed delivery；如果由 CI 主动部署，我会明确写 Push-based；当自动拉取与持续调谐也形成闭环时，再称为严格意义上的 Pull-based GitOps。

## 最后：不要用一个名词代替架构设计

“存到 Git 就是 GitOps 吗？”

我的答案是：**不是。它是必要基础之一，但不是充分条件。**

GitOps 真正改变的，不是 YAML 存放在哪里，而是谁持续负责让现实世界接近期望状态。

作为 DevOps 架构师，我更关心的从来不是系统能不能贴上一个流行标签。我关心的是：Git 是否真的是可信事实源，变更是否可审计，执行身份是否最小权限，线上漂移是否可见，以及系统在没有某个工程师手工介入时，能不能稳定回到我们声明的状态。

当这些问题都有明确答案，GitOps 才不只是仓库里的一批文件，而是一套真正运行起来的控制系统。

## 公开参考

- [OpenGitOps：GitOps Principles v1.0.0](https://opengitops.dev/)
