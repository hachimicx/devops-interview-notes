# DevOps自动化运维工具 — 面试题单(来源:面试小灶)

## DevOps 和 CI/CD 面试题(25题)

1. 你是怎么理解 DevOps 的？
2. 你都用过 Devops 哪些工具？
3. 简述 DevOps 工程师核心工作职责？
4. 什么是 CI/CD ？
5. CI/CD 流水线包含哪些阶段？
6. 简述你搭建过的 CI/CD，用到了哪些工具及解决了什么问题？
7. 灰度发布、蓝绿部署和滚动发布是什么？
8. 说一下你们公司网站是如何发版的？
9. 团队提交频繁导致 CI 队列拥堵，如何优化？
10. 在 CI/CD 流程中 Webhook 的作用是什么？
11. 如何在 CI 过程中提高代码质量？
12. SonarQube 在 CI 中的作用？
13. 什么是基础设施即代码（IaC）？
14. 简述“混沌工程”与 DevOps 有什么联系？
15. 在公司推行 DevOps 会遇到哪些阻力？
16. 你用过哪些 CI/CD 工具？
17. Gitlab CI/CD 是怎么工作的？
18. 在 DevOps 中，配置管理有什么作用？
19. DevOps 中的“左移”和“右移”是什么？
20. CI/CD 中自动化测试有哪些工作？
21. CI/CD 流程中应该监控哪些指标？
22. 如果你加入我们团队，如何开展 DevOps？
23. 混沌工程 怎么做故障注入？
24. GitLab Flow 了解过吗？
25. 怎么提升研发效能？

## Prometheus 监控面试题(29题)

1. Prometheus 有哪些核心组件及其作用？
2. Prometheus 有哪些数据指标类型及其应用场景？
3. Prometheus 支持哪些服务发现方式？
4. Pushgateway 的使用场景是什么？它解决了什么问题？
5. 什么时候会用 relabel_configs  ？
6. 你都用过哪些 Exporter ？
7. 你用过 Prometheus 哪些内置函数?
8. Prometheus 存储机制是怎么样的？
9. Prometheus 为什么不适合长期存储？
10. Prometheus 出现 OOM 的常见原因及解决方案？
11. Prometheus 有哪些主流高可用方案？
12. Thanos 实现高可用的核心原理是什么？
13. Thanos 有哪些核心组件及其作用？
14. VictoriaMetrics 实现高可用的核心原理是什么？
15. Alertmanager 通过哪些机制减少无效告警？
16. Prometheus 如何调整，可以缩短告警时间？
17. Alertmanager 高可用如何实现？
18. 简述 Exporter 开发思路和核心细节？
19. 监控一个 MySQL 是怎么样的流程？
20. 新上100台服务器，Prometheus 如何实现自动化监控？
21. Prometheus 性能调优的核心手段有哪些？
22. 监控上千台服务器，如何保障 Prometheus 整体采集与查询性能？
23. Prometheus 的 Pull 模型相比 Push 模型有什么优势和劣势？
24. Prometheus 如何实现对 Kubernetes 集群的全面监控？
25. Prometheus 查询突然变慢，可能的原因有哪些？
26. 如何评估 Prometheus 集群的容量？
27. Grafana 图表不显示数据，怎么排查？
28. Prometheus Operator 是什么？解决了什么问题？
29. Prometheus 与 Zabbix 有什么区别？

## Zabbix 监控面试题(18题)

1. 简述 Zabbix 核心组件及作用？
2. 简述 Zabbix 中 Host、Host Group、Template 的关系？
3. 简述 Zabbix 中 Item、Trigger、Action 的关系？
4. Zabbix Agent 的主动模式和被动模式有什么区别？
5. 简述 Zabbix 监控一台新的 Linux 主机流程？
6. Zabbix 如何监控一台 MySQL 服务器？
7. Zabbix UserParameter 有什么作用？
8. Zabbix 模板中的宏有什么作用？
9. Zabbix 告警机制是如何实现的？
10. Zabbix 监控图表出现中断，如何排查？
11. Zabbix 数据库膨胀很大，如何清理？
12. Zabbix 分布式架构如何实现的？
13. Zabbix 7.0 新增 Proxy HA 集群了解吗？
14. Zabbix 监控系统如何优化？
15. 自动发现和自动注册是什么？有什么应用场景？
16. 如何监控 Web 服务的可用性和响应时间？
17. Zabbix 如何实现自动监控上百台服务器？
18. Zabbix 如何有效避免告警风暴？

## Jenkins 持续集成面试题(17题)

1. Jenkins 在 CI/CD 中的作用是什么？
2. Jenkins 与 Gitlab CI/CD 功能有什么区别？
3. Jenkins 你都用过哪些插件？
4. Freestyle 项目与 Pipeline 项目有什么区别？
5. 简述 Jenkins 分布式构建的工作原理？
6. 简述 Jenkins Pipeline 的工作原理？
7. Jenkins 中的触发器有哪些类型？
8. Jenkins 大量构建作业如何管理与优化？
9. Jenkins 构建队列管理机制是什么？
10. Jenkins 如何自定义构建通知？
11. 如何在 Jenkins 中安全管理敏感信息？
12. Jenkins 如何实现多环境配置与管理？
13. Jenkins 如何实现跨项目参数传递？
14. 简述 Jenkins 共享库（Shared Library）作用？
15. Jenkins 如何应对大规模构建环境？
16. 如何实现代码提交自动触发 Jenkins CI ？
17. Jenkins 怎么进行数据备份？

## ELK 日志管理面试题(30题)

1. ELK Stack 都有哪些组件？
2. Elasticsearch 有哪些节点类型及其作用？
3. 简述 Elasticsearch 集群工作原理？
4. 简述  Elasticsearch 集群节点选举机制？
5. ES 集群健康状态 green / yellow / red 分别代表什么？
6. ES 集群 red 状态如何恢复？
7. ES 查询慢、写入慢怎么排查？
8. ES Mapping 有什么用途？
9. ES ILM 是什么？有什么应用场景？
10. ES 如何实现自动删除 30 天前索引？
11. ES 集群磁盘快满了，如何进行扩容？
12. ES 集群数据如何备份？
13. ES 如何性能优化？
14. ES 集群一般会监控哪些指标？
15. 简述 Logstash 工作流程？
16. Logstash 有哪些应用场景？
17. Logstash Pipelines（多管道）是什么？
18. Logstash 过滤器有哪些常用插件？
19. Logstash 如何性能优化？
20. Kibana 中的 Discover 功能是什么？
21. Kibana Dashboard 你用过哪些图表类型？
22. Filebeat 是如何读取日志文件的？
23. Filebeat 如何保证不丢数据、断点续传？
24. ELK 架构中，为什么要加消息队列？
25. ELK 架构加消息队列用Redis还是Kfaka？
26. 如何设计一个每天百GB日志量的ELK架构？
27. ELK 如何实现告警通知？
28. ELK 如何收集 K8s 集群中应用日志？
29. ELK Stack、Grafana Loki 和 Graylog 如何选择？
30. 简述 Grafana Loki  日志管理的工作流程？

## Ansible 自动化运维面试题(23题)

1. 简述 Ansible 工作架构和原理？
2. Ansible Inventory 是什么？
3. 动态 Inventory 有哪些应用场景？
4. Ansible 变量传递有哪几种方式？
5. Ansible Playbook 是什么？
6. Playbook 包含哪些核心字段？
7. Playbook 中 notify 和 handlers 的作用？
8. Playbook 中 tags 有什么用？
9. 如何调整 ansible-playbook 执行任务时的并发数？
10. Playbook 与 Role 什么关系？
11. 一个 Role 目录结构是怎样的？
12. Jinja2 模板是什么？
13. Ansible Facts 是什么？
14. Ansible 中如何处理敏感数据，如密码、密钥等？
15. Playbook 中 become / become_user 什么使用用？
16. Ansible 执行报错“Permission denied”的常见原因？
17. Ansible 部分主机显示 “unreachable” 是什么原因？
18. 管理大量主机时，如何提升 Ansible 执行效率？
19. Ansible 有哪些常用的模块？
20. Ansible 如何实现应用灰度发布？
21. Ansible 与 Terraform 有什么区别？
22. 简述 Terraform 工作原理？
23. 说下 Terraform 在阿里云上自动创建 ECS 的流程？

