<a id="jms"></a>

[](#jms)3\. JMS (Java 消息服务 )
----------------------------

Spring提供了一个JMS集成框架， 它简化了JMS API的使用， 就像Spring对JDBC API的集成一样。

JMS可以大致分为两个功能领域， 即消息的生产和消费。`JmsTemplate` 类用于消息生成和同步消息接收。对于类似于Java EE的消息驱动bean样式的异步接收， Spring提供了许多用于创建消息驱动的pojo(MDPs)的消息监听器容器。Spring还提供了一种创建消息监听器的声明方式。

`org.springframework.jms.core` 提供了使用JMS的核心功能。包含JMS模板类， 通过处理资源的创建和释放来简化JMS的使用， 就像`JdbcTemplate`对JDBC所做的那样。 Spring模板类的通用设计原则是提供帮助器方法来执行常见操作和更复杂的用法， 将处理任务的本质委托给用户实现的回调接口。JMS模板遵循相同的设计。这些类为发送消息提供了各种方便方法， 同步地使用消息， 并向用户公开JMS会话和消息生成器。

`org.springframework.jms.support` 提供`JMSException`的转换功能。转换将选中的`JMSException`层次结构转换为未选中的异常的镜像层次结构。如果已选中的`javax.jms.JMSException`有任何提供程序特定的子类， 则此异常将包装在未选中的`UncategorizedJmsException`中（选中：checked，未选中：unchecked）。

`org.springframework.jms.support.converter`提供了一个`MessageConverter`的抽象， 用于在Java对象和JMS消息之间进行转换。

`org.springframework.jms.support.destination` 为管理JMS目的地提供了各种策略， 例如为存储在JNDI中的目的地提供服务定位器。

`org.springframework.jms.annotation` 提供了必要的基础架构， 以支持使用`@JmsListener`的注解驱动的监听器端点。

`org.springframework.jms.config`为`jms`命名空间提供解析器实现， 以及Java配置支持来配置监听器容器和创建监听器端点。

最后, `org.springframework.jms.connection` 提供了适用于独立应用程序的`ConnectionFactory`的实现。它还包含一个Spring的`PlatformTransactionManager` JMS实现(命名为`JmsTransactionManager`)。这允许将JMS作为事务性资源无缝集成到Spring的事务管理机制中。

<a id="jms-using"></a>

### [](#jms-using)3.1. 使用 Spring JMS

本章节主要介绍如何使用 Spring JMS 组件.

<a id="jms-jmstemplate"></a>

#### [](#jms-jmstemplate)3.1.1. 使用 `JmsTemplate`

`JmsTemplate` 类 是JMS核心包中的中心类，它简化了JMS的使用， 因为它在发送或同步接收消息时处理资源的创建和释放。

使用`JmsTemplate`的代码只需实现回调接口， 就可以为它们提供明确定义的高级功能。`MessageCreator`回调接口在`JmsTemplate`的调用代码提供的会话中创建一条消息。 为了允许更复杂地使用JMS API， 回调`SessionCallback`为用户提供了JMS会话， 回调`ProducerCallback`公开了一个会话和 `MessageProducer` 配对。

JMS API公开了两种类型的发送方法， 一种是将传递模式、优先级和生存作为服务质量 (Qos) 参数， 另一个不采用使用默认值的Qos参数。由于 `JmsTemplate`中有许多发送方法， 所以QoS参数的设置已作为bean属性公开， 以避免发送方法的数量重复。同样， 使用属性`setReceiveTimeout`设置同步接收调用的超时值。

某些JMS提供程序允许通过`ConnectionFactory`的配置管理默认QoS值的设置。这会影响到对`MessageProducer`的`send`方法(`send(Destination destination, Message message)`)的调用将使用与JMS规范中指定的不同的QoS默认值。 为了提供对QoS值的一致管理，因此必须通过将布尔属性`isExplicitQosEnabled`设置为 `true`来明确地启用`JmsTemplate`以使用其自身的QoS值。

为方便起见， `JmsTemplate`还公开了一个基本的请求-答复操作， 允许发送消息并等待在作为操作一部分创建的临时队列上的答复。

`JmsTemplate`类的实例在配置后是线程安全的。这很重要， 因为这意味着您可以配置`JmsTemplate`的单个实例， 然后将此共享引用安全地插入到多个协作者中。 要清楚， `JmsTemplate`是有状态的， 因为它维护对 `ConnectionFactory`的引用， 但此状态不是会话状态。

在Spring 4.1框架中, `JmsMessagingTemplate` 是在`JmsTemplate`之上构建的， 它提供了与消息抽象(即 `org.springframework.messaging.Message`)的集成.这使您可以创建以通用方式发送的消息.

<a id="jms-connections"></a>

#### [](#jms-connections)3.1.2. Connections

`JmsTemplate` 需要`ConnectionFactory`的引用，`ConnectionFactory`是JMS规范的一部分，是使用JMS的入口点。它由客户端应用程序用作工厂来创建与JMS提供程序的连接，并封装各种配置参数，其中许多是特定于供应商的，如SSL配置选项。

在 EJB 中使用JMS时， 供应商提供JMS接口的实现， 以便它们可以参与声明性事务管理并执行连接和会话池。为了使用此实现， Java EE容器通常要求您将JMS连接工厂声明为EJB或Servlet部署描述符内的`resource-ref`。 为了确保将这些功能与EJB中的`JmsTemplate` 一起使用， 客户端应用程序应确保它引用`ConnectionFactory`的托管实现。

<a id="jms-caching-resources"></a>

##### [](#jms-caching-resources)缓存消息资源

标准API涉及创建许多中间对象，要发送消息， 请执行以 'API'步骤：

ConnectionFactory->Connection->Session->MessageProducer->send

在`ConnectionFactory`和`Send`操作之间， 有三个中间对象被创建和销毁。为优化资源使用和提高性能， 提供了两个 `ConnectionFactory`的实现。

<a id="jms-connection-factory"></a>

##### [](#jms-connection-factory)使用 `SingleConnectionFactory`

Spring提供了`ConnectionFactory`接口的实现， `SingleConnectionFactory`将在所有 `createConnection()`调用中返回相同的`Connection`， 并忽略`close()`的调用。 这对于测试和独立环境非常有用， 以便可以将相同的连接用于可能跨越任意数量事务的多个 `JmsTemplate` 调用，`SingleConnectionFactory`引用的标准`ConnectionFactory`通常来自JNDI。

<a id="jdbc-connection-factory-caching"></a>

##### [](#jdbc-connection-factory-caching)使用 `CachingConnectionFactory`

`CachingConnectionFactory`继承了`SingleConnectionFactory`的功能， 并添加了`Session`, `MessageProducer`, 和 `MessageConsumer`的缓存。初始缓存大小设置为`1`， 使用属性`sessionCacheSize`增加缓存会话的数量。请注意， 实际缓存会话的数目将超过该数目， 因为会话是根据其确认模式缓存的， 因此， 当`sessionCacheSize`被设置为一个、每个确认模式为一个时， 最多可以有4个缓存会话实例。 `MessageProducer` 和 `MessageConsumer` 在其所属会话中被缓存，， 并且在缓存时也考虑了生产者和使用者的独特属性。MessageProducers根据其目的地进行缓存，MessageConsumers是根据由目的地、选择器、noLocal传递标志和持久性订阅名称(如果创建耐用的使用者)组成的键进行缓存的。

<a id="jms-destinations"></a>

#### [](#jms-destinations)3.1.3. 目的地管理

目的地，如`ConnectionFactory`，是JMS管理的对象，可以在JNDI中进行存储和检索。配置Spring应用程序上下文时，可以使用JNDI工厂类`JndiObjectFactoryBean` /`<jee:jndi-lookup>`对对象对JMS目标的引用执行依赖项注入。 但是，如果应用程序中有大量目的地，或者JMS提供程序具有唯一的高级目的地管理功能，则此策略通常很麻烦。此类高级目的地管理的示例包括创建动态目的地或支持目的地的分层命名空间.`JmsTemplate`将目标名称的解析委托给JMS目标对象，以实现接口`DestinationResolver`。 `DynamicDestinationResolver`是`JmsTemplate`使用的默认实现，可容纳解析动态目标。还提供了一个`JndiDestinationResolver` ，它充当JNDI中包含的目标的服务定位器，并可选择退回到`DynamicDestinationResolver`中包含的行为。

在JMS应用程序中使用的目的地通常只在运行时才知道，因此在部署应用程序时无法进行管理创建。这通常是因为根据一个众所周知的命名约定，在运行时创建目的地的交互系统组件之间存在共享的应用程序逻辑。尽管创建动态目的地不是JMS规范的一部分，但大多数供应商都提供了此功能。动态目的地是使用用户定义的名称来创建的，它们将它们与临时目的地区分开来，并且通常不在JNDI中注册。 由于与目的地关联的属性是特定于供应商的，因此用于创建动态目的地的API因提供程序而异。但是，供应商有时所做的简单实现选择是忽略JMS规范中的警告，并使用`TopicSession`方法的 `createTopic(String topicName)`字符串topicName或`QueueSession`方法的`createQueue(String queueName)` 字符串queueName来创建具有默认目的地属性的新目的地。根据供应商实施情况， `DynamicDestinationResolver` 可能还会创建物理目的地，而不是仅解析一个。

布尔属性`pubSubDomain`用于配置`JmsTemplate`， 并了解所使用的JMS域。默认情况下， 此属性的值为false， 表示将使用点到点域( `Queues`)。`JmsTemplate`使用的此属性通过`DestinationResolver`接口的实现确定动态目的地解析的行为。

还可以通过属性`defaultDestination`将`JmsTemplate`配置为默认目的地，默认目的地将与不引用特定目的地的发送和接收操作一起使用。

<a id="jms-mdp"></a>

#### [](#jms-mdp)3.1.4. 消息监听容器

在EJB世界中，JMS消息最常见的一种用法是驱动消息驱动bean(mdb)。Spring提供了一种以不将用户与EJB容器绑定的方式创建消息驱动的pojo(mdp)的解决方案（请查看[异步接收:消息驱动的 POJOs](#jms-asynchronousMessageReception)，用于详细报道Spring的MDP支持）。 在Spring 4.1框架后,端点方法可以简单地用 `@JmsListener`注解，请参见 [注解驱动的监听器端点](#jms-annotated)。

消息监听器容器用于接收来自JMS消息队列的消息，并驱动向其注入的`MessageListener`。监听器容器负责消息接收的所有线程处理，并将其分派到监听器中以进行操作。消息监听器容器是MDP和消息传递提供程序之间的中介，它负责注册接收消息、参与事务、获取和释放资源、异常转换和类似的操作。 这样，您就可以作为应用程序开发人员编写与接收消息(可能响应)相关的(可能是复杂的)业务逻辑，并将样板JMS基础结构的关注委托给框架。

有两个标准的JMS消息监听器容器与Spring一起打包， 每个都有其专门的功能集

*   [`SimpleMessageListenerContainer`](#jms-mdp-simple)
    
*   [`DefaultMessageListenerContainer`](#jms-mdp-default)

<a id="jms-mdp-simple"></a>

##### [](#jms-mdp-simple)使用 `SimpleMessageListenerContainer`

SimpleMessageListenerContainer消息监听器容器是两种标准中比较简单的一种，它在启动时创建一个固定数量的JMS会话和用户， 使用标准的JMS 的`MessageConsumer.setMessageListener()` 方法注册监听器， 并将它留在JMS提供程序中以执行监听器回调。此变量不允许动态适应运行时要求或参与外部托管事务。 兼容性方面，它非常接近独立JMS规范的精神，但通常与Java EE的JMS限制不兼容。

虽然`SimpleMessageListenerContainer`不允许参与外部管理的事务，它支持本机JMS事务：只需将`sessionTransacted` 标志切换为`true` ，或在命名空间中，将`acknowledge`属性设置为 `transacted`。从您的监听器抛出的异常然后导致回滚，并重新传递消息。或者，考虑使用`CLIENT_ACKNOWLEDGE`模式，该模式在异常的情况下也提供重新传递，但不使用事务会话实例，因此不包括事务协议中的任何其他会话操作（例如发送响应消息）。

默认的 `AUTO_ACKNOWLEDGE`模式不提供适当的可靠性保证。 当监听器执行失败时（由于提供程序在监听器调用后自动确认每条消息，没有异常传播到提供程序）或监听器容器关闭时（可以通过设置 `acceptMessagesWhileStopping`标志来配置），消息可能会丢失。 确保在可靠性需求的情况下使用事务处理会话（例如，用于可靠的队列处理和持久主题订阅）。

<a id="jms-mdp-default"></a>

##### [](#jms-mdp-default)使用 `DefaultMessageListenerContainer`

DefaultMessageListenerContainer消息监听器容器是大多数情况下使用的。与`SimpleMessageListenerContainer`相比，此容器变量允许动态地适应运行时需求，并能够参与外部托管事务。 当配置了`JtaTransactionManager`时，每个收到的消息都在XA事务中注册。因此，处理可以利用XA事务语义。此监听器容器在JMS提供程序的低需求、高级功能(如参与外部托管事务以及与Java EE环境的兼容性)之间取得了良好的平衡。

您可以自定义容器的缓存级别。 请注意，如果未启用缓存，则会为每个消息接收创建新连接和新会话。将其与具有高负载的非持久性订阅相结合可能会导致消息丢失。请确保在这种情况下使用适当的缓存级别。

当代理发生故障时，此容器还具有可恢复的功能。 默认情况下，简单的`BackOff`实现每五秒重试一次。 您可以为更细粒度的恢复选项指定自定义`BackOff`实现。 有关示例，请参阅api-spring-framework/util/backoff/ExponentialBackOff.html \[`ExponentialBackOff`\]。

与它的同级的[`SimpleMessageListenerContainer`](#jms-mdp-simple)一样，`DefaultMessageListenerContainer`支持本机JMS事务并允许自定义确认模式。如果您的方案可行，则强烈建议在外部管理的事务上进行此操作；即，如果您可以在JVM死亡的情况下使用偶尔重复的消息。 业务逻辑中的自定义重复消息检测步骤可能涵盖此类情况，例如以业务实体存在检查或协议表检查的形式。任何此类安排都将比替代方案更有效。用XA事务包装您的整个处理(通过配置您的`DefaultMessageListenerContainer`与`JtaTransactionManager`，涵盖JMS消息的接收以及消息监听器(包括数据库操作等)中业务逻辑的执行。

默认的 `AUTO_ACKNOWLEDGE`模式不提供适当的可靠性保证。 当监听器执行失败时（由于提供程序在监听器调用后自动确认每条消息，没有异常传播到提供程序）或监听器容器关闭时（可以通过设置`acceptMessagesWhileStopping`标志来配置），消息可能会丢失。 确保在可靠性需求的情况下使用事务处理会话（例如，用于可靠的队列处理和持久主题订阅）。

<a id="jms-tx"></a>

#### [](#jms-tx)3.1.5. 事务管理

Spring提供了一个`JmsTransactionManager`，用于管理单个JMS `ConnectionFactory`事务。这使得JMS应用程序能够利用 [如数据访问章节的“事务管理”部分所述。](data-access.html#transaction) 。`JmsTransactionManager`执行本地资源事务，将JMSConnection/Session对从指定的`ConnectionFactory`绑定到线程。 `JmsTemplate` 自动检测此类事务性资源，并据此对其进行操作。

在Java EE环境中， `ConnectionFactory`将建立连接池和会话池，因此这些资源可以在事务之间有效地重用。在独立的环境中，使用Spring的`SingleConnectionFactory`将导致共享的JMS `Connection`，每个事务都有自己的独立`Session`。 或者，考虑使用提供程序特定的池适配器，如ActiveMQ的`PooledConnectionFactory`类。

`JmsTemplate`还可以与`JtaTransactionManager`和具有XA能力的JMSConnectionFactory一起使用，用于执行分布式事务。 请注意，这需要使用JTA事务管理器以及正确配置XA的`ConnectionFactory`。 （查看Java EE服务器或JMS提供程序的文档。）

使用JMS API从 `Connection`创建`Session`时，在托管和非托管事务环境中重用代码可能会造成混淆。 这是因为JMS API只有一个工厂方法来创建 `Session`，它需要事务和确认模式的值。 在托管环境中，设置这些值是环境的事务基础结构的责任，因此供应商的JMS连接包装器会忽略这些值。 在非托管环境中使用JmsTemplate时，可以通过使用属性`sessionTransacted`和 `sessionAcknowledgeMode`来指定这些值。 当您将 `PlatformTransactionManager`与`JmsTemplate`一起使用时，模板始终会被赋予事务性JMS `Session`。

<a id="jms-sending"></a>

### [](#jms-sending)3.2. 发送消息

`JmsTemplate` 包含许多发送消息的便捷方法。 Send方法使用`javax.jms.Destination`对象指定目标，其他方法通过在JNDI查找中使用 `String` 指定目标。 不带目标参数的`send`方法使用默认目标。

以下示例使用`MessageCreator` 回调从提供的`Session`对象创建文本消息：:

```java
import javax.jms.ConnectionFactory;
import javax.jms.JMSException;
import javax.jms.Message;
import javax.jms.Queue;
import javax.jms.Session;

import org.springframework.jms.core.MessageCreator;
import org.springframework.jms.core.JmsTemplate;

public class JmsQueueSender {

    private JmsTemplate jmsTemplate;
    private Queue queue;

    public void setConnectionFactory(ConnectionFactory cf) {
        this.jmsTemplate = new JmsTemplate(cf);
    }

    public void setQueue(Queue queue) {
        this.queue = queue;
    }

    public void simpleSend() {
        this.jmsTemplate.send(this.queue, new MessageCreator() {
            public Message createMessage(Session session) throws JMSException {
                return session.createTextMessage("hello queue world");
            }
        });
    }
}
```

在前面的示例中，通过将引用传递给 `ConnectionFactory`来构造`JmsTemplate` 。作为替代方法，提供了无参数构造函数和`connectionFactory`，可用于在JavaBean样式(使用`BeanFactory`或纯Java代码)中构造实例， 。或者，考虑从Spring的`JmsGatewaySupport`方便基类派生，它为JMS配置提供了预构建的bean属性。

`send(String destinationName, MessageCreator creator)` 方法允许您使用目的地的字符串名称发送消息。如果这些名称是在JNDI中注册的，则应将该模板的`destinationResolver`属性设置为 `JndiDestinationResolver`的实例。

如果创建了`JmsTemplate`并指定了默认目的地， 则`send(MessageCreator c)` 将向该目的地发送一条消息。

<a id="jms-msg-conversion"></a>

#### [](#jms-msg-conversion)3.2.1. 使用消息转换器

为了便于发送域模型对象， `JmsTemplate`有多种发送方法，它们将Java对象作为消息的数据内容的参数。`JmsTemplate` 中的重载方法`convertAndSend()` 和 `receiveAndConvert()`将转换过程委托给`MessageConverter`接口的实例。 此接口定义在Java对象和JMS消息之间转换的简单协定。默认实现`SimpleMessageConverter`支持`String` 和 `TextMessage`, `byte[]` 和 `BytesMesssage`, 和 `java.util.Map`和 `MapMessage`

沙箱当前包含一个`MapMessageConverter`，它使用反射在JavaBean和`MapMessage`之间进行转换。 您可能自己实现的其他流行实现选择是使用现有XML编组包（例如JAXB，Castor或XStream）的转换器来创建表示对象的`TextMessage`。

为了适应在转换器类中不能被全部封装的消息属性、报头和正文的设置，`MessagePostProcessor`接口在转换后，即在发送消息之前，为您提供对该消息的访问权限。下面的示例演示如何在将 `java.util.Map`转换为消息后修改消息头和属性。

```java
public void sendWithConversion() {
    Map map = new HashMap();
    map.put("Name", "Mark");
    map.put("Age", new Integer(47));
    jmsTemplate.convertAndSend("testQueue", map, new MessagePostProcessor() {
        public Message postProcessMessage(Message message) throws JMSException {
            message.setIntProperty("AccountID", 1234);
            message.setJMSCorrelationID("123-00001");
            return message;
        }
    });
}
```

这会产生以下形式的消息：

MapMessage={
    Header={
        ... standard headers ...
        CorrelationID={123-00001}
    }
    Properties={
        AccountID={Integer:1234}
    }
    Fields={
        Name={String:Mark}
        Age={Integer:47}
    }
}

<a id="jms-callbacks"></a>

#### [](#jms-callbacks)3.2.2. 使用 `SessionCallback` 和 `ProducerCallback`

虽然发送操作涉及许多常见的使用方案，但在需要对JMS会话或`MessageProducer`执行多个操作时，也会出现这种情况。 `SessionCallback`和`ProducerCallback`分别公开JMS `Session`和`Session` /`MessageProducer`对。`JmsTemplate`上的`execute()`方法执行这些回调方法。

<a id="jms-receiving"></a>

### [](#jms-receiving)3.3. 接收消息

这描述了如何在Spring中使用JMS接收消息。

<a id="jms-receiving-sync"></a>

#### [](#jms-receiving-sync)3.3.1. 同步接收

虽然JMS通常与异步处理相关联， 但可以同步使用消息。重载`receive(..)`方法可提供此功能。在同步接收过程中， 调用线程会一直阻塞， 直到消息变为可用。这可能是一个危险的操作， 因为调用线程可能会无限期地被阻塞。`receiveTimeout`属性指定接收方在放弃等待消息之前应等待的时间。

<a id="jms-asynchronousMessageReception"></a>

#### [](#jms-asynchronousMessageReception)3.3.2. 异步接收：消息驱动的POJO

Spring还通过使用`@JmsListener`注解支持带注解的监听器端点，并提供一个开放式基础架构以编程方式注册端。这是设置异步接收器的最方便的方法，有关更多详细信息，请参阅[启用监听器端点注解](#jms-annotated-support)。

以类似于EJB世界中的消息驱动Bean（MDB）的方式，消息驱动的POJO（MDP）充当JMS消息的接收器。 MDP上的一个限制（但请参阅[Using `MessageListenerAdapter`](#jms-receiving-async-message-listener-adapter)）是它必须实现 `javax.jms.MessageListener` 接口。 请注意，如果您的POJO在多个线程上收到消息，请务必确保您的实现是线程安全的。

以下示例显示了MDP的简单实现：

```java
import javax.jms.JMSException;
import javax.jms.Message;
import javax.jms.MessageListener;
import javax.jms.TextMessage;

public class ExampleListener implements MessageListener {

    public void onMessage(Message message) {
        if (message instanceof TextMessage) {
            try {
                System.out.println(((TextMessage) message).getText());
            }
            catch (JMSException ex) {
                throw new RuntimeException(ex);
            }
        }
        else {
            throw new IllegalArgumentException("Message must be of type TextMessage");
        }
    }
}
```

一旦实现了`MessageListener`，就可以创建一个消息监听器容器了。

以下示例显示如何定义和配置Spring附带的消息监听器容器之一（在本例中为`DefaultMessageListenerContainer`）：

```xml
<!-- this is the Message Driven POJO (MDP) -->
<bean id="messageListener" class="jmsexample.ExampleListener"/>

<!-- and this is the message listener container -->
<bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="connectionFactory" ref="connectionFactory"/>
    <property name="destination" ref="destination"/>
    <property name="messageListener" ref="messageListener"/>
</bean>
```

请参阅各种消息监听器容器的Spring javadoc（所有这些实现[MessageListenerContainer](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/jms/listener/MessageListenerContainer.html)）以获取每个实现支持的功能的完整描述。

<a id="jms-receiving-async-session-aware-message-listener"></a>

#### [](#jms-receiving-async-session-aware-message-listener)3.3.3.使用`SessionAwareMessageListener`接口

`SessionAwareMessageListener` 接口是一个特定于Spring的接口，它提供与JMS `MessageListener` 接口类似的契约，但也提供消息处理方法访问接收消息的JMS会话。 以下清单显示了`SessionAwareMessageListener`接口的定义：

```java
package org.springframework.jms.listener;

public interface SessionAwareMessageListener {

    void onMessage(Message message, Session session) throws JMSException;
}
```

如果希望您的MDP能够响应任何收到的消息(通过使用`onMessage(Message, Session)`方法中提供的会话)，您可以选择让您的MDP实现此接口(优先于标准的JMS `MessageListener` 接口)。 与Spring一起发送的所有消息监听器容器实现都支持实现`MessageListener`或`SessionAwareMessageListener`接口的MDP。实现`SessionAwareMessageListener`的类随之而来的警告是，它们将通过接口绑定到Spring。选择是否使用它完全由您作为应用程序开发人员或架构师来决定。

请注意，`SessionAwareMessageListener`接口的 `onMessage(..)`方法会抛出`JMSException`。 与标准JMS `MessageListener`接口相比，使用`SessionAwareMessageListener` 接口时，客户端代码负责处理任何抛出的异常。

<a id="jms-receiving-async-message-listener-adapter"></a>

#### [](#jms-receiving-async-message-listener-adapter)3.3.4. Using `MessageListenerAdapter`

`MessageListenerAdapter`类是Spring的异步消息传递支持中的最后一个组件。 简而言之， 它允许您将几乎任何类作为 MDP(当然也有一些限制)。

请考虑以下接口定义：

```java
public interface MessageDelegate {

    void handleMessage(String message);

    void handleMessage(Map message);

    void handleMessage(byte[] message);

    void handleMessage(Serializable message);
}
```

请注意，虽然接口既不扩展`MessageListener`也不扩展`SessionAwareMessageListener`接口，但您仍可以使用`MessageListenerAdapter`类将其用作MDP。还请注意各种消息处理方法是如何根据它们可以接收和处理的各种消息类型的内容来强类型化的。

现在考虑`MessageDelegate`接口的以下实现：

```java
public class DefaultMessageDelegate implements MessageDelegate {
    // implementation elided for clarity...
}
```

特别要注意`MessageDelegate`接口的前面实现（`DefaultMessageDelegate`类）根本没有JMS依赖。 它真的是一个POJO，我们可以通过以下配置进入MDP：

```xml
<!-- this is the Message Driven POJO (MDP) -->
<bean id="messageListener" class="org.springframework.jms.listener.adapter.MessageListenerAdapter">
    <constructor-arg>
        <bean class="jmsexample.DefaultMessageDelegate"/>
    </constructor-arg>
</bean>

<!-- and this is the message listener container... -->
<bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="connectionFactory" ref="connectionFactory"/>
    <property name="destination" ref="destination"/>
    <property name="messageListener" ref="messageListener"/>
</bean>
```

下面是另一个MDP的示例，它只能处理JMS `TextMessage`消息的接收。请注意，消息处理方法实际上被称为 `receive`(`MessageListenerAdapter`中的消息处理方法的名称默认为`handleMessage`),但它是可配置的(如下所示)。还请注意， `receive(..)`方法是如何强类型化的，以便只接收和响应JMS `TextMessage`消息。以下清单显示了`TextMessageDelegate`接口的定义：

```java
public interface TextMessageDelegate {

    void receive(TextMessage message);
}
```

以下清单显示了一个实现`TextMessageDelegate`接口的类:

```java
public class DefaultTextMessageDelegate implements TextMessageDelegate {
    // implementation elided for clarity...
}
```

随后`MessageListenerAdapter`的配置如下：

```xml
<bean id="messageListener" class="org.springframework.jms.listener.adapter.MessageListenerAdapter">
    <constructor-arg>
        <bean class="jmsexample.DefaultTextMessageDelegate"/>
    </constructor-arg>
    <property name="defaultListenerMethod" value="receive"/>
    <!-- we don't want automatic message context extraction -->
    <property name="messageConverter">
        <null/>
    </property>
</bean>
```

请注意，如果`messageListener`接收到`TextMessage`以外类型的JMS消息，则抛出`IllegalStateException`（随后又消除了）。 `MessageListenerAdapter`类的另一个功能是，如果处理程序方法返回非void值，则能够自动发回响应消息。 考虑以下接口和类：

```java
public interface ResponsiveTextMessageDelegate {

    // notice the return type...
    String receive(TextMessage message);
}
```

```java
public class DefaultResponsiveTextMessageDelegate implements ResponsiveTextMessageDelegate {
    // implementation elided for clarity...
}
```

如果上述`DefaultResponsiveTextMessageDelegate`与`MessageListenerAdapter`一起使用，则从执行`'receive(..)'` 方法返回的任何非null值将(在默认配置中)转换为`TextMessage`。然后，生成的`TextMessage`将被发送到原始消息的JMS答复属性中定义的目标(如果存在)， 或`MessageListenerAdapter`上的默认`Destination`(如果已配置)，如果找不到`Destination`，则会抛出`InvalidDestinationException`(请注意，此异常将不会被消除，并将传播到调用堆栈)。

<a id="jms-tx-participation"></a>

#### [](#jms-tx-participation)3.3.5. 处理事务内的消息

在事务内调用消息监听器只需要重新配置监听器容器

本地资源事务可以简单地通过监听器容器定义上的`sessionTransacted`标志来激活。然后，每个消息监听器调用将在活动的JMS事务内运行，并在监听器执行失败时回滚消息接收。 发送响应消息(通过`SessionAwareMessageListener`)将是同一本地事务的一部分，但任何其他资源操作(如数据库访问)都将独立运行。这通常需要监听器实现中的重复消息检测，包括数据库处理已提交但消息处理未能提交的情况。

考虑以下bean定义:

```xml
<bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="connectionFactory" ref="connectionFactory"/>
    <property name="destination" ref="destination"/>
    <property name="messageListener" ref="messageListener"/>
    <property name="sessionTransacted" value="true"/>
</bean>
```

要参与外部托管事务，您需要配置事务管理器并使用支持外部托管事务的监听器容器（通常为`DefaultMessageListenerContainer`）。

要为XA事务参与配置消息监听器容器，您需要配置一个`JtaTransactionManager`(默认情况下，它集成到Java EE服务器的事务子系统)。请注意，底层JMS `ConnectionFactory` 需要具有XA能力，并在您的JTA事务协调器中正确注册。 (请检查您的Java EE服务器的JNDI资源配置)。这允许消息接收，犹如数据库访问是同一事务的一部分(使用统一的提交语义，代价是XA事务日志开销)。

以下bean定义创建了一个事务管理器：

```xml
<bean id="transactionManager" class="org.springframework.transaction.jta.JtaTransactionManager"/>
```

然后我们需要将它添加到我们早期的容器配置中。 容器负责其余部分。 以下示例显示了如何执行此操作：

```xml
<bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="connectionFactory" ref="connectionFactory"/>
    <property name="destination" ref="destination"/>
    <property name="messageListener" ref="messageListener"/>
    <property name="transactionManager" ref="transactionManager"/> (1)
</bean>
```

**1**、Our transaction manager.

<a id="jms-jca-message-endpoint-manager"></a>

### [](#jms-jca-message-endpoint-manager)3.4. 用于支持JCA消息端点

从版本2.5开始, Spring还提供对JCA-based `MessageListener`容器的支持。`JmsMessageEndpointManager`将尝试自动从提供程序的`ResourceAdapter`类名称中确定`ActivationSpec`类名。 因此，通常可以只提供Spring的通用`JmsActivationSpecConfig`，如下面的示例所示：

```xml
<bean class="org.springframework.jms.listener.endpoint.JmsMessageEndpointManager">
    <property name="resourceAdapter" ref="resourceAdapter"/>
    <property name="activationSpecConfig">
        <bean class="org.springframework.jms.listener.endpoint.JmsActivationSpecConfig">
            <property name="destinationName" value="myQueue"/>
        </bean>
    </property>
    <property name="messageListener" ref="myMessageListener"/>
</bean>
```

或者，您可以用给定的`ActivationSpec`对象设置一个`JmsMessageEndpointManager`。`ActivationSpec`对象也可能来自JNDI查找（使用`<jee:jndi-lookup>`）。 以下示例显示了如何执行此操作：

```xml
<bean class="org.springframework.jms.listener.endpoint.JmsMessageEndpointManager">
    <property name="resourceAdapter" ref="resourceAdapter"/>
    <property name="activationSpec">
        <bean class="org.apache.activemq.ra.ActiveMQActivationSpec">
            <property name="destination" value="myQueue"/>
            <property name="destinationType" value="javax.jms.Queue"/>
        </bean>
    </property>
    <property name="messageListener" ref="myMessageListener"/>
</bean>
```

使用Spring的`ResourceAdapterFactoryBean`，您可以在本地配置目标`ResourceAdapter`，如以下示例所示：

```xml
<bean id="resourceAdapter" class="org.springframework.jca.support.ResourceAdapterFactoryBean">
    <property name="resourceAdapter">
        <bean class="org.apache.activemq.ra.ActiveMQResourceAdapter">
            <property name="serverUrl" value="tcp://localhost:61616"/>
        </bean>
    </property>
    <property name="workManager">
        <bean class="org.springframework.jca.work.SimpleTaskWorkManager"/>
    </property>
</bean>
```

指定的`WorkManager`也可以指向特定于环境的线程池，通常是通过`SimpleTaskWorkManager`的`asyncTaskExecutor`属性。如果碰巧使用多个适配器，请考虑为所有`ResourceAdapter`实例定义一个共享线程池。

在某些环境(如WebLogic 9+)中，整个`ResourceAdapter`对象可以从JNDI获得，而不是使用`<jee:jndi-lookup>`，然后，Spring消息监听器可与服务器托管的`ResourceAdapter`进行交互，也可以使用服务器的内置`WorkManager`。

有关更多详细信息，请参阅javadoc以获取[`JmsMessageEndpointManager`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/jms/listener/endpoint/JmsMessageEndpointManager.html), [`JmsActivationSpecConfig`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/jms/listener/endpoint/JmsActivationSpecConfig.html), 和 [`ResourceAdapterFactoryBean`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/jca/support/ResourceAdapterFactoryBean.html)

Spring还提供了一个通用的JCA消息端点管理器，它与JMS无关：`org.springframework.jca.endpoint.GenericMessageEndpointManager`。此组件允许使用任何消息监听器类型(如CCI `MessageListener`)和任何提供程序特定的`ActivationSpec`对象，查看您的JCA提供者的文档，了解连接的实际功能，并查阅[`GenericMessageEndpointManager`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/jca/endpoint/GenericMessageEndpointManager.html)的JavaDoc以了解Spring特定的配置详细信息。

JCA-based 消息端点管理非常类似于EJB2.1 消息驱动的bean。它使用相同的基础资源提供程序协定，与EJB2.1 MDB一样，您的JCA提供程序支持的任何消息监听器接口也可以在Spring上下文中使用。 不过，Spring为JMS提供了明确的"方便"支持，这仅仅是因为JMS是与JCA端管理合同一起使用的最常见的端API。

<a id="jms-annotated"></a>

### [](#jms-annotated)3.5. 注解驱动监听器端点

异步接收消息的最简单方法是使用带注解的监听器端点基础架构。简而言之， 它允许您将托管bean的方法作为JMS监听器端公开。以下示例显示了如何使用它：

```java
@Component
public class MyService {

    @JmsListener(destination = "myDestination")
    public void processOrder(String data) { ... }
}
```

前面示例的想法是，只要`javax.jms.Destination` `myDestination`上有消息可用，就会相应地调用`processOrder`方法（在这种情况下，使用JMS消息的内容，类似于[`MessageListenerAdapter`](#jms-receiving-async-message-listener-adapter)提供的内容）。

带注解的端点基础架构使用`JmsListenerContainerFactory`为每个带注解的方法在幕后创建一个消息监听器容器。此类容器不是针对应用程序上下文注册的，但可以很容易地为使用`JmsListenerEndpointRegistry` bean的管理目的而定位。

`@JmsListener`是Java 8上的一个可重复的注解,因此可以通过向它添加附加的@JmsListener声明将多个JMS目标关联到同一方法。

<a id="jms-annotated-support"></a>

#### [](#jms-annotated-support)3.5.1. 允许监听器端点注解

要启用对 `@JmsListener`注解的支持，请将`@EnableJms`添加到您其中一个`@Configuration`类中。

```java
@Configuration
@EnableJms
public class AppConfig {

    @Bean
    public DefaultJmsListenerContainerFactory jmsListenerContainerFactory() {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory());
        factory.setDestinationResolver(destinationResolver());
        factory.setSessionTransacted(true);
        factory.setConcurrency("3-10");
        return factory;
    }
}
```

默认情况下， 基础架构将查找名为`jmsListenerContainerFactory`的bean作用于创建消息监听器工厂的容器来源。在这种情况下， 如果忽略JMS基础架构设置，`processOrder`方法可以设置成3个核心线程和10个最大线程的调用。

可以使用注解自定义监听器容器工厂，或者通过实现`JmsListenerConfigurer`接口来配置显式的默认值。只有在没有特定容器工厂的情况下注册了至少一个端点时，才需要默认值。有关完整的详细信息和示例，请参考[`JmsListenerConfigurer`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/jms/annotation/JmsListenerConfigurer.html)。

如果你倾向于使用 [xml配置](#jms-namespace), 可以使用`<jms:annotation-driven>`。如下：

```xml
<jms:annotation-driven/>

<bean id="jmsListenerContainerFactory"
        class="org.springframework.jms.config.DefaultJmsListenerContainerFactory">
    <property name="connectionFactory" ref="connectionFactory"/>
    <property name="destinationResolver" ref="destinationResolver"/>
    <property name="sessionTransacted" value="true"/>
    <property name="concurrency" value="3-10"/>
</bean>
```

<a id="jms-annotated-programmatic-registration"></a>

#### [](#jms-annotated-programmatic-registration)3.5.2. 编程端点注册

`JmsListenerEndpoint`提供了一个JMS端点的模型，并负责为该模型配置容器。该基础架构使您能够以编程方式配置端点，此外还可以通过`JmsListener`注解检测到的端点。以下示例说明了如何执行此操作：

```java
@Configuration
@EnableJms
public class AppConfig implements JmsListenerConfigurer {

    @Override
    public void configureJmsListeners(JmsListenerEndpointRegistrar registrar) {
        SimpleJmsListenerEndpoint endpoint = new SimpleJmsListenerEndpoint();
        endpoint.setId("myJmsEndpoint");
        endpoint.setDestination("anotherQueue");
        endpoint.setMessageListener(message -> {
            // processing
        });
        registrar.registerEndpoint(endpoint);
    }
}
```

在上面的示例中，我们使用了`SimpleJmsListenerEndpoint`，它提供了实际的`MessageListener`调用，但您也可以构建自己的端变量来描述自定义调用机制。

请注意，您可以完全跳过对`@JmsListener`的使用， 并且只通过编程方式通过 `JmsListenerConfigurer`来注册端点。

<a id="jms-annotated-method-signature"></a>

#### [](#jms-annotated-method-signature)3.5.3. 注解端点方法签名

到目前为止， 我们已经在端点中注入了一个简单的 `String`， 但它实际上可以有一个非常灵活的方法签名。让我们重写它， 用一个自定义的标题注入`Order`：

```java
@Component
public class MyService {

    @JmsListener(destination = "myDestination")
    public void processOrder(Order order, @Header("order_type") String orderType) {
        ...
    }
}
```

这些是可以在JMS监听器端点中注入的主要元素：

*   原始`javax.jms.Message`或其任何子类（假设它与传入的消息类型匹配）。
    
*   用于对本机JMS API进行可选的`javax.jms.Session`，例如发送自定义应答。
    
*   表示传入的JMS消息的`org.springframework.messaging.Message` 。请注意，此消息包含自定义和标准标头(由`JmsHeaders`定义)。
    
*   `@Header`\-annotated 方法参数， 用于提取特定的标头值， 包括标准的JMS标头。
    
*   A `@Headers`\-annotated 方法注解参数，它还必须可赋给 `java.util.Map` 以获得对所有标题的访问权限。
    
*   非注解元素类型（例如`Message` 或`Session`) 是支持的，可以被处理。您可以通过用`@Payload`注解参数来显式使用它，您还可以通过添加额外的`@Valid`来启用验证。
    

注入Spring消息抽象的能力对于从存储在特定于传输的消息中的所有信息中受益， 而不依赖于特定于传输的API，这是非常有用的。 以下示例显示了如何执行此操作：

```
@JmsListener(destination = "myDestination")
public void processOrder(Message<Order> order) { ... }
```

`DefaultMessageHandlerMethodFactory`提供了方法参数的处理，您可以进一步自定义它以支持其他方法参数。 您也可以在那里自定义转换和验证支持。

例如，如果我们想在处理之前确保`Order`有效，我们可以使用 `@Valid`注解负载并配置必要的验证器，如下例所示： For instance, if we want to make sure our is valid before processing it, we can annotate the payload with and configure the necessary validator, as the following example shows:

```java
@Configuration
@EnableJms
public class AppConfig implements JmsListenerConfigurer {

    @Override
    public void configureJmsListeners(JmsListenerEndpointRegistrar registrar) {
        registrar.setMessageHandlerMethodFactory(myJmsHandlerMethodFactory());
    }

    @Bean
    public DefaultMessageHandlerMethodFactory myHandlerMethodFactory() {
        DefaultMessageHandlerMethodFactory factory = new DefaultMessageHandlerMethodFactory();
        factory.setValidator(myValidator());
        return factory;
    }
}
```

<a id="jms-annotated-response"></a>

#### [](#jms-annotated-response)3.5.4. 响应管理

[`MessageListenerAdapter`](#jms-receiving-async-message-listener-adapter)中的现有支持已经允许您的方法具有`void`的返回类型。在这种情况下，调用的结果将封装在原始消息的`JMSReplyTo`标头中指定的目的地中， 或者在监听器上配置的默认目的地`javax.jms.Message`中发送。现在可以使用消息抽象的`@SendTo`注解设置该默认目的地。

假设我们的`processOrder`方法现在应该返回一个`OrderStatus`， 那么可以按如下方式编写它以自动发送响应。

```java
@JmsListener(destination = "myDestination")
@SendTo("status")
public OrderStatus processOrder(Order order) {
    // order processing
    return status;
}
```

如果您有几个 `@JmsListener`注解的方法， 还可以将`@SendTo`注解放在类级别以共享默认的答复目的地。

如果需要以与传输无关的方式设置其他管理， 则可以返回 `Message`。 如下所示:

```java
@JmsListener(destination = "myDestination")
@SendTo("status")
public Message<OrderStatus> processOrder(Order order) {
    // order processing
    return MessageBuilder
            .withPayload(status)
            .setHeader("code", 1234)
            .build();
}
```

如果需要在运行时计算响应目的地， 则可以将响应封装在`JmsResponse`实例中， 它还提供在运行时使用的目的地。前面的示例可以重写如下:

```java
@JmsListener(destination = "myDestination")
public JmsResponse<Message<OrderStatus>> processOrder(Order order) {
    // order processing
    Message<OrderStatus> response = MessageBuilder
            .withPayload(status)
            .setHeader("code", 1234)
            .build();
    return JmsResponse.forQueue(response, "status");
}
```

最后，如果您需要为响应指定一些QoS值（例如优先级或生存时间），则可以相应地配置`JmsListenerContainerFactory`，如以下示例所示：

```java
@Configuration
@EnableJms
public class AppConfig {

    @Bean
    public DefaultJmsListenerContainerFactory jmsListenerContainerFactory() {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory());
        QosSettings replyQosSettings = new QosSettings();
        replyQosSettings.setPriority(2);
        replyQosSettings.setTimeToLive(10000);
        factory.setReplyQosSettings(replyQosSettings);
        return factory;
    }
}
```

<a id="jms-namespace"></a>

### [](#jms-namespace)3.6. JMS命名空间支持

Spring提供了一个用于简化JMS配置的XML命名空间。要使用JMS命名空间元素， 您需要引用JMS schema，如以下示例所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:jms="http://www.springframework.org/schema/jms" (1)
        xsi:schemaLocation="
            http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/jms http://www.springframework.org/schema/jms/spring-jms.xsd">

    <!-- bean definitions here -->

</beans>
```

**1**、Referencing the JMS schema.

命名空间包含三个顶级的元素: `<annotation-driven/>`, `<listener-container/>` 和 `<jca-listener-container/>`. `<annotation-driven/>` 允许使用[注解驱动的监听器端点](#jms-annotated). `<listener-container/>` 和 `<jca-listener-container/>` 定义了共享的监听器容器配置，并且可能包含`<listener/>`子元素 。下面是两个监听器的基本配置示例：

```xml
<jms:listener-container>

    <jms:listener destination="queue.orders" ref="orderService" method="placeOrder"/>

    <jms:listener destination="queue.confirmations" ref="confirmationLogger" method="log"/>

</jms:listener-container>
```

上面的示例等效于创建两个不同的监听器容器bean和两个不同的`MessageListenerAdapter` bean定义。，如[使用 `MessageListenerAdapter`](#jms-receiving-async-message-listener-adapter)中所示。 除了前面示例中显示的属性之外，`listener`元素还可以包含几个可选的属性。 下表描述了所有可用属性：

Table 3. Attributes of the JMS <listener> element  

|   属性   | 描述     |
| ---- | ---- |
|   `id`   |   监听器容器的Bean名称。 如果未指定，则自动生成bean名称。   |
|  `destination` (required)    |  此监听器的目的地，通过`DestinationResolver`策略解析。    |
|   `ref` (required)   | 处理程序对象的bean名称。     |
|   `method`   |  要调用的处理程序方法的名称。 如果`ref`属性指向`MessageListener`或Spring `SessionAwareMessageListener`，则可以省略此属性。    |
|    `response-destination`   | 要将响应消息发送到的默认响应目标的名称。 这适用于请求消息未携带 `JMSReplyTo`字段的情况。 此目标的类型由监听器容器的`response-destination-type`属性确定。     |
|  `subscription`    |  持久性订阅的名称（如果有）。    |
|   `selector`   |  此监听器的可选消息选择器。    |
|    `concurrency`   |   为此监听器启动的sessions 或consumers数量，可以是指示最大数量的一个简单的数字（例如，`5`），也可以是指示下限和上限的范围（例如， `3-5`）。 请注意，指定的最小值只是一个提示，可能在运行时被忽略。 默认值是容器提供的值。   |


`<listener-container/>` 元素也接受几个可选属性。这允许自定义各种策略(例如`taskExecutor` 和 `destinationResolver`)，以及基本的JMS设置和资源引用。使用这些属性，可以实现高度自定义的监听器容器，同时仍可从命名空间的方便性中获益。

通过指定要通过`factory-id`属性公开的bean `id`，可以自动将此类设置公开为`JmsListenerContainerFactory`。如下所示：

```xml
<jms:listener-container connection-factory="myConnectionFactory"
        task-executor="myTaskExecutor"
        destination-resolver="myDestinationResolver"
        transaction-manager="myTransactionManager"
        concurrency="10">

    <jms:listener destination="queue.orders" ref="orderService" method="placeOrder"/>

    <jms:listener destination="queue.confirmations" ref="confirmationLogger" method="log"/>

</jms:listener-container>
```

下表描述了所有可用属性。有关各个属性的详细信息， 请参阅[`AbstractMessageListenerContainer`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/jms/listener/AbstractMessageListenerContainer.html)及其具体子类的 javadocs。javadocs还提供了有关事务选择和消息重整方案的讨论。

Table 4. Attributes of the JMS <listener-container> element  

|  属性    | 描述     |
| ---- | ---- |
| `container-type`     |  此监听器容器的类型。 可用选项为`default`, `simple`, `default102`, 或 `simple102`（默认选项为 `default`）。    |
|   `container-class`   |  自定义监听器容器实现类作为完全限定的类名。根据`container-type`属性，默认值为Spring的标准`DefaultMessageListenerContainer`或`SimpleMessageListenerContainer`。    |
|  `factory-id`    |  将此元素定义的设置公开为具有指定 `id` 的 `JmsListenerContainerFactory`，以便它们可以与其他端点一起使用。    |
|   `connection-factory`   |   对JMS `ConnectionFactory` bean的引用（默认bean名称为`connectionFactory`）。   |
|   `task-executor`   |    对JMS监听器调用者的Spring `TaskExecutor`的引用。  |
|   `destination-resolver`   |   对`DestinationResolver`策略的引用，用于解析JMS `Destination`实例。   |
|   `message-converter`   |  对用于将JMS消息转换为监听器方法参数的`MessageConverter`策略的引用。 默认值为`SimpleMessageConverter`。    |
|   `error-handler`   |  对`ErrorHandler`策略的引用，用于处理在执行`MessageListener`期间可能发生的任何未捕获的异常。    |
|   `destination-type`	|   此监听器的JMS目标类型：`queue`, `topic`, `durableTopic`, `sharedTopic`或 `sharedDurableTopic`。 这可能会启用容器的`pubSubDomain`, `subscriptionDurable`和`subscriptionShared`属性。 默认值为`queue`（禁用这三个属性）。	|
|`response-destination-type`	|响应的JMS目标类型：`queue` or `topic`。 默认值是`destination-type` 属性的值。	|
|`client-id`	|此监听器容器的JMS客户端ID。 您必须在使用持久订阅时指定它。	|
|`cache`	|JMS资源的缓存级别：`none`, `connection`, `session`, `consumer`或`auto`。 默认情况下（`auto`），缓存级别实际上是`consumer`，除非指定了外部事务管理器 - 在这种情况下，有效默认值为`none` （假设Java EE样式的事务管理，其中给定的ConnectionFactory是XA感知的池）。	|
|  `acknowledge`   | 本机JMS确认模式：`auto`, `client`, `dups-ok`或`transacted`。 `transacted`激活本地`transacted`的会话。 作为替代方法，您可以指定`transaction-manager`属性，稍后将在表中进行说明。 默认为`auto`。     |
|  `transaction-manager`   |   对外部`PlatformTransactionManager`的引用（通常是基于XA的事务协调器，例如Spring的`JtaTransactionManager`）。 如果未指定，则使用本机确认（请参阅 `acknowledge` 属性）。   |
|  `concurrency`   |  为每个监听器启动的并发会话或使用者数。 它可以是指示最大数量的简单数字（例如，`5`），也可以是指示下限和上限的范围（例如，`3-5`）。 请注意，指定的最小值只是一个提示，可能在运行时被忽略。 默认值为 `1`.如果是主题监听器或队列排序很重要，则应将并发限制为 `1`。 考虑将其提升为一般队列。    |
|  `prefetch`   |  要加载到单个会话中的最大消息数。 请注意，提高此数字可能会导致并发消费者的饥饿。    |
|   `receive-timeout`  |   用于接收呼叫的超时（以毫秒为单位）。 默认值为`1000`（一秒）。 `-1`表示没有超时。  |
|   `back-off`   |    指定用于计算恢复尝试之间间隔的`BackOff`实例。 如果 `BackOffExecution`实现返回 `BackOffExecution#STOP`，则监听器容器不会进一步尝试恢复。 设置此属性时，将忽略`recovery-interval`值。 默认值为`FixedBackOff`，间隔为5000毫秒（即5秒）。   |
|  `recovery-interval`    |   指定恢复尝试之间的间隔（以毫秒为单位）。 它提供了一种使用指定间隔创建`FixedBackOff`的便捷方法。 有关更多恢复选项，请考虑指定`BackOff` 实例。 默认值为5000毫秒（即5秒）。    |
|   `phase`   |  此容器应开始和停止的生命周期阶段。 值越低，此容器启动越早，后者停止。 默认值为`Integer.MAX_VALUE`，表示容器尽可能晚启动并尽快停止。     |


使用jms模式支持配置基于JCA的监听器容器非常相似，如以下示例所示：

```xml
<jms:jca-listener-container resource-adapter="myResourceAdapter"
        destination-resolver="myDestinationResolver"
        transaction-manager="myTransactionManager"
        concurrency="10">

    <jms:listener destination="queue.orders" ref="myMessageListener"/>

</jms:jca-listener-container>
```

下表描述了JCA变量的可用配置选项:

Table 5. Attributes of the JMS <jca-listener-container/> element  

|      属性 |描述      |
| ---- | ---- |
|   `factory-id`   |   将此元素定义的设置公开为具有指定`id`的 `JmsListenerContainerFactory`，以便它们可以与其他端点一起使用。   |
|   `resource-adapter`   |  对JCA `ResourceAdapter` bean的引用（默认bean名称为`resourceAdapter`）。    |
|  `activation-spec-factory`    |  对`JmsActivationSpecFactory`的引用。 缺省情况是自动检测JMS提供程序及其`ActivationSpec` 类（请参阅[`DefaultJmsActivationSpecFactory`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/jms/listener/endpoint/DefaultJmsActivationSpecFactory.html)）。    |
|  `destination-resolver`    |  对`DestinationResolver`策略的引用，用于解析JMS `Destinations`。    |
|  `message-converter`    |  对用于将JMS消息转换为监听器方法参数的`MessageConverter`策略的引用。 默认值为`SimpleMessageConverter`。    |
|   `destination-type`   |  此监听器的JMS目标类型：`queue`, `topic`, `durableTopic`, `sharedTopic`或`sharedDurableTopic`。 这可能会启用容器的`pubSubDomain`, `subscriptionDurable`和`subscriptionShared`属性。 默认值为`queue` （禁用这三个属性）。    |
|  `response-destination-type`    | 响应的JMS目标类型：`queue` or `topic`。 默认值是`destination-type`属性的值。     |
|  `client-id`    |  此监听器容器的JMS客户端ID。 在使用持久订阅时需要指定它。    |
|  `acknowledge`    |  本机JMS确认模式：`auto`, `client`, `dups-ok`, or `transacted`。 `transacted`激活本地`transacted`的会话。 或者，您可以指定稍后描述的 `transaction-manager` 属性。 默认为`auto`。    |
| `transaction-manager`     |   对Spring `JtaTransactionManager`或`javax.transaction.TransactionManager` 的引用，用于为每个传入消息启动XA事务。 如果未指定，则使用本机确认（请参阅`acknowledge` 属性）。   |
|    `concurrency`  |   为每个监听器启动的并发会话或使用者数。 它可以是指示最大数量的简单数字（例如`5`）或指示下限和上限的范围（例如，`3-5`）。 请注意，指定的最小值只是一个提示，在运行时通常会在使用JCA监听器容器时被忽略。 默认值为1。 |
| `prefetch`     |   要加载到单个会话中的最大消息数。 请注意，提高此数字可能会导致并发消费者的饥饿。   |