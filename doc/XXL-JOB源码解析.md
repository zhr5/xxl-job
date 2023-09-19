# 架构图

![输入图片说明](images/XXL-JOB%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/img_Qohm.png)

# 源码结构

```
xxl-job-admin：调度中心
xxl-job-core：公共依赖
xxl-job-executor-samples：执行器Sample示例（选择合适的版本执行器，可直接使用，也可以参考其并将现有项目改造成执行器）
    ：xxl-job-executor-sample-springboot：Springboot版本，通过Springboot管理执行器，推荐这种方式；
    ：xxl-job-executor-sample-frameless：无框架版本；
```

## 调度中心启动流程

**xxl-job-admin模块是调度中心，用来管理调度任务。**

**xxl-job-core模块是公共的依赖，供调度中心以及调度任务依赖。**

`XxlJobAdminApplication` 启动后，根据日志

```
23:05:38.144 logback [main] INFO  c.x.j.a.c.scheduler.XxlJobScheduler - >>>>>>>>> init xxl-job admin success.
```

我们可以看出执行了

```
//调度器初始化
xxlJobScheduler.init();
```

![image-20230919230948353](images/XXL-JOB%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/image-20230919230948353.png)
