# DevOps自动化运维工具 — 面试参考答案

## DevOps 和 CI/CD 面试题

### 1. 你是怎么理解 DevOps 的？

**要点：**
- DevOps = 开发（Dev）+ 运维（Ops）的文化、实践与工具集合，目标是打破部门墙，缩短"代码写完"到"上线运行"的周期
- 核心是自动化整个交付链路：代码提交 → CI 构建/测试 → CD 部署 → 监控反馈
- 三要素：文化（共享责任、无指责复盘）、流程（CI/CD、小步快跑）、工具（Git/Jenkins/K8s/Prometheus）
- 度量用 DORA 四指标：部署频率、变更前置时间、MTTR、变更失败率

**一句话答：**
DevOps 就是把开发和运维拧成一条绳——用自动化流水线代替手工交接，让代码从提交到上线全链路自动流转，再靠监控把线上问题快速反馈回来，最终用 DORA 指标衡量交付速度和稳定性。

### 2. 你都用过 Devops 哪些工具？

**要点：**
- 代码管理与 CI/CD：GitLab、Jenkins、GitLab CI
- 配置管理与 IaC：Ansible、Terraform
- 容器与编排：Docker、Kubernetes、Harbor 镜像仓库
- 监控告警：Prometheus + Grafana + Alertmanager、Zabbix
- 日志：ELK（Elasticsearch/Logstash/Kibana/Filebeat）
- 其他：SonarQube 质量扫描、Nexus 制品库

**一句话答：**
代码用 GitLab，CI/CD 用 Jenkins 和 GitLab CI，部署用 Ansible 和 K8s，镜像走 Harbor，监控用 Prometheus+Grafana，日志用 ELK，代码质量用 SonarQube，每一环都实际落地过。

### 3. 简述 DevOps 工程师核心工作职责？

**要点：**
- 建设和维护 CI/CD 流水线，让交付自动化
- 基础设施与配置管理（Ansible/Terraform），保证环境一致
- 容器化与编排（Docker/K8s），应用上云与容器平台维护
- 监控告警与日志体系（Prometheus/ELK），保障可观测性
- 值班与故障处理：oncall、故障复盘、容量规划与性能优化
- 制定规范并赋能开发：发布规范、镜像规范、安全基线

**一句话答：**
一句话就是"让交付又快又稳"——搭流水线、管基础设施、做容器化和监控日志体系，平时处理线上故障并复盘优化，同时把规范沉淀成工具给开发自助使用。

### 4. 什么是 CI/CD ？

**要点：**
- CI 持续集成：开发频繁合码到主干，每次提交自动触发构建+测试，尽早暴露集成问题
- CD 持续交付：CI 之后自动打包成可发布制品，随时可一键发布生产
- 持续部署（Continuous Deployment）：更进一步，自动发布到生产，无需人工审批
- 价值：小步提交、快速反馈、减少手工操作和人为失误

**一句话答：**
CI 是每次提交代码自动构建加自动测试，尽早发现问题；CD 是构建产物随时可发布——持续交付是点一下按钮就上线，持续部署则连按钮都不用点，自动上线。

### 5. CI/CD 流水线包含哪些阶段？

**要点：**
- 代码检查（lint/SAST）→ 编译构建 → 单元测试 + 覆盖率 → 质量门禁（SonarQube）
- 制品打包（镜像/包）→ 推送制品库 → 部署测试环境 → 自动化接口测试
- 部署预发/生产（灰度或滚动）→ 冒烟验证 → 通知与归档
- 任一阶段失败即终止流水线并告警

**一句话答：**
典型流程是：拉代码 → 代码扫描 → 编译构建 → 单测/覆盖率 → 质量门禁 → 打镜像推仓库 → 部署测试环境跑自动化测试 → 灰度发布生产 → 冒烟验证 → 通知归档，每一步失败即终止并告警。

### 6. 简述你搭建过的 CI/CD，用到了哪些工具及解决了什么问题？

**要点：**
- GitLab 承载代码，Webhook 触发 Jenkins 流水线
- Jenkins 完成构建：Maven/前端打包，Docker 构建镜像，推 Harbor
- SonarQube 卡质量门禁，单测覆盖率达标才能继续
- 部署：测试环境 Ansible/K8s 自动发布，生产灰度+人工审批
- 解决的问题：手工发布易错、回滚慢、环境不一致；发布从半天缩短到十几分钟，可一键回滚

**一句话答：**
我们用 GitLab+Jenkins+Harbor+K8s 搭的流水线：提交触发构建，SonarQube 卡质量门禁，镜像推 Harbor 后由流水线发布到 K8s，生产走灰度加审批。解决了手工发布易错、环境不一致、回滚慢的问题，发布时间从半天降到十几分钟。

### 7. 灰度发布、蓝绿部署和滚动发布是什么？

**要点：**
- 滚动发布：分批替换旧实例（K8s 默认 RollingUpdate，用 maxSurge/maxUnavailable 控制节奏），省资源，但新旧版本同时在线
- 蓝绿部署：两套完整环境，蓝在跑流量，绿部署新版本，验证后整体切到绿；出问题秒切回蓝，但资源翻倍
- 灰度/金丝雀发布：先放少量真实流量到新版本（如 5%），观察指标正常再逐步放大，风险最小

**一句话答：**
滚动是分批替换实例、省资源；蓝绿是新旧两套环境整体切流量、切换快但费资源；灰度是先给新版本导一小股流量试探，指标没问题再逐步放量，生产上我最常用灰度加滚动结合。

### 8. 说一下你们公司网站是如何发版的？

**要点：**
- 开发提 MR → 评审合并到 main，自动触发流水线
- 构建镜像打版本 tag 推 Harbor，SonarQube+单测门禁通过才放行
- 先发布预发环境做冒烟，通过后生产灰度：先 1 个实例，观察 30 分钟错误率/延迟
- 正常则滚动到全量，异常立即回滚到上一版本镜像
- 发布过程钉钉/企业微信机器人通知，留操作记录

**一句话答：**
合并代码自动触发流水线，构建镜像推 Harbor，先发预发冒烟，生产先灰度一个实例观察错误率和延迟，正常再滚动全量，异常一键回滚，全程群机器人通知并留痕。

### 9. 团队提交频繁导致 CI 队列拥堵，如何优化？

**要点：**
- 扩容执行资源：增加 agent/executor，K8s 动态 agent 按需伸缩
- 让流水线更快：依赖缓存、并行 stage、拆分慢测试、增量构建
- 减少无效构建：路径过滤（文档改动不触发）、合并队列（merge queue）
- 分级流水线：MR 只跑 lint+单测，全量测试放到合并后或夜间
- 限制单项目并发数，低峰期调度重型任务

**一句话答：**
三个方向：一是扩容，加 agent 或用 K8s 动态 runner 按需扩；二是提速，加缓存、并行 stage、拆慢测试；三是减量，路径过滤跳过无关构建，MR 只跑轻量检查，重型测试挪到合并后或夜间跑。

### 10. 在 CI/CD 流程中 Webhook 的作用是什么？

**要点：**
- 代码托管平台在事件发生时（push/MR/tag）主动 POST 通知 CI 服务，事件驱动代替轮询
- 触发及时：提交后秒级开始构建，不用定时轮询 SCM
- 携带 payload（分支、commit、作者），流水线可据此过滤分支/路径
- Jenkins 侧配置 token 与 URL，GitLab 侧配置 Webhook 并可发送测试请求验证

**一句话答：**
Webhook 就是 Git 服务器在 push/MR 时主动回调 CI 的 HTTP 接口，把"该构建了"推给流水线，比定时轮询及时得多，payload 里还带分支和 commit 信息用于过滤。

### 11. 如何在 CI 过程中提高代码质量？

**要点：**
- 门禁前移：MR 触发 lint + 单测 + 覆盖率阈值，不达标不能合并
- SAST：SonarQube 扫描 bug/坏味道/安全漏洞，质量门禁卡严重问题
- Code Review 强制通过才可合并
- 制品安全：依赖与镜像漏洞扫描（如 Trivy 扫镜像）
- 主干保护：main 分支 protected，只允许流水线通过后合入

**一句话答：**
把质量卡点做进流水线：MR 必须 lint、单测、覆盖率达标，SonarQube 门禁通过，Code Review 通过才能合并，镜像再过 Trivy 漏洞扫描，让不合格代码根本进不了主干。

### 12. SonarQube 在 CI 中的作用？

**要点：**
- 静态代码分析：bug、漏洞、坏味道、重复率、测试覆盖率
- Quality Gate 定义阈值（如新增代码覆盖率达标、无阻断级问题），不通过则流水线失败
- 与 Jenkins 集成：sonar-scanner 扫描，结果回传，流水线判断门禁
- 增量分析只盯新增代码，避免历史债务卡死流水线

```bash
sonar-scanner \
  -Dsonar.projectKey=demo \
  -Dsonar.sources=src \
  -Dsonar.host.url=http://sonar:9000 \
  -Dsonar.token=xxxx
```

**一句话答：**
SonarQube 是流水线里的静态代码质量门禁——sonar-scanner 扫出 bug、漏洞、坏味道和覆盖率，按 Quality Gate 阈值判定通过与否，不通过就让构建失败，把质量问题拦在合码之前。

### 13. 什么是基础设施即代码（IaC）？

**要点：**
- 用代码（声明式配置）描述基础设施，代替手工在控制台点资源
- 配置进 Git：可评审、可回滚、可追溯
- 幂等：重复执行结果一致；Terraform plan 可预览变更
- 工具：Terraform（云资源）、Ansible（主机配置）、K8s manifests/Helm

**一句话答：**
IaC 就是把服务器、网络、负载均衡这些基础设施写成代码放进 Git，用 Terraform/Ansible 声明式管理，好处是环境可复现、变更有评审和记录、能一键重建和回滚。

### 14. 简述“混沌工程”与 DevOps 有什么联系？

**要点：**
- 混沌工程：主动向系统注入故障（杀实例、断网、打满 CPU），验证系统韧性和监控告警是否有效
- 是 DevOps/SRE "持续改进、拥抱变更"文化的延伸：与其等故障发生，不如主动演练
- 闭环：注入故障 → 验证监控告警/自愈/降级是否按预期工作 → 修复短板
- 常用工具：Chaos Mesh（K8s）、ChaosBlade、Litmus

**一句话答：**
混沌工程是把 DevOps 的持续改进做到极致——主动在生产或等价环境注入故障，验证监控能不能报、服务能不能自愈，把被动救火变成主动演练，和 CI/CD、监控共同构成高可用闭环。

### 15. 在公司推行 DevOps 会遇到哪些阻力？

**要点：**
- 文化墙：开发运维各管一段、责任互相推，KPI 冲突（开发要快、运维要稳）
- 习惯改变难：从手工工单到自动化，老员工有学习成本
- 存量系统改造难：老应用没有自动化接口，容器化/上流水线成本高
- 安全合规顾虑：权限放开、自助发布带来审计压力
- 应对：找痛点项目试点出效果、争取管理层支持、把规范做成工具降低使用门槛

**一句话答：**
最大阻力是人和流程：部门墙和 KPI 冲突让大家不愿共享责任，存量系统改造成本高，安全上怕失控。我的做法是挑一个痛点项目试点，用数据证明收益，再把规范固化成工具让大家低成本接入。

### 16. 你用过哪些 CI/CD 工具？

**要点：**
- Jenkins：插件生态最全，复杂流水线和异构老项目适配好
- GitLab CI：与 GitLab 一体，.gitlab-ci.yml 配置简单，自带 Runner
- 部署侧：Argo CD（K8s GitOps 持续部署）
- 选型考虑：团队规模、现有平台、维护成本

**一句话答：**
主力是 Jenkins 和 GitLab CI：Jenkins 插件多、适合复杂老项目；GitLab CI 配置简单、和代码库一体化；部署侧还用过 Argo CD 做 K8s 的 GitOps 自动发布。

### 17. Gitlab CI/CD 是怎么工作的？

**要点：**
- 仓库根目录放 .gitlab-ci.yml，声明 stages 和各 stage 的 jobs
- 提交代码后 GitLab 解析该文件生成流水线
- Runner（shell/docker/kubernetes executor）通过注册 token 从 GitLab 领取 job 执行
- 产物用 artifacts/cache 在 job 间传递，密钥在 CI/CD Settings 管理
- 触发：push、MR、tag、定时 schedule，也可手动

```yaml
stages: [test, build]
unit-test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn test
```

**一句话答：**
仓库里放 .gitlab-ci.yml 定义 stages 和 jobs，代码一提交 GitLab 就生成流水线，注册好的 Runner 领取 job 在容器里执行，artifacts 在 job 之间传递，push、MR、tag、定时都能触发。

### 18. 在 DevOps 中，配置管理有什么作用？

**要点：**
- 环境一致性：测试/生产用同一份配置代码拉起，避免"在我机器上是好的"
- 防止配置漂移：定期用 Ansible 巡检收敛
- 变更可追溯可回滚：配置进 Git，改了什么一目了然
- 效率：批量变更从逐台登录变成一条命令
- 应用运行时配置用配置中心（Nacos/Apollo）动态下发

**一句话答：**
配置管理就是把环境和配置代码化：Ansible/Terraform 保证多环境一致、防漂移，配置进 Git 可审计可回滚，批量变更一条命令完成；应用级动态配置则交给 Nacos/Apollo 这类配置中心。

### 19. DevOps 中的“左移”和“右移”是什么？

**要点：**
- 左移：把测试、安全、质量活动往流程早期移——开发阶段就写单测、跑 SAST/SCA、IaC 扫描
- 右移：把验证延伸到生产——灰度、A/B、混沌演练、生产监控与 RUM
- 目的一致：尽早发现问题、快速反馈，降低修复成本

**一句话答：**
左移是把测试和安全提前到开发阶段，提交时就跑单测和 SAST；右移是把验证延伸到线上，用灰度、混沌演练和监控在生产环境持续验证，两头都是为了更早发现问题。

### 20. CI/CD 中自动化测试有哪些工作？

**要点：**
- 单元测试：开发写，CI 每次提交跑，统计覆盖率
- 接口测试：pytest+requests、Postman/Newman，服务部署后自动跑
- UI 自动化：Selenium/Playwright，主要覆盖核心链路冒烟
- 非功能：性能压测（JMeter）、安全扫描（SAST/依赖扫描/镜像扫描）
- 分层策略：单测多而快、UI 少而精；失败用例隔离重试防 flaky

**一句话答：**
分层做：每次提交跑单测和覆盖率，部署到测试环境后自动跑接口测试，核心链路用 Selenium/Playwright 做 UI 冒烟，再加 JMeter 性能压测和镜像安全扫描，让测试金字塔又快又稳。

### 21. CI/CD 流程中应该监控哪些指标？

**要点：**
- 流水线指标：构建时长、成功率、排队时间、各 stage 耗时
- 质量指标：单测覆盖率、SonarQube 问题数、flaky 用例率
- 交付指标（DORA）：部署频率、变更前置时间、变更失败率、MTTR
- 部署后指标：错误率、延迟、资源占用，作为发布门禁

**一句话答：**
三类：流水线本身的时长、成功率、排队时间；质量类的覆盖率、静态扫描问题、flaky 率；还有 DORA 四指标——部署频率、前置时间、变更失败率和 MTTR，发布后再盯错误率和延迟。

### 22. 如果你加入我们团队，如何开展 DevOps？

**要点：**
- 先调研：梳理现有构建、发布、监控流程和痛点（访谈+看数据）
- 定基线：度量发布频率、失败率、构建时长，明确改进目标
- 快速赢：挑痛点（如手工发布）做自动化试点，快速出效果建立信任
- 推标准：流水线模板、镜像规范、监控告警基线，文档化并推广
- 持续运营：指标看板 + 例会复盘

**一句话答：**
我会先花时间摸清现有流程和痛点，用发布频率、失败率这些指标定基线；然后挑最痛的环节做自动化试点快速见效，再把流水线和规范模板化推广，最后用指标看板持续运营改进。

### 23. 混沌工程 怎么做故障注入？

**要点：**
- 先定稳态假设：明确正常指标基线（错误率、延迟），设好中止条件
- 控制爆炸半径：先在预发或小流量灰度注入，随时可一键终止
- 常见注入：杀 Pod、CPU/内存压力、网络延迟丢包、磁盘打满、依赖超时
- 工具：Chaos Mesh（K8s CRD）、ChaosBlade、Litmus
- 演练后复盘：验证告警/自愈/降级是否生效，形成改进项

```bash
# 用 tc 直接注入网络延迟/丢包（最基础的注入手段）
tc qdisc add dev eth0 root netem delay 100ms loss 1%
# 清除注入
tc qdisc del dev eth0 root
```

**一句话答：**
先定义稳态指标和中止条件，控制好爆炸半径，然后用 Chaos Mesh 或 ChaosBlade 注入故障——杀 Pod、打满 CPU、加网络延迟、填满磁盘——验证告警、自愈、降级是否按预期工作，最后复盘改进，整个过程随时能一键终止。

### 24. GitLab Flow 了解过吗？

**要点：**
- 介于 GitHub Flow（main+feature）和 Git Flow（develop/release/hotfix 多分支）之间的折中
- 核心思想 upstream first：feature 分支先合 main，再用分支表示"部署阶段"
- 环境分支：main → pre-production → production，代码只能向上游流动
- 发布场景：从 main 拉 1.x-stable 分支，bug 修复 cherry-pick 回各 release 分支

**一句话答：**
GitLab Flow 是在 main 加 feature 分支的基础上，用额外分支表示环境和发布：main 合入后依次流向 pre-production、production 环境分支，发版从 main 拉 stable 分支并用 cherry-pick 回补修复，比 Git Flow 轻、比 GitHub Flow 好管控发布。

### 25. 怎么提升研发效能？

**要点：**
- 让反馈更快：流水线提速（缓存、并行、增量），失败信息清晰可定位
- 减少等待和手工：自助式环境/发布、合并队列、自动化测试左移
- 减少打断：需求小批量、控制 WIP、代码评审 SLA
- 度量驱动：DORA 指标 + 定制看板，用数据找瓶颈
- 平台化沉淀：CI 模板、脚手架、共享库，降低重复劳动

**一句话答：**
核心是让"反馈快、等待少"：流水线提速加自动化测试左移，发布做成自助式，需求小批量流动，再用 DORA 指标找瓶颈持续改进，把重复工作沉淀成平台和模板。

## Prometheus 监控面试题

### 1. Prometheus 有哪些核心组件及其作用？

**要点：**
- Prometheus Server：抓取指标、TSDB 存储、PromQL 查询
- Exporter：把各类系统的指标暴露成 /metrics（node_exporter、mysqld_exporter 等）
- Alertmanager：告警去重、分组、路由、静默，发送通知
- Pushgateway：短生命周期任务推送指标的网关
- Grafana：可视化展示；配套还有服务发现、recording rules

**一句话答：**
核心是四件套——Prometheus Server 负责 pull 抓取、TSDB 存储和 PromQL 查询；Exporter 把各种系统指标暴露成 /metrics；Alertmanager 做告警分组、去重、路由和通知；Pushgateway 给短生命周期的批处理任务兜底，Grafana 负责展示。

### 2. Prometheus 有哪些数据指标类型及其应用场景？

**要点：**
- Counter：只增不减的计数器，用于 QPS、错误数，配 rate() 使用
- Gauge：可增可减的瞬时值：内存、并发数、队列长度
- Histogram：客户端分桶统计，服务端可算 P95/P99 延迟（histogram_quantile），可跨实例聚合
- Summary：客户端直接算分位数，无法多实例聚合，一般少用

**一句话答：**
四种：Counter 只增不减，适合错误数、QPS，配 rate 用；Gauge 是瞬时值如内存和连接数；Histogram 分桶统计延迟并算 P99；Summary 在客户端算分位数但不能跨实例聚合，所以延迟统计我优先用 Histogram。

### 3. Prometheus 支持哪些服务发现方式？

**要点：**
- file_sd：静态 JSON/YAML 文件，常配合 Ansible/CMDB 生成
- consul_sd：Consul 注册中心
- kubernetes_sd：K8s 环境（Pod/Service/Endpoints/Ingress 等角色）
- 云厂商 SD：ec2_sd、azure_sd、gce_sd、openstack_sd 等
- dns_sd、http_sd；发现的目标再用 relabel_configs 过滤和打标签

**一句话答：**
常用的有 file_sd（配 Ansible/CMDB 生成目标文件）、consul_sd、kubernetes_sd，云上的 EC2/Azure/GCE SD，还有 dns_sd 和 http_sd；发现到的目标再用 relabel_configs 过滤和打标签。

### 4. Pushgateway 的使用场景是什么？它解决了什么问题？

**要点：**
- 解决问题：Prometheus pull 抓不到的短生命周期任务（cron 脚本跑完就退出）
- 任务结束前把指标推到 Pushgateway，Prometheus 再 pull Pushgateway
- 缺陷：指标会一直保留，任务死了还报旧值，需在任务收尾时删除
- 不用于常规长期运行的应用监控

```bash
# 推送
echo "batch_job_success 1" | curl --data-binary @- \
  http://pushgateway:9091/metrics/job/batch/instance/host1
# 任务收尾删除，避免残留旧值
curl -X DELETE http://pushgateway:9091/metrics/job/batch/instance/host1
```

**一句话答：**
Pushgateway 是给 cron 脚本这类跑完就退出的短时任务兜底的：任务结束前把指标推给它，Prometheus 再来 pull 它；但指标会一直留存，所以任务收尾要主动 DELETE，避免残留旧值造成误判。

### 5. 什么时候会用 relabel_configs  ？

**要点：**
- 作用在抓取前的"目标"上：加/改/删标签、过滤目标（action: keep/drop）
- 典型：把服务发现的 __meta_* 元数据（consul/k8s）转成业务标签（env、team）
- 用 regex + keep 只保留需要的 target，减少无效抓取
- 区别 metric_relabel_configs：作用于抓回的样本（丢弃高基数指标）

```yaml
relabel_configs:
  - source_labels: [__meta_consul_tags]
    regex: .*,prod,.*
    action: keep
  - source_labels: [__meta_consul_service]
    target_label: service
```

**一句话答：**
relabel_configs 作用在抓取之前的目标上：把服务发现带出的元数据转成业务标签、给 instance 重命名、用 keep/drop 过滤目标；要删具体指标则用抓取之后的 metric_relabel_configs。

### 6. 你都用过哪些 Exporter ？

**要点：**
- 主机：node_exporter；容器：cAdvisor/kubelet、kube-state-metrics
- 数据库：mysqld_exporter、redis_exporter、postgresql_exporter
- 中间件/网络：blackbox_exporter（HTTP/TCP/Ping 探测）、nginx-exporter、kafka_exporter
- 设备类：SNMP exporter（交换机路由器）、dcgm-exporter（GPU）
- 部署：systemd 二进制或容器，Prometheus 配 target 抓取

**一句话答：**
主机用 node_exporter，容器和 K8s 用 cAdvisor 加 kube-state-metrics，数据库有 mysqld、redis、postgresql exporter，黑盒探测用 blackbox_exporter，网络设备用 SNMP exporter，基本都是起个 /metrics 端口接进 Prometheus。

### 7. 你用过 Prometheus 哪些内置函数?

**要点：**
- rate()/irate()：Counter 增长率（rate 看整段区间，irate 取最新两点看突刺）
- increase()：区间增量；histogram_quantile()：算分位数
- avg_over_time/max_over_time/min_over_time：区间聚合
- sum/avg 配 by 聚合；topk/bottomk；absent() 判断无数据（可用作拨测告警）
- predict_linear()：线性预测（如磁盘何时写满）

```promql
sum(rate(http_requests_total{job="api"}[5m])) by (handler)
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
predict_linear(node_filesystem_avail_bytes[6h], 24*3600) < 0
```

**一句话答：**
最常用的是 rate/irate 算 QPS、increase 算增量、histogram_quantile 算 P99 延迟，配合 sum/avg/by 做聚合、topk 取头部、avg_over_time 这类区间函数，还有 predict_linear 预测磁盘耗尽、absent 检测指标消失。

### 8. Prometheus 存储机制是怎么样的？

**要点：**
- 内置 TSDB：最近数据在内存 head block，WAL 保证进程重启不丢
- 每 2 小时的样本压缩成一个 block（chunks + index + meta.json）落盘
- 后台 compaction 把小块逐步合并成大块；样本用 XOR 压缩，一个 chunk 默认 120 个样本
- 查询同时查 head 和历史 block
- 本地保留期 --storage.tsdb.retention.time（默认 15d）

**一句话答：**
Prometheus 用内置 TSDB：最新 2 小时数据在内存 head 区并有 WAL 兜底，每 2 小时压缩成一个含 chunks 和倒排索引的 block 落盘，后台再把小块合并成大块，样本经 XOR 压缩很省空间，默认本地只留 15 天。

### 9. Prometheus 为什么不适合长期存储？

**要点：**
- 单机架构：存储和查询都在本机，横向扩展能力弱
- 本地磁盘容量有限，长保留意味着巨大磁盘成本
- 大时间范围查询要扫大量 block，会很慢
- 无副本：单份数据，盘坏数据丢
- 方案：remote_write 到 Thanos/VictoriaMetrics/InfluxDB 做长期存储

**一句话答：**
因为它是单机 TSDB：数据只在本地一份、容量受单盘限制、长区间查询要扫大量 block 会很慢、也没有副本容灾，所以生产上一般本地保留 15 到 30 天，历史数据用 remote_write 交给 Thanos 或 VictoriaMetrics 存。

### 10. Prometheus 出现 OOM 的常见原因及解决方案？

**要点：**
- 最常见：基数爆炸——label 值不可控（user_id、URL、container_id），series 数飙升
- 并发的大范围慢查询：一次查几万个 series 乘长区间
- 抓取目标暴增，或单个 /metrics 响应巨大
- compaction、remote_write 堆积也吃内存
- 方案：metric_relabel_configs 丢弃高基数指标、scrape 配置 sample_limit、控制查询并发（--query.max-concurrency）与样本量（--query.max-samples）、加内存、分片或上 Thanos

**一句话答：**
九成是基数爆炸：label 里带了 user_id、URL 这种不可枚举值，series 数失控；其次是并发大范围查询。治理上用 metric_relabel_configs 掐掉高基数指标、给 job 设 sample_limit、限制查询并发和样本数，内存顶不住就分片或上 Thanos。

### 11. Prometheus 有哪些主流高可用方案？

**要点：**
- 双份 Prometheus 抓相同目标（配置一致），查询侧双数据源或下游去重——简单但存储翻倍
- Thanos：sidecar 把 block 上传对象存储，全局查询+长期存储
- VictoriaMetrics：vmagent 采集 + VM 集群存储，资源成本更低
- Cortex/Mimir：多租户长期存储方案
- 注意：两台 Prometheus 数据非实时一致，靠下游按 replica 标签去重

**一句话答：**
入门是跑两台一模一样的 Prometheus 同时抓，简单但磁盘翻倍；生产更常用 Thanos——sidecar 把数据块上传对象存储，Query 层全局聚合去重；或者用 VictoriaMetrics 的 vmagent 加 vmcluster，资源成本低很多。

### 12. Thanos 实现高可用的核心原理是什么？

**要点：**
- Sidecar 与 Prometheus 同机，把 2 小时 block 上传对象存储（S3/OSS）
- 对象存储成为唯一事实源：本地短保留，长期数据在对象存储
- Query 层聚合多台 Prometheus 和 Store Gateway，按 external labels + replica 标签去重
- Compactor 负责合并压缩与降采样
- 查询层无状态可水平扩展，任一 Prometheus 挂掉不影响全局查询

**一句话答：**
Thanos 的思路是把 Prometheus 的 2 小时数据块通过 sidecar 上传到对象存储，让对象存储当长期事实源，Query 层把多台 Prometheus 和对象存储的数据聚合起来按 replica 标签去重，Compactor 负责压缩降采样——既高可用又能存很久。

### 13. Thanos 有哪些核心组件及其作用？

**要点：**
- Sidecar：随 Prometheus 部署，上传 block、代理本地实时查询
- Query：无状态查询网关，聚合多数据源并去重
- Store Gateway：查询对象存储里的历史 block
- Compactor：块合并压缩、降采样（5m/1h）、清理
- Ruler：独立执行告警/记录规则；Receive：接收 remote_write 的推送模式

**一句话答：**
六个组件：Sidecar 负责上传数据块和代理实时查询，Query 做全局查询和去重，Store Gateway 读对象存储里的历史块，Compactor 做压缩降采样，Ruler 独立跑告警规则，Receive 支持推送写入。

### 14. VictoriaMetrics 实现高可用的核心原理是什么？

**要点：**
- 集群版组件分离：vminsert 写入、vmselect 查询、vmstorage 存储，无状态层前置负载均衡即可扩展
- 存储分片部署，数据分布在多个 vmstorage
- 高可用：-replicationFactor=N 多写几份，查询端自动 dedup 合并
- vmagent 负责采集：支持 Prometheus 服务发现、relabel、多 remote_write
- 单机版也能扛较大规模，压缩率高、内存占用低

**一句话答：**
VictoriaMetrics 把写入 vminsert、查询 vmselect、存储 vmstorage 拆开，存储层分片部署，无状态层加负载均衡即可水平扩展；高可用靠 replicationFactor 多写几份，查询时自动去重；采集用兼容 Prometheus 配置的 vmagent。

### 15. Alertmanager 通过哪些机制减少无效告警？

**要点：**
- group_by/group_wait/group_interval：同标签告警聚合成一条通知，等一小段时间凑批
- repeat_interval：未恢复的告警不会频繁重复轰炸
- inhibit_rules：高级别告警抑制低级别（如节点 down 抑制该节点所有衍生告警）
- silence：维护窗口静默；route 树按标签分流到不同接收人
- 配合 Prometheus 侧 for 持续时间防抖

```yaml
route:
  group_by: [alertname, cluster]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
inhibit_rules:
  - source_matchers: ['alertname = NodeDown']
    target_matchers: ['severity = warning']
    equal: ['instance']
```

**一句话答：**
靠分组聚合把同源告警合并成一条、group_wait 和 repeat_interval 控制节奏防轰炸、inhibit_rules 让"节点宕机"这种根因告警抑制下游一堆衍生告警，再加上静默和路由分流，让值班只看到该看的信息。

### 16. Prometheus 如何调整，可以缩短告警时间？

**要点：**
- 缩短抓取间隔 scrape_interval（如 30s→15s）和规则评估 evaluation_interval
- 减小关键告警的 for 持续时间（要权衡误报率）
- Alertmanager 侧：group_wait 调小、通知渠道直连
- 核心指标单独用更细粒度采集，或用 recording rules 预计算
- 逐环节评估耗时：抓取 → 评估 → AM 分组 → 通知

**一句话答：**
把链路上每个等待环节都缩短：抓取间隔和规则评估间隔调小、关键告警的 for 时长调短、Alertmanager 的 group_wait 减小；注意防抖和误报的平衡，核心指标可以单独用更密的采集间隔。

### 17. Alertmanager 高可用如何实现？

**要点：**
- 部署多实例（建议 3 个）组成 gossip 集群：--cluster.peer 指向彼此（集群端口 9094）
- Prometheus 把同一条告警同时发给所有 AM 实例
- 实例间通过 gossip 同步通知状态，保证同一条告警只发一次（去重）
- 本身无状态，挂掉部分实例告警不丢

**一句话答：**
跑至少 3 个 Alertmanager，用 --cluster.peer 组成 gossip 集群，Prometheus 把同一条告警同时发给每个实例，实例间同步通知状态做去重，这样挂掉一两台告警也不丢、也不会重复发。

### 18. 简述 Exporter 开发思路和核心细节？

**要点：**
- 用 client_golang：实现 Collector 接口的 Describe/Collect，或直接注册 Gauge/Counter 等指标
- 暴露 /metrics（promhttp.Handler），自定义 registry 避免混入无关运行时指标
- Collect 里查询外部系统要设超时、可加缓存，避免单次 scrape 被拖死
- 指标语义正确：累计值用 Counter、当前值用 Gauge、延迟分布用 Histogram
- label 保持低基数（instance、type 这类），避免高基数拖垮 TSDB

```go
func main() {
    up := prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "my_app_up", Help: "app availability"})
    prometheus.MustRegister(up)
    up.Set(1)
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":9105", nil)
}
```

**一句话答：**
用 Go 的 client_golang 写一个 Collector：Collect 方法里去查目标系统，把结果填进 Counter、Gauge 或 Histogram，注意查询带超时、label 保持低基数，最后暴露 /metrics 给 Prometheus 抓，官方大多数 exporter 就是这么写的。

### 19. 监控一个 MySQL 是怎么样的流程？

**要点：**
- 建最小权限监控账号：PROCESS、REPLICATION CLIENT、SELECT
- 部署 mysqld_exporter，配置连接信息，起服务暴露 9104
- Prometheus 加 target，Grafana 导入现成 MySQL 看板
- 配告警：连接数、Threads_running、慢查询增速、主从延迟、复制中断
- 补充：慢日志分析、业务自定义指标

```sql
CREATE USER 'exporter'@'%' IDENTIFIED BY 'xxx' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
```

**一句话答：**
流程是：给 mysqld_exporter 建一个只有 PROCESS、REPLICATION CLIENT、SELECT 权限的监控账号，部署 exporter 暴露 9104，Prometheus 加 target 抓取，Grafana 导入现成 MySQL 看板，再配上连接数、慢查询、主从延迟这些告警规则。

### 20. 新上100台服务器，Prometheus 如何实现自动化监控？

**要点：**
- 批量装 node_exporter：Ansible playbook 一键部署 + systemd 注册
- 目标管理：Ansible 生成 targets JSON 给 file_sd，或注册到 Consul/CMDB 对接 http_sd
- 统一配置：同一 job、统一采集间隔；告警规则和 Grafana 看板用统一模板
- 新机器上线流程自动纳管（装机脚本/云 autoscaling 钩子触发）

**一句话答：**
用 Ansible 批量部署 node_exporter 并生成 targets JSON 文件给 Prometheus 的 file_sd，主机分组进文件就自动被监控；告警规则和看板用统一模板，之后新机器只要进装机流程就自动纳管，也可以对接 CMDB 或 Consul 做动态发现。

### 21. Prometheus 性能调优的核心手段有哪些？

**要点：**
- 控制基数：删掉无用 label/指标，metric_relabel_configs 丢弃高基数 series
- 采集粒度分级：核心服务 15s，普通机器 30~60s
- 查询治理：禁用大范围裸查询，热点查询建 recording rules 预聚合
- 参数：--query.max-concurrency、--query.max-samples、合理设置 retention
- 资源：SSD、充足内存；到瓶颈就 hashmod 分片或 remote 存储卸载

**一句话答：**
抓的就是"基数、粒度、查询"三件事：把高基数指标和无效 label 掐掉，非核心目标放宽采集间隔，热点查询用 recording rules 预计算，再配好查询并发和保留期参数；单机顶不住就 hashmod 分片或者把存储卸给 Thanos/VM。

### 22. 监控上千台服务器，如何保障 Prometheus 整体采集与查询性能？

**要点：**
- 水平分片：按 hashmod 把 target 分给多个 Prometheus 实例
- 存储卸载：remote_write 到 Thanos/VM，Prometheus 本地只留短 retention
- 采集端减负：合理 scrape_interval、按需装 exporter、避免重复采集
- 查询端：看板尽量走预聚合 recording rules，避免面板直查原始大指标
- 容量观测：samples/sec、head series 数、查询延迟做成常驻监控

```yaml
relabel_configs:
  - source_labels: [__address__]
    modulus: 4
    target_label: __tmp_hash
    action: hashmod
  - source_labels: [__tmp_hash]
    regex: ^0$
    action: keep
```

**一句话答：**
上千台的核心是分片和分层：用 hashmod 把目标均分给多台 Prometheus 各抓一份，数据 remote_write 到 Thanos 或 VictoriaMetrics 做长期存储和全局查询，看板尽量用预聚合规则，同时把 samples/s 和 head series 监控起来防基数失控。

### 23. Prometheus 的 Pull 模型相比 Push 模型有什么优势和劣势？

**要点：**
- Pull 优势：服务发现与健康检查天然内聚（抓不到就是 down）；防止恶意/异常客户端灌数据；配置集中在服务端
- Pull 劣势：短时任务抓不到（要 Pushgateway 兜底）；NAT/防火墙后的目标不可达；数据时效受采集间隔限制
- Push 优势：适合事件驱动、短任务、客户端在隔离网络主动出站的场景
- Prometheus 以 Pull 为主，Pushgateway/remote_write 补充

**一句话答：**
Pull 的好处是目标健康状态一目了然、数据来源可控、配置集中在服务端，坏处是短命任务和 NAT 后面的目标抓不到，得靠 Pushgateway 兜底；Push 则相反，适合任务制和防火墙内场景。Prometheus 以 Pull 为主，两者互补。

### 24. Prometheus 如何实现对 Kubernetes 集群的全面监控？

**要点：**
- node_exporter（DaemonSet）监控节点；kubelet/cAdvisor 暴露容器指标
- kube-state-metrics：Deployment/Pod/Node 等对象状态
- 组件指标：apiserver、etcd、coredns、kube-proxy
- Prometheus Operator + ServiceMonitor/PodMonitor 自动发现目标；kube-prometheus-stack 一键部署
- 告警用 kubernetes-mixin 规则集，Grafana 有现成集群看板

**一句话答：**
标准做法是 kube-prometheus-stack 一套拿下：node_exporter 收节点指标，kubelet/cAdvisor 收容器指标，kube-state-metrics 收 K8s 对象状态，再加上 apiserver、etcd、coredns 组件指标；Prometheus Operator 通过 ServiceMonitor 自动发现新服务，告警直接用 kubernetes-mixin 规则。

### 25. Prometheus 查询突然变慢，可能的原因有哪些？

**要点：**
- 查询本身：label 匹配过宽（无 job 过滤的裸指标）、正则匹配、超长 range、高基数 group by
- 负载：并发重查询挤爆 CPU/内存，OOM 后 head 缓存失效
- 存储层：磁盘 IO 瓶颈、正在 compaction、block 数量过多
- 基数爆炸：最近上线了高基数指标
- 排查：TSDB Status 的 top cardinality、机器资源、慢查询日志

**一句话答：**
先看查询本身是不是变重了——大范围、宽匹配、高基数聚合；再看系统——是不是有并发重查询把 CPU 内存吃满、磁盘 IO 到瓶颈、后台在做 compaction；最后查 TSDB Status 里的 top cardinality，看是不是新上了高基数指标。

### 26. 如何评估 Prometheus 集群的容量？

**要点：**
- 采集侧：samples/sec ≈ Σ(每个 target 的 series 数 ÷ scrape_interval)
- 磁盘 ≈ samples/s × 2 bytes × 保留秒数（压缩后经验值约 1~2B/样本）
- 内存：看 head 里的 series 数（每 series 约几 KB），另留大查询峰值
- 例：60000 samples/s、保留 15 天 ≈ 150GB 磁盘
- 持续观测 prometheus_tsdb_head_series 与样本速率，预留 30~50% 余量

**一句话答：**
抓取容量用"所有目标的 series 数除以采集间隔"估出每秒样本数，磁盘按每样本约 2 字节乘保留时长估算，内存看 head 里的 series 数；比如 6 万 samples/s 保留 15 天大概 150GB 磁盘，实际再留三成余量，并把 head series 和样本速率做成常驻监控。

### 27. Grafana 图表不显示数据，怎么排查？

**要点：**
- 数据源：Test 连通性（URL、认证、代理设置）；确认 Grafana 服务器能访问 Prometheus
- 查询：把面板 PromQL 拷到 Prometheus 原生界面直接执行验证
- 时间范围/变量：时间选择器、时区、模板变量取值是否为空
- 有数据但图空：单位/legend/格式化设置；F12 看接口返回区分"no data"和查询报错
- 权限/版本：数据源权限、Grafana 与数据源版本兼容性

**一句话答：**
按链路查：先在数据源设置里点 Test 确认 Grafana 能通 Prometheus，然后把面板的 PromQL 拷到 Prometheus 原生界面直接跑——能跑通就是面板的时间范围、变量或单位设置问题，跑不通就是查询或数据本身的问题，再看浏览器 F12 里接口报错。

### 28. Prometheus Operator 是什么？解决了什么问题？

**要点：**
- K8s 上的 Prometheus 管理方案：用 CRD 声明式管理监控配置
- 核心对象：Prometheus、Alertmanager、ServiceMonitor、PodMonitor、PrometheusRule
- 解决问题：手工维护 prometheus.yml 在 K8s 动态环境下不可扩展
- Operator 监听 CR 变更，自动生成配置并热加载，新服务打标签即被监控
- 社区标配部署方式：kube-prometheus-stack Helm chart

**一句话答：**
Prometheus Operator 把 prometheus.yml 变成 K8s CRD：ServiceMonitor/PodMonitor 声明要抓谁，PrometheusRule 声明告警规则，Operator 监听这些对象自动生成配置并热加载，新服务打上标签就自动被监控，不用再手工维护配置文件。

### 29. Prometheus 与 Zabbix 有什么区别？

**要点：**
- 数据模型：Prometheus 多维 label + 时序库 + PromQL；Zabbix 主机+item 模型，关系库存储
- 采集：Prometheus pull + exporter 生态，容器云原生友好；Zabbix agent/SNMP，传统设备和网络设备强
- 告警：Prometheus 规则 + Alertmanager 分组抑制；Zabbix trigger + action，界面化配置
- 扩展性：Prometheus 水平扩展需 Thanos/VM；Zabbix proxy 分布式成熟
- 选型：云原生/微服务选 Prometheus；网络设备和传统 IDC 混合环境 Zabbix 更顺手，二者常共存

**一句话答：**
Zabbix 是主机加监控项的模型、UI 配置、对网络设备和传统服务器支持成熟；Prometheus 是多维时序模型、PromQL 灵活、对容器和微服务生态原生友好，但长期存储要配 Thanos/VM。我们云上业务用 Prometheus，网络设备和老机房用 Zabbix，两者互补。

## Zabbix 监控面试题

### 1. 简述 Zabbix 核心组件及作用？

**要点：**
- Server：核心进程，负责数据采集调度、触发器评估、告警执行
- Database：存储配置、监控数据和事件（MySQL/PostgreSQL 等）
- Web 前端：PHP 编写，配置与展示界面
- Agent：装在被监控机，采集数据（主动/被动两种模式）
- Proxy：分布式场景代理采集，缓解 Server 压力
- Java Gateway：监控 JMX；Sender/Get：命令行收发测试工具

**一句话答：**
核心是 Server 加数据库加 Web 前端三件套：Server 负责采集调度和触发告警，数据库存配置和数据，Web 做配置展示；被监控端装 Agent，大规模时加 Proxy 分担采集压力，Java 环境再配 Java Gateway 走 JMX。

### 2. 简述 Zabbix 中 Host、Host Group、Template 的关系？

**要点：**
- Template 定义监控项、触发器、图形的集合，链接到 Host 后主机继承全部内容
- Host 是被监控对象，可链接多个模板，也可单独加自定义 item
- Host Group 是主机的逻辑分组，用于权限控制（用户组×主机组）和批量操作
- 模板改一处，所有链接它的主机生效——规模化监控的基础

**一句话答：**
模板是监控项、触发器、图形的集合，链接到主机后主机继承全部监控内容，模板改一处全部生效；主机组是主机的逻辑分组，主要用来做用户权限控制和批量操作，三者构成"模板→主机→组"的层级。

### 3. 简述 Zabbix 中 Item、Trigger、Action 的关系？

**要点：**
- Item 采集数据；Trigger 基于 item 数据定义阈值表达式，状态在 OK/PROBLEM 间翻转
- Action 监听 trigger 事件，满足条件时执行：发通知或远程命令
- 链路：Item 采数 → Trigger 判定异常产生事件 → Action 按条件路由通知

**一句话答：**
Item 负责采数据，Trigger 拿 item 的数据按表达式判断是否异常并产生事件，Action 监听事件按条件发通知或执行远程命令，是"采集→判定→响应"的三级流水线。

### 4. Zabbix Agent 的主动模式和被动模式有什么区别？

**要点：**
- 被动：Server/Proxy 来连 agent 的 10050 端口要数据；agent 配 Server 白名单
- 主动：agent 按 ServerActive 配置周期性连 10051 上报；item 类型为 "Zabbix agent (active)"
- 选择：被动便于服务端集中管控和排障；主动适合 agent 藏在 NAT/防火墙后、或目标数千台要给 Server 减负
- 大规模环境推荐主动模式：server 端不用维护海量并发连接

**一句话答：**
被动模式是 Server 来连 agent 的 10050 拉数据，主动模式是 agent 定时连 Server 的 10051 上报。被动便于集中管控，主动适合 NAT 后的主机和大规模环境——让 server 不用维护海量连接，生产上千台以上我优先主动模式。

### 5. 简述 Zabbix 监控一台新的 Linux 主机流程？

**要点：**
- 安装对应版本的 zabbix-agent（yum/apt），配置 Server/ServerActive 指向 Server 或 Proxy
- 防火墙放行 10050（被动）或 agent 主动出站 10051
- Web 端创建主机：填 IP、归属主机组、链接 "Linux by Zabbix agent" 模板
- 确认 ZBX 图标变绿（可用性）；批量场景用自动注册或 Ansible 推部署

```bash
# RHEL 系
yum install zabbix-agent
# /etc/zabbix/zabbix_agentd.conf
# Server=10.0.0.1
# ServerActive=10.0.0.1
# Hostname=web01
systemctl enable --now zabbix-agent
```

**一句话答：**
机器上装 zabbix-agent，配置里 Server 和 ServerActive 指向 Server 并放行 10050，Web 端建主机链接 Linux agent 模板，等 ZBX 图标变绿即可；批量机器就用 Ansible 推部署加自动注册自动纳管。

### 6. Zabbix 如何监控一台 MySQL 服务器？

**要点：**
- 创建专用监控账号并授权
- 用 "MySQL by Zabbix agent 2" 模板：agent2 内置 MySQL 插件，宏配连接信息（MYSQL.DSN、用户、密码）
- 经典 agent 1 方式：userparameter_mysql.conf + .my.cnf 提供 mysqladmin/queue 探测
- 关注指标：连接数、QPS、主从延迟、慢查询、缓冲池命中率
- 告警：复制中断、连接数过高、服务不可达

```sql
CREATE USER 'zbx_monitor'@'%' IDENTIFIED BY 'xxx';
GRANT REPLICATION CLIENT, PROCESS, SHOW DATABASES ON *.* TO 'zbx_monitor'@'%';
```

**一句话答：**
给 agent2 建 zbx_monitor 账号授权 REPLICATION CLIENT 和 PROCESS，链接 "MySQL by Zabbix agent 2" 模板，在主机宏里配连接串和密码，模板自带连接数、主从延迟、慢查询等监控项，再对复制中断和连接数配告警。

### 7. Zabbix UserParameter 有什么作用？

**要点：**
- 自定义监控项：agent 配置里定义 key → 执行的命令/脚本，服务端像普通 item 一样调用
- 支持位置参数 $1..$9，一个 UserParameter 复用多种用途
- 放在 /etc/zabbix/zabbix_agentd.d/*.conf，改后重启 agent
- 服务端用 zabbix_agentd -t key 本地测试验证

```ini
UserParameter=nginx.status[*],curl -s http://127.0.0.1/status | grep '$1' | awk '{print $$NF}'
# 服务端取值：nginx.status[active]
```

**一句话答：**
UserParameter 让 agent 能执行自定义脚本并作为监控项 key 返回，比如定义 nginx.status[*] 抓 nginx 状态页各字段，服务端就能像内置 item 一样取值，是扩展 Zabbix 监控任何东西的基本手段。

### 8. Zabbix 模板中的宏有什么作用？

**要点：**
- 宏是变量占位符：{$MACRO}，在 item/trigger/模板和主机层级引用
- 继承与覆盖：主机级宏 > 模板级 > 全局，同一模板适配不同环境
- 典型用途：{$MYSQL.DSN}、{$CPU.UTIL.CRIT} 阈值、{$URL} 地址
- 带上下文宏 {$MACRO:"value"} 可对不同对象设不同阈值

**一句话答：**
宏就是模板里的变量：模板定义 {$THRESHOLD} 这类占位符，各主机按自己的环境覆盖值，这样同一套模板在不同主机上用不同阈值和地址，避免每台机器复制一套模板。

### 9. Zabbix 告警机制是如何实现的？

**要点：**
- Trigger 表达式周期评估，异常时生成 PROBLEM 事件
- Action 按条件匹配事件，执行 Operations：发消息（介质：邮件/钉钉/企微 webhook）、升级规则（escalation 未恢复逐步升级到主管）、远程命令
- Recovery operations 恢复时通知；依赖关系防告警风暴（父 trigger 依赖）
- 维护期抑制告警

**一句话答：**
Trigger 按表达式周期判断产生事件，Action 匹配事件后按操作发通知或跑远程命令，支持升级策略——问题持续未恢复自动升级到上级，配合触发器依赖和维护期来控制噪音。

### 10. Zabbix 监控图表出现中断，如何排查？

**要点：**
- 看 item 最新数据有没有值：没有则查 agent 可用性（ZBX 红绿）
- Server/Proxy 日志：first network error、cannot connect 类错误
- 检查 item 类型/间隔/key 是否变更、是否到达数据收集限制
- 服务端压力大：Housekeeper 忙、数据库 IO 高、队列（Administration → Queue）堆积
- 网络/防火墙抖动、DNS 解析问题

**一句话答：**
先看 item 最新数据还有没有值、agent 可用性是否变红，再看 server/proxy 日志里的连接错误，然后查 Administration 的 Queue 是否堆积——很多时候是 server 压力大或数据库慢导致采集中断，也可能是网络抖动或 item 被改动。

### 11. Zabbix 数据库膨胀很大，如何清理？

**要点：**
- 调整保留期：Administration → Housekeeping，history（默认 90d）和 trends（默认 365d）按需缩短
- Housekeeper 删除压力大会拖慢库：大数据量场景关闭自动清理，改用定时任务按分区 DROP
- MySQL 按天/月对 history/trends 相关表分区，过期 DROP PARTITION 秒删
- 一次大 DELETE 反而放大 ibdata；磁盘空间回收需 OPTIMIZE TABLE
- 降低数据量：延长采集间隔、减少无价值 item、开启压缩（PG 或引擎层）

**一句话答：**
先在 Housekeeping 里缩短 history/trends 保留期；数据量大时关掉内置清理，改成按时间分区、过期直接 DROP PARTITION，避免大 DELETE 拖垮库；同时从源头减少数据：拉长采集间隔、砍掉无价值 item。

### 12. Zabbix 分布式架构如何实现的？

**要点：**
- Proxy 部署在分支/机房侧：代替 Server 采集数据并暂存，再回传 Server
- 两种模式：主动 proxy（proxy 连 server 拉配置、推数据，推荐）与被动 proxy
- 配置代理指向：agent 的 ServerActive 指向 proxy；server 端建 proxy 并把主机指给 proxy
- 好处：跨地域集中监控、server 采集压力下放、网络隔离环境可用
- Zabbix 7.0 起 proxy 支持负载均衡 + HA

**一句话答：**
用 Proxy 把采集压力下放到各机房：proxy 主动连 server 拉配置、采集后暂存再回传，agent 指向 proxy 而不是 server，server 端建好 proxy 并把主机关联过去；这样跨地域集中监控，server 不用直面几千个 agent。

### 13. Zabbix 7.0 新增 Proxy HA 集群了解吗？

**要点：**
- 同一组多个 proxy 可指向同一个 server（最多 25 个成员组成一个 proxy 组）
- server 端配置 load balancing + failover：多个 proxy 分担同一组主机
- 某 proxy 掉线，其主机自动由组内其他 proxy 接管，无需人工切换
- 适用于多地部署容错；监控目标自动在组内均衡

**一句话答：**
Zabbix 7.0 让多个 proxy 组成一个 proxy 组共同服务一组主机：server 自动负载均衡，某台 proxy 挂了组内其他 proxy 自动接管它的主机，不用再靠 keepalived 之类的手工方案做 proxy 高可用。

### 14. Zabbix 监控系统如何优化？

**要点：**
- 采集：多用主动模式 agent、调大不可用项的采集间隔、关闭无用 item
- 触发器：表达式避免 avg(#1h) 这类重计算，控制 trigger 数量
- 数据库：history/trends 保留期精简、分区表 + DROP PARTITION、独立数据库服务器、PG+压缩
- Server：增加 poller 进程数、proxy 分担采集
- Web：开启缓存、前端与 API 调用频率控制

**一句话答：**
从采集、触发器、数据库三层优化：采集上多用主动模式和 proxy，砍掉无价值 item；触发器避免长时间窗口聚合；数据库是最常见瓶颈——独立部署、分区表按天删除、精简保留期，必要时加大 poller 进程。

### 15. 自动发现和自动注册是什么？有什么应用场景？

**要点：**
- 网络自动发现（discovery）：server 按规则扫描 IP 段（zabbix_get 探测），发现后按条件执行 action 添加主机并链模板
- 自动注册（active agent auto-registration）：主动模式 agent 上线后向 server 报到，action 按 HostMetadata 匹配自动添加主机
- 场景：云上自动扩容的机器、批量装机、IDC 大量主机统一纳管
- 自动注册更精准安全，不依赖网段扫描

**一句话答：**
自动发现是 server 主动扫网段找到新机器并按规则加监控；自动注册是主动模式 agent 上线后带 HostMetadata 向 server 报到，server 按 action 自动建主机链模板。云上弹性扩容、批量装机场景基本都靠这两个机制自动纳管。

### 16. 如何监控 Web 服务的可用性和响应时间？

**要点：**
- Zabbix Web 场景（Web scenario）：配置步骤访问 URL，检查响应码/内容关键字
- 指标：每步响应时间、下载速度、HTTP 状态码；失败可触发告警
- 用宏变量管理登录场景（{$USER}/{$PASS}）模拟登录后检测
- 也可配合外部拨测（httping/curl 脚本 + UserParameter）

**一句话答：**
用 Zabbix 的 Web scenario：配置一个场景按步骤访问页面，校验响应码和页面关键字，自动生成响应时间、状态码监控项，失败即告警；需要登录的页面用宏传账号密码模拟登录后再检测。

### 17. Zabbix 如何实现自动监控上百台服务器？

**要点：**
- 统一模板 + 主机组：同类角色（web/db/cache）共用模板，阈值用宏差异化
- 批量装 agent：Ansible playbook 推 zabbix-agent + 配置
- 自动注册：HostMetadata 标记角色（如 linux-web），action 自动分组、链对应模板
- 网络 HUD 差异化监控：分角色模板叠加
- 完成后统一看 Host 可用性大屏

**一句话答：**
Ansible 批量装 agent，配置 HostMetadata 标注角色，agent 一上线自动注册——action 按角色自动分组并链接对应模板，阈值用主机宏差异化，上百台机器基本零手工就能全部纳管。

### 18. Zabbix 如何有效避免告警风暴？

**要点：**
- 触发器依赖：机房/宿主机故障作为父 trigger，下游主机全部依赖它，父告警抑制子告警
- Action 侧：同源告警聚合（升级间隔拉长）、按严重度分级路由
- 维护期：变更窗口静默
- 触发器防抖：用 nodata(#3)、连续 N 次判定而非单次阈值
- 通知渠道分级：warning 只进群，high 以上才电话

**一句话答：**
核心是触发器依赖加分级：宿主机或交换机故障做父触发器，抑制下游一串子告警；action 按严重度分流——warning 进群、high 以上才打电话；变更窗口用维护期静默，触发器再用连续多次判定防抖。

## Jenkins 持续集成面试题

### 1. Jenkins 在 CI/CD 中的作用是什么？

**要点：**
- CI/CD 编排引擎：接收触发（webhook/定时/手动），按 Pipeline 定义执行构建、测试、部署
- 插件生态庞大：对接 Git、Maven、Docker、K8s、SonarQube、通知等几乎一切工具
- 分布式构建：master 调度 + agent 执行，可弹性扩展
- 本质价值：把"从代码到上线"的手工步骤固化成可重复的自动化流水线

**一句话答：**
Jenkins 是流水线的编排引擎：代码一提交通过 webhook 触发，它按 Jenkinsfile 调度构建、测试、镜像、部署各环节，靠插件对接 Git、Docker、SonarQube、K8s 等工具，再通过 agent 做分布式执行，把发布流程完全自动化。

### 2. Jenkins 与 Gitlab CI/CD 功能有什么区别？

**要点：**
- Jenkins：独立服务，插件 1800+，适配异构老项目和复杂流程，需要自己维护 master/agent
- GitLab CI：GitLab 内置，.gitlab-ci.yml 声明式 YAML，与代码库/权限/MR 一体化，维护成本低
- Jenkins UI 功能多但配置分散在插件里；GitLab CI 配置即代码，review 友好
- 选型：已有 GitLab 且流程标准 → GitLab CI；复杂异构、多源仓库、老技术栈 → Jenkins

**一句话答：**
Jenkins 靠插件堆功能，什么都能接但得自己维护服务；GitLab CI 是 GitLab 内置的，.gitlab-ci.yml 和代码库、MR、权限天然一体，维护成本低。一般标准流程用 GitLab CI，异构复杂的老项目和多源仓库场景用 Jenkins。

### 3. Jenkins 你都用过哪些插件？

**要点：**
- Git / GitLab / GitHub：源码对接与 webhook 触发
- Pipeline / Blue Ocean：流水线与可视化
- Credentials Binding / SSH Pipeline Steps：凭据与远程执行
- Docker / Kubernetes / Kubernetes CLI：镜像构建与 K8s 动态 agent
- SonarQube Scanner、JUnit、Jacoco：质量与测试报告
- Email Extension、钉钉/企业微信通知插件：构建通知
- Role-based Authorization Strategy：权限管理

**一句话答：**
常用的是 Git 和 GitLab 插件做源码对接，Pipeline 加 Blue Ocean 写流水线，Credentials 管凭据，Kubernetes 插件做动态 agent，SonarQube Scanner 和 JUnit 卡质量，再加上 Email Extension 和企业微信插件发通知，权限用 Role-based 插件。

### 4. Freestyle 项目与 Pipeline 项目有什么区别？

**要点：**
- Freestyle：表单点选配置，逻辑散在 UI 里，无法版本化，复杂流程受限
- Pipeline：Jenkinsfile（Groovy DSL）描述整个流水线，可进 Git 版本管理、可 code review
- Pipeline 支持 stage 并行、when 条件、post 失败处理、共享库复用
- 团队协作与审计需求下基本都选 Pipeline（Pipeline as Code）

**一句话答：**
Freestyle 是在界面上点选配置，没法版本化、逻辑复杂了就乱；Pipeline 用 Jenkinsfile 描述流水线，能进 Git 做 review 和复用，支持并行、条件、失败后处理，现在基本都 Pipeline as Code。

### 5. 简述 Jenkins 分布式构建的工作原理？

**要点：**
- Master（controller）：管理 UI、任务调度、配置存储，不跑重构建
- Agent：执行节点，通过 JNLP（inbound）或 SSH（outbound）连接 master
- 任务按 label 匹配 agent：node('build-node') 指定执行节点
- Agent 类型：常驻（物理机/虚机）或 K8s Pod 动态创建销毁
- 工作空间：每个 agent 上维护 workspace，构建产物按需归档回 master

**一句话答：**
Master 只管调度和 UI，构建分发到 agent 执行：agent 用 JNLP 或 SSH 注册到 master，任务按 label 路由到对应节点，K8s 环境下 agent 可以是按需创建的 Pod，跑完即销毁，这样 master 不会被重构建拖垮。

### 6. 简述 Jenkins Pipeline 的工作原理？

**要点：**
- Jenkinsfile 定义 stages/steps，Pipeline 从 SCM 拉取后解析执行
- 声明式：pipeline { agent any / stages / post }；脚本式：node { } 更灵活
- 执行模型：Groovy 解析成 CPS 流程，step 逐个在 agent 上执行
- stage 视图可视化，post 块处理成功/失败/always
- 支持共享库抽取公共逻辑，参数化构建 input 审批

```groovy
pipeline {
  agent any
  stages {
    stage('Build') { steps { sh 'mvn -B package' } }
    stage('Test')  { steps { sh 'mvn test'; junit '**/target/surefire-reports/*.xml' } }
  }
  post { failure { emailext subject: '构建失败', to: 'dev@team.com' } }
}
```

**一句话答：**
Jenkinsfile 用 stages 和 steps 描述流水线，Jenkins 从仓库拉取后解析成可执行流程，step 逐个下发给 agent 执行，stage 视图实时展示进度，post 块统一处理成败通知，公共逻辑还能抽到共享库复用。

### 7. Jenkins 中的触发器有哪些类型？

**要点：**
- Webhook：Git push/MR 即时触发（最常用）
- 轮询 SCM：Jenkins 定时 poll（配置 */5 * * * * 之类）
- 定时构建：cron 表达式（如夜间回归 H 2 * * *）
- 上游任务触发：build after other projects are built（job 链）
- 手动/参数化触发、GitLab MR 事件触发

**一句话答：**
主要有五类：Git 仓库的 webhook 即时触发、定时 cron 构建、轮询 SCM、上游 job 完成后链式触发，加上手动参数化触发；生产上优先 webhook，夜间回归用定时。

### 8. Jenkins 大量构建作业如何管理与优化？

**要点：**
- 用 Jenkins Template/Job DSL 或多分支流水线批量生成 job，避免手工几百个
- Pipeline 共享库收敛公共逻辑，改一处全局生效
- 分文件夹/视图管理，命名规范（项目-环境-类型）
- 清理：discarded builds 策略、废弃 job 定期归档删除
- 负载：agent 分池（按项目/环境 label），控制并发与队列

**一句话答：**
关键是别手工建 job：用 Job DSL 或多分支流水线模板化生成，公共逻辑进共享库，job 按项目分文件夹管理，构建记录设保留策略，执行资源按 label 分池控制并发。

### 9. Jenkins 构建队列管理机制是什么？

**要点：**
- 触发的构建先进 Queue 等待可用 executor
- 可配置：job 级并发限制、agent 上的 executor 数、节点独占（instance cap）
- 优先级插件可调整任务权重；静默期（quiet period）合并短时间内的连续触发
- 队列拥堵治理：扩 agent、加 executor、K8s 动态 agent、削减无效触发

**一句话答：**
构建先排队等 executor：Jenkins 按 agent 的 executor 数和 job 的并发限制调度，quiet period 会把短时间的连续触发合并，队列堵了就加 agent、用 K8s 动态扩容或限制单 job 并发。

### 10. Jenkins 如何自定义构建通知？

**要点：**
- post 块 + 插件：emailext 邮件、钉钉/企业微信/Slack 插件发群消息
- webhook 机器人：curl 直接 POST 自定义内容（模板灵活）
- 通知内容：任务、分支、构建结果、耗时、日志链接
- 按需分层：失败必发、恢复时发、成功只在部署类 job 发

```groovy
post {
  failure {
    sh '''
      curl -s -X POST "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx" \
        -H 'Content-Type: application/json' \
        -d '{"msgtype":"text","text":{"content":"构建失败: '${JOB_NAME}' #'${BUILD_NUMBER}'"}}'
    '''
  }
}
```

**一句话答：**
在 pipeline 的 post 块里做：用 emailext 发邮件，或直接 curl 企业微信/钉钉群机器人 webhook 推送自定义内容，把 job 名、分支、结果和日志链接带上，失败必发、恢复再发，成功只在部署类任务发，避免刷屏。

### 11. 如何在 Jenkins 中安全管理敏感信息？

**要点：**
- Credentials Store 存密码/SSH key/token，流水线用 withCredentials 引用，日志自动打码
- 禁止明文写进 Jenkinsfile 或全局环境变量
- 企业级可对接 Vault；K8s 场景用 Secret 挂载
- 权限收敛：Role-based 插件按项目分权，凭据使用有审计
- 凭据定期轮换，最小权限原则

```groovy
withCredentials([usernamePassword(credentialsId: 'git-cred',
                  usernameVariable: 'U', passwordVariable: 'P')]) {
  sh 'git clone https://${U}:${P}@git.example.com/team/app.git'
}
```

**一句话答：**
凭据统一放 Jenkins Credentials Store，流水线用 withCredentials 注入，日志会自动脱敏，绝不把密码写进 Jenkinsfile；权限上用 Role-based 插件按项目分权，要求更高就对接 Vault 并定期轮换。

### 12. Jenkins 如何实现多环境配置与管理？

**要点：**
- 参数化流水线：deploy_env 参数（dev/staging/prod）决定部署目标和审批
- 配置分离：各环境配置放独立分支/目录或配置中心（Nacos/Apollo），不打进镜像
- 凭据按环境分 credentialsId，避免混用
- 生产环境加 input 人工审批 + 具体 agent 部署
- 环境差异用 Ansible inventory 或 K8s 多 namespace 隔离

**一句话答：**
用参数化构建区分环境：deploy_env 参数决定发布到哪个集群和用哪套凭据，应用配置不进镜像、走配置中心或独立配置文件管理，测试环境自动发，生产环境加 input 审批节点，环境间用 K8s namespace 或 inventory 隔离。

### 13. Jenkins 如何实现跨项目参数传递？

**要点：**
- Trigger 参数化构建插件：上游 build job: 'downstream', parameters: [...] 传参
-下游用 parameters 声明接收
- 归档传递：archiveArtifacts + Copy Artifact 插件拿上游产物
- API 方式：上游通过 REST 触发下游并 POST 参数
- 复杂编排建议直接写成一个 pipeline 的多个 stage，避免 job 链碎片化

```groovy
build job: 'deploy-service', parameters: [
  string(name: 'IMAGE_TAG', value: "${env.GIT_COMMIT[0..7]}"),
  string(name: 'DEPLOY_ENV', value: 'staging')
]
```

**一句话答：**
常用两种：build job 传 parameters 触发下游并带参，下游用 parameters 声明接收；或者上游 archiveArtifacts 存产物、下游用 Copy Artifact 插件取。如果是强关联流程，我更愿意合并成一个 pipeline 的多个 stage，比 job 链好维护。

### 14. 简述 Jenkins 共享库（Shared Library）作用？

**要点：**
- 把公共 pipeline 逻辑抽成独立 Git 仓库：vars/（全局函数）、src/（Groovy 类）、resources/
- Jenkins 系统级配置全局共享库，Jenkinsfile 里 @Library('mylib') 引用
- 收益：几十个项目的 Jenkinsfile 瘦身成几行标准调用；规范统一、一次修复全局生效
- 支持版本化（tag/branch），升级可控

```groovy
// vars/deployK8s.groovy
def call(Map cfg) {
  sh "kubectl set image deploy/${cfg.name} ${cfg.name}=${cfg.image} -n ${cfg.ns}"
}
// Jenkinsfile 中：deployK8s(name: 'api', image: 'reg/app:v1.2', ns: 'prod')
```

**一句话答：**
共享库是把所有项目的公共流水线逻辑抽到一个 Git 仓库，vars 放自定义 step、src 放类，Jenkinsfile 里 @Library 引用后一行函数就能完成部署，改一处全部项目生效，Jenkinsfile 从几百行瘦身到十几行。

### 15. Jenkins 如何应对大规模构建环境？

**要点：**
- Master 只调度不构建，构建全下放 agent（K8s Pod agent 按需伸缩最理想）
- 构建提速：依赖缓存（Maven/npm 代理仓库 Nexus）、并行 stage、增量构建
- 治理：job 模板化（Job DSL/共享库）、构建记录清理、磁盘与 workspace 定期回收
- 高可用：JCasC 配置即代码 + 定期备份 JENKINS_HOME；按团队/环境拆多个 Jenkins 实例也是常见解法
- 监控 Jenkins 自身：队列长度、executor 利用率、内存

**一句话答：**
核心思路是让 master 只做调度、构建全放 agent，用 K8s 动态 Pod agent 按需伸缩；再配 Nexus 缓存依赖、并行化流水线提速，job 用共享库模板化管理，配置用 JCasC 加定期备份，实在大了就按业务线拆多个 Jenkins 实例。

### 16. 如何实现代码提交自动触发 Jenkins CI ？

**要点：**
- GitLab/GitHub 仓库配置 Webhook 指向 Jenkins（GitLab 插件 URL + token）
- Jenkins job 配置 GitLab trigger 或 Generic Webhook Trigger
- 事件过滤：分支、MR 事件、路径过滤（docs 变更跳过）
- 网络不可达时退化为轮询 SCM（polling）
- 验证：仓库侧发送测试请求，Jenkins 日志确认触发源

**一句话答：**
在 Git 仓库配 Webhook 指向 Jenkins 的 GitLab 插件端点，Jenkins 侧勾选对应触发器并配置分支和路径过滤，push 或 MR 一进来就自动构建；网络不通的环境就用轮询 SCM 兜底。

### 17. Jenkins 怎么进行数据备份？

**要点：**
- 核心是备份 JENKINS_HOME：config.xml、jobs/、credentials、plugins 清单
- 频率：每天定时；轻量做法排除 workspace、builds 大目录
- 工具：ThinBackup 插件（定时备份到 NFS/对象存储）、或 rsync/打包脚本 + crontab
- 配置即代码：JCasC 把主配置代码化，插件清单 plugins.txt，恢复更快
- 演练恢复：定期验证备份可用

```bash
tar czf jenkins_home_$(date +%F).tar.gz -C /var/lib/jenkins \
  --exclude='workspace' --exclude='*/builds' config.xml jobs/ users/ secrets/
```

**一句话答：**
备份 JENKINS_HOME 就是备份一切：用 ThinBackup 插件或 tar 定时打包 config.xml、jobs、users、secrets，排除 workspace 和历史构建，传到 NFS 或对象存储；再配合 JCasC 和 plugins.txt 把配置代码化，恢复时先起容器再导回即可，并定期演练。

## ELK 日志管理面试题

### 1. ELK Stack 都有哪些组件？

**要点：**
- Elasticsearch：分布式搜索/分析引擎，存储和查询日志
- Logstash：采集、解析、转换日志
- Kibana：可视化查询与看板
- Beats 家族：轻量采集器（Filebeat 日志文件、Metricbeat 指标、Auditbeat 审计）
- ELK + Beats 合称 Elastic Stack；大架构常加 Kafka/Redis 做缓冲层

**一句话答：**
核心三件套是 Elasticsearch 负责存储检索、Logstash 负责解析加工、Kibana 负责查询展示，采集端现在普遍用 Filebeat 这类轻量 Beats 代替 Logstash 直连，量大时中间再加 Kafka 削峰。

### 2. Elasticsearch 有哪些节点类型及其作用？

**要点：**
- Master eligible：参与选主、管理集群状态（分片分配、索引创建）
- Data：存数据、执行 CRUD 与聚合查询（可分 data_hot/warm/cold 分层）
- Ingest：写入前的管道预处理（类似轻量 logstash）
- Coordinating：请求路由与结果归并（所有节点默认都是 coordinating）
- 生产建议：专用 master（3 台奇数）+ 专用 data，各司其职

**一句话答：**
主要有四类：master eligible 负责集群元数据和选主，data 负责存数据跑查询，ingest 负责写入前预处理，coordinating 负责请求路由和结果合并；生产上建议 3 台专用 master 加独立 data 节点分离部署。

### 3. 简述 Elasticsearch 集群工作原理？

**要点：**
- 多节点组成集群，数据按 index → shard 分布，每个分片是独立 Lucene 索引
- 主分片数建索引时定死，副本分片数可动态调整；分片分布在各 data 节点
- 写入：请求到 coordinating 节点，按路由（默认 _id hash）转发到主分片，同步副本后返回
- 查询：scatter-gather——广播到所有相关分片，各节点局部排序，coordinating 归并
- Master 维护集群状态并下发到各节点

**一句话答：**
ES 把索引拆成多个分片分布到 data 节点，主分片定死、副本可调；写入时 coordinating 节点按路由规则转发主分片、等副本确认；查询用 scatter-gather 两阶段——各分片本地查、coordinating 归并排序，master 统一管理集群元数据。

### 4. 简述  Elasticsearch 集群节点选举机制？

**要点：**
- master-eligible 节点通过投票选出 master；7.x 起基于 Raft 类算法（Coordination 模块），需奇数个投票节点
- cluster.initial_master_nodes 仅首次组网用；之后由集群状态记录
- 选举 quorum 要求：投票配置中过半节点可用，否则集群不可写（防脑裂）
- 故障时剩余 eligible 节点重新选举，恢复后自动重新加入
- 相关：voting config exclusions（缩容 eligible 节点前先排除）

**一句话答：**
7.x 之后 eligible 节点基于 Raft 类投票机制选 master，必须凑齐投票配置的过半节点才能选出主并写入，天然防脑裂，所以 master 建议三台奇数部署；master 挂了剩余节点自动重新选举。

### 5. ES 集群健康状态 green / yellow / red 分别代表什么？

**要点：**
- green：所有主分片和副本分片都正常分配
- yellow：主分片全在，有副本分片未分配（常见：单节点集群有副本索引）
- red：有主分片丢失/未分配——该索引部分数据不可读不可写
- 查看命令：_cat/health、_cat/shards?v 查未分配分片及原因（_cluster/allocation/explain）

**一句话答：**
green 全部主副本分片正常；yellow 是主分片完好但有副本没分配，通常是单节点集群却配了副本，不影响数据完整性；red 是有主分片丢了，对应索引数据残缺，要立刻处理。

### 6. ES 集群 red 状态如何恢复？

**要点：**
- 先定位：GET _cluster/allocation/explain 看分片为什么没分配（磁盘水位/节点离线/损坏）
- 节点重启类原因：节点回来后分片自动恢复；确认集群 watermark 未触顶
- 磁盘满：扩容或清理，必要时临时调高 flood_stage 水位恢复写入
- 节点彻底丢了：可对丢失副本的索引用 _shrink 之外的方式——实际操作是重索引或接受丢数据；日志类可缩短保留期直接删 red 索引
- 校验：恢复后 _cat/health 回 green/yellow，验证业务读写

**一句话答：**
先用 _cluster/allocation/explain 找原因：多数是节点离线或磁盘满——节点恢复或清理磁盘后分片会自动分配回来；如果是节点彻底坏掉的日志索引，权衡后直接删除重建；恢复后用 _cat/health 确认回到 green。

### 7. ES 查询慢、写入慢怎么排查？

**要点：**
- 慢查询：开启 slowlog（search.slowlog.threshold），看 query/fetch 阶段耗时
- 常见原因：深分页（from+size 大翻页）、 wildcard 前缀通配、大聚合、过多分片扇出
- 写入慢：refresh_interval 过短（默认 1s）、副本数多、mapping 过多字段、bulks 太小或并行不够
- 分片不均/热点：_cat/shards 检查；节点资源 CPU/IO 打满
- index sorting、合理分片大小（单分片 10-50GB 量级）

**一句话答：**
先开 slowlog 定位慢在哪：查询慢多见深分页、前缀通配查询、大聚合；写入慢常见 refresh_interval 太短、副本太多、bulk 批次太小，再用 _cat/shards 看分片是否倾斜、单分片是否过大，最后看节点 CPU 和磁盘 IO。

### 8. ES Mapping 有什么用途？

**要点：**
- 定义字段类型（text/keyword/date/numeric/nested）与分析方式——类似数据库表结构
- text 分词可搜全文，keyword 不分词可精确匹配、排序、聚合
- dynamic mapping 自动推断，但可能不符合预期（数字变 long、日志 message 只给 text）
- 显式 mapping + 合理 index 模板是日志场景标配；mapping 字段数过多影响性能

**一句话答：**
Mapping 就是索引的表结构：定义每个字段什么类型、是否分词——message 用 text 分词检索，service 用 keyword 做精确过滤聚合；生产上要用显式 mapping 和索引模板，避免 dynamic mapping 把类型推错。

### 9. ES ILM 是什么？有什么应用场景？

**要点：**
- Index Lifecycle Management：索引生命周期策略，自动管理 hot → warm → cold → delete
- 基于 rollover/max_age/min_size 条件自动切换阶段
- 日志场景标配：热节点写 1 天 → 滚动 → 7 天后转 warm → 30 天删除
- 需要 index template + ILM policy + rollover 别名配合
- 减少 shard 数、控制成本、免维护脚本

**一句话答：**
ILM 是索引生命周期管理：定义 hot、warm、cold、delete 四个阶段的策略，日志索引写满一天或到一定大小自动 rollover，超期自动转温冷存储、到保留期自动删除，全程无需人工跑脚本。

### 10. ES 如何实现自动删除 30 天前索引？

**要点：**
- 方案一：ILM——policy 的 delete 阶段 min_age: 30d，索引模板绑定该 policy
- 方案二：定时任务脚本按日期匹配删除（date math 也支持 <logs-{now/d-30d}>）
- curator（Elastic 官方 Python 工具）delete_indices 按年龄删除
- 用 action 而非手删：先 delete 再确认磁盘水位回落

```json
PUT _ilm/policy/logs-30d
{ "policy": { "phases": {
  "delete": { "min_age": "30d", "actions": { "delete": {} } }
}}}
```

**一句话答：**
首选 ILM：建一个 delete 阶段设 min_age 30d 的 policy，通过索引模板绑到日志索引上自动删；没有 ILM 的老版本就用 curator 或 crontab 脚本按日期匹配删除 30 天前的索引。

### 11. ES 集群磁盘快满了，如何进行扩容？

**要点：**
- 应急：先删/缩旧索引（ILM 提前删除期）、清空回收站、必要时临时调高磁盘水位（watermark）恢复写入
- 短期：加节点会触发自动 rebalance（控制 recovery 速度避免拖垮集群）
- 存量数据迁移：_shrink 或 reindex 到新磁盘节点
- 长期：扩磁盘、设置 ILM 滚动避免单索引过大、监控 watermark 告警提前介入
- 谨慎：watermark 是保护机制，关闭前先评估 OOM 风险

**一句话答：**
先应急清理：删旧索引、临时调高 watermark 恢复写入；然后加数据节点扩容，ES 会自动平衡分片，注意用 cluster.routing.allocation 控制 recovery 速度；长期要靠 ILM 控制索引规模和磁盘告警提前介入。

### 12. ES 集群数据如何备份？

**要点：**
- 官方方案 snapshot：注册共享文件系统仓库（fs/NFS）或对象存储（S3/HDFS/OSS）
- 全量 + 增量快照，可恢复到原集群或新集群
- 8.x 起 elastic 无需 license 即可用（Basic license 即可）
- 定期 crontab snapshot + 保留策略清理旧快照
- 恢复：restore API 可选索引、重命名索引恢复

```bash
PUT _snapshot/my_backup
{ "type": "fs", "settings": { "location": "/mount/backup" } }
PUT _snapshot/my_backup/snap_2026_09_10?wait_for_completion=true
```

**一句话答：**
用 snapshot 快照机制：注册 fs 或 S3 类仓库，定时做全量加增量快照并设保留期，恢复时用 restore API 选择性恢复到原集群或新集群，这是官方唯一推荐的备份方式。

### 13. ES 如何性能优化？

**要点：**
- 硬件/部署：SSD、内存给系统 cache 留一半、专用 master、按角色分节点
- 分片：单分片 10-50GB、总数适中避免 over-sharding；副本按可用性需求
- 写入：bulk 批量写、refresh_interval 调大（30s）、必要时副本先 0 后 1（大批量导入）
- 查询：filter 上下文（可缓存）、避免深分页（search_after）、keyword 做 terms 聚合
- 日志场景：ILM 滚动、必要字段才索引、关闭 _source 或压缩不可取（谨慎）

**一句话答：**
几层优化：部署上专用 master 加 SSD、内存留一半给文件缓存；分片控制在 10 到 50GB 一个；写入用 bulk、调大 refresh_interval；查询多用 filter 缓存、深分页换 search_after；日志场景配 ILM 滚动和字段精简。

### 14. ES 集群一般会监控哪些指标？

**要点：**
- 集群健康：green/yellow/red、节点数、未分配分片数
- 资源：各节点 CPU、JVM 堆使用率与 GC、磁盘使用率与水位
- 写入/查询：indexing rate/latency、search rate/latency、thread pool 拒绝数（write/search rejected）
- 分片：单节点分片数、分片大小分布、relocating/unassigned
- 熔断：circuit breaker 触发、FiELDDATA/SHIndex cache 占用

**一句话答：**
重点盯四类：健康度和未分配分片、节点资源尤其 JVM 堆和 GC、读写延迟加线程池拒绝数、还有分片分布是否均衡和磁盘水位，再配上 circuit breaker 触发情况。

### 15. 简述 Logstash 工作流程？

**要点：**
- 三段式管道：input → filter → output，队列衔接
- input 插件接收/拉取数据（beats/tcp/file/kafka）
- filter 解析加工：grok 正则解析、mutate 字段处理、date 时间解析、geoip
- output 输出：elasticsearch、kafka、file 等
- pipeline.workers（并发数）、batch.size、pipeline.batch.delay 决定吞吐

```
input  { beats { port => 5044 } }
filter {
  grok { match => { "message" => "%{COMBINEDAPACHELOG}" } }
  date { match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"] }
}
output { elasticsearch { hosts => ["http://es:9200"] } }
```

**一句话答：**
Logstash 是 input、filter、output 三段管道：从 beats 或 kafka 收数据，filter 里用 grok、date、mutate 解析字段，最后批量写进 ES，吞吐由 pipeline workers 数和 batch size 控制。

### 16. Logstash 有哪些应用场景？

**要点：**
- 日志结构化：grok 解析 nginx/java/系统日志，拆出状态码、耗时等字段
- 数据清洗富化：geoip IP 转地理位置、用户代理解析、字段增删改
- 数据搬运枢纽：kafka→ES、多源汇聚、格式转换
- 与 Beats 分工：Beats 轻采集，Logstash 做"重处理"

**一句话答：**
主要做日志的结构化解析——grok 把 nginx、Java 异常日志拆成字段，geoip 富化 IP 归属地，以及多源数据的汇聚转换，比如从 Kafka 消费后写入 ES，跟 Filebeat 是"轻采集加重处理"的配合。

### 17. Logstash Pipelines（多管道）是什么？

**要点：**
- 一个 Logstash 进程跑多条独立 pipeline（pipelines.yml 定义），各自 input/filter/output
- 隔离：不同管道互不影响（一条堵不拖累其他），配置独立管理
- 资源共享：省掉多进程部署的资源开销
- 适用：多类型日志不同解析逻辑、多环境汇聚

```yaml
# pipelines.yml
- pipeline.id: nginx
  path.config: "/etc/logstash/conf.d/nginx.conf"
- pipeline.id: app
  path.config: "/etc/logstash/conf.d/app.conf"
```

**一句话答：**
多管道是在一个 Logstash 进程里通过 pipelines.yml 跑多条互不干扰的 pipeline，每条有独立的 input、filter、output 和 worker 配置，不同日志走不同解析逻辑又共享进程资源。

### 18. Logstash 过滤器有哪些常用插件？

**要点：**
- grok：正则模式解析非结构化日志（内置 COMBINEDAPACHELOG 等模式）
- mutate：字段增删改、类型转换、重命名
- date：解析日志时间替代 @timestamp
- geoip：IP 归属地解析
- kv / json / csv：键值对、JSON、CSV 格式拆解
- dissect：定界符快速拆分（比 grok 快但灵活性低）
- drop / prune：丢弃事件或字段

**一句话答：**
最常用 grok 做正则解析，mutate 改字段和类型，date 修正 @timestamp，geoip 解析 IP 归属地，还有 kv、json 拆格式，dissect 做定界符快速切分，性能敏感就用 dissect 加少量 grok。

### 19. Logstash 如何性能优化？

**要点：**
- 纵向：pipeline.workers = CPU 核数、batch.size 适当调大（125→250/500）、JVM 堆 4-8GB
- 横向：多实例 + Kafka 分区并行消费
- 解析降本：dissect 替代复杂 grok、能用 json 就别正则
- 磁盘持久化 queue（persisted queue）替代内存队列防丢
- 监控 pipeline 滞后，慢了先看 filter 阶段

**一句话答：**
先是单机调参：workers 跟 CPU 核数一致、batch.size 调大、JVM 堆给 4 到 8G；解析上能 dissect、json 就不用 grok 正则；量再大就横向扩多实例配 Kafka 分区并行，另外开持久化队列防数据丢失。

### 20. Kibana 中的 Discover 功能是什么？

**要点：**
- 数据探索界面：按时间范围浏览原始文档，支持 KQL/Lucene 查询过滤
- 字段侧栏可展开查看 Top 值分布，点击字段快速过滤
- 时间直方图概览事件量趋势，快速定位异常时间点
- 可保存为搜索并用于可视化和告警的查询基础

**一句话答：**
Discover 是 Kibana 的日志查询入口：选索引模式和时间范围，用 KQL 过滤文档，字段侧栏能看分布、一键加过滤，配合时间直方图快速定位异常时段，查到的条件可以直接保存复用到看板和告警。

### 21. Kibana Dashboard 你用过哪些图表类型？

**要点：**
- Lens：拖拽式主推工具，快速出柱状/折线/饼图
- TSVB / Timelion：时间序列高级分析（多序列、函数运算、移动平均）
- Data Table / Metric：明细表、单值大盘（QPS、错误数）
- Map：地理分布（配 geoip）
- Logs UI：流式日志查看

**一句话答：**
日常用 Lens 拖拽出图，时间序列复杂计算用 TSVB 或 Timelion，大盘核心指标用 Metric 单值图加 Data Table 明细，有 geoip 就配 Map 地图，告警相关用 Alerting 面板展示触发记录。

### 22. Filebeat 是如何读取日志文件的？

**要点：**
- Harvester：每个文件一个 harvester 逐行读取，发送到 spooler 聚合
- Registry 文件记录每个文件的状态：inode/设备号 + offset（断点）
- Prospector/input 周期扫描 paths 新文件，为新文件启动 harvester
- 新内容检测：读到 EOF 后定期 stat 检查文件是否更新
- inode 识别文件：轮转改名不重读；close_inactive 释放不活跃文件的 harvester

**一句话答：**
Filebeat 为每个文件起一个 harvester 逐行读取，内容进 spooler 聚合后发送，registry 文件记录每个文件的 inode 和 offset，轮转改名也能靠 inode 识别不重复采集，不活跃文件会被 close_inactive 释放 harvester。

### 23. Filebeat 如何保证不丢数据、断点续传？

**要点：**
- offset 记录在 registry（默认 data/registry），发送确认（ACK）后才更新，重启从上次 offset 续读
- 内部队列 + 重试：发送失败退避重试，backoff 策略
- at-least-once 语义：可能重复（如 ES 写入后 ACK 丢失），配 ES 侧 doc_id 去重
- 磁盘满/网络断：数据缓存在内存队列，必要时启用磁盘队列
- 输出确认机制：ES/Kafka/LibLogstash 都有 ACK 支持

**一句话答：**
靠 registry 里的 offset 断点：只有收到输出端的 ACK 才更新进度，没确认就重试，重启从上次 offset 续传，是 at-least-once 语义——极端情况可能重复，但不会丢，重读场景一般能接受。

### 24. ELK 架构中，为什么要加消息队列？

**要点：**
- 削峰填谷：日志洪峰时 Kafka 先扛住，ES 按自己的节奏消费，防 ES 写爆
- 解耦与缓冲：ES/Logstash 挂了数据不丢，恢复后追平
- 多消费者：一份数据同时给 ES、告警系统、数据团队
- 顺序与分区：按 host/service 分区保证局部有序
- 代价：多维护一套组件，增加延迟（秒级）

**一句话答：**
加 Kafka 主要为削峰和解耦：日志洪峰先落 Kafka，ES 按能力消费不会被写爆，ES 或 Logstash 故障时数据在队列里不丢，还能一份数据多系统消费，代价是多维护一套中间件。

### 25. ELK 架构加消息队列用Redis还是Kfaka？

**要点：**
- Redis（list/pub-sub）：轻量、部署简单；适合小规模（GB 级/天）；持久化弱、堆积能力有限、单点风险
- Kafka：高吞吐（十万级/s）、磁盘持久化、可回溯、分区水平扩展、多消费者；需要 ZK/KRaft 和运维成本
- 百 GB 级别/天以上必须 Kafka；小集群用 Redis 更简单
- 中间量级可用 Redis Cluster，但长期演进一般都上 Kafka

**一句话答：**
小规模日志量用 Redis 部署简单够用；但生产上百 GB 量级必须 Kafka——高吞吐、磁盘持久化、堆积不丢、可回溯重放还能多消费者，Redis 有堆积上限和持久化弱的风险，只适合小体量过渡。

### 26. 如何设计一个每天百GB日志量的ELK架构？

**要点：**
- 采集层：Filebeat 装应用机（轻量、断点续传），容器环境 DaemonSet
- 缓冲层：Kafka（3 broker、按服务分区），削峰 + 故障缓冲
- 处理层：Logstash 集群消费 Kafka（workers=batch 调优），grok 等解析
- 存储层：ES 集群——3 专用 master + 若干 data（SSD、堆内存一半），ILM hot/warm + 30 天删除
- 展示层：Kibana；监控：ES 自带 exporter + 告警（磁盘水位、JVM、拒绝数）

**一句话答：**
经典四层：Filebeat 采集进 Kafka 削峰，Logstash 集群消费解析后写 ES，ES 用 3 台专用 master 加多个 SSD data 节点并配 ILM 热温分层 30 天删除，Kibana 查询展示，再对磁盘水位和 JVM 监控告警，各层都可独立水平扩展。

### 27. ELK 如何实现告警通知？

**要点：**
- 原生方案：Elastic Alerting（Kibana Watcher/Alerts）——按 ES 查询规则周期检测（错误率突增、关键字），通知 webhook/邮件
- 开源方案：ElastAlert2——规则 YAML（frequency、spike 等类型），独立进程查 ES 发告警
- 复用统一告警：把规则结果推给 Prometheus Alertmanager 或钉钉/企微机器人
- 关注告警治理：聚合、静默、分级，避免告警风暴

```yaml
# ElastAlert2 频次告警示例
name: error-spike
type: frequency
index: logs-*
num_events: 50
timeframe:
  minutes: 5
alert:
  - "post"
http_post_url: "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx"
```

**一句话答：**
两条路线：用 Elastic 原生 Alerting 在 Kibana 里配规则周期检测 ES 数据发通知；或用 ElastAlert2 独立跑 YAML 规则推 webhook；实际生产更常把告警统一汇到钉钉/企微机器人或 Alertmanager 做分级聚合。

### 28. ELK 如何收集 K8s 集群中应用日志？

**要点：**
- 容器日志落盘 /var/log/containers/*.log（stdout/stderr 由 kubelet 软链到 /var/lib/docker/containers）
- Filebeat DaemonSet：hostPath 挂载日志目录 + 容器运行时目录
- 解析：filebeat autodiscover 按容器注解/标签路由解析配置；或 Logstash 按 namespace/pod 过滤
- 字段：kubernetes.pod.name/namespace/labels 元数据自动关联（filebeat kubernetes processor/add_kubernetes_metadata）
- 应用文件日志：emptyDir + sidecar 或直挂 hostPath 采集

**一句话答：**
标准方案是 Filebeat 跑 DaemonSet：hostPath 挂 /var/log/containers 采集容器 stdout 日志，用 kubernetes 元数据 processor 自动关联 pod、namespace、标签，再按标签路由到不同索引；应用写文件日志的话用 sidecar 或直接采集挂出来的文件。

### 29. ELK Stack、Grafana Loki 和 Graylog 如何选择？

**要点：**
- ELK：功能最全（全文检索、聚合分析、APM 生态）；成本高（副本+倒排索引）、运维重
- Loki：只索引标签不索引全文，成本极低、与 Grafana/Prometheus 无缝配合；查询语言 LogQL 弱于 KQL，全文检索能力有限
- Graylog：开箱即用的日志平台（告警、stream、权限完善）；社区版功能够用、企业版收费
- 选型：重度检索分析选 ELK；云原生指标关联+低成本选 Loki；需要一体化日志平台快速落地选 Graylog

**一句话答：**
要全文检索和复杂分析选 ELK，代价是存储和运维成本高；已用 Grafana 监控体系且日志主要按标签查就选 Loki，成本极低；想要开箱即用的日志平台带告警和权限就选 Graylog，我们按"成本、查询方式、现有生态"三个维度定。

### 30. 简述 Grafana Loki  日志管理的工作流程？

**要点：**
- 采集端：Promtail（DaemonSet）或 Grafana Agent/Alloy 抓日志，打上标签（pod、namespace、job）
- 写入： distributor 接收并校验，ingester 内存构建倒排（标签→chunks）+ 后端存储（S3/对象存储）
- 查询： querier（+ query-frontend）按标签选择器（LogQL）定位 chunk，再流式过滤内容
- 单二进制模式适合小规模；微服务模式水平扩展
- 与 Prometheus 标签体系统一，LogQL 和 PromQL 语法相似

**一句话答：**
Loki 是标签索引模型：Promtail 收集日志打上和 Prometheus 一致的标签，发给 distributor，ingester 把日志按标签切成 chunk 存到对象存储，查询时用 LogQL 按标签快速定位 chunk 再做内容过滤，因为不索引全文所以存储成本远低于 ES。

## Ansible 自动化运维面试题

### 1. 简述 Ansible 工作架构和原理？

**要点：**
- 无 agent 架构：控制节点通过 SSH（Linux）/WinRM（Windows）连接被管节点，无需装客户端
- 默认 SSH + sftp 传输模块代码，模块在远端以临时 Python 脚本执行后删除
- 控制节点把 playbook 编译成任务列表，按 inventory 分组、forks 并发执行
- 幂等性：大多数模块检测目标状态，已满足则不做变更
- Python 只需控制端和目标端（Python 2.7+/3.x）有解释器

**一句话答：**
Ansible 是无 agent 的：控制节点通过 SSH 把模块代码推到目标机临时执行后即删，配合 inventory 分组和 forks 并发，大多数模块幂等——重复执行结果一致，所以只要能 SSH 通就能管，不需要在被管机装任何东西。

### 2. Ansible Inventory 是什么？

**要点：**
- 被管主机清单：默认 /etc/ansible/hosts，INI 或 YAML 格式
- 主机分组管理，组可嵌套（children）；支持主机范围展开 web[01:50]
- 主机/组级变量：ansible_host、ansible_user、ansible_port 等
- 单台、多台、组、全all 匹配执行；-i 指定清单文件

```ini
[webservers]
web01 ansible_host=10.0.0.11
web[02:04] ansible_host=10.0.0.[12:14]

[databases]
db01 ansible_host=10.0.1.11

[prod:children]
webservers
databases
```

**一句话答：**
Inventory 是主机清单：把目标机分组（组还能嵌套），每台可定义 IP、端口、用户等变量，执行时按组或通配符匹配，默认读 /etc/ansible/hosts，也支持动态生成。

### 3. 动态 Inventory 有哪些应用场景？

**要点：**
- 主机经常变化（云 autoscaling、容器、K8s），静态清单无法及时同步
- 实现：可执行脚本或插件输出 JSON（_meta.hosts 带变量），Ansible 调用获取
- 常见对接：云厂商插件（EC2/Aliyun）、CMDB（接口拉取）、Consul、K8s
- 云上自动扩缩容场景：新机器拉起后自动纳入管理

**一句话答：**
动态 Inventory 是用一个脚本或插件实时输出 JSON 主机清单，对接云厂商 API、CMDB 或 Consul，适合云上弹性伸缩这类主机频繁变化的场景——新机器一拉起自动就能被 Ansible 管理。

### 4. Ansible 变量传递有哪几种方式？

**要点：**
- Inventory 变量：host_vars/、group_vars/ 目录文件
- Play 变量：vars、vars_files、vars_prompt
- extra-vars：-e "k=v" 命令行注入（优先级最高）
- 事实变量：gather_facts 收集的 ansible_*；注册变量 register
- set_fact 运行中定义、include_vars 动态加载；角色 defaults/ 与 vars/
- 优先级：extra-vars > play vars > role vars > group_vars > host_vars > role defaults

**一句话答：**
常用的有：group_vars/host_vars 文件、playbook 里的 vars 和 vars_files、命令行 -e 传参、gather_facts 的事实变量、register 捕获的执行结果，加上 set_fact 动态定义；记住优先级——命令行 extra-vars 最高，role 的 defaults 最低。

### 5. Ansible Playbook 是什么？

**要点：**
- YAML 格式的任务编排文件：把多个 play（任务集合）按顺序定义
- Play 指定 hosts、vars、tasks、roles，原子任务调用模块
- 相比 ad-hoc 命令：可重复、可版本化、可编排复杂流程
- 支持 handlers、tags、when 条件、循环、模板渲染

```yaml
- hosts: webservers
  become: yes
  tasks:
    - name: install nginx
      apt: { name: nginx, state: present }
    - name: start nginx
      service: { name: nginx, state: started, enabled: yes }
```

**一句话答：**
Playbook 是 YAML 写的任务编排剧本：一个 play 指定目标主机组、变量和任务列表，任务调用模块按顺序执行，支持条件、循环、handlers 和模板，相比临时命令可以版本化复用。

### 6. Playbook 包含哪些核心字段？

**要点：**
- name：play/任务描述；hosts：目标主机组
- become：提权（sudo）；vars / vars_files：变量
- tasks：任务列表（模块调用）；handlers：事件触发的任务
- roles：引用角色；pre_tasks / post_tasks：角色前后任务
- tags：打标签；ignore_errors、when、loop 等任务级控制

**一句话答：**
核心字段：hosts 指定目标、become 提权、vars 定义变量、tasks 列任务、handlers 存重启类延迟任务、roles 引角色，还有 pre_tasks/post_tasks 和 tags，一个 playbook 就是这些字段组合出的多个 play。

### 7. Playbook 中 notify 和 handlers 的作用？

**要点：**
- notify 在任务"发生实际变更"时触发对应 handler（未变更不触发）
- handlers 在所有任务跑完后统一执行一次（去重，多个任务通知同一 handler 只跑一次）
- 典型场景：改配置文件 → notify restart nginx
- listen 关键字：一个事件可触发多个 handler；play 结束时强制 flush_handlers 可立即执行

```yaml
tasks:
  - name: update config
    copy: { src: nginx.conf, dest: /etc/nginx/nginx.conf }
    notify: restart nginx
handlers:
  - name: restart nginx
    service: { name: nginx, state: restarted }
```

**一句话答：**
notify 是任务发生变更时给 handler 发信号，handlers 在 play 结束时统一执行且自动去重——比如改了配置文件才重启 nginx，没变更就不重启，避免无谓重启。

### 8. Playbook 中 tags 有什么用？

**要点：**
- 给任务/play 打标签，执行时只跑指定部分
- --tags "config" 只执行带该标签任务；--skip-tags 跳过；--list-tags 查看
- always 标签：除非 --skip-tags always，否则总是执行
- 大 playbook 调试与局部变更利器

**一句话答：**
tags 用来选择性执行：给任务打标签后，ansible-playbook --tags 只跑指定部分、--skip-tags 跳过某些部分，大 playbook 里改配置不想全量重跑时就靠它。

### 9. 如何调整 ansible-playbook 执行任务时的并发数？

**要点：**
- 默认并发 forks=5；命令行 -f/--forks 调整（如 -f 20）
- 配置文件 ansible.cfg [defaults] forks = 20 持久化
- 并发过高注意：目标机 SSH 连接数、控制机资源、数据库连接池冲击
- serial 字段：控制每批主机数（灰度/滚动场景），与 forks 是两回事

**一句话答：**
用 -f 或 --forks 调并发，默认 5，也可以写进 ansible.cfg 持久化；注意别把目标机 SSH 或共享服务压垮，滚动发布场景还要配合 serial 控制每批主机数。

### 10. Playbook 与 Role 什么关系？

**要点：**
- Role 是按功能组织的可复用单元：tasks、handlers、templates、files、vars、defaults 按约定目录拆分
- Playbook 是编排入口：通过 roles 字段引用一个或多个 role
- 角色 = 组件化（nginx、mysql 各一个），playbook = 组装（web 服务器 = nginx + php + redis）
- Ansible Galaxy 分享/复用角色；ansible-galaxy init 生成骨架

**一句话答：**
Role 是把一类功能的任务、模板、变量按目录约定封装成的可复用组件，Playbook 是编排入口负责把 role 组装起来——比如 nginx、mysql 各做一个 role，playbook 里按主机角色分别引用。

### 11. 一个 Role 目录结构是怎样的？

**要点：**
- tasks/：主任务文件（main.yml 入口）
- handlers/：处理器；templates/：Jinja2 模板；files/：静态文件
- vars/：高优先级变量；defaults/：默认变量（可被覆盖）
- meta/：依赖与元信息；README 说明
- defaults 优先级低于 vars，覆盖场景常用

```
roles/nginx/
├── tasks/main.yml
├── handlers/main.yml
├── templates/nginx.conf.j2
├── files/
├── vars/main.yml
├── defaults/main.yml
└── meta/main.yml
```

**一句话答：**
标准结构是 tasks、handlers、templates、files、vars、defaults、meta 七个目录各放 main.yml 入口：tasks 写任务，templates 放模板，defaults 放可被覆盖的默认变量，vars 放高优先级变量，meta 声明角色依赖。

### 12. Jinja2 模板是什么？

**要点：**
- Ansible template 模块用 Jinja2 渲染配置：{{ var }} 变量、{% if %}/{% for %} 逻辑
- filters 管道处理：default、upper、to_json、ipaddr 等
- 配置模板化：端口、地址、参数从变量注入，一份模板多环境复用
- lookup 插件可读外部数据；模板以 .j2 结尾约定

```
server {
  listen {{ nginx_port }};
  {% for upstream in nginx_upstreams %}
  server {{ upstream.host }}:{{ upstream.port }};
  {% endfor %}
}
```

**一句话答：**
Jinja2 是 Ansible 的模板引擎：用 {{ 变量 }} 和 {% if %}/{% for %} 把配置参数化，配合 template 模块渲染出各环境的配置文件，比如同一份 nginx.conf.j2 按变量生成不同端口的配置。

### 13. Ansible Facts 是什么？

**要点：**
- gather_facts（默认开启）时目标机自动上报的系统信息：OS、IP、CPU、内存、磁盘、hostname
- 变量形如 ansible_distribution、ansible_default_ipv4.address、ansible_memtotal_mb
- 用途：模板渲染、条件判断（when: ansible_os_family == "RedHat"）
- setup 模块单独收集；自定义 fact：/etc/ansible/facts.d/*.fact JSON 文件
- gather_facts: no 关闭可加速不需要 facts 的 playbook

**一句话答：**
Facts 是 Ansible 自动采集的目标机系统信息——系统版本、IP、CPU、内存这些，模板和条件判断都靠它，比如按 ansible_os_family 区分 yum 还是 apt；不需要时 gather_facts: no 能加快执行。

### 14. Ansible 中如何处理敏感数据，如密码、密钥等？

**要点：**
- ansible-vault 加密变量文件/整个 playbook：ansible-vault create/encrypt/decrypt/view/rekey
- 执行时 --ask-vault-pass 或 vault-password-file 提供解密口令
- 变量命名约定（如 db_password）配合 no_log: true 防止任务输出泄露
- 企业级对接 HashiCorp Vault/云 KMS 拉取密钥
- 私钥、密码不进 Git 明文

```bash
ansible-vault create group_vars/prod/vault.yml
ansible-playbook site.yml --ask-vault-pass
```

**一句话答：**
用 ansible-vault 加密敏感变量文件，执行时带 --ask-vault-pass 解密，敏感任务输出再加 no_log: true 防止日志泄露，更严格的场景对接 HashiCorp Vault 动态取密钥，绝不把明文密码提交进 Git。

### 15. Playbook 中 become / become_user 什么使用用？

**要点：**
- become: yes 切换提权执行（默认 sudo，可配 su/其他）
- become_user 指定切换到的用户（默认 root）
- play 级全局提权、task 级局部提权均可
- 需要 become_password（--ask-become-pass）或免密 sudo 配置
- 场景：普通运维账号登录，root 权限执行系统操作

```yaml
- hosts: all
  become: yes
  become_user: root
  tasks:
    - name: only this task as postgres
      become_user: postgres
      command: pg_dump mydb
```

**一句话答：**
become 开启提权，become_user 指定以哪个用户执行，默认 root——生产上一般用普通账号 SSH 登录，系统级任务 become 到 root，个别任务比如数据库备份再切到 postgres 用户，提权密码可免密 sudo 或 --ask-become-pass。

### 16. Ansible 执行报错“Permission denied”的常见原因？

**要点：**
- SSH 认证问题：密钥未分发/权限过大（600）、目标用户无登录权限、known_hosts 变化
- 目标文件/目录权限不足：需要 become: yes 提权而未加
- sudo 规则不允许该用户/命令（visudo 配置）
- SELinux 上下文、远端只读文件系统
- 排查顺序：ansible -m ping 通不通 → 手动 ssh 同账号试 → 加 -vvv 看 ssh 命令详情

**一句话答：**
常见三类：SSH 层——密钥没分发或权限不对；权限层——任务需要 root 但没加 become，或 sudoers 限制了该用户；系统层——SELinux 或文件属主问题。先用 -m ping 验证连通，再手动 SSH 复现，最后 -vvv 看 ssh 细节。

### 17. Ansible 部分主机显示 “unreachable” 是什么原因？

**要点：**
- SSH 连不上：网络/防火墙（22 端口）、主机宕机、SSH 服务挂
- 认证失败：密钥/密码错、目标机用户不存在
- inventory 变量错：ansible_host/IP 变更、端口不对
- Python 解释器缺失（新系统默认路径不在 /usr/bin/python）
- 处理：重试 --limit 到问题机器、手动 ssh 验证、修正 inventory；retries 可自动重试

**一句话答：**
unreachable 就是 SSH 连不通：先看主机是不是挂了、22 端口和防火墙，再验证密钥认证和 inventory 里的 IP 端口用户对不对，新系统还可能是找不到 Python 解释器，用 --limit 单独重试问题机器定位。

### 18. 管理大量主机时，如何提升 Ansible 执行效率？

**要点：**
- SSH 复用（pipelining = True，ControlMaster/ControlPersist 连接复用）
- 调大 forks 并发；不需要 facts 的任务 gather_facts: false
- 模块执行优化：shell/command 模块无 Python 传输开销（但少用，破坏幂等）
- strategy: free 让快主机不等待慢主机；async 异步长任务轮询
- 清单分层批量执行、Mitogen 加速器（需评估）、控制机就近部署

**一句话答：**
主要三板斧：开 SSH pipelining 和连接复用、按机器性能调大 forks、不需要 facts 就关掉；再加上 strategy: free 和 async 异步执行避免慢主机拖全局，量大时配合动态 inventory 分批跑。

### 19. Ansible 有哪些常用的模块？

**要点：**
- 文件：copy、template、file（属性）、fetch、lineinfile、blockinfile
- 包管理：yum/apt/package、pip
- 服务：systemd/service；用户组：user、group
- 执行：command、shell、script、raw
- 源码与编排：git、unarchive、uri、cron、firewalld/iptables、mount
- 云：ec2/aliyun 系列模块

**一句话答：**
文件操作用 copy、template、lineinfile、file，软件用 yum/apt/pip，服务用 systemd，执行命令用 command 和 shell，拉代码用 git，还有 uri、cron、user 这些，覆盖了日常运维九成场景。

### 20. Ansible 如何实现应用灰度发布？

**要点：**
- 分批策略：serial: "20%" 或按批次分组，先发一小批
- 批间验证：每批后用 uri 模块探测健康检查接口，失败即 fail 中止
- 负载均衡摘除：发布前把节点从 LB 摘除（调 API），发布验证后再加回
- 版本回滚：rollback playbook + 上一版本号变量
- 配合 CI：Jenkins 流水线调用 ansible-playbook 逐批执行

```yaml
- hosts: webservers
  serial: "20%"
  tasks:
    - name: 从负载均衡摘除节点
      uri: { url: "http://lb/api/remove/{{ inventory_hostname }}", method: POST }
    - name: 部署新版本
      unarchive: { src: "app-{{ app_version }}.tar.gz", dest: /opt/app }
    - name: 健康检查
      uri: { url: "http://{{ inventory_hostname }}:8080/health", status_code: 200 }
      register: hc
      until: hc.status == 200
      retries: 10
      delay: 6
```

**一句话答：**
用 serial 控制每批主机比例，每批部署前先从负载均衡摘节点，部署后 uri 探活确认健康再加回流量，任何一批验证失败立即中止，配合版本变量和回滚 playbook 实现灰度和快速回退。

### 21. Ansible 与 Terraform 有什么区别？

**要点：**
- 目标不同：Terraform 管"基础设施"（云资源创建），Ansible 管"配置"（装软件、改配置、部署应用）
- Terraform 声明式 + 状态文件（state）跟踪资源实际状态，支持 plan 预览
- Ansible 面向已有机器，SSH 执行，过程式为主但模块幂等
- 常配合：Terraform 建好 ECS → 输出 IP → Ansible 接手配置

**一句话答：**
Terraform 管基础设施的创建和变更，声明式加 state 状态文件，plan 一下就能预览改动；Ansible 管机器内部的配置和部署，走 SSH 执行。实践中一般 Terraform 先建云资源，Ansible 再上去做配置，各管一段。

### 22. 简述 Terraform 工作原理？

**要点：**
- 声明式 HCL：.tf 文件描述"资源最终长什么样"，不写过程
- state 状态文件记录资源与真实基础设施的映射（本地或远程 S3/OSS + 锁）
- 执行流程：refresh 读真实状态 → plan 对比期望与实际算出变更 diff → apply 执行变更并更新 state
- 通过 Provider 插件对接各云 API（AWS/阿里云/K8s）
- 依赖图自动排序资源创建顺序

```hcl
provider "alicloud" {
  region = "cn-hangzhou"
}
resource "alicloud_instance" "web" {
  instance_type   = "ecs.c7.xlarge"
  image_id        = "aliyun_3_x64_20G_alibase_20240528.vhd"
  security_groups = [alicloud_security_group.sg.id]
  vswitch_id      = var.vswitch_id
}
```

**一句话答：**
Terraform 用 HCL 声明期望的资源状态，靠 state 文件记住上次创建了什么，plan 时对比期望和真实基础设施算出差异，apply 时通过 provider 调云 API 把变更落下去，全程可预览、可回滚、幂等。

### 23. 说下 Terraform 在阿里云上自动创建 ECS 的流程？

**要点：**
- 准备：安装 Terraform、配置 AK/SK（环境变量）、初始化 provider
- 编写 main.tf/variables.tf：provider "alicloud"、resource alicloud_instance（镜像/规格/VPC/安全组）
- terraform init 拉取 provider → plan 预览变更 → apply 创建 → destroy 释放
- state 存远程（OSS + 锁）便于团队协作
- 进阶：配合 module 复用、输出 IP 给 Ansible 做后续配置

```bash
export ALICLOUD_ACCESS_KEY=xxx ALICLOUD_SECRET_KEY=xxx
terraform init && terraform plan && terraform apply
```

**一句话答：**
流程是：配好 AK/SK 写 main.tf，provider 指定 alicloud，resource 里声明 ECS 的规格、镜像、VPC 和安全组，然后 terraform init 初始化、plan 预览、apply 创建，state 建议放 OSS 加锁，创建完把 IP 输出交给 Ansible 继续配置。
