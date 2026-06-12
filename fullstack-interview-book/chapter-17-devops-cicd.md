# 第十七章：DevOps 与工程化

很多候选人在算法、框架题上准备充分，但在 DevOps、CI/CD、交付流程、监控值班这些“工程化能力”上明显薄弱。可现实里，资深 Android、前端、Node.js、全栈工程师能否独立负责项目，往往就体现在这些题上。面试官通过这一章的问题，本质上在判断：你是“会写代码的人”，还是“能把软件可靠交付到生产环境的人”。

---

## 一、Git 工作流与版本控制

### 1. Git Flow、GitHub Flow、Trunk-Based Development 有什么区别？

| 工作流 | 核心分支 | 特点 | 适用场景 |
|---|---|---|---|
| Git Flow | `main`、`develop`、feature、release、hotfix | 流程完整，分支较重 | 发布节奏明确、传统团队 |
| GitHub Flow | `main` + 短期分支 + PR | 简洁、强调持续部署 | Web 团队、频繁上线 |
| Trunk-Based Development | 主干开发（trunk）+ 极短生命周期分支 | 强调小步快跑与快速集成 | 高效 CI/CD、平台团队 |

**面试回答建议：**

- Git Flow 适合版本发布复杂、审批链较长的组织；
- GitHub Flow 适合 SaaS 与持续交付；
- Trunk-Based Development 更依赖自动化测试、feature flags、代码评审纪律。

面试加分点：不是说哪个“更高级”，而是说**组织结构、发布节奏、风险承受能力**决定工作流选择。

### 2. Merge 与 Rebase 有什么区别？什么时候用？

**Merge**

- 保留完整分叉历史；
- 不重写历史；
- 可能产生 merge commit。

**Rebase**

- 把提交“平移”到新基线之上；
- 历史更线性；
- 会重写提交历史。

**常见建议：**

- 本地整理个人分支时可用 rebase；
- 已共享分支谨慎 rebase；
- 保护主分支时通常通过 PR merge 完成。

一句高质量回答：

> 我把 rebase 当作“整理个人工作历史”的工具，把 merge 当作“保留团队协作历史”的工具。

### 3. Cherry-pick、Interactive Rebase、Bisect 分别用于什么？

**`git cherry-pick`**

- 选择性把某个提交摘到当前分支；
- 常用于 hotfix 回迁、多分支补丁同步。

**`git rebase -i`**

- 整理提交历史；
- squash、reword、fixup；
- 在发 PR 前让提交更可读。

**`git bisect`**

- 二分定位引入 bug 的提交；
- 对“最近某次改动导致回归”极其高效。

面试中可举实战例子：

> 某性能回退问题出现于过去两周，我会先写可重复验证脚本，然后用 `git bisect run` 自动二分定位。

### 4. 什么是 Conventional Commits？

Conventional Commits 是一种约定式提交规范，典型格式：

```text
feat(auth): add refresh token rotation
fix(api): handle null user profile
chore(ci): cache pnpm store
docs(book): update chapter 17
```

常见类型：

- `feat`
- `fix`
- `docs`
- `refactor`
- `test`
- `chore`
- `perf`
- `build`
- `ci`

价值：

- 自动生成 changelog；
- 规范团队协作；
- 便于语义化版本发布。

### 5. Monorepo 与 Polyrepo 如何选？

| 维度 | Monorepo | Polyrepo |
|---|---|---|
| 代码发现性 | 高 | 中 |
| 跨项目重构 | 容易 | 成本更高 |
| CI 复杂度 | 高 | 中 |
| 权限隔离 | 较弱 | 更容易 |
| 依赖统一 | 强 | 较难 |

**Monorepo 适合：**

- 前端 + Node.js + shared package 强耦合；
- 需要统一 lint、test、build；
- 团队有 Nx、Turborepo、Bazel、pnpm workspace 等基础设施能力。

**Polyrepo 适合：**

- 组织边界明确；
- 服务独立性强；
- 权限和发布节奏差异大。

---

## 二、CI/CD 基础与 GitHub Actions

### 6. 你如何解释 CI 与 CD？

- **CI（Continuous Integration，持续集成）**：频繁合并代码，通过自动化 lint、test、build 尽早发现问题。
- **CD（Continuous Delivery / Deployment）**：
  - Delivery：代码随时可发布；
  - Deployment：通过流水线自动发布到生产。

面试官关心的不只是概念，而是你是否理解 **“缩短反馈回路”** 才是 CI/CD 的核心价值。

### 7. 一条成熟的流水线一般有哪些阶段？

高频标准链路：

```text
lint -> unit test -> integration test -> build -> security scan -> package -> deploy -> smoke test
```

若时间有限，至少要能讲清楚：

1. 静态检查；
2. 单元测试；
3. 构建产物；
4. 部署；
5. 发布后验证。

### 8. GitHub Actions Workflow 基本结构是什么？

典型 YAML：

```yaml
name: ci
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm test
```

面试应会解释：

- `on`：触发条件；
- `jobs`：并行或串行任务；
- `steps`：具体执行步骤；
- `uses`：复用 action；
- `run`：执行 shell 命令。

### 9. 什么是 Matrix Build？为什么有用？

Matrix build 是让同一工作流在多个维度组合下运行，例如：

```yaml
strategy:
  matrix:
    node: [18, 20]
    os: [ubuntu-latest, windows-latest]
```

用途：

- 多版本 Node.js 兼容性验证；
- 多平台构建；
- Android 不同 API / JDK 组合验证。

### 10. GitHub Actions 缓存应该怎么做？

常见缓存对象：

- `npm` / `pnpm` / `yarn` 依赖；
- Gradle 缓存；
- Docker layer；
- 编译中间产物。

但缓存不是越多越好，需要权衡：

- key 是否稳定；
- 命中率是否高；
- 恢复时间是否大于压缩/上传时间；
- 是否会引入脏缓存问题。

### 11. Secrets 在 CI 中如何安全使用？

建议回答：

- 存在 GitHub Secrets、环境密钥管理系统中；
- 最小权限原则；
- 不打印到日志；
- 按环境隔离（dev/staging/prod）；
- 优先使用短期凭证或 OIDC，而不是长期静态密钥。

这道题可直接体现你的安全意识。

### 12. 什么是 Reusable Workflows？

当多个仓库或多个 workflow 重复使用同一套逻辑时，可封装成可复用 workflow：

- 降低复制粘贴；
- 统一标准；
- 便于集中升级。

在大型组织中，它是平台工程（platform engineering）的常见实践。

---

## 三、测试体系与质量门禁

### 13. 什么是测试金字塔（Testing Pyramid）？

```text
        E2E
     Integration
       Unit Test
```

含义：

- 单元测试数量最多、速度最快；
- 集成测试验证模块协作；
- 端到端测试覆盖核心用户路径，但数量少、维护成本高。

高质量回答不是背图，而是说明：

- 为什么不能全靠 E2E；
- 为什么单测不能完全替代集成测试；
- 不同层次失败反馈速度不同。

### 14. Code Coverage 应该怎么看？

覆盖率（coverage）不是越高越好，而是要看：

- 核心路径是否被覆盖；
- 错误分支、边界条件是否覆盖；
- 覆盖率是否被“无意义测试”刷高；
- 是否结合 mutation testing、质量门禁。

你可以说：

> 我更关注关键模块的有效覆盖率，而不是盲目追求全局 100%。

### 15. 什么是 Quality Gate？

质量门禁指代码合入或发布前必须满足的一组条件，例如：

- lint 通过；
- 单测通过；
- 覆盖率不低于阈值；
- 关键安全扫描无高危；
- 构建成功；
- 审核通过。

现实里，质量门禁是工程纪律的自动化体现。

---

## 四、Docker 面试高频题

### 16. Dockerfile 最佳实践有哪些？

重点一定要会：

1. **多阶段构建（multi-stage build）**
2. **利用层缓存（layer caching）**
3. **最小化基础镜像**
4. **避免以 root 运行**
5. **减少镜像内容**

示例：

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["dist/index.js"]
```

### 17. 为什么多阶段构建重要？

因为它能把构建工具链与运行时分离：

- 构建阶段需要编译器、依赖、源码；
- 运行阶段只需要最终产物。

收益：

- 镜像更小；
- 攻击面更小；
- 拉取更快；
- 发布更稳。

### 18. Docker Compose 用于什么？

Docker Compose 适合本地开发和集成测试场景，用一份声明式配置启动多容器：

- app
- database
- redis
- message queue

例如本地 Node.js + Postgres + Redis 联调。  
面试中可强调：**Compose 更偏开发与测试环境，生产集群通常看 Kubernetes 或其他编排系统。**

### 19. Alpine、Distroless 镜像怎么选？

- **Alpine**：小、常见、包管理方便，但 musl libc 可能带来兼容性坑；
- **Distroless**：更精简、更安全、无 shell，适合生产运行；
- **Debian/Ubuntu**：兼容性更强，镜像通常更大。

### 20. 容器网络怎么理解？

关键点：

- 容器有自己的网络命名空间；
- 同一个 Docker network 中容器可通过服务名互通；
- 端口映射把容器端口暴露到宿主机；
- 容器到外部网络通常通过 NAT。

面试时，能把“容器内监听 0.0.0.0 与 localhost 的区别”讲清楚会加分。

---

## 五、Kubernetes 基础

### 21. Pod、Deployment、Service、Ingress 分别是什么？

| 资源 | 作用 |
|---|---|
| Pod | 最小部署单元，包含一个或多个容器 |
| Deployment | 管理 Pod 副本、滚动更新、自愈 |
| Service | 给一组 Pod 提供稳定访问入口 |
| Ingress | 管理 HTTP/HTTPS 七层路由 |

**面试常见错误**：把 Service 当成“负载均衡器本体”。更准确地说，Service 是服务发现与访问抽象，底层可能配合 kube-proxy、iptables/ipvs、云 LB。

### 22. ConfigMap 与 Secret 有什么区别？

- ConfigMap：普通配置；
- Secret：敏感信息，如 token、密码、证书。

但要说明：

- Secret 默认只是 base64 编码，不等于真正加密；
- 生产环境通常还会结合 KMS、External Secrets、Vault。

### 23. 什么是 HPA（Horizontal Pod Autoscaler）？

HPA 根据指标自动扩缩容 Pod 副本数，常见依据：

- CPU 使用率；
- 内存；
- 自定义业务指标；
- QPS、队列长度。

成熟回答会提到：

- 扩容不解决所有问题；
- 冷启动、连接预热、数据库瓶颈都可能成为限制。

### 24. Liveness、Readiness、Startup Probe 有什么区别？

- **Liveness Probe**：我是不是已经挂了，需要重启？
- **Readiness Probe**：我是否准备好接流量？
- **Startup Probe**：我是不是还在慢启动阶段，不要太早判死。

这道题非常高频，尤其在 Java/Node.js/Android 后端岗位里。

### 25. 滚动发布时为什么 readiness 很关键？

因为 Deployment 在滚动更新时需要知道新 Pod 何时真正可接流量。  
如果 readiness 配置不当，就会出现：

- 新实例还没初始化完成就开始接流量；
- 旧实例过早摘除导致容量下降；
- 短时间大量 5xx。

---

## 六、监控、日志与告警

### 26. Prometheus 有哪些指标类型？

必须熟：

- Counter：单调递增计数器
- Gauge：可增可减瞬时值
- Histogram：分桶统计
- Summary：分位数近似统计

**高频追问**：Histogram 与 Summary 怎么选？

- Histogram 适合聚合后统一算分位数；
- Summary 由客户端直接计算分位数，跨实例聚合不友好。

### 27. PromQL 面试会问什么？

常见问题：

- QPS：`rate(http_requests_total[5m])`
- 错误率：`sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))`
- P95 延迟：基于 histogram bucket 计算

你不一定要背很多语法，但要知道：

- `rate` 用于 counter；
- 聚合要分清 label；
- 仪表盘和告警常依赖窗口期计算。

### 28. Grafana 的价值是什么？

Grafana 不是“只是画图工具”，而是：

- 统一展示多数据源；
- 构建团队共享仪表盘；
- 把系统状态可视化；
- 支撑发布观测、值班排障。

好的回答会提到“黄金指标（Golden Signals）”：延迟、流量、错误、饱和度。

### 29. ELK / Loki 这类日志系统如何比较？

**ELK**

- Elasticsearch + Logstash + Kibana；
- 能力完整，搜索强；
- 成本与运维复杂度较高。

**Loki**

- 标签索引 + 对象存储；
- 与 Grafana 集成好；
- 通常更轻。

### 30. 告警系统应该如何设计，避免“告警风暴”？

原则：

- 只对可行动事件告警；
- 以用户影响为中心；
- 合理设置窗口和去抖；
- 分级：info / warning / critical；
- 合并同类告警；
- 提供 runbook。

一句高质量表达：

> 没有处理动作的告警，本质上不是告警，而是噪音。

---

## 七、基础设施即代码（IaC）

### 31. Terraform 的核心概念有哪些？

- Provider：云平台或服务提供方插件；
- Resource：具体资源声明；
- State：当前基础设施状态快照；
- Plan：变更预览；
- Apply：执行变更。

面试关键点：

- 为什么需要 state；
- 为什么 state 要安全存储和加锁；
- 为什么 Terraform 适合“声明式资源管理”。

### 32. Terraform 的 State 为什么敏感？

因为 state 中可能包含：

- 资源 ID；
- 网络信息；
- 甚至明文敏感值。

生产实践：

- 远端存储；
- 访问控制；
- 加锁；
- 状态备份；
- 最小权限。

### 33. Ansible 适合什么场景？

Ansible 更偏配置管理与远程执行：

- 批量装软件；
- 下发配置；
- 滚动执行运维操作；
- 初始化机器。

它与 Terraform 常见分工：

- Terraform：创建资源；
- Ansible：配置资源。

---

## 八、部署策略

### 34. Blue-Green Deployment 是什么？

蓝绿发布维护两套几乎等价环境：

- Blue：当前生产；
- Green：新版本。

切流时把流量从蓝切到绿。  
优点：回滚快。  
缺点：成本高，需要双倍环境。

### 35. Canary Deployment 是什么？

金丝雀发布是先给一小部分流量打到新版本，观察指标：

- 错误率；
- 延迟；
- 业务转化率；
- 资源占用。

若一切正常，再逐步扩大流量。

### 36. Rolling Update 与 Blue-Green 有何不同？

Rolling Update 是逐批替换旧实例，不需要整套双环境，成本低，但回滚与状态一致性更复杂。  
Blue-Green 回滚最直接，但资源成本更高。

### 37. Feature Flags 为什么重要？

Feature Flags（功能开关）允许“代码发布”和“功能上线”解耦。

典型收益：

- 灰度放量；
- 快速关闭问题功能；
- 针对用户群做 A/B test；
- 降低 trunk-based development 风险。

常见平台：

- LaunchDarkly
- Unleash
- 自研配置中心

---

## 九、综合工程化面试题

### 38. 如果让你设计一条前端 / Node.js 项目的 CI/CD，你会怎么做？

可以按下面结构回答：

1. PR 触发：lint、type check、unit test；
2. 合并主干：build、integration test、镜像构建；
3. 安全扫描：依赖漏洞、镜像扫描、secret scan；
4. 部署到 staging；
5. smoke test；
6. 审批后生产发布；
7. 发布后监控与自动回滚。

### 39. 发布失败后怎么回滚？

取决于部署方式：

- Docker/K8s：回滚镜像 tag 或 Deployment revision；
- Blue-Green：切回旧环境；
- Canary：停止放量，退回旧版本；
- Feature Flag：直接关闭功能。

好的回答会强调：

- 回滚前先确认是代码问题还是环境问题；
- 数据库 schema 变更要前向兼容；
- 回滚演练与 runbook 非常重要。

### 40. 为什么说“可观测性是 CI/CD 的最后一公里”？

因为“能发布”不等于“发布成功”。  
真正闭环应该是：

```text
代码提交 -> 自动验证 -> 自动部署 -> 上线观测 -> 指标确认 -> 稳定运行
```

如果没有监控、日志、trace、告警，再漂亮的流水线也只是把风险自动化地送进生产。

### 41. 面试里如何体现你真的做过工程化？

不要只背概念，尽量用下面句式：

- “我们之前主干每天发布十几次，所以采用 GitHub Flow + feature flag。”
- “因为 E2E 太慢，我们把回归拆成 PR 阶段的 smoke + 夜间全量。”
- “发布事故后我们补了 readiness probe 和自动回滚阈值。”
- “为降低 CI 时间，我们引入缓存和 affected build。”

这类带背景、决策、权衡、结果的数据化表达最有说服力。

## 本章要点

- DevOps 面试不是工具名词考试，而是交付能力、风险控制和自动化思维的体现。
- Git 工作流没有唯一标准，关键看团队规模、发布频率、质量保障能力。
- CI/CD 核心是缩短反馈回路，并把质量门禁自动化。
- Docker 重点是多阶段构建、镜像安全、体积优化和运行时隔离。
- Kubernetes 必须熟悉 Pod、Deployment、Service、Ingress、Probe、HPA。
- 监控与告警的目标不是“看起来很全”，而是帮助发现并解决真实问题。
- Terraform 与 Ansible 分别偏资源声明和配置管理。
- 蓝绿、金丝雀、滚动发布、功能开关都是“降低发布风险”的不同手段。

## 延伸阅读

- GitHub Actions 官方文档
- Google《Site Reliability Engineering》
- Kubernetes 官方文档与 Production Best Practices
- Docker 官方 Best Practices
- Terraform 官方 Learn 教程与 HashiCorp 文档
