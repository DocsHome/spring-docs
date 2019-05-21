<a id="remoting"></a>

[](#remoting)1\. Spring的远程处理和Web服务
----------------------------------

Spring为使用各种技术的远程支持的集成类提供了功能，远程处理支持可轻松使用常用(Spring)pojo实现的、由您提供的服务开发。目前，Spring支持以下远程技术：

*   **远程方法调用(RMI)**:通过使用`RmiProxyFactoryBean`和`RmiServiceExporter`。Spring支持传统的RMI（使用 `java.rmi.Remote`接口和`java.rmi.RemoteException`）以及通过RMI调用程序（使用任何Java接口）进行透明的远程处理。
    
*   **Spring的 HTTP 调用**: Spring提供了一种特殊的远程处理策略，允许通过HTTP进行Java序列化，支持任何Java接口（如RMI调用者所做的那样）。 相应的支持类是`HttpInvokerProxyFactoryBean`和`HttpInvokerServiceExporter`.
    
*   **Hessian**:通过使用Spring的`HessianProxyFactoryBean` 和`HessianServiceExporter`，您可以使用Caucho提供的轻量级二进制http协议透明地公开您的服务。
    
*   **JAX-WS**: Spring通过JAX-WS 为Web服务提供了远程处理支持(如Java EE 5和Java 6所介绍的那样， 它继承了JAX-RPC).
    
*   **JMS**: 通过`JmsInvokerServiceExporter` 和 `JmsInvokerProxyFactoryBean`类支持使用JMS作为底层协议进行远程处理。
    
*   **AMQP**:Spring AMQP项目支持使用AMQP作为底层协议进行远程处理。

在讨论Spring的远程处理功能时，我们使用以下域模型和相应的服务：

```java
public class Account implements Serializable{

    private String name;

    public String getName(){
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

}

public interface AccountService {

    public void insertAccount(Account account);

    public List<Account> getAccounts(String name);

}

// the implementation doing nothing at the moment
public class AccountServiceImpl implements AccountService {

    public void insertAccount(Account acc) {
        // do something...
    }

    public List<Account> getAccounts(String name) {
        // do something...
    }

}
```

```java

```

本节首先使用RMI将服务公开给远程客户端，然后再谈谈使用RMI的缺点。 然后继续使用Hessian作为协议的示例。

<a id="remoting-rmi"></a>

### [](#remoting-rmi)1.1. 使用RMI公开服务

使用Spring对RMI的支持， 您可以通过RMI架构透明地公开服务。在用此设置之后， 您基本上拥有一个类似于远程EJB的配置， 但不存在安全上下文传播或远程事务传播的标准支持这一事实。Spring在使用RMI调用器时提供了这样的附加调用上下文的钩子， 因此您可以在此处插入安全框架或自定义安全凭据。

<a id="remoting-rmi-server"></a>

#### [](#remoting-rmi-server)1.1.1. 使用`RmiServiceExporter`暴露服务

使用`RmiServiceExporter`，我们可以将AccountService对象的接口公开为RMI对象。可以使用`RmiProxyFactoryBean`访问该接口，或者在传统RMI服务的情况下通过普通RMI访问该接口。 `RmiServiceExporter`明确支持通过RMI调用程序公开任何非RMI服务。

我们首先必须在Spring容器中设置我们的服务。 以下示例显示了如何执行此操作：

```xml
<bean id="accountService" class="example.AccountServiceImpl">
    <!-- any additional properties, maybe a DAO? -->
</bean>
```

接下来，我们必须使用`RmiServiceExporter`暴露我们的服务。 以下示例显示了如何执行此操作：

```xml
<bean class="org.springframework.remoting.rmi.RmiServiceExporter">
    <!-- does not necessarily have to be the same name as the bean to be exported -->
    <property name="serviceName" value="AccountService"/>
    <property name="service" ref="accountService"/>
    <property name="serviceInterface" value="example.AccountService"/>
    <!-- defaults to 1099 -->
    <property name="registryPort" value="1199"/>
</bean>
```

在前面的示例中，我们覆盖RMI注册表的端口。通常情况下， 您的应用服务器还维护一个RMI注册表， 因此最好不要干预它。此外， 服务名称用于绑定服务。因此，在前面的示例中，服务绑定在 `'rmi://HOST:1199/AccountService'`。 我们稍后使用此URL链接客户端的服务。 .

`servicePort` 属性已被省略（默认为0）。这意味着将使用匿名端口与服务进行通信。

<a id="remoting-rmi-client"></a>

#### [](#remoting-rmi-client)1.1.2. 连接客户端和服务

我们的客户端是一个使用 `AccountService`管理帐户的简单对象，如以下示例所示：

```java
public class SimpleObject {

    private AccountService accountService;

    public void setAccountService(AccountService accountService) {
        this.accountService = accountService;
    }

    // additional methods using the accountService

}
```

要在客户端上链接服务， 我们将创建一个单独的Spring容器， 其中包含简单对象和连接配置位的服务。

```xml
<bean class="example.SimpleObject">
    <property name="accountService" ref="accountService"/>
</bean>

<bean id="accountService" class="org.springframework.remoting.rmi.RmiProxyFactoryBean">
    <property name="serviceUrl" value="rmi://HOST:1199/AccountService"/>
    <property name="serviceInterface" value="example.AccountService"/>
</bean>
```

这就是支持客户端连接远程帐户服务所需做的全部工作，Spring将透明地创建一个调用， 并通过 `RmiServiceExporter`远程启用帐户服务。在客户端， 我们在其链接中使用 `RmiProxyFactoryBean`。

<a id="remoting-caucho-protocols"></a>

### [](#remoting-caucho-protocols)1.2. 使用Hessian通过HTTP远程调用服务

Hessian提供基于HTTP的二进制远程协议。 它由Caucho开发，您可以在 [http://www.caucho.com](http://www.caucho.com)找到有关Hessian本身的更多信息。

<a id="remoting-caucho-protocols-hessian"></a>

#### [](#remoting-caucho-protocols-hessian)1.2.1.使用`DispatcherServlet` 连接Hessian

Hessian通过HTTP进行通信，并使用自定义servlet进行通信。通过使用Spring的`DispatcherServlet` 原则（参见[\[webmvc#mvc-servlet\]](webmvc.html#mvc-servlet)），我们可以连接这样的servlet以公开您的服务。 首先，我们必须在我们的应用程序中创建一个新的servlet，如以下摘自`web.xml`所示:

```xml
<servlet>
    <servlet-name>remoting</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>remoting</servlet-name>
    <url-pattern>/remoting/*</url-pattern>
</servlet-mapping>
```

如果您熟悉Spring的`DispatcherServlet`原则，您可能知道现在必须在 `WEB-INF`目录中创建一个名为`remoting-servlet.xml`的Spring容器配置资源（在您的servlet名称之后）。 应用程序上下文将在下一节中使用。

或者， 考虑使用Spring更简单的`HttpRequestHandlerServlet`。这使您可以在根应用程序上下文中嵌入远程公开定义(默认情况下为`WEB-INF/applicationContext.xml`)， 并将单个Servlet定义指向特定的公开bean。在这种情况下， 每个Servlet名称都需要匹配其目标公开的bean名称。

<a id="remoting-caucho-protocols-hessian-server"></a>

#### [](#remoting-caucho-protocols-hessian-server)1.2.2. 使用`HessianServiceExporter`暴露您的Bean

在新创建的名为 `remoting-servlet.xml`的应用程序上下文中，我们创建了一个`HessianServiceExporter` 来暴露我们的服务，如下例所示:

```xml
<bean id="accountService" class="example.AccountServiceImpl">
    <!-- any additional properties, maybe a DAO? -->
</bean>

<bean name="/AccountService" class="org.springframework.remoting.caucho.HessianServiceExporter">
    <property name="service" ref="accountService"/>
    <property name="serviceInterface" value="example.AccountService"/>
</bean>
```

现在, 我们已经准备好在客户端上链接服务了.未指定显式处理程序映射， 将请求URL映射到服务上， 因此将使用`BeanNameUrlHandlerMapping`。 因此， 该服务将在包含 `DispatcherServlet`映射(如上所述)的URL中通过其bean名称指示的网址公开:`[http://HOST:8080/remoting/AccountService](http://HOST:8080/remoting/AccountService)`

或者, 在您的根应用程序上下文中创建一个`HessianServiceExporter`(例如在 `WEB-INF/applicationContext.xml`)。如下所示:

```xml
<bean name="accountExporter" class="org.springframework.remoting.caucho.HessianServiceExporter">
    <property name="service" ref="accountService"/>
    <property name="serviceInterface" value="example.AccountService"/>
</bean>
```

在后一种情况下， 在`web.xml`中为该公开定义一个相应的Servlet， 并使用相同的最终结果： 公开程序被映射到请求路径`/remoting/AccountService`。请注意， Servlet名称需要与目标公开方的bean名称匹配。

```xml
<servlet>
    <servlet-name>accountExporter</servlet-name>
    <servlet-class>org.springframework.web.context.support.HttpRequestHandlerServlet</servlet-class>
</servlet>

<servlet-mapping>
    <servlet-name>accountExporter</servlet-name>
    <url-pattern>/remoting/AccountService</url-pattern>
</servlet-mapping>
```

<a id="remoting-caucho-protocols-hessian-clien"></a>

#### [](#remoting-caucho-protocols-hessian-client)1.2.3. 在客户端的服务中链接

通过使用`HessianProxyFactoryBean`，我们可以链接客户端的服务。同样的原则适用于RMI示例。我们创建一个单独的bean工厂或应用程序上下文，并提到以下bean，其中`SimpleObject`是通过使用`AccountService`来管理帐户，如以下示例所示：:

```xml
<bean class="example.SimpleObject">
    <property name="accountService" ref="accountService"/>
</bean>

<bean id="accountService" class="org.springframework.remoting.caucho.HessianProxyFactoryBean">
    <property name="serviceUrl" value="http://remotehost:8080/remoting/AccountService"/>
    <property name="serviceInterface" value="example.AccountService"/>
</bean>
```

<a id="remoting-caucho-protocols-security"></a>

#### [](#remoting-caucho-protocols-security)1.2.4. 通过Hessian公开的服务来应用HTTP基本的验证

Hessian的一个优点是我们可以轻松应用HTTP基本身份验证，因为这两种协议都是基于HTTP的。例如，可以通过使用 `web.xml`安全功能来应用常规HTTP服务器安全性机制。 通常，您无需在此处使用每用户安全凭据。 相反，您可以使用在`HessianProxyFactoryBean`级别定义的共享凭证（类似于JDBC `DataSource`），如以下示例所示:

```xml
<bean class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping">
    <property name="interceptors" ref="authorizationInterceptor"/>
</bean>

<bean id="authorizationInterceptor"
        class="org.springframework.web.servlet.handler.UserRoleAuthorizationInterceptor">
    <property name="authorizedRoles" value="administrator,operator"/>
</bean>
```

在前面的示例中，我们明确提到了 `BeanNameUrlHandlerMapping`并设置了一个拦截器，只允许管理员和操作员调用此应用程序上下文中提到的bean。

前面的示例并没有显示一种灵活的安全基础结构。有关安全性的更多选项，请查看[http://projects.spring.io/spring-security/](https://projects.spring.io/spring-security/)上的Spring Security项目。

<a id="remoting-httpinvoker"></a>

### [](#remoting-httpinvoker)1.3. 使用HTTP调用公开服务

与Hessian相反，Spring HTTP调用者都是轻量级协议，Hessian使用自身序列化机制，而Spring HTTP调用者使用标准的Java序列化机制通过HTTP公开服务。如果您的参数和返回类型是使用Hessian使用的序列化机制无法序列化的复杂类型，那么这具有巨大的优势（当您选择远程处理技术时，请参阅下一节以了解更多注意事项）。

在底层，Spring使用JDK或Apache `HttpComponents`提供的标准工具来执行HTTP调用。 如果您需要更高级且更易于使用的功能，请使用后者。 有关更多信息，请参阅[hc.apache.org/httpcomponents-client-ga/](https://hc.apache.org/httpcomponents-client-ga/)。

由于不安全的Java反序列化而造成的漏洞:操作的输入流可能导致在反序列化步骤中在服务器上执行不需要的代码。因此， 不要将HTTP调用方终结点公开给不受信任的客户端， 而只应该在您自己的服务之间。通常， 我们强烈推荐任何其他消息格式(例如JSON)。

如果您担心由于Java序列化引起的安全漏洞， 请考虑核心JVM级别的generalpurpose序列化筛选器机制， 它最初是为JDK 9开发的。但同时又向后移植到JDK 8,7和6。 请参阅[https://blogs.oracle.com/java-platform-group/entry/incoming\_filter\_serialization\_data\_a](https://blogs.oracle.com/java-platform-group/entry/incoming_filter_serialization_data_a) 和 [http://openjdk.java.net/jeps/290](http://openjdk.java.net/jeps/290).

<a id="remoting-httpinvoker-server"></a>

#### [](#remoting-httpinvoker-server)1.3.1. 公开服务对象

为服务对象设置HTTP调用的架构类似于使用Hession，方式也是相同的。正如Hession支持提供 是`HessianServiceExporter`。 Spring的HttpInvoker支持提供了`org.springframework.remoting.httpinvoker.HttpInvokerServiceExporter`。

要在Spring Web MVC `DispatcherServlet`中暴露`AccountService`（前面提到过），需要在调度程序的应用程序上下文中使用以下配置，如以下示例所示：:

```xml
<bean name="/AccountService" class="org.springframework.remoting.httpinvoker.HttpInvokerServiceExporter">
    <property name="service" ref="accountService"/>
    <property name="serviceInterface" value="example.AccountService"/>
</bean>
```

这样一个公开的定义将通过 `DispatcherServlet`的标准映射设备， 如在关于[Hession的章节](#remoting-caucho-protocols)中解释。

或者，您可以在根应用程序上下文中创建`HttpInvokerServiceExporter`（例如，在`'WEB-INF/applicationContext.xml'`中），如以下示例所示:

```xml
<bean name="accountExporter" class="org.springframework.remoting.httpinvoker.HttpInvokerServiceExporter">
    <property name="service" ref="accountService"/>
    <property name="serviceInterface" value="example.AccountService"/>
</bean>
```

此外，在`web.xml`中为该公开定义一个对应的Servlet， 其Servlet名称与目标公开的bean名称相匹配。如下所示:

```xml
<servlet>
    <servlet-name>accountExporter</servlet-name>
    <servlet-class>org.springframework.web.context.support.HttpRequestHandlerServlet</servlet-class>
</servlet>

<servlet-mapping>
    <servlet-name>accountExporter</servlet-name>
    <url-pattern>/remoting/AccountService</url-pattern>
</servlet-mapping>
```

<a id="remoting-httpinvoker-client"></a>

#### [](#remoting-httpinvoker-client)1.3.2. 在客户端链接服务

同样，从客户端链接服务非常类似于使用Hessian时的方式。通过使用代理，Spring可以将对HTTP POST请求的调用转换为指向公开服务的URL。以下示例显示如何配置:

```xml
<bean id="httpInvokerProxy" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
    <property name="serviceUrl" value="http://remotehost:8080/remoting/AccountService"/>
    <property name="serviceInterface" value="example.AccountService"/>
</bean>
```

如前所述，您可以选择要使用的HTTP客户端。默认情况下，`HttpInvokerProxy`使用JDK的HTTP功能，但您也可以通过设置 `httpInvokerRequestExecutor`属性来使用Apache `HttpComponents`客户端。以下示例显示了如何执行此操作:

```xml
<property name="httpInvokerRequestExecutor">
    <bean class="org.springframework.remoting.httpinvoker.HttpComponentsHttpInvokerRequestExecutor"/>
</property>
```

<a id="remoting-web-services"></a>

### [](#remoting-web-services)1.4. Web Services

Spring提供对标准Java Web服务API的完全支持:

*   使用JAX-WS暴露服务
    
*   使用JAX-WS访问Web服务
    

除了在Spring的核心中支持JAX-WS外，Spring的portfolio也具有Spring Web服务的特点。这是一种以文档驱动的[Spring Web服务](http://www.springframework.org/spring-ws)为基础的解决方案， 它极力推荐用于构建现代的、经得起未来考验的Web服务。

<a id="remoting-web-services-jaxws-export-servlet"></a>

#### [](#remoting-web-services-jaxws-export-servlet)1.4.1. 使用JAX-WS公开基于Servlet的Web服务

Spring为JAX-WS Servlet端实现提供了一个方便的基类- `SpringBeanAutowiringSupport`。为了公开我们的`AccountService`， 我们继承了Spring的`SpringBeanAutowiringSupport`类并在这里实现我们的业务逻辑， 通常是将调用委托给业务层。我们将简单地使用spring的`@Autowired`注解来表达对Spring管理bean的这种依赖性。以下示例显示了扩展`SpringBeanAutowiringSupport`的类:

```java
/**
 * JAX-WS compliant AccountService implementation that simply delegates
 * to the AccountService implementation in the root web application context.
 *
 * This wrapper class is necessary because JAX-WS requires working with dedicated
 * endpoint classes. If an existing service needs to be exported, a wrapper that
 * extends SpringBeanAutowiringSupport for simple Spring bean autowiring (through
 * the @Autowired annotation) is the simplest JAX-WS compliant way.
 *
 * This is the class registered with the server-side JAX-WS implementation.
 * In the case of a Java EE 5 server, this would simply be defined as a servlet
 * in web.xml, with the server detecting that this is a JAX-WS endpoint and reacting
 * accordingly. The servlet name usually needs to match the specified WS service name.
 *
 * The web service engine manages the lifecycle of instances of this class.
 * Spring bean references will just be wired in here.
 */
import org.springframework.web.context.support.SpringBeanAutowiringSupport;

@WebService(serviceName="AccountService")
public class AccountServiceEndpoint extends SpringBeanAutowiringSupport {

    @Autowired
    private AccountService biz;

    @WebMethod
    public void insertAccount(Account acc) {
        biz.insertAccount(acc);
    }

    @WebMethod
    public Account[] getAccounts(String name) {
        return biz.getAccounts(name);
    }

}
```

我们的`AccountServiceEndpoint`需要在与Spring上下文相同的Web应用程序中运行，以允许访问Spring的工具。 默认情况下，在Java EE 5环境中使用JAX-WS servlet端点部署的标准协定就是这种情况。 有关详细信息，请参阅各种Java EE 5 Web服务教程。

<a id="remoting-web-services-jaxws-export-standalone"></a>

#### [](#remoting-web-services-jaxws-export-standalone)1.4.2. 使用JAX-WS公开独立Web服务

Oracle JDK附带的内置JAX-WS提供程序通过使用JDK中包含的内置HTTP服务器支持Web服务的公开。Spring的`SimpleJaxWsServiceExporter`在Spring应用程序上下文中检测所有`@WebService`注解的bean，并通过默认的JAX-WS服务器（JDK HTTP服务器）公开它们。

在这种情况下， 端实例被定义并作为Spring bean来管理。它们将在JAX-WS引擎中注册， 但它们的生命周期将由Spring应用程序上下文来实现。这意味着像显式依赖注入这样的Spring功能可以应用到端点实例。 当然， 通过`@Autowired`的注解驱动的注入也会起作用。以下示例显示了如何定义这些bean:

```xml
<bean class="org.springframework.remoting.jaxws.SimpleJaxWsServiceExporter">
    <property name="baseAddress" value="http://localhost:8080/"/>
</bean>

<bean id="accountServiceEndpoint" class="example.AccountServiceEndpoint">
    ...
</bean>

...
```

`AccountServiceEndpoint`可以不用继承Spring的`SpringBeanAutowiringSupport`，因为此示例中的端点是完全由Spring管理的bean。 这意味着端点实现可以如下（没有声明任何超类 - 而且Spring的`@Autowired`配置注解仍然被可用）:

```java
@WebService(serviceName="AccountService")
public class AccountServiceEndpoint {

    @Autowired
    private AccountService biz;

    @WebMethod
    public void insertAccount(Account acc) {
        biz.insertAccount(acc);
    }

    @WebMethod
    public List<Account> getAccounts(String name) {
        return biz.getAccounts(name);
    }

}
```

<a id="remoting-web-services-jaxws-export-ri"></a>

#### [](#remoting-web-services-jaxws-export-ri)1.4.3. 使用JAX-WS RI的Spring支持公开Web服务

Oracle的JAX-WS RI, 作为GlassFish项目的一部分，通过Spring的支持也作为JAX-WS Commons项目的一部分。这允许将 JAX-WS端点定义为Spring管理bean， 类似于[上一节](#remoting-web-services-jaxws-export-standalone)中讨论的独立模式。 但这一次是在Servlet环境中进行的

这在Java EE 5环境中不可移植。 它主要用于非EE环境，例如Tomcat，它将JAX-WS RI作为Web应用程序的一部分嵌入。

与Servlet端的标准样式的不同之处在于，端实例本身的生命周期将由Spring来管理，并且在`web.xml`中将只定义一个JAX-WS Servlet。使用标准的Java EE 5样式 (如上所述)， 每个服务端都有一个Servlet定义， 每个端通常委派给Spring bean(通过使用`@Autowired`，如上所示)。

有关设置和使用方式的详细信息，请参阅[https://jax-ws-commons.java.net/spring/](https://jax-ws-commons.java.net/spring/)。

<a id="remoting-web-services-jaxws-access"></a>

#### [](#remoting-web-services-jaxws-access)1.4.4. 使用JAX-WS访问Web服务

Spring提供了两个工厂bean来创建JAX-WS Web服务代理，即`LocalJaxWsServiceFactoryBean` 和 `JaxWsPortProxyFactoryBean`。前者只能返回一个JAX-WS服务类供我们使用。后者是完整版本，可以返回实现我们的业务服务接口的代理。在以下示例中，我们使用`JaxWsPortProxyFactoryBean`为`AccountService`端点创建代理（再次）：:

```xml
<bean id="accountWebService" class="org.springframework.remoting.jaxws.JaxWsPortProxyFactoryBean">
    <property name="serviceInterface" value="example.AccountService"/> (1)
    <property name="wsdlDocumentUrl" value="http://localhost:8888/AccountServiceEndpoint?WSDL"/>
    <property name="namespaceUri" value="http://example/"/>
    <property name="serviceName" value="AccountService"/>
    <property name="portName" value="AccountServiceEndpointPort"/>
</bean>
```

**1**、其中`serviceInterface` 是客户端使用的业务接口。

`wsdlDocumentUrl`是WSDL文件的URL。Spring需要这样的启动时间来创建 JAX-WS服务。`namespaceUri`对应于.wsdl文件中的`targetNamespace` 。`serviceName`对应于.wsdl文件中的服务名称。 `portName`对应于.wsdl文件中的端口名称。

访问Web服务现在非常容易， 因为我们有一个bean工厂， 它将公开它作为`AccountService`接口。我们可以在Spring中配置:

```xml
<bean id="client" class="example.AccountClientImpl">
    ...
    <property name="service" ref="accountWebService"/>
</bean>
```

从客户端代码中， 我们可以访问Web服务，就好像它是一个普通类一样:

```java
public class AccountClientImpl {

    private AccountService service;

    public void setService(AccountService service) {
        this.service = service;
    }

    public void foo() {
        service.insertAccount(...);
    }
}
```

上述情况略有简化， 因为JAX-WS需要端点接口和实现类来注解`@WebService`、`@SOAPBinding`等注解。这意味着您不能(很容易)使用普通的Java接口和实现类作为JAX-WS端点工件；首先您需要相应地对它们进行注解。 有关这些要求的详细信息 请参阅JAX-WS文档。

<a id="remoting-jms"></a>

### [](#remoting-jms)1.5. 通过JMS公开服务

您还可以使用JMS作为底层通信协议透明地公开服务。 JMS的远程支持在Spring Framework中是非常基础。它在同一个线程和同一个非事务性会话中发送和接收。 并且这样的吞吐量将非常依赖于实现， 请注意， 这些单线程和非事务性约束仅适用于Spring的JMS远程处理支持。有关Spring对JMS消息传递的丰富支持的信息， 请参见[JMS (Java Message Service)](#jms)

以下接口用于服务器端和客户端:

```java
package com.foo;

public interface CheckingAccountService {

    public void cancelAccount(Long accountId);

}
```

在服务器端使用上述接口的以下简单实现:

```java
package com.foo;

public class SimpleCheckingAccountService implements CheckingAccountService {

    public void cancelAccount(Long accountId) {
        System.out.println("Cancelling account [" + accountId + "]");
    }

}
```

此配置文件包含在客户端和服务器上共享的JMS基础结构bean:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="connectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
        <property name="brokerURL" value="tcp://ep-t43:61616"/>
    </bean>

    <bean id="queue" class="org.apache.activemq.command.ActiveMQQueue">
        <constructor-arg value="mmm"/>
    </bean>

</beans>
```

<a id="remoting-jms-server"></a>

#### [](#remoting-jms-server)1.5.1. 服务端的配置

在服务器上， 您只需使用`JmsInvokerServiceExporter`公开服务对象:
```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd">
    
        <bean id="checkingAccountService"
                class="org.springframework.jms.remoting.JmsInvokerServiceExporter">
            <property name="serviceInterface" value="com.foo.CheckingAccountService"/>
            <property name="service">
                <bean class="com.foo.SimpleCheckingAccountService"/>
            </property>
        </bean>
    
        <bean class="org.springframework.jms.listener.SimpleMessageListenerContainer">
            <property name="connectionFactory" ref="connectionFactory"/>
            <property name="destination" ref="queue"/>
            <property name="concurrentConsumers" value="3"/>
            <property name="messageListener" ref="checkingAccountService"/>
        </bean>
    
    </beans>
```
```java
    package com.foo;
    
    import org.springframework.context.support.ClassPathXmlApplicationContext;
    
    public class Server {
    
        public static void main(String[] args) throws Exception {
            new ClassPathXmlApplicationContext(new String[]{"com/foo/server.xml", "com/foo/jms.xml"});
        }
    
    }
```

<a id="remoting-jms-client"></a>

#### [](#remoting-jms-client)1.5.2. 客户端方面的配置

客户端只需要创建一个实现约定接口（`CheckingAccountService`）的客户端代理。

以下示例定义了可以注入其他客户端对象的bean（代理负责通过JMS将调用转发到服务器端对象）:
```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd">
    
        <bean id="checkingAccountService"
                class="org.springframework.jms.remoting.JmsInvokerProxyFactoryBean">
            <property name="serviceInterface" value="com.foo.CheckingAccountService"/>
            <property name="connectionFactory" ref="connectionFactory"/>
            <property name="queue" ref="queue"/>
        </bean>
    
    </beans>
```
```java
    package com.foo;
    
    import org.springframework.context.ApplicationContext;
    import org.springframework.context.support.ClassPathXmlApplicationContext;
    
    public class Client {
    
        public static void main(String[] args) throws Exception {
            ApplicationContext ctx = new ClassPathXmlApplicationContext(
                    new String[] {"com/foo/client.xml", "com/foo/jms.xml"});
            CheckingAccountService service = (CheckingAccountService) ctx.getBean("checkingAccountService");
            service.cancelAccount(new Long(10));
        }
    
    }
```

<a id="remoting-amqp"></a>

### [](#remoting-amqp)1.6. AMQP

有关详细信息，请参阅[Spring AMQP参考指南的“使用AMQP进行Spring Remoting”部分](https://docs.spring.io/spring-amqp/docs/current/reference/html/_reference.html#remoting) 。

远程接口未实现自动检测

远程接口不发生自动检测实现的接口的主要原因是避免向远程调用方打开太多的门，目标对象可能实现内部回调接口，例如`InitializingBean`或`DisposableBean`，它们不希望向调用者公开。

在本地情况下， 向代理提供由目标实现的所有接口通常并不重要。但是， 当公开远程服务时， 应公开特定的服务接口， 并使用特定的操作来进行远程使用。 除内部回调接口外， 目标可能实现多个业务接口， 其中仅有一个用于远程公开。由于这些原因，， 我们需要指定这样的服务接口。

这是配置方便和意外暴露内部方法风险之间的权衡，总是指定一个服务接口并不费劲， 并把你放在安全方面的控制公开的特定方法。

<a id="remoting-considerations"></a>

### [](#remoting-considerations)1.7. 选择技术时的注意事项

这里介绍的每项技术都有其缺点。 在选择技术时，您应该仔细考虑您的需求，您公开的服务以及通过网络发送的对象。

当使用RMI，不可能通过HTTP协议来访问到这些对象，除非你使用了RMI的隧道。RMI是一个重量级的协议，支持全功能的序列化策略，这在使用需要在线上进行序列化的复杂数据模型时非常重要。 但是， RMI-JRMP与Java客户端绑定在一起，它是一种Java-to-Java远程处理解决方案。

如果您需要基于HTTP的远程处理，而且还依赖于Java序列化，那么Spring的HTTP调用程序是一个不错的选择。它与RMI调用共享基本的基础结构， 只是使用HTTP作为传输。 请注意，HTTP调用程序不仅限于Java-to-Java远程处理，还包括客户端和服务器端的Spring。（后者也适用于Spring的非RMI接口的RMI调用程序。）

在跨平台中使用时，Hessian是很好的选择，因为它们明确允许非Java客户端。但是，非Java支持仍然有限。已知问题包括Hibernate对象的序列化以及延迟初始化的集合。如果您有这样的数据模型，请考虑使用RMI或HTTP调用程序而不是Hessian。

JMS可用于提供服务群集， 并允许JMS代理处理负载平衡、发现和自动故障转移。 默认情况下， 在使用JMS远程处理时使用Java序列化， 但JMS提供程序可以对格式使用不同的机制， 例如XStream以允许在其他技术中实现服务器。

最后但并非最不重要的是，EJB具有优于RMI的优势，因为它支持标准的基于角色的身份验证和授权以及远程事务传播。有可能让RMI调用者或HTTP调用者也支持安全上下文传播，尽管核心Spring没有提供。 Spring仅提供适当的挂钩，用于插入第三方或自定义解决方案。

<a id="rest-client-access"></a>

### [](#rest-client-access)1.8. REST 端点

Spring Framework为调用REST端点提供了两种选择：

*   [使用 `RestTemplate`](#rest-resttemplate): 原始的Spring REST客户端与在Spring中API相似于其他模板类，例如JdbcTemplate
    
*   [WebClient](web-reactive.html#webflux-client): 非阻塞，响应式替代方案，支持同步和异步以及流方案。
    

从5.0开始，非阻塞，反应式 `WebClient`提供了`RestTemplate`的现代替代方案，同时有效支持同步和异步以及流方案。 `RestTemplate`将在未来版本中弃用，并且不会在未来添加主要的新功能。

<a id="rest-resttemplate"></a>

#### [](#rest-resttemplate)1.8.1. 使用 `RestTemplate`

`RestTemplate`在HTTP客户端库上提供了更高级别的API，它使得在一行中调用REST端点变得容易。 它公开了以下重载方法组：

Table 1. RestTemplate 方法  

|  Method group    | Description     |
| ---- | ---- |
|  `getForObject`    | 通过GET获取响应。     |
|  `getForEntity`    | 使用GET获取`ResponseEntity` (即status, headers和body)     |
|  `headForHeaders`    | Retrieves all headers for a resource by using HEAD.     |
|  `postForLocation`   | 使用POST创建新资源，并从响应中返回`Location`标头。     |
|  `postForObject`   | 使用POST创建新资源并从响应中返回表示。     |
|  `postForEntity`    | 使用POST创建新资源并从响应中返回表示。     |
|  `put`    |  使用PUT创建或更新资源。    |
|  `patchForObject`    | 使用PATCH更新资源并从响应中返回表示。 请注意，JDK `HttpURLConnection`不支持`PATCH`，但Apache HttpComponents和其他一样。     |
|  `delete`   |  使用DELETE删除指定URI处的资源。|
| `optionsForAllow`     |  使用ALLOW检索资源的允许HTTP方法。    |
|  `exchange`    | 上述方法的更通用（且不太固定）的版本，在需要时提供额外的灵活性。 它接受`RequestEntity`（包括HTTP方法，URL，headers和正文作为输入）并返回`ResponseEntity`。这些方法允许使用`ParameterizedTypeReference`而不是`Class`来指定具有泛型的响应类型。   |
|  `execute`    |  执行请求的最通用方式，通过回调接口完全控制请求准备和响应提取。    |


<a id="rest-resttemplate-create"></a>

##### [](#rest-resttemplate-create)初始化

默认构造函数使用 `java.net.HttpURLConnection`来执行请求。 您可以使用 `ClientHttpRequestFactory`的实现切换到不同的HTTP库。 内置支持以下内容：

*   Apache HttpComponents
    
*   Netty
    
*   OkHttp
    

例如，要切换到Apache HttpComponents，您可以使用以下命令：

```java
RestTemplate template = new RestTemplate(new HttpComponentsClientHttpRequestFactory());
```

每个`ClientHttpRequestFactory`都公开特定于底层HTTP客户端库的配置选项 - 例如，用于凭据，连接池和其他详细信息。

请注意，HTTP请求的`java.net`实现在访问表示错误的响应的状态（例如401）时可能引发异常。 如果这是一个问题，请切换到另一个HTTP客户端库。

<a id="rest-resttemplate-uri"></a>

##### [](#rest-resttemplate-uri)URIs

许多`RestTemplate`方法接受URI模板和URI模板变量，可以是`String`变量参数，也可以是`Map<String,String>`。

以下示例使用`String`变量参数：

```java
String result = restTemplate.getForObject(
        "http://example.com/hotels/{hotel}/bookings/{booking}", String.class, "42", "21");
```

以下示例使用`Map<String, String>` :

```java
Map<String, String> vars = Collections.singletonMap("hotel", "42");

String result = restTemplate.getForObject(
        "http://example.com/hotels/{hotel}/rooms/{hotel}", String.class, vars);
```

请记住，URI模板是自动编码的，如以下示例所示：

```java
restTemplate.getForObject("http://example.com/hotel list", String.class);

// Results in request to "http://example.com/hotel%20list"
```

您可以使用`RestTemplate`的`uriTemplateHandler`属性来自定义URI的编码方式。 或者，您可以准备`java.net.URI`并将其传递给接受`URI`的`RestTemplate`方法之一。

有关使用和编码URI的更多详细信息，请参阅[URI Links](web.html#mvc-uri-building)。

<a id="rest-template-headers"></a>

##### [](#rest-template-headers)Headers

您可以使用`exchange()`方法指定请求头，如以下示例所示：

```java
String uriTemplate = "http://example.com/hotels/{hotel}";
URI uri = UriComponentsBuilder.fromUriString(uriTemplate).build(42);

RequestEntity<Void> requestEntity = RequestEntity.get(uri)
        .header(("MyRequestHeader", "MyValue")
        .build();

ResponseEntity<String> response = template.exchange(requestEntity, String.class);

String responseHeader = response.getHeaders().getFirst("MyResponseHeader");
String body = response.getBody();
```

您可以通过返回 `ResponseEntity`的许多`RestTemplate`方法变体获取响应头。

<a id="rest-template-body"></a>

##### [](#rest-template-body)Body

传递到`RestTemplate`方法并从RestTemplate方法返回的对象在 `HttpMessageConverter`的帮助下转换为原始内容和从原始内容转换。

在POST上，输入对象被序列化到请求主体，如以下示例所示：

URI location = template.postForLocation("http://example.com/people", person);

您无需显式设置请求的`Content-Type`头。在大多数情况下，您可以找到基于源对象类型的兼容消息转换器，并且所选消息转换器会相应地设置内容类型。 如有必要，您可以使用`exchange`方法显式提供`Content-Type`请求头，这反过来会影响选择的消息转换器。

在GET上，响应的主体被反序列化为输出`Object`，如以下示例所示：

Person person = restTemplate.getForObject("http://example.com/people/{id}", Person.class, 42);

不需要显式设置请求的`Accept`头。在大多数情况下，可以根据预期的响应类型找到兼容的消息转换器，然后有助于填充`Accept`头。如有必要，可以使用 `exchange`方法显式提供Accept头。

默认情况下， `RestTemplate`会注册所有[消息转换器](#rest-message-conversion)，具体取决于有助于确定存在哪些可选转换库的类路径检查。 您还可以将消息转换器设置为显式使用。

<a id="rest-message-conversion"></a>

##### [](#rest-message-conversion)消息转换器

[Same as in Spring WebFlux](web-reactive.html#webflux-codecs)

`spring-web` 模块包含 `HttpMessageConverter`，用于通过`InputStream`和`OutputStream`读取和写入HTTP请求和响应的主体。 `HttpMessageConverter`实例用于客户端（例如，在`RestTemplate`中）和服务器端（例如，在Spring MVC REST控制器中）。

框架中提供了主要媒体（MIME）类型的具体实现，默认情况下，它们在客户端注册`RestTemplate` ，在服务器端注册`RequestMethodHandlerAdapter`（请参阅[配置消息转换器](web.html#mvc-config-message-converters)）。

`HttpMessageConverter`的实现将在以下部分中介绍。 对于所有转换器，使用默认媒体类型，但您可以通过设置`supportedMediaTypes` bean属性来覆盖它。 下表描述了每种实现：

Table 2. HttpMessageConverter 实现  

|  消息转换器    |  描述    |
| ---- | ---- |
| `StringHttpMessageConverter`     | 一个`HttpMessageConverter`实现，可以从HTTP请求和响应中读取和写入`String`实例。 默认情况下，此转换器支持所有文本媒体类型（`text/*`），并使用`Content-Type` 为`text/plain`进行写入。     |
|  `FormHttpMessageConverter`    | 一个`HttpMessageConverter`实现，可以从HTTP请求和响应中读取和写入表单数据。 默认情况下，此转换器读取和写入 `application/x-www-form-urlencoded`媒体类型。表单数据从`MultiValueMap<String, String>`读取并写入。     |
|  `ByteArrayHttpMessageConverter`    |  一个`HttpMessageConverter`实现，可以从HTTP请求和响应中读取和写入字节数组。 默认情况下，此转换器支持所有媒体类型（`*/*`），并使用`Content-Type` 为`application/octet-stream`进行写入。 您可以通过设置`supportedMediaTypes`属性并覆盖`getContentType(byte[])`来覆盖它。    |
|  `MarshallingHttpMessageConverter`    | 一个`HttpMessageConverter`实现，可以使用`org.springframework.oxm`包中的Spring的`Marshaller`和`Unmarshaller`抽象来读写XML。 该转换器需要`Marshaller`和`Unmarshaller`才能使用。 您可以通过构造函数或bean属性注入这些。 默认情况下，此转换器支持`text/xml`和 `application/xml`。     |
| `MappingJackson2HttpMessageConverter`     | 一个`HttpMessageConverter`实现，可以使用Jackson的`ObjectMapper`读写JSON。您可以根据需要通过使用Jackson提供的注解来自定义JSON映射。 当您需要进一步控制时（对于需要为特定类型提供自定义JSON序列化器/反序列化器的情况），您可以通过`ObjectMapper`属性注入自定义`ObjectMapper`。 默认情况下，此转换器支持`application/json`.。     |
| `MappingJackson2XmlHttpMessageConverter`     | 一个`HttpMessageConverter`实现，可以使用 [Jackson XML](https://github.com/FasterXML/jackson-dataformat-xml)扩展的 `XmlMapper`读写XML。 您可以根据需要通过使用JAXB或Jackson提供的注解来自定义XML映射。 当您需要进一步控制时（对于需要为特定类型提供自定义XML序列化器/反序列化器的情况），您可以通过`ObjectMapper` 属性注入自定义`XmlMapper`。 默认情况下，此转换器支持`application/xml`。     |
| `SourceHttpMessageConverter`     | 一个`HttpMessageConverter`实现，可以从HTTP请求和响应中读取和写入`javax.xml.transform.Source`。 仅支持`DOMSource`，`SAXSource`和`StreamSource`。 默认情况下，此转换器支持`text/xml` 和 `application/xml`。     |
| `BufferedImageHttpMessageConverter`     | 一个`HttpMessageConverter`实现，可以从HTTP请求和响应中读取和写入 `java.awt.image.BufferedImage`。 此转换器读取和写入Java I/O API支持的媒体类型。|

<a id="rest-template-jsonview"></a>

##### [](#rest-template-jsonview)Jackson JSON 视图

您可以指定[Jackson JSON视图](http://wiki.fasterxml.com/JacksonJsonViews)以仅序列化对象属性的子集，如以下示例所示：

```java
MappingJacksonValue value = new MappingJacksonValue(new User("eric", "7!jd#h23"));
value.setSerializationView(User.WithoutPasswordView.class);

RequestEntity<MappingJacksonValue> requestEntity =
    RequestEntity.post(new URI("http://example.com/user")).body(value);

ResponseEntity<String> response = template.exchange(requestEntity, String.class);
```

<a id="rest-template-multipart"></a>

##### [](#rest-template-multipart)Multipart

要发送多部分数据，您需要提供`MultiValueMap<String, ?>` ，其值是表示部件内容的`Object`实例或表示部件的内容和标头的`HttpEntity` 实例。 `MultipartBodyBuilder`提供了一个方便的API来准备多部分请求，如下例所示：

```java
    MultipartBodyBuilder builder = new MultipartBodyBuilder();
    builder.part("fieldPart", "fieldValue");
    builder.part("filePart", new FileSystemResource("...logo.png"));
    builder.part("jsonPart", new Person("Jason"));

    MultiValueMap<String, HttpEntity<?>> parts = builder.build();
```

在大多数情况下，您不必为每个部件指定`Content-Type` 。内容类型是根据选择用于序列化它的 `HttpMessageConverter`自动确定的，或者在有资源的情况下，基于文件扩展名自动确定。 如有必要，您可以通过其中一个重载的构建器`part` 方法显式提供要用于每个部件的 `MediaType`。

一旦`MultiValueMap`准备就绪，您可以将其传递给`RestTemplate`，如以下示例所示：

```java
    MultipartBodyBuilder builder = ...;
    template.postForObject("http://example.com/upload", builder.build(), Void.class);
```

如果`MultiValueMap`包含至少一个非`String`值，该值也可以表示常规表单数据（即`application/x-www-form-urlencoded`），则无需将 `Content-Type`设置为`multipart/form-data`。 当您使用确保`HttpEntity` 包装器的 `MultipartBodyBuilder`时，情况总是如此。

<a id="rest-async-resttemplate"></a>

#### [](#rest-async-resttemplate)1.8.2. 使用 `AsyncRestTemplate` (已废弃)

不推荐使用`AsyncRestTemplate`。 对于您可能考虑使用`AsyncRestTemplate`的所有用例，请改用[WebClient](web-reactive.html#webflux-client) 。