<a id="scheduling"></a>

[](#scheduling)7\. 执行任务和任务计划
----------------------------

Spring框架分别为异步执行、`TaskExecutor`的任务调度和`TaskScheduler`接口提供了抽象。Spring还具有支持线程池或委派到应用程序服务器环境CommonJ的接口实现。最终， 在Java SE 5、Java SE 6和Java EE有差异的环境都实现了一套公共的抽象接口。

Spring还具有集成类，支持使用`Timer`（JDK自1.3以来的一部分）和Quartz Scheduler（[http://quartz-scheduler.org](http://quartz-scheduler.org)）进行调度。 您可以使用`FactoryBean`同时分别对`Timer`或`Trigger`实例进行可选引用来设置这两个调度程序。 此外，还提供了Quartz Scheduler和 `Timer`的便捷类，它允许您调用现有目标对象的方法（类似于正常的`MethodInvokingFactoryBean`操作）。

<a id="scheduling-task-executor"></a>

### [](#scheduling-task-executor)7.1. Spring `TaskExecutor` 抽象

Executors是JDK中使用的线程池的名字，executor意思是无法保证底层的实现实际是一个池，一个executor可以是单线程的或者是同步的。Spring的抽象隐藏了Java SE和Java EE环境之间的实现细节。

Spring的`TaskExecutor`接口和`java.util.concurrent.Executor`接口是相同的。实际上，他存在的主要原因是在使用线程池时对Java 5抽象的程度不同。该接口有一个 (`execute(Runnable task)`)方法,它根据线程池的语义和配置接受执行任务.

最初创建`TaskExecutor`是为了给其他Spring组件提供所需的线程池抽象。诸如`ApplicationEventMulticaster`，JMS的`AbstractMessageListenerContainer`和Quartz集成之类的组件都使用 `TaskExecutor`抽象来池化线程。 但是，如果您的bean需要线程池行为，您也可以根据自己的需要使用此抽象。

<a id="scheduling-task-executor-types"></a>

#### [](#scheduling-task-executor-types)7.1.1. `TaskExecutor` 类型

Spring包含许多TaskExecutor的预构建实现。很可能，你永远不需要实现自己的。 Spring提供的变体如下:

*   `SyncTaskExecutor`: 此实现不会异步执行调用。 相反，每次调用都发生在调用线程中。 它主要用于不需要多线程的情况，例如在简单的测试用例中。
    
*   `SimpleAsyncTaskExecutor`: 此实现不会重用任何线程。 相反，它为每次调用启动一个新线程。 但是，它确实支持并发限制，该限制会阻止任何超出限制的调用，直到释放一个插槽。 如果您正在寻找真正的池，请参阅此列表中稍后的`ThreadPoolTaskExecutor`。
    
*   `ConcurrentTaskExecutor`: 此实现是 `java.util.concurrent.Executor`对象的适配器。有一个可选的`ThreadPoolTaskExecutor`，它将 `Executor`配置参数作为bean属性公开。很少需要使用到`ConcurrentTaskExecutor`，但如果`ThreadPoolTaskExecutor`不够灵活，那么你就需要`ConcurrentTaskExecutor`。
    
*   `ThreadPoolTaskExecutor`: 这种实现是最常用的。 它公开了bean属性，用于配置`java.util.concurrent.ThreadPoolExecutor`并将其包装在`TaskExecutor`中。 如果您需要适应不同类型的`java.util.concurrent.Executor`，我们建议您使用`ConcurrentTaskExecutor`。
    
*   `WorkManagerTaskExecutor`: 此实现使用CommonJ `WorkManager`作为其后备服务提供程序，并且是在Spring应用程序上下文中在WebLogic或WebSphere上设置基于CommonJ的线程池集成的中心便利类。
    
*   `DefaultManagedTaskExecutor`: 此实现在JSR-236兼容的运行时环境（例如Java EE 7+应用程序服务器）中使用JNDI获取的`ManagedExecutorService`，为此目的替换CommonJ WorkManager。
    

<a id="scheduling-task-executor-usage"></a>

#### [](#scheduling-task-executor-usage)7.1.2. 使用 `TaskExecutor`

Spring的 `TaskExecutor`实现用作简单的JavaBeans。 在下面的示例中，我们定义了一个使用`ThreadPoolTaskExecutor`异步打印出一组消息的bean:

```java
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

如您所见，您可以将`Runnable`添加到队列中，而不是从池中检索线程并自行执行。 然后，`TaskExecutor`使用其内部规则来确定任务何时执行。

要配置`TaskExecutor`使用的规则，我们公开了简单的bean属性:

```xml
<bean id="taskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
    <property name="corePoolSize" value="5"/>
    <property name="maxPoolSize" value="10"/>
    <property name="queueCapacity" value="25"/>
</bean>

<bean id="taskExecutorExample" class="TaskExecutorExample">
    <constructor-arg ref="taskExecutor"/>
</bean>
```

<a id="scheduling-task-scheduler"></a>

### [](#scheduling-task-scheduler)7.2. Spring `TaskScheduler` 抽象

除了`TaskExecutor`抽象之外，Spring 3.0还引入了一个`TaskScheduler`，它具有各种方法，可以在将来的某个时刻调度任务。 以下清单显示了`TaskScheduler`接口定义:

```java
public interface TaskScheduler {

    ScheduledFuture schedule(Runnable task, Trigger trigger);

    ScheduledFuture schedule(Runnable task, Instant startTime);

    ScheduledFuture schedule(Runnable task, Date startTime);

    ScheduledFuture scheduleAtFixedRate(Runnable task, Instant startTime, Duration period);

    ScheduledFuture scheduleAtFixedRate(Runnable task, Date startTime, long period);

    ScheduledFuture scheduleAtFixedRate(Runnable task, Duration period);

    ScheduledFuture scheduleAtFixedRate(Runnable task, long period);

    ScheduledFuture scheduleWithFixedDelay(Runnable task, Instant startTime, Duration delay);

    ScheduledFuture scheduleWithFixedDelay(Runnable task, Date startTime, long delay);

    ScheduledFuture scheduleWithFixedDelay(Runnable task, Duration delay);

    ScheduledFuture scheduleWithFixedDelay(Runnable task, long delay);
}
```

最简单的方法是一个名为`schedule`的方法，它只接受`Runnable`和`Date`。 这会导致任务在指定时间后运行一次。 所有其他方法都能够安排任务重复运行。 通过这些方法，在简单的周期中需要以固定频率和固定时间间隔方法执行任务是实现的，但接受`Trigger`会更方便。

<a id="scheduling-trigger-interface"></a>

#### [](#scheduling-trigger-interface)7.2.1. `Trigger` 接口

`Trigger`接口基本上受到JSR-236的启发，从Spring 3.0开始，它尚未正式实现。Trigger的基本思想是可以基于过去的执行结果或甚至任意条件来确定执行时间。如果这些确定考虑到了前一次执行的结果， 则该信息在`TriggerContext`中可用。`Trigger` 接口本身非常简单：:

```java
public interface Trigger {

    Date nextExecutionTime(TriggerContext triggerContext);
}
```

`TriggerContext`是最重要的部分。 它封装了所有相关数据，如有必要，将来可以进行扩展。 `TriggerContext`是一个接口（默认情况下使用 `SimpleTriggerContext`实现）。 以下清单显示了`Trigger`实现的可用方法。

```java
public interface TriggerContext {

    Date lastScheduledExecutionTime();

    Date lastActualExecutionTime();

    Date lastCompletionTime();
}
```

<a id="scheduling-trigger-implementations"></a>

#### [](#scheduling-trigger-implementations)7.2.2. `Trigger` 实现

Spring提供了两个`Trigger`接口的实现 ， 最有趣的是`CronTrigger`。 它支持基于cron表达式调度任务。例如，以下任务被安排在每小时超过15分钟， 但仅在工作日的早上9时到下午5时"营业时间"内运行:

```
scheduler.schedule(task, new CronTrigger("0 15 9-17 * * MON-FRI"));
```

另一个现成的实现是`PeriodicTrigger`,它接受一个固定的周期、一个可选的初始延迟值和一个布尔值来指示该期间是否应解释为固定速率或固定延迟。由于`TaskScheduler`接口已经定义了以固定速率或固定延迟来调度任务的方法，因此应该尽可能直接使用这些方法。 `PeriodicTrigger`实现的价值在于，它可以在依赖于`Trigger`抽象的组件中使用。例如，允许周期性触发器、cron-based触发器、甚至是可互换使用的自定义触发器实现，可能会很方便。此类组件可以利用依赖项注入，这样可以在外部配置此类`Triggers`，因此容易修改或扩展。

<a id="scheduling-task-scheduler-implementations"></a>

#### [](#scheduling-task-scheduler-implementations)7.2.3. `TaskScheduler` 实现

与Spring的`TaskExecutor`抽象一样， `TaskScheduler`的主要好处是，依赖于调度行为的代码不必与特定的调度程序实现耦合。当在应用程序服务器环境中运行时，不应由应用程序本身直接创建线程，因此提供的灵活性尤其重要。对于这种情况， Spring提供了一个`TimerManagerTaskScheduler`，它委托给WebLogic或WebSphere上的CommonJ `TimerManager`，以及一个委托给Java EE 7+环境中的JSR-236 `ManagedScheduledExecutorService`的更新的`DefaultManagedTaskScheduler`。 两者通常都配置有JNDI查找。

每当外部线程管理不是必需的时候，更简单的替代方案是应用程序中的本地`ScheduledExecutorService`设置，可以通过Spring的`ConcurrentTaskScheduler`进行调整。 为方便起见，Spring还提供了一个`ThreadPoolTaskScheduler`，它在内部委托给`ScheduledExecutorService`，以提供沿`ThreadPoolTaskExecutor`行的公共bean样式配置。些变体适用于宽松应用程序服务器环境中的本地嵌入式线程池设置，特别是在Tomcat和Jetty上。

<a id="scheduling-annotation-support"></a>

### [](#scheduling-annotation-support)7.3. 对调度和异步执行的注解支持

Spring为任务调度和异步方法执行提供了注解支持

<a id="scheduling-enable-annotation-support"></a>

#### [](#scheduling-enable-annotation-support)7.3.1. 启用调度注解

要启用对`@Scheduled`和`@Async`注解的支持，请将`@EnableScheduling`和`@EnableAsync`添加到您的`@Configuration` 类中。如下例所示：:

```java
@Configuration
@EnableAsync
@EnableScheduling
public class AppConfig {
}
```

您可以自由选择您的应用程序的相关注解。例如，如果您只需要支持`@Scheduled`，那么就省略`@EnableAsync`。对于更多的fine-grained控制，您可以另外实现[`SchedulingConfigurer`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/scheduling/annotation/SchedulingConfigurer.html)和/或[`AsyncConfigurer`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/scheduling/annotation/AsyncConfigurer.html)接口。有关详细信息，请参阅对应的javadocs。

如果您喜欢XML配置， 请使用`<task:annotation-driven>` 元素。如下:

```xml
<task:annotation-driven executor="myExecutor" scheduler="myScheduler"/>
<task:executor id="myExecutor" pool-size="5"/>
<task:scheduler id="myScheduler" pool-size="10"/>
```

请注意，在上面的XML中，提供了一个执行器引用来处理与`@Async`注解的方法对应的那些任务，并提供了调度程序引用来管理那些用`@Scheduled`注解的方法。

处理`@Async`注解的默认建议模式是`proxy`，它允许仅通过代理拦截调用。 同一类中的本地调用不能以这种方式截获。 对于更高级的拦截模式，请考虑结合编译时或加载时编织切换到`aspectj`模式。

<a id="scheduling-annotation-support-scheduled"></a>

#### [](#scheduling-annotation-support-scheduled)7.3.2. `@Scheduled` 注解

可以将 `@Scheduled`注解与触发器元数据一起添加到方法中。例如， 下面的方法每隔5秒调用一次固定的延迟， 这意味着该期间将从每次调用的完成时间计算:

```java
@Scheduled(fixedDelay=5000)
public void doSomething() {
    // something that should execute periodically
}
```

如果需要安装固定的速度来执行，只需简单的改变注解中的属性名即可执行。下面可以每5秒执行速度在每个调用开始后：

```java
@Scheduled(fixedRate=5000)
public void doSomething() {
    // something that should execute periodically
}
```

对于固定延迟和固定速率任务， 可以指定初始延迟， 指示在第一次执行该方法之前等待的毫秒数。如下面的 `fixedRate`示例所示:

```java
@Scheduled(initialDelay=1000, fixedRate=5000)
public void doSomething() {
    // something that should execute periodically
}
```

如果简单的周期调度不够表达你的意愿， 则可以提供cron表达式。例如， 以下任务将只在工作日执行。

```java
@Scheduled(cron="*/5 * * * * MON-FRI")
public void doSomething() {
    // something that should execute on weekdays only
}
```

你可以使用 `zone`属性来指定解析cron表达式的时区

请注意，要计划的方法必须具有void返回，并且不能期望为任何参数。如果该方法需要与应用程序上下文中的其他对象进行交互， 则通常是通过依赖项注入提供的。

在Spring 4.3框架中， 任何范围的bean都支持`@Scheduled` 方法。

请确保在运行时不会在同一个`@Scheduled`注解类上初始化多个实例，除非您确实希望为每个此类实例都安排回调。与此相关，请确保不要在bean类上使用`@Configurable`，并将其与容器`@Scheduled`并注册为常规的Spring bean。如果是这样， 程序将会双重初始化，否则，一旦通过容器，并通过@Configurable切面，每个`@Scheduled`方法的结果都将被调用两次。

<a id="scheduling-annotation-support-async"></a>

#### [](#scheduling-annotation-support-async)7.3.3. `@Async` 注解

可以在方法上提供`@Async`注解，以便发生该方法的异步调用。换言之，调用方将在调用时立即返回，并且该方法的实际执行将发生在已提交到Spring `TaskExecutor`的任务中。在最简单的情况下，注解可以应用于`void`返回方法。如下：

```java
@Async
void doSomething() {
    // this will be executed asynchronously
}
```

与使用`@Scheduled`注解的注解方法不同，这些方法可以有预期参数的，因为调用方将在运行时以“normal”方式调用它们，而不是由容器管理的计划任务。例如，以下是`@Async`注解的合法应用:

```java
@Async
void doSomething(String s) {
    // this will be executed asynchronously
}
```

即使返回值的方法也可以异步调用，但是，此类方法需要具有`Future`类型的返回值。这仍然提供了异步执行的好处，以便调用者可以在调用`Future`上的`get()`之前执行其他任务。以下示例显示如何在返回值的方法上使用`@Async` :

```java
@Async
Future<String> returnSomething(int i) {
    // this will be executed asynchronously
}
```

`@Async`方法不仅可以声明一个常规的`java.util.concurrent.Future`返回类型，而且还可能是spring的`org.springframework.util.concurrent.ListenableFuture` 或者如Spring 4.2版本后,存在于JDK8的`java.util.concurrent.CompletableFuture`。用于与异步任务进行更丰富的交互，以及通过进一步的处理步骤进行组合操作。

`@Async`不能与生命周期回调(如 `@PostConstruct`)一起使用。若要异步初始化Spring bean，则当前必须使用单独的初始化Spring bean，然后在目标上调用`@Async`注解方法。如下:

```java
public class SampleBeanImpl implements SampleBean {

    @Async
    void doSomething() {
        // ...
    }

}

public class SampleBeanInitializer {

    private final SampleBean bean;

    public SampleBeanInitializer(SampleBean bean) {
        this.bean = bean;
    }

    @PostConstruct
    public void initialize() {
        bean.doSomething();
    }

}
```

没有直接的XML配置等价于`@Async`，因为这些方法应该首先设计为异步执行，而不是外部得来的。但是，您可以手动设置Spring的`AsyncExecutionInterceptor`与Spring AOP结合使用自定义切点。

<a id="scheduling-annotation-support-qualification"></a>

#### [](#scheduling-annotation-support-qualification)7.3.4. 使用 `@Async`的Executor的条件

默认情况下，在方法上指定`@Async`时，使用的执行程序是 [启用异步支持时配置](#scheduling-enable-annotation-support)的执行程序，例如，如果使用XML或`AsyncConfigurer`实现（如果有），则为“annotation-driven”元素。但是，当需要指示执行给定方法时，应使用非默认的执行器时，可以使用`@Async`注解的`value`属性。以下示例显示了如何执行此操作:

```java
@Async("otherExecutor")
void doSomething(String s) {
    // this will be executed asynchronously by "otherExecutor"
}
```

在这种情况下，`"otherExecutor"`可以是Spring容器中任何`Executor` bean的名称，也可以是与任何`Executor` 关联的限定符的名称（例如，使用`<qualifier>`元素或Spring的`@Qualifier`注解指定）

<a id="scheduling-annotation-support-exception"></a>

#### [](#scheduling-annotation-support-exception)7.3.5. 使用`@Async`的异常管理

当`@Async`方法有`Future`类型的返回值时，可以很容易地管理在方法执行期间引发的异常，因为当调用对`Future`结果的`get`时将引发此异常。但是，对于 `void`返回类型，异常是无法捕获的，无法传输的。对于这些情况，可以提供`AsyncUncaughtExceptionHandler`来处理此类异常。 以下示例显示了如何执行此操作：:

```java
public class MyAsyncUncaughtExceptionHandler implements AsyncUncaughtExceptionHandler {

    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        // handle exception
    }
}
```

默认情况下，仅记录异常。 您可以使用`AsyncConfigurer`或`<task:annotation-driven/>`XML元素定义自定义`AsyncUncaughtExceptionHandler`。

<a id="scheduling-task-namespace"></a>

### [](#scheduling-task-namespace)7.4. `task` 命名空间

从Spring 3.0开始，有一个用于配置`TaskExecutor`和`TaskScheduler`实例的XML命名空间。

<a id="scheduling-task-namespace-scheduler"></a>

#### [](#scheduling-task-namespace-scheduler)7.4.1. 'scheduler' 元素

下面的元素将创建具有指定线程池大小的`ThreadPoolTaskScheduler` 实例:

```xml
<task:scheduler id="scheduler" pool-size="10"/>
```

为`id`属性提供的值将用作池中线程名称的前缀，`scheduler` 元素相对非常简单。如果不提供`pool-size` 属性，则默认的线程池将只有一个线程。计划程序再也没有其他配置选项。

<a id="scheduling-task-namespace-executor"></a>

#### [](#scheduling-task-namespace-executor)7.4.2. `executor` 元素

以下创建一个`ThreadPoolTaskExecutor` 实例:

```xml
<task:executor id="executor" pool-size="10"/>
```

与[上面](#scheduling-task-namespace-scheduler)的调度程序一样，为`id`属性提供的值将用作池中线程名称的前缀。就池大小而言， `executor`元素支持比`scheduler`元素更多的配置选项。首先，`ThreadPoolTaskExecutor`的线程池本身更具可配置。执行器的线程池可能有不同的核心值和最大大小，，而不仅仅是单个的大小。如果提供了单个值，则执行器将具有固定大小的线程池(核心和最大大小相同)。 但是，`executor` 元素的`pool-size` 属性也接受以`min-max`形式的范围。以下示例将最小值设置为`5`，最大值设置为`25`:

```xml
<task:executor
        id="executorWithPoolSizeRange"
        pool-size="5-25"
        queue-capacity="100"/>
```

从该配置中可以看出，还提供了`queue-capacity` (队列容量)值。还应根据执行者的队列容量来考虑线程池的配置，有关池大小和队列容量之间关系的详细说明，请参阅[`ThreadPoolExecutor`](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ThreadPoolExecutor.html)的文档。 主要的想法是，在提交任务时，如果活动线程的数目当前小于核心大小，执行器将首先尝试使用一个空闲线程。如果已达到核心大小，则只要尚未达到其容量，就会将该任务添加到队列中。只有这样，如果已达到队列的容量，执行器将创建一个超出核心大小的新线程。如果还达到最大大小，，则执行器将拒绝该任务。

默认情况下，队列是无限制的，但一般不会这样配置，因为当所有池线程都运行时，如果将很多的任务添加到该队列中，则会导致`OutOfMemoryErrors`。此外，如果队列是无界的，则最大大小根本没有效果。由于执行器总是在创建超出核心大小的新线程之前尝试该队列，因此队列必须具有有限的容量，以使线程池超出核心大小(这就是为什么在使用无界队列时，固定大小池是唯一合理的情况)。

如上所述，在任务被拒绝时考虑这种情况。默认情况下，当任务被拒绝时，线程池执行程序会抛出`TaskRejectedException`。但是，拒绝策略实际上是可配置的。使用默认拒绝策略时抛出异常，即`AbortPolicy`实现。对于可以在高负载下跳过某些任务的应用程序，您可以改为配置`DiscardPolicy`或`DiscardOldestPolicy`。另一个适用于需要在高负载下限制提交任务的应用程序的选项是`CallerRunsPolicy`。该策略不是抛出异常或丢弃任务，而是强制调用submit方法的线程自己运行任务。这个想法是这样的调用者在运行该任务时很忙，并且不能立即提交其他任务。因此，它提供了一种简单的方法来限制传入的负载，同时保持线程池和队列的限制。通常，这允许执行程序“赶上”它正在处理的任务，从而释放队列，池中或两者中的一些容量。您可以从`executor`元素上的`rejection-policy`属性的可用值枚举中选择任何这些选项。

以下示例显示了一个`executor` 元素，其中包含许多属性以指定各种行为：:

```xml
<task:executor
        id="executorWithCallerRunsPolicy"
        pool-size="5-25"
        queue-capacity="100"
        rejection-policy="CALLER_RUNS"/>
```

最后, `keep-alive`设置确定在终止之前线程可能保持空闲的时间限制(以秒为单位)。如果池中当前有多个线程的核心数目，则在等待此时间量后不处理任务，多余的线程将被终止。时间值为零将导致多余的线程在执行任务后立即终止，而不需要在任务队列中保留后续工作。以下示例将`keep-alive`值设置为两分钟：:

```xml
<task:executor
        id="executorWithKeepAlive"
        pool-size="5-25"
        keep-alive="120"/>
```

<a id="scheduling-task-namespace-scheduled-tasks"></a>

#### [](#scheduling-task-namespace-scheduled-tasks)7.4.3. 'scheduled-tasks' 元素

Spring的任务命名空间的最强大功能是支持配置在Spring应用程序上下文中安排的任务。这遵循了类似于Spring中其他“method-invokers”的方法，例如由JMS命名空间提供的用于配置消息驱动的pojo。 基本上， `ref`属性可以指向任何Spring管理的对象， `method` 属性提供要在该对象上调用的方法的名称。 以下清单显示了一个简单示例：:

```xml
<task:scheduled-tasks scheduler="myScheduler">
    <task:scheduled ref="beanA" method="methodA" fixed-delay="5000"/>
</task:scheduled-tasks>

<task:scheduler id="myScheduler" pool-size="10"/>
```

Tscheduler由外部元素引用，每个单独的任务都包括其触发器元数据的配置。在前面的示例中，该元数据定义了一个定期触发器，它具有固定的延迟，表示每个任务执行完成后等待的毫秒数。另一个选项是"`fixed-rate`"，表示无论以前执行多长时间，方法的执行频率。此外， ，对于`fixed-delay`和`fixed-rate`任务，可以指定'initial-delay'参数，指示在第一次执行该方法之前等待的毫秒数。为了获得更多的控制，可以改为提供一个`cron`属性。下面是一个演示其他选项的示例:

```xml
<task:scheduled-tasks scheduler="myScheduler">
    <task:scheduled ref="beanA" method="methodA" fixed-delay="5000" initial-delay="1000"/>
    <task:scheduled ref="beanB" method="methodB" fixed-rate="5000"/>
    <task:scheduled ref="beanC" method="methodC" cron="*/5 * * * * MON-FRI"/>
</task:scheduled-tasks>

<task:scheduler id="myScheduler" pool-size="10"/>
```

<a id="scheduling-quartz"></a>

### [](#scheduling-quartz)7.5. 使用Quartz的Scheduler

Quartz使用`Trigger`, `Job`, 和 `JobDetail`等对象来进行各种类型的任务调度。关于Quartz的基本概念，请参阅[http://quartz-scheduler.org](http://quartz-scheduler.org)。为了让基于Spring的应用程序方便使用，Spring提供了一些类来简化quartz的用法。

<a id="scheduling-quartz-jobdetail"></a>

#### [](#scheduling-quartz-jobdetail)7.5.1. 使用`JobDetailFactoryBean`

Quartz `JobDetail` 对象保存运行一个任务所需的全部信息。Spring提供一个叫作 `JobDetailFactoryBean`的类让JobDetail能对一些有意义的初始值(从XML配置)进行初始化，让我们来看个例子:

```xml
<bean name="exampleJob" class="org.springframework.scheduling.quartz.JobDetailFactoryBean">
    <property name="jobClass" value="example.ExampleJob"/>
    <property name="jobDataAsMap">
        <map>
            <entry key="timeout" value="5"/>
        </map>
    </property>
</bean>
```

job detail 配置拥有所有运行job(`ExampleJob`)的必要信息。 可以通过job的data map来指定timeout。Job的data map可以通过`JobExecutionContext`（在运行时传递给你)来得到，但是 `JobDetail`同时把从job的data map中得到的属性映射到实际job中的属性中去。 所以，如果`ExampleJob`中包含一个名为 `timeout`的属性，`JobDetail`将自动为它赋值:

```java
package example;

public class ExampleJob extends QuartzJobBean {

    private int timeout;

    /**
     * Setter called after the ExampleJob is instantiated
     * with the value from the JobDetailFactoryBean (5)
     */
    public void setTimeout(int timeout) {
        this.timeout = timeout;
    }

    protected void executeInternal(JobExecutionContext ctx) throws JobExecutionException {
        // do the actual work
    }

}
```

data map中的所有附加属性当然也可以使用的

使用`name`和`group`属性，你可以分别修改job在哪一个组下运行和使用什么名称。默认情况下，job的名称等于`JobDetailFactoryBean`的名称（在上面的例子中为`exampleJob`)。

<a id="scheduling-quartz-method-invoking-job"></a>

#### [](#scheduling-quartz-method-invoking-job)7.5.2. 使用 `MethodInvokingJobDetailFactoryBean`

通常情况下，你只需要调用特定对象上的一个方法即可实现任务调度。你可以使用`MethodInvokingJobDetailFactoryBean`准确的做到这一点:

```xml
<bean id="jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
    <property name="targetObject" ref="exampleBusinessObject"/>
    <property name="targetMethod" value="doIt"/>
</bean>
```

上面例子将调用`exampleBusinessObject`中的`doIt` 方法如下:

```java
public class ExampleBusinessObject {

    // properties and collaborators

    public void doIt() {
        // do the actual work
    }
}
```

```
<bean id="exampleBusinessObject" class="examples.ExampleBusinessObject"/>
```

使用`MethodInvokingJobDetailFactoryBean`你不需要创建只有一行代码且只调用一个方法的job， 你只需要创建真实的业务对象来包装具体的细节的对象。

默认情况下，Quartz Jobs是无状态的，可能导致jobs之间互相的影响。如果你为相同的`JobDetail`指定两个Trigger， 很可能当第一个job完成之前，第二个job就开始了。如果`JobDetail`对象实现了`Stateful`接口，就不会发生这样的事情。 第二个job将不会在第一个job完成之前开始。为了使得jobs不并发运行，设置`MethodInvokingJobDetailFactoryBean`中的`concurrent`标记为`false`:

```xml
<bean id="jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
    <property name="targetObject" ref="exampleBusinessObject"/>
    <property name="targetMethod" value="doIt"/>
    <property name="concurrent" value="false"/>
</bean>
```

默认情况下，jobs在并发的方式下运行。

<a id="scheduling-quartz-cron"></a>

#### [](#scheduling-quartz-cron)7.5.3. 使用triggers和`SchedulerFactoryBean`来织入任务

我们已经创建了job details，jobs。我们同时回顾了允许你调用特定对象上某一个方法的便捷的bean。 当然我们仍需要调度这些jobs。这需要使用triggers和`SchedulerFactoryBean`来完成。 Quartz中提供了几个triggers，Spring提供了两个带有方便默认值的Quartz `FactoryBean`实现：`CronTriggerFactoryBean`和`SimpleTriggerFactoryBean`。

Triggers也需要被调度。Spring提供`SchedulerFactoryBean`来公开一些属性来设置triggers。`SchedulerFactoryBean`负责调度那些实际的triggers

以下清单使用`SimpleTriggerFactoryBean`和`CronTriggerFactoryBean`:

```xml
<bean id="simpleTrigger" class="org.springframework.scheduling.quartz.SimpleTriggerFactoryBean">
    <!-- see the example of method invoking job above -->
    <property name="jobDetail" ref="jobDetail"/>
    <!-- 10 seconds -->
    <property name="startDelay" value="10000"/>
    <!-- repeat every 50 seconds -->
    <property name="repeatInterval" value="50000"/>
</bean>

<bean id="cronTrigger" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
    <property name="jobDetail" ref="exampleJob"/>
    <!-- run every morning at 6 AM -->
    <property name="cronExpression" value="0 0 6 * * ?"/>
</bean>
```

现在我们创建了两个triggers，其中一个开始延迟10秒以后每50秒运行一次，另一个每天早上6点钟运行。 我们需要创建一个`SchedulerFactoryBean`来最终实现上述的一切:

```xml
<bean class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
    <property name="triggers">
        <list>
            <ref bean="cronTrigger"/>
            <ref bean="simpleTrigger"/>
        </list>
    </property>
</bean>
```

更多的属性你可以通过`SchedulerFactoryBean`来设置，例如job details使用的日期， 用来订制Quartz的一些属性以及其它相关信息。 你可以查阅[`SchedulerFactoryBean`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/scheduling/quartz/SchedulerFactoryBean.html)的Javadoc。