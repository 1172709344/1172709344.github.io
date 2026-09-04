---
layout: article
title: "为什么 GitHub Actions 有 50 多个 Jobs，Runner 调度器却只看见 30 个？"
description: "一次动态 self-hosted runner 排障实践：从 queued Job、Jobs API 默认分页和 matrix 展开入手，建立调度器完整性检查、临时止血与长期修复方法。"
date: 2026-09-04
category: DevOps / Platform Engineering
reading_time: 12
canonical: /articles/github-actions-runner-dispatcher-pagination/
permalink: /articles/github-actions-runner-dispatcher-pagination/
---

有一类 GitHub Actions 故障很容易把人带错方向：

- workflow 中的大部分 Jobs 正常；
- 只有某个云环境或某组 matrix Jobs 一直 `queued`；
- 页面显示 “Waiting for a runner to pick up this job”；
- 动态 runner 服务本身还在运行；
- 应用和集群也没有明显异常。

第一反应通常是查 runner label、云资源配额、容器启动日志，甚至直接查 Helm 和 Kubernetes。但这次真正的问题发生得更早：调度器压根没有看见那些 Jobs。

原因很朴素，也很隐蔽：GitHub Actions Jobs API 默认每页只返回 30 条，调度器只读取了第一页。

## 先说结论：total_count 不等于当前数组长度

GitHub Actions 的 Jobs 列表响应可以简化成：

```json
{
  "total_count": 53,
  "jobs": ["...当前页的 Jobs..."]
}
```

这里最危险的误解是：看到 `total_count=53`，就以为当前 `jobs` 数组也包含 53 条。

实际上，如果请求没有指定分页参数，当前数组通常只有第一页的 30 条：

```text
服务端总数: 53
当前响应数组: 30
调度器实际可见范围: 第 1-30 个 Job
目标 Job 所在位置: 第 40 多位
```

于是形成一种非常迷惑的现场：

- GitHub 正确生成了 Job；
- Job 正确携带动态 runner label；
- Dispatcher 轮询没有报错；
- 云平台也没有创建失败记录；
- 但 Job 永远等不到 runner。

不是创建 runner 失败，而是创建流程从未被触发。

## 为什么只有部分 Jobs 受影响

问题通常在 matrix workflow 扩大后出现。

例如：

```yaml
strategy:
  matrix:
    region: [region-a, region-b, region-c]
    service: [api, web, worker, scheduler]
    environment: [test, stage, production]
```

Matrix 会在调度前展开成多个独立 Jobs。再加上 lint、test、build、security scan 和多个部署阶段，一个 workflow 很容易超过 30 个 Jobs。

如果动态 runner Jobs 恰好排在第一页，它们会正常运行；当新的 service、region 或检查项把它们推到第 31 位以后，同一套代码突然就“偶发失效”。

这也是为什么问题容易被误判成：

- 某个云环境不稳定；
- 某个 runner label 有问题；
- 某个 matrix 组合不受支持；
- 某个时间段云 API 创建资源失败。

真正决定是否触发的变量，可能只是 Job 在 API 列表中的位置。

## max-parallel 为什么救不了这个问题

有人会尝试：

```yaml
strategy:
  max-parallel: 1
  matrix:
    # ...
```

它只能限制同时执行多少个 Jobs，不能减少 matrix 展开后的 Jobs 总数。

假设 workflow 仍然生成 53 个 Jobs，那么即使一次只运行 1 个：

- 第 1 至 30 个仍在第一页；
- 第 31 至 53 个仍在第二页；
- 只读第一页的调度器仍然看不到后面的 Jobs。

因此，`max-parallel` 可以降低资源峰值，但不能修复数据读取不完整。

## 用三层状态机定位，不要一上来查应用

动态 runner 链路至少有三个独立状态层：

```text
GitHub Actions Job 状态
          |
          v
Dispatcher 扫描与匹配状态
          |
          v
临时 runner 资源生命周期
```

它们回答的是三个不同问题：

| 层次       | 核心问题                           | 典型证据                           |
| ---------- | ---------------------------------- | ---------------------------------- |
| GitHub     | Job 是否已生成并等待 runner？      | status、labels、runner name、steps |
| Dispatcher | 调度器是否读取并匹配了 Job？       | run、Job ID、label、分页数量日志   |
| Runner     | 临时资源是否创建、注册并领取 Job？ | 资源状态、注册日志、runner name    |

只有 Job 获得 runner 并开始 steps 后，才应该进入第四层：部署脚本、集群和应用。

一个实用判断是：

```text
runner_name 为空 + steps 为空
= Job 尚未开始执行
= Helm、kubectl 和应用代码不是当前第一排查对象
```

## 第一步：完整读取 Jobs

下面的 Bash 示例依赖 GitHub CLI 和 `jq`。`gh api --paginate` 会沿 GitHub 返回的 `Link` header 读取全部页面，`--slurp` 再把各页响应合并为一个 JSON 数组：

```bash
repo='<owner>/<repository>'
run_id='<workflow-run-id>'

pages="$(
  gh api --paginate --slurp \
    "/repos/${repo}/actions/runs/${run_id}/jobs?per_page=100"
)"

all_jobs="$(jq -c '[.[].jobs[]]' <<<"${pages}")"
total_count="$(jq -r '.[0].total_count // 0' <<<"${pages}")"
collected="$(jq 'length' <<<"${all_jobs}")"
page_count="$(jq 'length' <<<"${pages}")"

printf 'TotalCount=%s\nCollected=%s\nPages=%s\n' \
  "${total_count}" "${collected}" "${page_count}"
```

然后加入一个不能省略的断言：

```bash
if (( collected != total_count )); then
  printf 'Incomplete Jobs response: collected %s of %s\n' \
    "${collected}" "${total_count}" >&2
  exit 1
fi
```

这个断言会把“静默丢数据”变成显式故障。

没有它，调度器可能持续输出“扫描成功”，实际上每次都在处理不完整的数据。

## 第二步：找到目标 Job 的全局位置

完整收集后，列出等待动态 runner 的 Jobs：

```bash
dynamic_label='<dynamic-runner-label>'

jq -r --arg label "${dynamic_label}" '
  ["POSITION", "ID", "NAME", "RUNNER_NAME", "STEPS"],
  (
    to_entries[]
    | select(.value.status == "queued")
    | select((.value.labels // []) | index($label))
    | [
        (.key + 1),
        .value.id,
        .value.name,
        (.value.runner_name // ""),
        ((.value.steps // []) | length)
      ]
  )
  | @tsv
' <<<"${all_jobs}"
```

如果所有异常 Jobs 都位于第 31 位以后，而 dispatcher 日志只出现前 30 个，因果链已经很强：

```text
目标 Job 已由 GitHub 生成
          +
Job 位于默认分页之后
          +
Dispatcher 仅记录第一页
          +
没有任何 runner 创建事件
          =
分页遗漏导致 Job 未被调度
```

## 第三步：让调度器记录数据完整性

仅记录 “Found 53 jobs” 不够。这个数字可能只是 `total_count`，并不代表代码收到了 53 个对象。

每轮扫描至少应该记录：

```text
repository
run_id
total_count
returned_count
page_count
queued_dynamic_job_count
```

建议直接建立不变量：

```text
returned_count == total_count
```

如果不成立：

- 不应该继续把本次结果当成完整扫描；
- 应该输出 request ID 和分页信息；
- 应该产生告警；
- 不应该把读取失败解释成“没有待处理 Job”。

## 临时修复：per_page=100 有用，但别把它叫完整分页

最小热修复通常是：

```text
/repos/{owner}/{repo}/actions/runs/{run_id}/jobs?per_page=100
```

它能覆盖 31 至 100 个 Jobs 的常见场景，改动也很小，适合快速恢复业务。

但它有明确边界：第 101 个 Job 以后仍然会被遗漏。

因此文档和变更说明应该准确表述为：

```text
短期：把单页上限提高到 100，恢复当前规模
长期：实现完整分页，覆盖任意合理规模
```

把 `per_page=100` 描述成“已经解决分页”会给未来留下同样的事故，只是触发阈值从 30 变成 100。

## 长期修复：遍历 Link，而不是猜最大数量

更稳妥的实现应遵循 GitHub 返回的 `Link` header，持续读取 `rel="next"`，直到没有下一页。

伪代码如下：

```go
func listAllJobs(ctx context.Context, firstURL string) ([]Job, error) {
  jobs := make([]Job, 0)
  nextURL := firstURL
  totalCount := -1

  for nextURL != "" {
    response, err := requestJobsPage(ctx, nextURL)
    if err != nil {
      return nil, err
    }

    if totalCount < 0 {
      totalCount = response.TotalCount
    }

    jobs = append(jobs, response.Jobs...)
    nextURL = response.NextURL
  }

  if totalCount >= 0 && len(jobs) < totalCount {
    return nil, fmt.Errorf(
      "incomplete jobs response: collected %d of %d",
      len(jobs), totalCount,
    )
  }

  return jobs, nil
}
```

实现时还要考虑：

- 429 和 5xx 的有界重试；
- `Retry-After`；
- context timeout；
- 每页 request ID；
- 响应解码失败；
- workflow 状态在分页期间变化；
- rerun attempt 对 Job 列表语义的影响。

分页不是在 URL 后加一个参数就结束了，它是一份数据完整性契约。

## 别忘了另一个陷阱：去重必须允许失败后重试

动态 runner 调度器通常每隔几秒轮询一次。为了防止同一个 Job 被重复创建多个 runner，需要并发去重：

```text
key = repository + run_id + job_id
```

但去重只能解决一半问题。

如果逻辑是：

1. 先写入 “正在创建” key；
2. 异步创建 runner；
3. 创建失败；
4. key 永远不删除；

那么一次瞬时失败就会变成永久不重试。

正确的状态至少要区分：

```text
not-started -> creating -> registered -> completed
                    |
                    v
                  failed -> retryable
```

工程上需要做到：

- 创建函数返回明确的 `error`；
- 创建失败后释放或带 TTL 保留 key；
- 成功注册后按生命周期更新状态；
- 超时任务能够被重新评估；
- 多个轮询并发时仍只能有一个创建者。

只有去重、没有失败恢复，会把“重复创建风险”换成“永久卡住风险”。

## 临时止血时，最重要的是避免重复部署

修复调度器后，之前 queued 的 Job 可能立刻被扫描并开始执行。

如果你同时手工触发了一个替代部署 workflow，就可能出现两个部署并行执行。

安全的止血顺序是：

1. 确认原 run 和受影响 Jobs；
2. 取消原 run，或明确保证目标 Jobs 不会继续执行；
3. 选择一个恢复路径；
4. 仅触发受影响 cloud/environment/service；
5. 验证实际部署版本；
6. 再部署 dispatcher 修复。

可选恢复路径包括：

- 使用已有的单环境手工 workflow；
- 由有权限的 owner 重新运行特定失败范围；
- 先部署 `per_page=100` 热修复，再创建一个新的 run；
- 在批准的变更窗口内临时使用固定 runner。

没有仓库或生产平台写权限时，不要通过替代凭证绕过边界。把证据、最小操作和回滚条件交给有权限的操作者执行。

## 发布成功，不等于生产正在运行新版本

动态 runner dispatcher 通常还跨越源码、镜像和运行时资源三层：

```text
源码 commit
    -> 镜像构建
    -> 镜像 tag/digest
    -> 部署 workflow
    -> 运行中的 dispatcher image
```

以下证据都不能单独完成验收：

- PR 已合并；
- 镜像构建 workflow 成功；
- 部署 workflow 成功；
- IaC apply 返回成功。

最终必须从运行中的 ECI、Pod、VM 或 service 读取实际 image tag/digest，并确认：

- 版本与批准版本一致；
- 资源处于 Running/Ready；
- 重启次数正常；
- 新日志已经包含分页数量；
- 第 31 位之后的 Job 能获得 runner。

还要检查定时部署任务。一个隐藏的旧版本变量，可能在几小时后把刚修好的 dispatcher 覆盖回旧镜像。

## 测试应该覆盖哪些边界

分页缺陷最适合使用边界值测试，而不是只测一个常规响应。

建议最小集合：

| Jobs 数 | 目的                        |
| ------- | --------------------------- |
| 0       | 空 workflow 或过滤后无结果  |
| 1       | 最小正常响应                |
| 30      | 默认页边界                  |
| 31      | 必须进入第二页的第一个场景  |
| 100     | 最大单页边界                |
| 101     | `per_page=100` 热修复失效点 |
| 150+    | 多页聚合与数量断言          |

同时测试：

- 第二页请求失败；
- 第二页 JSON 解码失败；
- GitHub 返回 429；
- 第一次 runner 创建失败，下轮轮询成功；
- 两次扫描同时命中同一 Job；
- workflow 在分页期间被取消；
- 已完成 Job 不再创建 runner。

## 一套可复用的排障顺序

遇到动态 self-hosted runner Job 长时间 queued，可以按以下顺序：

### 1. 看 Job 是否开始

检查 `runner_name` 和 `steps`。都为空时，暂时不要查部署步骤。

### 2. 完整分页读取 Jobs

比较 `total_count` 与收集数量，找到目标 Job 的全局位置。

### 3. 对照 dispatcher 日志

确认日志是否出现目标 run、Job ID 和 label，而不只是 workflow 总数。

### 4. 再查 runner 资源

只有 dispatcher 已发起创建，才进入云资源、注册、网络、配额和镜像排障。

### 5. 控制止血并发

取消原 queued run，避免替代 workflow 与自动恢复同时部署。

### 6. 验证运行版本

从实际运行资源读取 image tag/digest，并用第 31 位之后的 Job 做回归。

## 最后：把分页当成可靠性问题

这次问题表面上只是漏了一个 query 参数，背后却是一个更普遍的工程事实：

> 调度器对外部 API 的读取完整性，和资源创建成功率一样重要。

一个没有报错的 200 响应，不代表数据完整；一个仍在 Running 的 dispatcher，也不代表它看见了所有待处理任务。

真正可靠的实现需要同时具备：

- 完整分页；
- 数量不变量；
- 有界重试；
- 可观测的 request/page 信息；
- 并发去重；
- 失败后可恢复；
- 运行版本验收；
- 超过分页边界的真实回归。

当这些机制都存在时，“等待 runner”才不再是一条模糊提示，而会变成一条可以逐层定位、快速止血、持续验证的工程链路。

## 公开参考

- [GitHub REST API：List jobs for a workflow run](https://docs.github.com/en/rest/actions/workflow-jobs?apiVersion=2022-11-28#list-jobs-for-a-workflow-run)
- [GitHub REST API：Using pagination in the REST API](https://docs.github.com/en/rest/using-the-rest-api/using-pagination-in-the-rest-api)
