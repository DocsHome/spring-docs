<a id="appendix"></a>

[](#appendix)9\. 附录
-------------------

<a id="xsd-schemas"></a>

### [](#xsd-schemas)9.1. XML Schemas

附录的这一部分列出了与集成技术相关的XML Schemas。

<a id="xsd-schemas-jee"></a>

#### [](#xsd-schemas-jee)9.1.1. `jee` Schema

`jee`元素处理与Java EE（Java Enterprise Edition）配置相关的问题，例如查找JNDI对象和定义EJB引用。

要使用`jee` schema中的元素，您需要在Spring XML配置文件的顶部包含以下前导码。 以下代码段中的文本引用了正确的schema，以便`jee`命名空间中的元素可供您使用：:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jee="http://www.springframework.org/schema/jee" xsi:schemaLocation="
        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/jee http://www.springframework.org/schema/jee/spring-jee.xsd">

    <!-- bean definitions here -->
</beans>
```

<a id="xsd-schemas-jee-jndi-lookup"></a>

##### [](#xsd-schemas-jee-jndi-lookup)<jee:jndi-lookup/> (simple)

以下示例显示如何使用JNDI在没有`jee` schema的情况下查找数据源:

```xml
<bean id="dataSource" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiName" value="jdbc/MyDataSource"/>
</bean>
<bean id="userDao" class="com.foo.JdbcUserDao">
    <!-- Spring will do the cast automatically (as usual) -->
    <property name="dataSource" ref="dataSource"/>
</bean>
```

以下示例显示如何使用JNDI使用`jee` schema查找数据源:

```xml
<jee:jndi-lookup id="dataSource" jndi-name="jdbc/MyDataSource"/>

<bean id="userDao" class="com.foo.JdbcUserDao">
    <!-- Spring will do the cast automatically (as usual) -->
    <property name="dataSource" ref="dataSource"/>
</bean>
```

<a id="xsd-schemas-jee-jndi-lookup-environment-single"></a>

##### [](#xsd-schemas-jee-jndi-lookup-environment-single)`<jee:jndi-lookup/>` (使用单个JNDI环境设置)

以下示例显示如何使用JNDI查找没有`jee`的环境变量 :

```xml
<bean id="simple" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiName" value="jdbc/MyDataSource"/>
    <property name="jndiEnvironment">
        <props>
            <prop key="ping">pong</prop>
        </props>
    </property>
</bean>
```

以下示例显示如何使用JNDI使用`jee`查找环境变量:

```xml
<jee:jndi-lookup id="simple" jndi-name="jdbc/MyDataSource">
    <jee:environment>ping=pong</jee:environment>
</jee:jndi-lookup>
```

<a id="xsd-schemas-jee-jndi-lookup-evironment-multiple"></a>

##### [](#xsd-schemas-jee-jndi-lookup-evironment-multiple)`<jee:jndi-lookup/>` (多个JNDI环境设置)

以下示例显示如何使用JNDI在没有`jee`的情况下查找多个环境变量:

```xml
<bean id="simple" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiName" value="jdbc/MyDataSource"/>
    <property name="jndiEnvironment">
        <props>
            <prop key="sing">song</prop>
            <prop key="ping">pong</prop>
        </props>
    </property>
</bean>
```

The following example shows how to use JNDI to look up multiple environment variables with `jee`:

```xml
<jee:jndi-lookup id="simple" jndi-name="jdbc/MyDataSource">
    <!-- newline-separated, key-value pairs for the environment (standard Properties format) -->
    <jee:environment>
        sing=song
        ping=pong
    </jee:environment>
</jee:jndi-lookup>
```

<a id="xsd-schemas-jee-jndi-lookup-complex"></a>

##### [](#xsd-schemas-jee-jndi-lookup-complex)`<jee:jndi-lookup/>` (复杂)

以下示例显示如何使用JNDI使用`jee`查找多个环境变量:

```xml
<bean id="simple" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiName" value="jdbc/MyDataSource"/>
    <property name="cache" value="true"/>
    <property name="resourceRef" value="true"/>
    <property name="lookupOnStartup" value="false"/>
    <property name="expectedType" value="com.myapp.DefaultThing"/>
    <property name="proxyInterface" value="com.myapp.Thing"/>
</bean>
```

以下示例显示如何使用JNDI在没有`jee`的情况下查找数据源和许多不同的属性:

```xml
<jee:jndi-lookup id="simple"
        jndi-name="jdbc/MyDataSource"
        cache="true"
        resource-ref="true"
        lookup-on-startup="false"
        expected-type="com.myapp.DefaultThing"
        proxy-interface="com.myapp.Thing"/>
```

<a id="xsd-schemas-jee-local-slsb"></a>

##### [](#xsd-schemas-jee-local-slsb)`<jee:local-slsb/>` (简单)

`<jee:local-slsb/>` 元素配置对本地EJB Stateless SessionBean的引用。

以下示例显示如何在没有`jee`的情况下配置对本地EJB Stateless SessionBean的引用:

```xml
<bean id="simple"
        class="org.springframework.ejb.access.LocalStatelessSessionProxyFactoryBean">
    <property name="jndiName" value="ejb/RentalServiceBean"/>
    <property name="businessInterface" value="com.foo.service.RentalService"/>
</bean>
```

以下示例显示如何使用`jee`配置对本地EJB Stateless SessionBean的引用:

```xml
<jee:local-slsb id="simpleSlsb" jndi-name="ejb/RentalServiceBean"
        business-interface="com.foo.service.RentalService"/>
```

<a id="xsd-schemas-jee-local-slsb-complex"></a>

##### [](#xsd-schemas-jee-local-slsb-complex)`<jee:local-slsb/>` (复杂)

`<jee:local-slsb/>`元素配置对本地EJB Stateless SessionBean的引用。

以下示例显示如何配置对本地EJB Stateless SessionBean的引用以及许多不带`jee`的属性:

```xml
<bean id="complexLocalEjb"
        class="org.springframework.ejb.access.LocalStatelessSessionProxyFactoryBean">
    <property name="jndiName" value="ejb/RentalServiceBean"/>
    <property name="businessInterface" value="com.example.service.RentalService"/>
    <property name="cacheHome" value="true"/>
    <property name="lookupHomeOnStartup" value="true"/>
    <property name="resourceRef" value="true"/>
</bean>
```

以下示例显示如何使用`jee`配置对本地EJB Stateless SessionBean和许多属性的引用:

```xml
<jee:local-slsb id="complexLocalEjb"
        jndi-name="ejb/RentalServiceBean"
        business-interface="com.foo.service.RentalService"
        cache-home="true"
        lookup-home-on-startup="true"
        resource-ref="true">
```

<a id="xsd-schemas-jee-remote-slsb"></a>

##### [](#xsd-schemas-jee-remote-slsb)<jee:remote-slsb/>

`<jee:remote-slsb/>` 元素配置对`remote`EJB Stateless SessionBean的引用。

以下示例显示如何在不使用`jee`的情况下配置对远程EJB Stateless SessionBean的引用

```xml
<bean id="complexRemoteEjb"
        class="org.springframework.ejb.access.SimpleRemoteStatelessSessionProxyFactoryBean">
    <property name="jndiName" value="ejb/MyRemoteBean"/>
    <property name="businessInterface" value="com.foo.service.RentalService"/>
    <property name="cacheHome" value="true"/>
    <property name="lookupHomeOnStartup" value="true"/>
    <property name="resourceRef" value="true"/>
    <property name="homeInterface" value="com.foo.service.RentalService"/>
    <property name="refreshHomeOnConnectFailure" value="true"/>
</bean>
```

以下示例显示如何使用 `jee`配置对远程EJB Stateless SessionBean的引用:

```xml
<jee:remote-slsb id="complexRemoteEjb"
        jndi-name="ejb/MyRemoteBean"
        business-interface="com.foo.service.RentalService"
        cache-home="true"
        lookup-home-on-startup="true"
        resource-ref="true"
        home-interface="com.foo.service.RentalService"
        refresh-home-on-connect-failure="true">
```

<a id="xsd-schemas-jms"></a>

#### [](#xsd-schemas-jms)9.1.2. `jms` Schema

`jms`元素处理配置与JMS相关的bean，例如Spring的[Message Listener Containers](#jms-mdp)。 这些元素在[JMS命名空间支持](#jms-namespace)支持一节中详细介绍。 有关此支持和`jms`元素本身的完整详细信息，请参阅该章节。

为了完整性，要使用`jms` schema中的元素，您需要在Spring XML配置文件的顶部包含以下前导码。 以下代码段中的文本引用了正确的schema，以便您可以使用jms命名空间中的元素:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jms="http://www.springframework.org/schema/jms" xsi:schemaLocation="
        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/jms http://www.springframework.org/schema/jms/spring-jms.xsd">

    <!-- bean definitions here -->
</beans>
```

<a id="xsd-schemas-context-mbe"></a>

#### [](#xsd-schemas-context-mbe)9.1.3. 使用 `<context:mbean-export/>`

[配置基于注解的MBean导出](#jmx-context-mbeanexport)中详细介绍了此元素。

<a id="xsd-schemas-cache"></a>

#### [](#xsd-schemas-cache)9.1.4. `cache` Schema

您可以使用`cache`元素来启用对Spring的`@CacheEvict`, `@CachePut`,和`@Caching`注释的支持。 它还支持基于声明的基于XML的缓存。 有关详细信息，请参阅[启用缓存注解](#cache-annotation-enable)和 [基于XML的声明性缓存](#cache-declarative-xml)。

要使用`cache` schema中的元素，需要在Spring XML配置文件的顶部包含以下前导码。 以下代码段中的文本引用了正确的schema，以便您可以使用`cache`命名空间中的元素:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:cache="http://www.springframework.org/schema/cache" xsi:schemaLocation="
        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd">

    <!-- bean definitions here -->
</beans>
```