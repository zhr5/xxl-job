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

这里使用

这里可以看出`XxlJobAdminConfig` 实现了`InitializingBean` ，bean初始化后会执行`afterPropertiesSet`

## 路由策略算法解析

### 1.概述

xxl-job就是因为内涵丰富的调度策略，使得框架的多样性，灵活性更高。现在就开始讲解xxl-job的核心路由策略算法，总共有10种路由策略，对于以后想从事分布式微服务开发，任务调度的学习是很有必要的。

### 2.路由策略种类

- 第一个
- 最后一个
- 随机选取
- 轮询选取
- 一致性hash
- 最不经常使用 (LFU)
- 最近最久未使用（LRU）
- 故障转移
- 忙碌转移
- 分片广播 
以上就是xxl-job内部封装的路由策略算法，也是很常见的路由算法，学习掌握之后对自己设计分布式架构很有帮助。

### 3.路由策略详解

#### 3.1 第一个

```java
package com.xxl.job.admin.core.route.strategy;

import com.xxl.job.admin.core.route.ExecutorRouter;
import com.xxl.job.core.biz.model.ReturnT;
import com.xxl.job.core.biz.model.TriggerParam;

import java.util.List;

/**
 * Created by xuxueli on 17/3/10.
 */
public class ExecutorRouteFirst extends ExecutorRouter {

    @Override
    public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList){
        return new ReturnT<String>(addressList.get(0));
    }

}
```

#### 3.2 最后一个

```java
package com.xxl.job.admin.core.route.strategy;

import com.xxl.job.admin.core.route.ExecutorRouter;
import com.xxl.job.core.biz.model.ReturnT;
import com.xxl.job.core.biz.model.TriggerParam;

import java.util.List;

/**
 * Created by xuxueli on 17/3/10.
 */
public class ExecutorRouteLast extends ExecutorRouter {

    @Override
    public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        return new ReturnT<String>(addressList.get(addressList.size()-1));
    }

}
```

#### 3.3 随机

```java
package com.xxl.job.admin.core.route.strategy;

import com.xxl.job.admin.core.route.ExecutorRouter;
import com.xxl.job.core.biz.model.ReturnT;
import com.xxl.job.core.biz.model.TriggerParam;

import java.util.List;
import java.util.Random;

/**
 * Created by xuxueli on 17/3/10.
 */
public class ExecutorRouteRandom extends ExecutorRouter {

    private static Random localRandom = new Random();

    @Override
    public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        String address = addressList.get(localRandom.nextInt(addressList.size()));//随机选取
        return new ReturnT<String>(address);
    }

}
```

#### 3.4 轮询

```java
package com.xxl.job.admin.core.route.strategy;

import com.xxl.job.admin.core.route.ExecutorRouter;
import com.xxl.job.core.biz.model.ReturnT;
import com.xxl.job.core.biz.model.TriggerParam;

import java.util.List;
import java.util.Random;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Created by xuxueli on 17/3/10.
 */
public class ExecutorRouteRound extends ExecutorRouter {
    //<JobId%addressList.size(),count>
    private static ConcurrentMap<Integer, AtomicInteger> routeCountEachJob = new ConcurrentHashMap<>();
    private static long CACHE_VALID_TIME = 0;

    private static int count(int jobId) {
        // cache clear
        if (System.currentTimeMillis() > CACHE_VALID_TIME) {
            routeCountEachJob.clear();
            CACHE_VALID_TIME = System.currentTimeMillis() + 1000*60*60*24;
        }

        AtomicInteger count = routeCountEachJob.get(jobId);
        if (count == null || count.get() > 1000000) {
            // 初始化时主动Random一次，缓解首次压力
            count = new AtomicInteger(new Random().nextInt(100));
        } else {
            // count++
            count.addAndGet(1);
        }
        routeCountEachJob.put(jobId, count);
        return count.get();
    }

    @Override
    public ReturnT<String> route(TriggerParam triggerParam, List<String> addressList) {
        String address = addressList.get(count(triggerParam.getJobId())%addressList.size());
        return new ReturnT<String>(address);
    }

}
```

这里注意到创建了一个静态的ConcurrentMap对象，这个routeCountEachJob就是用来存放路由任务的，而且还设置了缓存时间，有效期为24小时，当超过24小时的时候，自动的清空当前的缓存。

其中ConcurrentMap的key为jobId，value为当前jobId所对应的计数器，每访问一次就自增一，最大增到100000，然后又从[0，100)的随机数开始重新自增。

这个算法的思想就是取余数，每次先计算出当前jobId所对应的计数器的值，然后 计数器的值 % addressList.size() 求得这一次轮询的地址。


#### 3.5 一致性hash

```

```

