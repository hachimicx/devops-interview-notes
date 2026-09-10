# DevOps运维开发 — 面试题单(来源:面试小灶)

## Python 运维开发面试题(53题)

1. 为什么说 Python “一切皆为对象” ？
2. Python 里常见可变 / 不可变对象分别有哪些？
3. Python 浅拷贝 & 深拷贝 有什么区别？
4. List（列表） 和 Tuple（元组） 的核心区别是什么？
5. Set（集合）有哪些特性？怎么实现去重的？
6. Dict（字典）的底层实现原理是什么？
7. *args 和 **kwargs 的作用是什么？
8. 什么匿名函数（Lambda）？
9. 匿名函数（Lambda）有哪些应用场景？
10. 什么是高阶函数？
11. 装饰器解决了什么问题？
12. 什么是闭包？
13. 简述 Python 的 LEGB 作用域规则？
14. 生成器与迭代器有什么区别？
15. 如何高效处理 GB 级别大文件？
16. 什么是面向对象编程（OOP）？
17. __new__ 与 __init__ 的区别是什么？
18. @classmethod 类方法、@staticmethod 静态方法的区别？
19. hasattr() 和 getattr() 有什么作用？
20. super() 工作原理是什么？
21. with 语句的原理是什么？
22. 多进程、多线程、协程的区别是什么？
23. GIL（全局解释器锁）有什么作用？
24. 简述 Python 垃圾回收机制工作原理？
25. Python 如何递归遍历目录并统计总文件大小？
26. Python 如何执行 Shell 命令？
27. Python 如何实现在100台服务器上执行脚本？
28. Socket 是什么？有哪些应用场景？
29. 你都用 Python 调用过哪些 API 及应用场景？
30. 如何排查 Python 程序的内存泄漏问题？
31. SQLAlchemy 是什么？
32. Python 运维自动化脚本你都用过哪些模块？
33. Python 如何统计 Nginx 日志 Top 10 访问IP？
34. Python 如何获取 CPU、内存、硬盘、网络 使用率？
35. Python 有哪些 Web 框架及各自特点？
36. 简述 Django MVT 架构？
37. Django ORM 是什么？
38. Django ORM 有哪些常用查询方法？
39. Django REST Framework（DRF） 是什么？
40. DRF ModelViewSet 的父类有哪些？
41. Django 中间件的作用是什么？
42. 简述 Django 中间件自定义流程？
43. DRF 限流是怎么实现的？
44. DRF 认证和权限怎么实现的？
45. Celery 是什么？什么情况下用？
46. 简述 Django CSRF 工作原理？
47. Django 如何编写测试用例？
48. Websocket 与 HTTP 有什么区别？
49. Python 怎么使用正则？
50. Python 正则中有哪些常用的字符及作用？
51. 都做过哪些运维开发项目？
52. 简单说下你对运维开发的理解？
53. 说一下开发一个运维Web平台的大致流程？

## Golang 运维开发面试题(46题)

1. Go 语言有哪些核心特点与主要应用场景？
2. Go 强类型、静态类型体现在哪里？
3. Go  数组（Array） 和 切片（Slice）什么区别？
4. 简述 Go 切片（Slice）底层结构原理？
5. make 和 new 函数的区别及适用场景？
6. 简述 Go Map 底层实现原理？
7. Go 里 struct 是什么？和其他语言的类有什么区别？
8. struct 定义方法时，值接收者和指针接收者有什么区别？
9. 什么是指针？指针的作用是什么？
10. Go 函数是值传递还是引用传递？
11. defer 有哪些应用场景及执行顺序？
12. 什么是高阶函数？有哪些常见应用场景？
13. 什么是匿名函数？有哪些应用场景？
14. 什么是闭包？有哪些应用场景？
15. Go 怎么实现面向对象编程？
16. Go 如何判断两个 切片（slice） 内容是否相同？
17. Go 异常处理如何实现的？
18. 什么是 Goroutine（协程）？和线程有什么区别？
19. 多个 Goroutine 如何同步？常用方式有哪些？
20. Channel 是什么？有哪些常见应用场景？
21. Select 是什么？有哪些常见应用场景？
22. Context 在 Goroutine 中的作用是什么？
23. Goroutine 泄漏有哪些常见原因？
24. 大量 Goroutine 时如何优化？
25. Go Modules 有什么作用？解决了什么问题？
26. Go 项目开发中，你经常用哪些模块？
27. 你使用 Go写过哪些自动化运维脚本？
28. Go 处理文件大文件时如何优化？
29. Go 如何实现 WebSocket 服务？
30. 如何排查 Go 内存泄露？有哪些常见原因与解决方法？
31. Go 有哪些 Web 框架及各自特点？
32. Gin 获取 URL 参数、路由参数、表单参数、JSON 参数分别用什么方法？
33. ShouldBind 和 Bind 有什么区别？
34. 简述 Gin 统一处理返回格式与错误码的实现流程？
35. gin.New() 和 gin.Default() 什么区别？
36. Gin 中间件的作用是什么？
37. 简述 Gin 中间件自定义流程？
38. Gin 如何实现跨域 CORS？
39. ORM 如何定义一对多、多对多模型关系？
40. GORM 的 First、Last、Take、Find 有什么区别？
41. gorm.Model 默认有哪几个字段？
42. Gin 如何实现微服务架构？
43. 简述 Go 垃圾回收实现机制？
44. 如何用 Channel 实现两个 Goroutine 交替打印数字？
45. client-go 客户端库的核心组件有哪些？
46. 简述 client-go 客户端库 Informer 工作机制？

## Vue 前端开发面试题(21题)

1. 简述 Vue 的响应式原理？
2. v-for 中为什么建议使用 key？
3. v-if 和 v-show 有什么区别？
4. computed 和 watch 有什么区别及应用场景？
5. Vue 的双向绑定（v-model ）是如何实现的？
6. Vue 生命周期钩子函数有哪些？
7. 如何在 Vue 项目中引入第三方库？
8. Vue 中的 $nextTick 与 setTimeout 有什么区别？
9. Vue 组件间有哪些通信方式？
10. Vue 中的 ref 有什么作用？
11. Vue Router 中 hash 模式和 history 模式有什么区别？
12. Vue Router 路由守卫有什么作用？
13. 如何在 Vue 中实现跨组件的状态管理？
14. Vue 如何实现异步加载组件？
15. Vue3 中 Composition API 和 Options API 有什么区别？
16. Vue 项目如何做性能优化？
17. keep-alive 是什么？有什么作用？
18. Vue 中 this 是什么意思？
19. Vue 中有哪些常用的事件修饰符？
20. 基于 Vue 你都用过哪些 UI 组件库？
21. Nginx 部署的 Vue 项目，访问 404 怎么解决？

## Git 代码管理面试题(12题)

1. Git 和 SVN 的区别有哪些？
2. Git 的工作区、暂存区和本地仓库分别是什么？
3. git fetch 和 git pull 的区别是什么？
4. Git 如何解决分支合并冲突？
5. 项目开发中常用的分支有哪些？各有什么作用？
6. Git Hook 有哪些常用类型？有哪些应用场景？
7. 简述将代码提交到远程仓库的流程？
8. 多人协同开发时，如何规范使用分支？
9. 项目中如何配置忽略文件 .gitignore？
10. 当 GitLab 仓库出现大文件、仓库膨胀时，你会如何处理？
11. Git 中 head、HEAD、head^、head~1 分别表示什么？
12. 在 GitLab 中如何保护分支并强制要求 Merge Request？

