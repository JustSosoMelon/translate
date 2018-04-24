## 27 [任务执行和定时器](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling)

## 27.1 [介绍](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-introduction)

Spring Framework提供了`TaskExecutor`和`TaskScheduler`抽象，分别用于任务异步执行和定时执行。Spring还支持在应用服务器环境中支持这些接口的线程池或委派到CommonJ的实现。最终基于接口操作具体的实现，抽象了Java SE 5, Java SE 6和Java EE环境的区别。

Spring也提供了集成类，支持使用`Timer`（JDK 1.3后的类）和Quartz Scheduler([http://quartz-scheduler.org](http://quartz-scheduler.org/))进行调度。这两种调度器都是使用“FactoryBean”设置的，分别引用`Timer`或“`Trigger`实例，此外，对于Quartz Scheduler 和`Timer`都有一个方便的类，它允许你调用已有目标对象的方法(类似于`MethodInvokingFactoryBean`操作)。

## 27.2 [Spring的TaskExecutor抽象](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-task-executor)

spring 2.0引入了一个处理执行器的新抽象，Executor是线程池概念的Java5名称，这个executor命名是由于没有保证底层实现是一个池，一个executor可能是单线程或同步的，Spring的抽象隐藏了Java SE 1.4, Java SE 5和Java EE环境的不同实现。

Spring的`TaskExecutor`接口`java.util.concurrent.Executor`是一致的，实际上，它存在的主要原因是使用Java1.5之前的线程池时抽象出Java5的Executor接口，这个接口有单一的方法`execute(Runnable task)`，接收一个task，基于线程池的语义和配置执行。

`TaskExecutor`的引入原本是为了给其他的Spring组件一个线程池的抽象，比如`ApplicationEventMulticaster`, JMS的`AbstractMessageListenerContainer`和Quartz的集成都使用`TaskExecutor`的抽象共用（pool）线程，但是如果你自定义的bean需要线程池行为，也可以根据你自己的需求使用`TaskExecutor`抽象。

### 27.2.1 [TaskExecutor类型](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-task-executor-types)

Spring的发布版本中包含一些预先构建的`TaskExecutor`实现，大多数情况下，你不需要实现自己的`TaskExecutor`。

-   `SimpleAsyncTaskExecutor`

这个实现不重用任何线程，而是为每次调用启动一个新线程，但是它有一个并发上限，达到上限后将会阻塞任何的调用，直到有调用结束后并发数降低，如果你寻找真正的线程池，在本章继续往下看。
    
-   `SyncTaskExecutor`
    
这个实现不会异步执行方法调用，而是都在当前访问线程执行，它主要用于不需要多线程的场景，比如简单的测试场景
    
-   `ConcurrentTaskExecutor`
    
这个实现是`java.util.concurrent.Executor`的包装，另一个选择，`ThreadPoolTaskExecutor`，它以bean属性的方式暴露`Executor`的配置参数，很少需要用`ConcurrentTaskExecutor`，但是如果[`ThreadPoolTaskExecutor`](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#threadPoolTaskExecutor)不够稳定，不满足你的需求，`ConcurrentTaskExecutor`是一个选择。
    
-   `SimpleThreadPoolTaskExecutor`
    
这个实现实际上是Quartz的`SimpleThreadPool`类的一个子类，监听Spring生命周期过程中的回调。有一个线程池需要被Quartz和非Quartz组件共享的场景是该实现的典型应用场景。
    
-   `ThreadPoolTaskExecutor`
    
这个实现不能和`java.util.concurrent`包的任何兼容或可选版本共同使用，Doug Lea's和Dawid Kurzyniec's的实现都使用了不同的包结构，这将阻止他们正常工作。

这个实现仅可以用在Java5环境，但也是该环境最常用的实现，它暴露bean属性用于配置一个`java.util.concurrent.ThreadPoolExecutor`，将其包装在一个`TaskExecutor`中。如果你需要一些高级的，比如`ScheduledThreadPoolExecutor`，建议你使用[`ConcurrentTaskExecutor`](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#concurrentTaskExecutor)代替。
    
-   `TimerTaskExecutor`
    
这个实现使用单一的`TimerTask`作为底层的实现，和[`SyncTaskExecutor`](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#syncTaskExecutor)不同，方法的调用在一个单独线程中执行，但是所有调用在这个单独线程中也是同步的。
    
-   `WorkManagerTaskExecutor`
    
CommonJ是一组由BEA和IBM共同开发的实现，这些实现不是J2EE的规范，但是BEA和IBM的应用服务器实现规范。
    
这个实现使用CommonJ WorkManager作为底层支持，是在Spring Context中设置CommonJ WorkManager引用的中心方便类，类似[`SimpleThreadPoolTaskExecutor`](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#simpleThreadPoolTaskExecutor)，这个类实现了WorkManager接口，因此也可以直接当作WorkManager使用。
    

### 27.2.2 [使用TaskExecutor](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-task-executor-usage)

Spring的`TaskExecutor`的实现被作为简单JavaBean使用，在下面的例子中，我们定义了一个bean，使用`ThreadPoolTaskExecutor`去异步打印一组消息。
```
import org.springframework.core.task.TaskExecutor;

public class TaskExecutorExample {

  private class MessagePrinterTask implements Runnable {

    private String message;

    public MessagePrinterTask(String message) {
      this.message = message;
    }

    public void run() {
      System.out.println(message);
    }

  }

  private TaskExecutor taskExecutor;

  public TaskExecutorExample(TaskExecutor taskExecutor) {
    this.taskExecutor = taskExecutor;
  }

  public void printMessages() {
    for(int i = 0; i < 25; i++) {
      taskExecutor.execute(new MessagePrinterTask("Message" + i));
    }
  }
}
```
如你所见，并不需要主动从线程池中获取线程并执行，添加`Runnable`实例到队列中，这个`TaskExecutor`使用它内部的规则去决定何时执行task。

`ThreadPoolTaskExecutor`暴露了bean属性，让你可以配置`TaskExecutor`使用的规则。
```
<bean id="taskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
  <property name="corePoolSize" value="5" />
  <property name="maxPoolSize" value="10" />
  <property name="queueCapacity" value="25" />
</bean>

<bean id="taskExecutorExample" class="TaskExecutorExample">
  <constructor-arg ref="taskExecutor" />
</bean>
```

## 27.3 [Spring的TaskScheduler抽象](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-task-scheduler)

除了`TaskExecutor`的抽象，Spring 3.0引入了`TaskScheduler`，带来很多调度任务的方法。
```
public interface TaskScheduler {

    ScheduledFuture schedule(Runnable task, Trigger trigger);

    ScheduledFuture schedule(Runnable task, Date startTime);

    ScheduledFuture scheduleAtFixedRate(Runnable task, Date startTime, long period);

    ScheduledFuture scheduleAtFixedRate(Runnable task, long period);

    ScheduledFuture scheduleWithFixedDelay(Runnable task, Date startTime, long delay);

    ScheduledFuture scheduleWithFixedDelay(Runnable task, long delay);

}
```
最简单的是schedule方法，仅需`Runnable`  and  `Date`两个参数，能让任务在某个固定时刻运行一次，所有其他的方法可以让任务重复运行，固定频率和固定延迟的方法用于简单的周期执行，接受Trigger参数的schedule方法最灵活。


### 27.3.1 [Trigger接口](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-trigger-interface)

`Trigger`接口受JSR-236的启发，在Spring 3.0时代还未被官方实现。`Trigger`的基本思想是任务执行可能由过去执行的结果或任何条件决定，如果这些决定考虑了前一次的执行结果，可以从`TriggerContext`中获取这些信息。`Trigger`接口本身很简单：
```
public interface Trigger {

    Date nextExecutionTime(TriggerContext triggerContext);

}
```

如你所见，`TriggerContext`是最重要的部分，它封装了所有相关的数据，且对未来的扩展开放。`TriggerContext`是一个接口，默认使用`SimpleTriggerContext`，`TriggerContext`中的方法如下：
```
public interface TriggerContext {

    Date lastScheduledExecutionTime();

    Date lastActualExecutionTime();

    Date lastCompletionTime();

}
```

### 27.3.2 [Trigger实现](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-trigger-implementations)

Spring为Trigger接口提供了两种实现，最有意思的是`CronTrigger`，它让任务调度支持cron表达式，比如下面的task将会每小时的第15分钟执行一次，执行区间为工作日的9点到17点。
```
scheduler.schedule(task, new CronTrigger("0 15 9-17 * * MON-FRI"));
```

另一个开箱即用的实现是`PeriodicTrigger`，接受一个固定的周期，一个可选的初始延迟，和一个boolean值指示周期是固定执行频率还是固定执行延迟（执行结束后等待一个固定时间），由于`TaskScheduler`接口已经定义了固定频率或固定延迟调度任务的方法，这些方法直接使用，`PeriodicTrigger`的实现的价值在于可以在依赖`Trigger`抽象的组件中使用，比如它允许周期触发，cron触发和自定义触发交互使用的场景方便的实现，像这样的组件可以利用依赖注入的优势，`Triggers`可以在外部配置，容易修改和扩展。

### 27.3.3 [TaskScheduler实现](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-task-scheduler-implementations)

与Spring的TaskExecutor抽象一样，TaskScheduler的主要优点是依赖于调度行为的代码不需要与特定的调度器实现耦合。当应用服务器环境运行过程中线程不应该由应用本身创建的情况下（比如由应用服务器组件提供TaskScheduler实现），这种灵活性特别有意义，对于这种情形，Spring提供了`TimerManagerTaskScheduler`，代理一个CommonJ TimerManager实例，通常配置为JNDI查找。

每当不需要外部线程管理时，`ThreadPoolTaskScheduler`是一种更简单的选择，它在内部代理了一个`ScheduledExecutorService`实例，`ThreadPoolTaskScheduler` 实际也实现了Spring的`TaskExecutor`接口，这样，一个实例就可以被用于异步执行，并且尽可能早地执行，并且有可能重复执行。


## 27.4 [调度和异步执行注解支持](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-annotation-support)

Spring同时为任务调度和异步方法执行提供了注解支持。

### 27.4.1[开启调度注解](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#secheduling-enable-annotation-support)

为了`@Scheduled`和`@Async`注解的支持，分别添加@EnableScheduling`和`@EnableAsync` 在一个Configuration类上。
```
_@Configuration_
_@EnableAsync_
_@EnableScheduling_
public class AppConfig {
}
```

你可以为你的应用自由选取相关注解，比如，如果你只需要`@Scheduled`的支持，忽略掉`@EnableAsync`。如果需要更细粒度的控制，你可以额外实现`SchedulingConfigurer`和/或`AsyncConfigurer`接口，详请参考Javadoc。

如果你更喜欢XML配置，使用`<task:>`命名空间相关元素：
<task:annotation-driven executor="myExecutor" scheduler="myScheduler"/>
<task:executor id="myExecutor" pool-size="5"/>
<task:scheduler id="myScheduler" pool-size="10"/>}

注意以上XML中提供了一个executor用于处理`@Async`注释的方法调用产生的任务，一个scheduler用于管理被`@Scheduled`注解的方法调用。

### 27.4.2 [@Scheduled注解](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-annotation-support-scheduled)

@Scheduled注解可与触发器元数据一起加在方法上，比如下面的方法将会每5秒固定延迟调用一次，固定延迟意味着周期从每次调用结束时刻开始计时5s再触发下一次调用。
```
_@Scheduled(fixedDelay=5000)_
public void doSomething() {
    // something that should execute periodically
}
```

如果你想要一个固定频率的执行，更换注解中的属性为fixedRate，以下方法会每隔5s调用一次，从一次调用的开始时刻计时5s触发下一次调用。
```
_@Scheduled(fixedRate=5000)_
public void doSomething() {
    // something that should execute periodically
}
```

对于固定延迟和固定频率任务，都可以设置一个初始延迟，指定第一次方法被调度前的等待时间。
```
_@Scheduled(initialDelay=1000, fixedRate=5000)_
public void doSomething() {
    // something that should execute periodically
}
```

如果简单的定时调度不足以表达你的需求，可以提供一个cron表达式，比如下面的方法会每个工作日中每隔5s执行。
```
_@Scheduled(cron="*/5 * * * * MON-FRI")_
public void doSomething() {
    // something that should execute on weekdays only
}
```

注意调度schedule的方法必须是void返回值，且没有入参，如果这个方法需要和ApplicationContext中其他对象交互，可以通过依赖注入引入。

> 确保你没有初始化多个@Scheduled注解的实例，除非你希望对每个这样的实例安排回调。类似地，确保你没有在spring容器中管理的@Scheduled Bean上使用@Configurable注解：将会造成两次初始化，一次通过容器，一次通过@Configurable切面，带来的结果是每个@Scheduled注解的方法会被调用两次。

### 27.4.3 [@Async 注解](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-annotation-support-async)

`@Async`注解为需要异步执行的方法提供支持，异步的意思是调用者调用方法后会立即返回，而方法的实际执行将被提交到一个Spring `TaskExecutor`中。最简单的情况是用于void返回类型的方法上：
```
_@Async_
void doSomething() {
    // this will be executed asynchronously
}
```

和`@Scheduled`注解的方法不一样，这些`@Async`注解的方法可以有参数，因为他被调用者使用正常的方式调用，而不是被容器管理的调度任务调用，下面的`@Async`用法是合法的：
```
_@Async_
void doSomething(String s) {
    // this will be executed asynchronously
}
```

虽然有返回值的方法可以被异步调用，但是这些方法返回值必须使用一个Future范型封装，这也提供了异步执行的特性，便于调用者在调用Feature的`get()`前可以执行其他的任务。
```
_@Async_
Future<String> returnSomething(int i) {
    // this will be executed asynchronously
}
```

`@Async`  can not be used in conjunction with lifecycle callbacks such as  `@PostConstruct`. To asynchronously initialize Spring beans you currently have to use a separate initializing Spring bean that invokes the  `@Async`  annotated method on the target then.
`@Async`不能和生命周期回调如`@PostConstruct`一起使用，想要异步初始化Spring bean，当前你必须使用一个单独的初始化bean去调用目标上@Async注解的方法。
```
public class SampleBeanImpl implements SampleBean {

  _@Async_
  void doSomething() { … }
}

public class SampleBeanInititalizer {

  private final SampleBean bean;

  public SampleBeanInitializer(SampleBean bean) {
    this.bean = bean;
  }

  _@PostConstruct_
  public void initialize() {
    bean.doSomething();
  }
}
```

### 27.4.4 [@Async指定Executor](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-annotation-support-qualification)

默认@Async注解的方法被执行，使用的executor是提供给'annotation-driven'元素的（如前文所述），然而该注解的value属性可以用于指定一个executor，不使用默认executor：
```
_@Async("otherExecutor")_
void doSomething(String s) {
    // this will be executed asynchronously by "otherExecutor"
}
```
 如上情况，"otherExecutor"可能是spring容器中的bean名字，或bean上指定的qualifier名字，即`@Qualifier`或`<qualifier>`元素指定的value。

## 27.5 [Task 命名空间](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-task-namespace)

主要用于xml形式的spring task配置，暂不翻译。

## 27.6 [使用Quartz Scheduler](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-quartz)

Quartz使用`Trigger`、`Job`和`JobDetail`对象实现所有类型的任务，Quartz的基本概念请参考 [quartz官网](http://quartz-scheduler.org/). 为方便，Spring提供一些类简化Spring应用中的Quartz应用。

### 27.6.1 [使用JobDetailBean](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-quartz-jobdetail)

`JobDetail`对象包含了允许任务的所有信息，Spring框架提供`JobDetailBean`，提供了合理的默认值，更易使用，请看下面的例子：
```
<bean name="exampleJob" class="org.springframework.scheduling.quartz.JobDetailBean">
  <property name="jobClass" value="example.ExampleJob" />
  <property name="jobDataAsMap">
    <map>
      <entry key="timeout" value="5" />
    </map>
  </property>
</bean>
```

JobDetail bean拥有任务执行需要的所有信息（`ExampleJob`），timeout在任务数据map中指定，任务数据map在`JobExecutionContext`中可用（在执行时传递过来），但是`JobDetailBean`也将job data map中的属性映射到实际job，因此在这个情形下，如果`ExampleJob`包含一个timeout属性，`JobDetailBean`会自动填充这个属性。
```
package example;

public class ExampleJob extends QuartzJobBean {

  private int timeout;

  **/**
   * Setter called after the ExampleJob is instantiated
   * with the value from the JobDetailBean (5)
   */**
  public void setTimeout(int timeout) {
    this.timeout = timeout;
  }

  protected void executeInternal(JobExecutionContext ctx) throws JobExecutionException {
      _// do the actual work_
  }
}
```
所有`JobDetailBean`中额外的设置都将可用。

> 注意：使用`name`  and  `group` 属性，你可以分别修改job的name和group，默认job的名字匹配`JobDetailBean`的名字，如上面的例子，name即为`exampleJob`。

### 27.6.2 [使用MethodInvokingJobDetailFactoryBean](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-quartz-method-invoking-job)

通常你仅仅需要调用某个指定对象上的一个方法，使用`MethodInvokingJobDetailFactoryBean`，你恰好可以达到这个效果：
```
<bean id="jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
  <property name="targetObject" ref="exampleBusinessObject" />
  <property name="targetMethod" value="doIt" />
</bean>
```
The above example will result in the  `doIt`  method being called on the  `exampleBusinessObject`  method (see below):
以上的例子会使得`exampleBusinessObject`实例上的`doIt`方法被调用，结合下面的示例：
```
public class ExampleBusinessObject {

  _// properties and collaborators_

  public void doIt() {
    _// do the actual work_
  }
}

<bean id="exampleBusinessObject" class="examples.ExampleBusinessObject"/>
```

使用`MethodInvokingJobDetailFactoryBean`，你不需要去创建只调用一个方法的单行job，你只需要创建实际的业务对象，与MethodInvokingJobDetailFactoryBean bean连接。

默认情况下，Quartz job是无状态的，带来任务相互干扰的可能，比如你为一个`JobDetail`指定了两个trigger，那第一个job结束前，第二个job就可能会开始，如果`JobDetail`类实现了`Stateful`接口，这个情况就不会发生。为了使得`MethodInvokingJobDetailFactoryBean`生产的job是非并发的，将`concurrent`标志设为false。
```
<bean id="jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
  <property name="targetObject" ref="exampleBusinessObject" />
  <property name="targetMethod" value="doIt" />
  <property name="concurrent" value="false" />
</bean>
```

> 默认情况下，job会使用并发方式执行。

### 27.6.3 [使用Trigger和SchedulerFactoryBean连接多个Job](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/scheduling.html#scheduling-quartz-cron)

我们已经学会创建job details和job，我们也看到了允许你调用具体对象上的某一方法的便利bean，当然，我们仍然需要自己去调度job，这件事由trigger和SchedulerFactoryBean来做，Quartz已经提供了一些trigger，Spring也提供了两个Quartz FactoryBean，实现两个便利的行为：`CronTriggerFactoryBean`  and  `SimpleTriggerFactoryBean`.

Trigger需要被调度，Spring提供了SchedulerFactoryBean，包含Triggers属性，且暴露了set方法，SchedulerFactoryBean实际上就是通过这些triggers来调度Job。

找到一些例子，如下：
```
<bean id="simpleTrigger" class="org.springframework.scheduling.quartz.SimpleTriggerFactoryBean">
    <!-- see the example of method invoking job above -->
    <property name="jobDetail" ref="jobDetail" />
    <!-- 10 seconds -->
    <property name="startDelay" value="10000" />
    <!-- repeat every 50 seconds -->
    <property name="repeatInterval" value="50000" />
</bean>

<bean id="cronTrigger" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
    <property name="jobDetail" ref="exampleJob" />
    <!-- run every morning at 6 AM -->
    <property name="cronExpression" value="0 0 6 * * ?" />
</bean>

Now we've set up two triggers, one running every 50 seconds with a starting delay of 10 seconds and one every morning at 6 AM. To finalize everything, we need to set up the  `SchedulerFactoryBean`:

<bean class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
    <property name="triggers">
        <list>
            <ref bean="cronTrigger" />
            <ref bean="simpleTrigger" />
        </list>
    </property>
</bean>
```
更多SchedulerFactoryBean的属性，比如job detail使用的calendars，自定义结合Quartz的属性，等等，参考[SchedulerFactoryBean Javadoc](http://static.springsource.org/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/quartz/SchedulerFactoryBean.html) 

----------

[Prev](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/mail.html)

[Up](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/spring-integration.html)

[Next](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/dynamic-language.html)

26. Email

[Home](https://docs.spring.io/spring/docs/3.2.x/spring-framework-reference/html/index.html)

28. Dynamic language support
