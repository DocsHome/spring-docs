<a id="ejb"></a>

[](#ejb)2\. 企业级JavaBean(EJB) 集成
-------------------------------

作为轻量级容器， Spring通常被视为EJB的替代者。我们确实认为， 对于许多应用程序和用例来说， Spring作为一个容器， 加上其在事务、ORM 和JDBC访问方面的丰富支持功能， 比通过EJB容器和EJB实现等效功能更有选择。

但是， 重要的是要注意， 使用Spring不会阻止您使用EJB。实际上， Spring使访问EJB和在其中实现EJB和功能变得更加容易。此外， 通过使用Spring来访问EJB提供的服务， 可以在本地EJB、远程EJB或POJO(普通的Java对象）变体之间切换这些服务的实现， 而无需更改客户端代码。

在本章中， 我们来看看Spring如何帮助您访问和实现EJB。Spring在访问无状态会话bean(SLSBs)时提供了特定的值， 因此我们将从讨论这个开始。

<a id="ejb-access"></a>

### [](#ejb-access)2.1. 访问 EJB

本节介绍如何访问EJB.

<a id="ejb-access-concepts"></a>

#### [](#ejb-access-concepts)2.1.1. 概念

若要在本地或远程无状态会话bean上调用方法， 客户端代码通常必须执行JNDI查找以获取(本地或远程)EJB主对象， 然后对该对象使用`create`方法调用以获取实际(本地或远程)EJB对象，然后在EJB上调用一个或多个方法。

为避免重复的低级代码，许多EJB应用程序使用Service Locator（服务定位器模式）和Business Delegate（业务委托模式）。这些都比在整个客户端代码中充满JNDI查找更好，但它们通常的实现具有明显的缺点：

*   通常，使用EJB的代码依赖于Service Locator或Business Delegate单例，这使得很难进行测试。
    
*   对于没有业务代表使用的服务定位器模式，应用程序代码仍然必须在EJB主目录上调用`create()`方法并处理生成的异常。 因此，它仍然与EJB API和EJB编程模型的复杂性联系在一起。
    
*   实现业务委托模式通常会导致严重的代码重复，我们必须编写大量在EJB上调用相同方法的方法。
    

Spring方法是允许创建和使用代理对象， 通常在Spring容器中配置， 作为无需代码的业务委派。您不需要在手动编码的业务委派中编写另一个服务定位器、另一个JNDI查找或重复方法， 除非实际上在这些代码中添加了实际值。

<a id="ejb-access-local"></a>

#### [](#ejb-access-local)2.1.2. 访问本地的SLSBs

假设我们有一个需要使用本地EJB的Web控制器。我们将遵循最佳实践并使用EJB业务方法接口模式， 以便EJB的本地接口继承非EJB特定的业务方法接口。我们将此业务方法称为`MyComponent`接口。 以下示例显示了这样的接口：

```java
public interface MyComponent {
    ...
}
```

使用业务方法接口模式的主要原因之一是确保本地接口中的方法签名和bean实现类之间的同步是自动的。另一个原因是， 它后来使我们更容易切换到POJO(普通、旧的Java对象)的服务实现，如果它有意义的这样做。 当然， 我们还需要实现本地 home 接口， 并提供实现`SessionBean`和`MyComponent` 业务方法接口的实现类。现在， 我们需要做的唯一的Java编码来将我们的Web层控制器与EJB实现挂钩， 就是在控制器上公开一个类型 `MyComponent`的setter方法。这将将引用保存为控制器中的实例变量：

```java
private MyComponent myComponent;

public void setMyComponent(MyComponent myComponent) {
    this.myComponent = myComponent;
}
```

随后， 我们可以在控制器中的任何业务方法中使用此实例变量。现在假设我们从Spring容器中获取控制器对象， 我们可以 (在同一上下文中)配置一个 `LocalStatelessSessionProxyFactoryBean`实例， 它将是EJB代理对象。 代理的配置以及控制器的 `myComponent`属性的设置是通过配置项完成的， 例如：

```java
<bean id="myComponent"
        class="org.springframework.ejb.access.LocalStatelessSessionProxyFactoryBean">
    <property name="jndiName" value="ejb/myBean"/>
    <property name="businessInterface" value="com.mycom.MyComponent"/>
</bean>

<bean id="myController" class="com.mycom.myController">
    <property name="myComponent" ref="myComponent"/>
</bean>
```

有很多工作发生在幕后， 这得益于Spring的AOP框架， 虽然你不被迫使用AOP的概念来享受结果。`myComponent` bean定义为EJB创建了一个代理， 它实现了业务方法接口。 EJB本地主页是在启动时缓存的， 因此只有一个JNDI查找。每次调用EJB 时， 代理都调用本地AOP上的`classname` 方法， 并调用AOP上相应的业务方法。

`myController` bean定义将控制器类的`myComponent`属性设置为EJB代理。

或者(最好是在许多这样的代理定义的情况下), 考虑使用Spring的“jee” 命名空间中的`<jee:local-slsb>` 配置元素。 以下示例说明了如何执行此操作:

```xml
<jee:local-slsb id="myComponent" jndi-name="ejb/myBean"
        business-interface="com.mycom.MyComponent"/>

<bean id="myController" class="com.mycom.myController">
    <property name="myComponent" ref="myComponent"/>
</bean>
```

此EJB访问机制提供了应用程序代码的巨大简化， Web层代码(或其他EJB客户端代码)不依赖于EJB的使用。如果要用POJO或mock对象或其他测试存根替换此EJB引用， 我们只需更改`myComponent` bean定义， 而不更改一行Java代码。此外， 我们还不必编写一行JNDI查找或其他EJB管道代码作为应用程序的一部分。

在实际应用中的基准和经验表明， 这种方法(涉及对目标EJB的反射调用)的性能开销是最小的， 通常在典型的使用中是无法检测到的。请记住， 无论如何， 我们不希望对EJB进行fine-grained调用， 因为在应用程序服务器中存在与EJB基础结构相关的成本。

关于JNDI查找有一个警告。在bean容器中， 此类通常作为单一实例最好使用(根本没有理由使其成为原型)。但是， 如果该bean容器pre-instantiates单变量(与各种XML `ApplicationContext`变体一样)， 则在EJB容器加载目标EJB之前装载 bean容器时可能会出现问题。这是因为JNDI查找将在该类的`init()` 方法中执行, 然后缓存, 但EJB还没有被绑定到目标位置.解决方案是不pre-instantiate此工厂对象, 但允许它在首次使用时创建 .在XML容器中, 这是通过`lazy-init`属性来控制的。

虽然大多数Spring用户不感兴趣，但那些使用EJB编程AOP的人可能希望查看`LocalSlsbInvokerInterceptor`。

<a id="ejb-access-remote"></a>

#### [](#ejb-access-remote)2.1.3.访问远程的SLSBs

访问远程EJB与访问本地EJB基本相同， 只是使用了`SimpleRemoteStatelessSessionProxyFactoryBean`或 `<jee:remote-slsb>` 配置元素。当然， 无论有无Spring， 远程调用语义都适用。在另一台计算机的另一个VM中对某个对象的方法的调用有时必须在使用方案和失败处理方面得到不同的处理。

Spring的EJB客户端支持在非Spring方法上增加了一个优势。通常， EJB客户端代码在本地或远程调用EJB之间很容易来回切换是有问题的。这是因为远程接口方法必须声明它们抛出`RemoteException`， 而客户端代码必须处理此问题， 而本地接口方法则不这样做。 为本地EJB编写的客户端代码需要移动到远程EJB， 通常必须进行修改， 以便为远程异常添加处理， 为需要移动到本地EJB的远程EJB编写的客户端代码， 可以保持不变， 但会对远程异常进行大量不必要的处理， 或者需要修改以删除该代码。 使用Spring远程EJB代理， 您可以不在业务方法接口中声明任何抛出的`RemoteException`， 也不能实现EJB代码， 它具有一个完全相同的远程接口， 但它确实抛出了`RemoteException`， 并且依赖于代理来动态处理这两个接口， 就好像它们是相同的一样。 即 客户端代码不必处理已检查的`RemoteException`类。在EJB调用期间抛出的任何实际`RemoteException`都将引发为non-checked `RemoteAccessException`类， 这是`RuntimeException`的一个子类别。 然后， 可以将目标服务切换到本地EJB或远程EJB(甚至是普通Java对象) 实现之间， 而无需客户端代码知道或关心。当然， 这是可选的。没有什么能阻止您在业务界面中声明`RemoteExceptions`。

<a id="ejb-access-ejb2-ejb3"></a>

#### [](#ejb-access-ejb2-ejb3)2.1.4. 访问EJB 2.x SLSBs 和 EJB 3 SLSBs

通过Spring访问EJB 2.x 会话bean和EJB 3会话bean在很大程度上是透明的。Spring的EJB访问器(包括 `<jee:local-slsb>`和 `<jee:remote-slsb>`功能) 在运行时透明地适应实际组件。 如果找到了一个主接口(EJB 2.x 样式)， 或者如果没有可用的主接口(EJB 3 样式)， 则执行直接组件调用。

Note: 对于EJB 3会话bean， 您可以有效地使用 `JndiObjectFactoryBean`/`<jee:jndi-lookup>`， 因为完全可用的组件引用会在那里公开进行纯JNDI查找。 定义显式`<jee:local-slsb>` 或 `<jee:remote-slsb>`查找只是提供一致且更显式的EJB访问配置。