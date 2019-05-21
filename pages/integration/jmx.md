<a id="jmx"></a>

[](#jmx)4\. JMX
---------------

Spring中的JMS支持为您提供了方便、透明地将Spring应用程序集成到JMS基础架构中的功能。

JMX?

本章不是JMX的介绍。 它并不试图解释您为什么要使用JMX。 如果您是JMX的新手，请参阅本章末尾的 [更多资源](#jmx-resources) 。

具体来说，Spring的JMX支持提供了四个核心功能：

*   将任何Spring bean自动注册为JMX MBean。
    
*   一种灵活的机制，用于控制bean的管理界面。
    
*   声明远程公开MBeans、JSR-160连接
    
*   本地和远程MBean资源的简单代理
    

这些功能设计为在不将应用程序组件连接到Spring或JMX接口和类的情况下工作。实际上， 在大多数情况下， 您的应用程序类不必知道Spring或JMX，就可以利用Spring JMX功能。

<a id="jmx-exporting"></a>

### [](#jmx-exporting)4.1. 公开你的bean给JMX

Spring的JMX框架中的核心类是`MBeanExporter`。这个类是负责收集您的Spring bean和注册他们到JMX `MBeanServer`， 例如，请考虑以下类:

```java
package org.springframework.jmx;

public class JmxTestBean implements IJmxTestBean {

    private String name;
    private int age;
    private boolean isSuperman;

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public int add(int x, int y) {
        return x + y;
    }

    public void dontExposeMe() {
        throw new RuntimeException();
    }
}
```

若要将此bean的属性和方法作为MBean的属性和操作公开， 只需在配置文件中配置`MBeanExporter`类的实例并传入bean。 如下所示:

```xml
<beans>
    <!-- this bean must not be lazily initialized if the exporting is to happen -->
    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter" lazy-init="false">
        <property name="beans">
            <map>
                <entry key="bean:name=testBean1" value-ref="testBean"/>
            </map>
        </property>
    </bean>
    <bean id="testBean" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>
</beans>
```

来自上述配置代码段的相关bean定义是`exporter` bean 。`beans` 属性告诉`MBeanExporter`必须将您的哪些bean公开到JMX `MBeanServer`。在默认配置中，`beans` `Map`中每个条目的键用作对应条目值所引用的bean的 `ObjectName`。您可以更改此行为，如[Controlling `ObjectName`](#jmx-naming)实例中所述。

使用此配置，`testBean`在`ObjectName` bean下公开为MBean：`bean:name=testBean1`。 默认情况下，bean的所有 `public` 属性都作为属性公开，所有 `public` 方法（从`Object`类继承的公共方法除外）都作为操作公开。

`MBeanExporter`是一个有生命周期的bean(查看 [启动和关闭回调](core.html#beans-factory-lifecycle-processor)).默认情况下，在应用程序生命周期中尽可能晚地公开mbean。通过设置`autoStartup`标志，可以配置公开发生的阶段或禁用自动注册。`phase` at which

<a id="jmx-exporting-mbeanserver"></a>

#### [](#jmx-exporting-mbeanserver)4.1.1. 创建一个MBeanServer

[上面](#jmx-exporting)的配置假定应用程序在已运行的一个(且只有一个)`MBeanServer`的环境中运行。在这种情况下，Spring将尝试定位运行的`MBeanServer`，并将您的bean注册到该服务器(如果有的话)。 当应用程序在诸如Tomcat或IBMWebSphere等具有自己的`MBeanServer`的容器中运行时，此行为非常有用。

但是，这种方法在独立的环境中是没有用的，或者在不提供`MBeanServer`的容器内运行。为了解决这个问题，您可以通过将`org.springframework.jmx.support.MBeanServerFactoryBean`类的实例添加到您的配置中以声明方式创建一个`MBeanServer`实例。 还可以通过将`MBeanExporter`的`server`属性的值设置为`MBeanServerFactoryBean`返回的`MBeanServer`值来确保使用特定的`MBeanServer`。例如：

```xml
<beans>

    <bean id="mbeanServer" class="org.springframework.jmx.support.MBeanServerFactoryBean"/>

    <!--
    this bean needs to be eagerly pre-instantiated in order for the exporting to occur;
    this means that it must not be marked as lazily initialized
    -->
    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="beans">
            <map>
                <entry key="bean:name=testBean1" value-ref="testBean"/>
            </map>
        </property>
        <property name="server" ref="mbeanServer"/>
    </bean>

    <bean id="testBean" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>

</beans>
```

在前面的示例中，`MBeanServer`的实例由`MBeanServerFactoryBean`创建，并通过`server`属性提供给`MBeanExporter`。当您提供自己的`MBeanServer`实例时，`MBeanExporter`不会尝试查找正在运行的`MBeanServer`并使用提供的`MBeanServer`实例。 为了使其正常工作，您必须在类路径上具有JMX实现。

<a id="jmx-mbean-server"></a>

#### [](#jmx-mbean-server)4.1.2. 重用已有的`MBeanServer`

如果未指定服务器，`MBeanExporter`将尝试自动检测正在运行的`MBeanServer`。在大多数环境中，只有一个`MBeanServer`实例被使用，但是当存在多个实例时，公开者可能会选择错误的服务器。 在这种情况下，应使用`MBeanServer` `agentId`指示要使用的实例。如下所示：

```xml
<beans>
    <bean id="mbeanServer" class="org.springframework.jmx.support.MBeanServerFactoryBean">
        <!-- indicate to first look for a server -->
        <property name="locateExistingServerIfPossible" value="true"/>
        <!-- search for the MBeanServer instance with the given agentId -->
        <property name="agentId" value="MBeanServer_instance_agentId>"/>
    </bean>
    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="server" ref="mbeanServer"/>
        ...
    </bean>
</beans>
```

对于现有 `MBeanServer` 具有通过查找方法检索的动态（或未知）`agentId`的平台或情况，应使用[factory-method](core.html#beans-factory-class-static-factory-method)，如以下示例所示：

```xml
<beans>
    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="server">
            <!-- Custom MBeanServerLocator -->
            <bean class="platform.package.MBeanServerLocator" factory-method="locateMBeanServer"/>
        </property>
    </bean>

    <!-- other beans here -->

</beans>
```

<a id="jmx-exporting-lazy"></a>

#### [](#jmx-exporting-lazy)4.1.3. 延迟初始化的MBean

如果您配置的bean中还配置了用于延迟初始化的`MBeanExporter`，则`MBeanExporter`将不会中断此关系，并将避免对bean进行实例处理。相反，它将向`MBeanServer`注册一个代理，并将从容器中获取bean，直到出现对代理的第一个调用。

<a id="jmx-exporting-auto"></a>

#### [](#jmx-exporting-auto)4.1.4. 自动注册MBeans

任何通过`MBeanExporter`公开的bean和已经有效的mbean都是在`MBeanExporter`的情况下注册到`MBeanServer`。通过将`autodetect`属性设置为`true`，`MBeanExporter`可以自动检测mbean：

```xml
<bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
    <property name="autodetect" value="true"/>
</bean>

<bean name="spring:mbean=true" class="org.springframework.jmx.export.TestDynamicMBean"/>
```

在前面的示例中，名为`spring:mbean=true`的bean已经是一个有效的JMX MBean，并由Spring自动注册。默认情况下，Spring bean在JMX注册时自动使用他们的name作为`ObjectName`。您可以覆盖此行为，如[控制Bean的 `ObjectName`实例中](#jmx-naming)所述。

<a id="jmx-exporting-registration-behavior"></a>

#### [](#jmx-exporting-registration-behavior)4.1.5. 控制注册的行为

请考虑这样一个场景，即Spring `MBeanExporter`尝试使用对象`bean:name=testBean1`将MBean注册到`MBeanServer`。如果`MBean`实例已在同一`ObjectName`下注册，则默认行为是失败(并引发`InstanceAlreadyExistsException`)。

可以控制`MBean`在`MBeanServer`中注册时所发生的行为，Spring的JMX支持允许三种不同的注册行为在注册过程发现`MBean`已在同一`ObjectName`下注册时控制注册行为。下表汇总了这些注册行为：

Table 6. 注册行为  

|  注册行为    | 说明     |
| ---- | ---- |
|  `FAIL_ON_EXISTING`    |  这是默认的注册行为。 如果`MBean`实例已在同一`ObjectName`下注册，则未注册正在注册的`MBean`，并抛出`InstanceAlreadyExistsException`。 现有MBean不受影响。    |
|  `IGNORE_EXISTING`    | 如果`MBean`实例已在同一`ObjectName`下注册，则未注册正在注册的`MBean`。 现有`MBean`不受影响，并且不会抛出异常。 这在多个应用程序想要在共享`MBeanServer`中共享公共`MBean`的设置中很有用。     |
|  `REPLACE_EXISTING`    |  如果`MBean`实例已在同一 `ObjectName`下注册，则先前注册的现有`MBean`将取消注册，并且新`MBean`将在其位置注册（新`MBean`将有效替换先前的实例）。    |


上表中的值在`RegistrationPolicy`类中定义为枚举。 如果要更改默认注册行为，则需要将`MBeanExporter`定义上的`registrationPolicy`属性值设置为其中一个值。

以下示例显示如何从默认注册行为更改为`REPLACE_EXISTING`行为：

```xml
<beans>

    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="beans">
            <map>
                <entry key="bean:name=testBean1" value-ref="testBean"/>
            </map>
        </property>
        <property name="registrationPolicy" value="REPLACE_EXISTING"/>
    </bean>

    <bean id="testBean" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>

</beans>
```

<a id="jmx-interface"></a>

### [](#jmx-interface)4.2. 控制您的Bean的管理界面

在[上一节的示例](#jmx-exporting-registration-behavior)中，您几乎无法控制bean的管理接口。每个导出bean的所有`public`属性和方法分别作为JMX属性和操作公开。 为了精确控制导出的bean的哪些属性和方法实际上作为JMX属性和操作公开，Spring JMX提供了一个全面且可扩展的机制来控制bean的管理接口。

<a id="jmx-interface-assembler"></a>

#### [](#jmx-interface-assembler)4.2.1. 使用`MBeanInfoAssembler` 接口

在幕后，`MBeanExporter`委托给`org.springframework.jmx.export.assembler.MBeanInfoAssembler`接口的实现，该接口负责定义公开的每个bean的管理接口。默认实现`org.springframework.jmx.export.assembler.SimpleReflectiveMBeanInfoAssembler`简单定义了一个管理接口， 它公开了所有公共属性和方法(如前面的示例中所看到的)。Spring提供了`MBeanInfoAssembler`接口的两个附加实现，允许您使用源级元数据或任意接口控制生成的管理接口。

<a id="jmx-interface-metadata"></a>

#### [](#jmx-interface-metadata)4.2.2. 使用源码级别的元数据：Java注解

使用`MetadataMBeanInfoAssembler`可以使用源代码级元数据为bean定义管理接口。元数据的读取由`org.springframework.jmx.export.metadata.JmxAttributeSource`接口封装。Spring JMX提供了一个使用Java标注的默认实现， 即`org.springframework.jmx.export.annotation.AnnotationJmxAttributeSource`。`MetadataMBeanInfoAssembler`必须配置为`JmxAttributeSource`接口的实现实例才能正常工作(没有默认值)。

要将bean标记为要公开的JMX，应使用`ManagedResource`注解对bean类进行注解。要作为操作公开的每个方法都必须用`ManagedOperation` 注解进行标记，并且要公开的每个属性都必须用`ManagedAttribute`注解标记。在标记属性时，可以忽略getter的批注或setter来分别创建只写或只读属性。

一个`ManagedResource`注解的bean必须是public的，因为方法需要公开操作或属性。

以下示例显示了我们在[创建MBeanServer](#jmx-exporting-mbeanserver)中使用的`JmxTestBean`类的注解形式：

```java
package org.springframework.jmx;

import org.springframework.jmx.export.annotation.ManagedResource;
import org.springframework.jmx.export.annotation.ManagedOperation;
import org.springframework.jmx.export.annotation.ManagedAttribute;

@ManagedResource(
        objectName="bean:name=testBean4",
        description="My Managed Bean",
        log=true,
        logFile="jmx.log",
        currencyTimeLimit=15,
        persistPolicy="OnUpdate",
        persistPeriod=200,
        persistLocation="foo",
        persistName="bar")
public class AnnotationTestBean implements IJmxTestBean {

    private String name;
    private int age;

    @ManagedAttribute(description="The Age Attribute", currencyTimeLimit=15)
    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @ManagedAttribute(description="The Name Attribute",
            currencyTimeLimit=20,
            defaultValue="bar",
            persistPolicy="OnUpdate")
    public void setName(String name) {
        this.name = name;
    }

    @ManagedAttribute(defaultValue="foo", persistPeriod=300)
    public String getName() {
        return name;
    }

    @ManagedOperation(description="Add two numbers")
    @ManagedOperationParameters({
        @ManagedOperationParameter(name = "x", description = "The first number"),
        @ManagedOperationParameter(name = "y", description = "The second number")})
    public int add(int x, int y) {
        return x + y;
    }

    public void dontExposeMe() {
        throw new RuntimeException();
    }

}
```

在这里，您可以看到`JmxTestBean` 类是用`ManagedResource`注解标记的，并且该`ManagedResource`注解是用一组属性配置的。这些属性可用于配置由`MBeanExporter` 生成的MBean的各个方面，稍后将在[源级元数据类型](#jmx-interface-metadata-types)中进行更详细的说明。

你也将注意到`age`和`name`属性被注解使用了 `ManagedAttribute`注解，但是由于`age`属性，只有getter属性被标记。这将导致这些属性被包括在管理接口作为属性，但是`age`属性将是只读。

最后， `add(int, int)`方法用`ManagedOperation`属性标记，而`dontExposeMe()`方法则没有。 这会导致管理接口在使用 `MetadataMBeanInfoAssembler`时仅包含一个操作（`add(int, int)`）。

以下配置显示了如何配置`MBeanExporter`以使用`MetadataMBeanInfoAssembler`：

```xml
<beans>
    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="assembler" ref="assembler"/>
        <property name="namingStrategy" ref="namingStrategy"/>
        <property name="autodetect" value="true"/>
    </bean>

    <bean id="jmxAttributeSource"
            class="org.springframework.jmx.export.annotation.AnnotationJmxAttributeSource"/>

    <!-- will create management interface using annotation metadata -->
    <bean id="assembler"
            class="org.springframework.jmx.export.assembler.MetadataMBeanInfoAssembler">
        <property name="attributeSource" ref="jmxAttributeSource"/>
    </bean>

    <!-- will pick up the ObjectName from the annotation -->
    <bean id="namingStrategy"
            class="org.springframework.jmx.export.naming.MetadataNamingStrategy">
        <property name="attributeSource" ref="jmxAttributeSource"/>
    </bean>

    <bean id="testBean" class="org.springframework.jmx.AnnotationTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>
</beans>
```

在前面的示例中，`MetadataMBeanInfoAssembler` bean已使用`AnnotationJmxAttributeSource`类的实例进行配置，并通过汇编程序属性传递给`MBeanExporter`。 这就是利用面向Spring的MBean的元数据驱动管理接口所需的全部内容。

<a id="jmx-interface-metadata-types"></a>

#### [](#jmx-interface-metadata-types)4.2.3. 源码级别的元数据类型

以下源级别元数据类型可在Spring JMX中使用:

Table 7. 源码级别的元数据类型  

|  目的    |  注解    |  注解类型    |
| ---- | ---- | ---- |
| 将`Class`的所有实例标记为JMX托管资源。     |   `@ManagedResource`   |  Class    |
|  将方法标记为JMX操作。    |  `@ManagedOperation`    |   Method   |
|  将getter或setter标记为JMX属性的一半。    |  `@ManagedAttribute`    | Method (only getters and setters)     |
|  定义操作参数的描述。    |  `@ManagedOperationParameter` and `@ManagedOperationParameters`    |    Method  |


下表描述了可在这些源级元数据类型上使用的配置参数：:

Table 8. 源码级别的元数据参数  

|  参数    |  描述    |  应用于    |
| ---- | ---- | ---- |
| `ObjectName`     | 由`MetadataNamingStrategy`用于确定受管资源的`ObjectName`。     | `ManagedResource`     |
|  `description`    | 设置资源，属性或操作的友好描述。     |  `ManagedResource`, `ManagedAttribute`, `ManagedOperation`, or `ManagedOperationParameter`    |
|  `currencyTimeLimit`    | 设置`currencyTimeLimit`描述符字段的值。     |  `ManagedResource` or `ManagedAttribute`    |
|  `defaultValue`    | 设置`defaultValue`描述符字段的值。     |  `ManagedAttribute`  |
|  `log`    | 设置`log`描述符字段的值。     | `ManagedResource`     |
|  `logFile`    | 设置`logFile`描述符字段的值。     | `ManagedResource`     |
|  `persistPolicy`    | 设置`persistPolicy`描述符字段的值。     |  `ManagedResource` |
|  `persistPeriod`    |  设置`persistPeriod`描述符字段的值。  | `ManagedResource`    |
| `persistLocation`    | 设置`persistLocation`描述符字段的值。     |`ManagedResource`|
|  `persistName`    |  设置`persistName`描述符字段的值。    |  `ManagedResource`    |
|   `name`   |  设置操作参数的显示名称。    | `ManagedOperationParameter`     |
|   `index`   |  设置操作参数的索引。    | `ManagedOperationParameter`     |


<a id="jmx-interface-autodetect"></a>

#### [](#jmx-interface-autodetect)4.2.4. 使用`AutodetectCapableMBeanInfoAssembler` 接口

为了进一步简化配置，Spring引入了扩展`MBeanInfoAssembler`接口的`AutodetectCapableMBeanInfoAssembler`接口，以添加对MBean资源自动检测的支持。如果您将`MBeanExporter`配置为`AutodetectCapableMBeanInfoAssembler`实例，则允许对包含在JMX中的bean进行 “vote”。

`AutodetectCapableMBeanInfo`接口唯一一个开箱即用的实现是`MetadataMBeanInfoAssembler`，它将投票包括任何用`ManagedResource`属性标记的bean。在这种情况下，默认的方法是使用bean名称作为`ObjectName`，从而产生如下的配置：

```xml
<beans>

    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <!-- notice how no 'beans' are explicitly configured here -->
        <property name="autodetect" value="true"/>
        <property name="assembler" ref="assembler"/>
    </bean>

    <bean id="testBean" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>

    <bean id="assembler" class="org.springframework.jmx.export.assembler.MetadataMBeanInfoAssembler">
        <property name="attributeSource">
            <bean class="org.springframework.jmx.export.annotation.AnnotationJmxAttributeSource"/>
        </property>
    </bean>

</beans>
```

请注意，在此配置中没有将bean传递给`MBeanExporter`。但是，`JmxTestBean` 仍将注册，因为它是用`ManagedResource`属性标记的，而`MetadataMBeanInfoAssembler`检测到此值并将其包括在内。这种方法的唯一问题是，`JmxTestBean` 的名称现在具有业务含义。您可以通过更改[控制bean的`ObjectName`](#jmx-naming)中定义的`ObjectName`创建的默认行为来解决此问题。

<a id="jmx-interface-java"></a>

#### [](#jmx-interface-java)4.2.5. 使用Java接口定义管理接口

除了`MetadataMBeanInfoAssembler`， Spring还包括有`InterfaceBasedMBeanInfoAssembler`， 它允许您根据接口集合中定义的方法集来约束所公开的方法和属性。

尽管用于公开mbean的标准机制是使用接口和简单的命名方案，但`InterfaceBasedMBeanInfoAssembler`通过删除对命名约定的需求来扩展此功能，从而允许您使用多个接口并消除对bean的需要以实现MBean接口。

请考虑此接口， 用于为先前看到的`JmxTestBean`类定义管理接口:

```java
public interface IJmxTestBean {

    public int add(int x, int y);

    public long myOperation();

    public int getAge();

    public void setAge(int age);

    public void setName(String name);

    public String getName();

}
```

此接口定义将在JMXMBean上作为操作和属性公开的方法和属性。下面的代码演示如何将Spring JMX配置为使用此接口作为管理接口的定义:

```xml
<beans>

    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="beans">
            <map>
                <entry key="bean:name=testBean5" value-ref="testBean"/>
            </map>
        </property>
        <property name="assembler">
            <bean class="org.springframework.jmx.export.assembler.InterfaceBasedMBeanInfoAssembler">
                <property name="managedInterfaces">
                    <value>org.springframework.jmx.IJmxTestBean</value>
                </property>
            </bean>
        </property>
    </bean>

    <bean id="testBean" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>

</beans>
```

在这里您可以看到，`InterfaceBasedMBeanInfoAssembler`配置为在为任何bean构建管理接口时使用`IJmxTestBean`接口。重要的是要了解， `InterfaceBasedMBeanInfoAssembler`处理的bean不需要实现用于生成JMX管理界面的接口。

在上面的例子中，`IJmxTestBean`接口用于构造所有bean的所有管理接口。在许多情况下，这不是所需的行为，您可能希望对不同的bean使用不同的接口。在这种情况下， 您可以通过`interfaceMappings`属性传递`InterfaceBasedMBeanInfoAssembler``Properties`实例用于该bean，其中每个条目的键是bean名称，每个条目的值是一个逗号分隔的接口名称列表。

如果没有通过`managedInterfaces`或`interfaceMappings`属性指定管理接口，则`InterfaceBasedMBeanInfoAssembler`将反映在bean上，并使用该bean实现的所有接口来创建管理接口。

<a id="jmx-interface-methodnames"></a>

#### [](#jmx-interface-methodnames)4.2.6. 使用 `MethodNameBasedMBeanInfoAssembler`

`MethodNameBasedMBeanInfoAssembler` 允许您指定将作为属性和操作公开给JMX的方法名称的列表。下面的代码显示了此示例的配置:

```xml
<bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
    <property name="beans">
        <map>
            <entry key="bean:name=testBean5" value-ref="testBean"/>
        </map>
    </property>
    <property name="assembler">
        <bean class="org.springframework.jmx.export.assembler.MethodNameBasedMBeanInfoAssembler">
            <property name="managedMethods">
                <value>add,myOperation,getName,setName,getAge</value>
            </property>
        </bean>
    </property>
</bean>
```

在这里您可以看到，方法`add`和`myOperation`将公开为JMX操作，`getName()`, `setName(String)`, 和 `getAge()`将公开为JMX属性的一部分。在上面的代码中， 方法映射适用于向JMX公开的bean。要想逐个bean控制方法的公开，请使用 `MethodNameMBeanInfoAssembler`的`methodMappings`属性将bean名称映射到方法名称的列表。

<a id="jmx-naming"></a>

### [](#jmx-naming)4.3. 为你的bean控制`ObjectName`实例

在幕后，`MBeanExporter`代表了`ObjectNamingStrategy`的实现，以获取它所注册的每个bean的`ObjectName`。缺省情况下，默认情况下，`KeyNamingStrategy`将使用 `beans` `Map` 的键作为`ObjectName`。此外， `KeyNamingStrategy`可以将`beans` `Map`的键映射到`Properties`文件(或文件)中的项，以解析 `ObjectName`。除了`KeyNamingStrategy`，Spring还提供了两个附加的`ObjectNamingStrategy`实现：`IdentityNamingStrategy`（基于bean的JVM标识构建`ObjectName`）和`MetadataNamingStrategy` （使用源级元数据来获取`ObjectName`）。

<a id="jmx-naming-properties"></a>

#### [](#jmx-naming-properties)4.3.1. 从属性中读取`ObjectName`

你可以配置你自己的`KeyNamingStrategy`实例，并将其配置为从`Properties`实例读取`ObjectName`， 而不是使用 bean key。 `KeyNamingStrategy`将会尝试定位`Properties`找到一个对应于bean key的项。如果未找到任何项， 或者`Properties`实例为`null`， 则使用bean key本身。

以下代码显示了`KeyNamingStrategy`的示例配置:

```xml
<beans>

    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="beans">
            <map>
                <entry key="testBean" value-ref="testBean"/>
            </map>
        </property>
        <property name="namingStrategy" ref="namingStrategy"/>
    </bean>

    <bean id="testBean" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>

    <bean id="namingStrategy" class="org.springframework.jmx.export.naming.KeyNamingStrategy">
        <property name="mappings">
            <props>
                <prop key="testBean">bean:name=testBean1</prop>
            </props>
        </property>
        <property name="mappingLocations">
            <value>names1.properties,names2.properties</value>
        </property>
    </bean>

</beans>
```

在这里,`KeyNamingStrategy`的一个实例配置了一个 `Properties` 实例，它从由映射 `Properties` 定义的属性实例和位于由映射属性定义的路径中的属性文件进行合并。在此配置中，`testBean` bean将被赋予`ObjectName` `bean:name=testBean1`，因为这是 `Properties` 实例中具有对应于bean key的项。

如果无法找到`Properties`实例中的条目， 则使用bean key的名称作为`ObjectName`。

<a id="jmx-naming-metadata"></a>

#### [](#jmx-naming-metadata)4.3.2. 使用 `MetadataNamingStrategy`

`MetadataNamingStrategy`使用每个bean上的`ManagedResource`属性的对象属性来创建`objectName`。下面的代码显示了`MetadataNamingStrategy`的配置：

```xml
<beans>

    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="beans">
            <map>
                <entry key="testBean" value-ref="testBean"/>
            </map>
        </property>
        <property name="namingStrategy" ref="namingStrategy"/>
    </bean>

    <bean id="testBean" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>

    <bean id="namingStrategy" class="org.springframework.jmx.export.naming.MetadataNamingStrategy">
        <property name="attributeSource" ref="attributeSource"/>
    </bean>

    <bean id="attributeSource"
            class="org.springframework.jmx.export.annotation.AnnotationJmxAttributeSource"/>

</beans>
```

如果没有为`ManagedResource`提供 `objectName`属性，则会使用下面的格式创建 `objectName`：_\[fully-qualified-package-name\]:type=\[short-classname\],name=\[bean-name\]_。例如，下面的bean生成的`ObjectName`将会是`com.example:type=MyClass,name=myBean`：

```xml
<bean id="myBean" class="com.example.MyClass"/>
```

<a id="jmx-context-mbeanexport"></a>

#### [](#jmx-context-mbeanexport)4.3.3. 配置注解基于MBean导出

如果您更喜欢使用[基于注解的方法](#jmx-interface-metadata)来定义您的管理接口，那么就可以使用`MBeanExporter`的方便子类：`AnnotationMBeanExporter`。定义此子类的实例时，不再需要`namingStrategy`, `assembler`, 和 `attributeSource`配置，因为它将始终使用标准的Java基于元数据(自动检测也总是启用)。实际上， `@EnableMBeanExport` `@Configuration`注解支持更简单的语法，而不是定义`MBeanExporter` bean。如下例所示：

```java
@Configuration
@EnableMBeanExport
public class AppConfig {

}
```

如果您更喜欢基于XML的配置，则`<context:mbean-export/>`元素具有相同的用途，如下所示:

```xml
<context:mbean-export/>
```

如果需要，可以提供对特定MBean `server`的引用，`defaultDomain`属性(`AnnotationMBeanExporter`的属性)接受生成的MBean `ObjectName`字段的替代值。这将用于代替[MetadataNamingStrategy](#jmx-naming-metadata)（上一节中描述的完全限定的软件包名称)。如以下示例所示：

```java
@EnableMBeanExport(server="myMBeanServer", defaultDomain="myDomain")
@Configuration
ContextConfiguration {

}
```

以下示例显示了前面基于注解的示例的XML等效项：:

```xml
<context:mbean-export server="myMBeanServer" default-domain="myDomain"/>
```

不要将基于的AOP代理与bean类中的JMX注解自动检测结合使用。基于代理“hide”目标类，它还隐藏JMX托管资源注解。因此，在这种情况下使用目标类代理:通过在`<aop:config/>`上设置 'proxy-target-class'标志， `<tx:annotation-driven/>`等。否则，您的JMXbean在启动时可能会被悄悄忽略。

<a id="jmx-jsr160"></a>

### [](#jmx-jsr160)4.4. 使用 JSR-160 连接器

对于远程访问，SpringJMX模块在`org.springframework.jmx.support`包中提供了两个 `FactoryBean` 实现，用于创建服务器端和客户端连接器。

<a id="jmx-jsr160-server"></a>

#### [](#jmx-jsr160-server)4.4.1. 服务端的连接器

要使SpringJMX创建、启动和公开JSR-160 `JMXConnectorServer`, 请使用以下配置:

```xml
<bean id="serverConnector" class="org.springframework.jmx.support.ConnectorServerFactoryBean"/>
```

默认情况下， `ConnectorServerFactoryBean`创建一个绑定到 `service:jmx:jmxmp://localhost:9875`的`JMXConnectorServer` 。因此， `serverConnector` bean 通过JMXMP端口9875上的本地主机协议向客户端公开`MBeanServer` 。请注意，JMXMP协议被JSR160规范标记为可选的：目前，主要的开源JMX实现有：MX4J和一个JDK不支持的JMXMP。

要指定另一个URL并在`MBeanServer`中注册`JMXConnectorServer`本身，请分别使用 `serviceUrl` 和 `ObjectName`属性。如下:

```xml
<bean id="serverConnector"
        class="org.springframework.jmx.support.ConnectorServerFactoryBean">
    <property name="objectName" value="connector:name=rmi"/>
    <property name="serviceUrl"
            value="service:jmx:rmi://localhost/jndi/rmi://localhost:1099/myconnector"/>
</bean>
```

如果`ObjectName`属性设置为Spring，将自动在该 `ObjectName`下的`MBeanServer`中注册您的连接器。下面的示例显示了在创建`JMXConnector`时可以传递给`ConnectorServerFactoryBean`的完整参数集：

```xml
<bean id="serverConnector"
        class="org.springframework.jmx.support.ConnectorServerFactoryBean">
    <property name="objectName" value="connector:name=iiop"/>
    <property name="serviceUrl"
        value="service:jmx:iiop://localhost/jndi/iiop://localhost:900/myconnector"/>
    <property name="threaded" value="true"/>
    <property name="daemon" value="true"/>
    <property name="environment">
        <map>
            <entry key="someKey" value="someValue"/>
        </map>
    </property>
</bean>
```

请注意，在使用 RMI-based 连接器时，需要启动查找服务(`tnameserv`或`rmiregistry`)才能完成名称注册。如果您使用Spring向您通过RMI导出远程服务，则Spring将已经构建了RMI注册表。如果不是，您可以使用下面的配置片段轻松地启动注册表。如下：

```xml
<bean id="registry" class="org.springframework.remoting.rmi.RmiRegistryFactoryBean">
    <property name="port" value="1099"/>
</bean>
```

<a id="jmx-jsr160-client"></a>

#### [](#jmx-jsr160-client)4.4.2. 客户端连接器

要创建`MBeanServerConnection`到远程JSR-160启用的`MBeanServer`, 请使用`MBeanServerConnectionFactoryBean`，如下所示:

```xml
<bean id="clientConnector" class="org.springframework.jmx.support.MBeanServerConnectionFactoryBean">
    <property name="serviceUrl" value="service:jmx:rmi://localhost/jndi/rmi://localhost:1099/jmxrmi"/>
</bean>
```

<a id="jmx-jsr160-protocols"></a>

#### [](#jmx-jsr160-protocols)4.4.3. 基于 Hessian/SOAP的JMX

JSR-160允许扩展到客户端和服务器之间进行通信的方式.前面部分中显示的示例使用JSR-160规范（IIOP和JRMP）和（可选）JMXMP所需的基于RMI的强制实现。 通过使用其他提供程序或JMX实现(如[MX4J](http://mx4j.sourceforge.net))，您可以利用SOAP或Hessian等协议优先于简单的HTTP或SSL等协议，如以下示例所示:

```xml
<bean id="serverConnector" class="org.springframework.jmx.support.ConnectorServerFactoryBean">
    <property name="objectName" value="connector:name=burlap"/>
    <property name="serviceUrl" value="service:jmx:burlap://localhost:9874"/>
</bean>
```

在前面的示例中，我们使用了MX4J 3.0.0。 有关更多信息，请参阅官方MX4J文档。

<a id="jmx-proxy"></a>

### [](#jmx-proxy)4.5. 通过代理访问MBeans

SpringJMX允许您创建代理，以便将使用远程或本地的`MBeanServer`中注册的MBean。这些代理为您提供了一个标准的Java接口，通过它可以与您的MBean进行交互。下面的代码演示如何为在本地`MBeanServer`中运行的MBean配置代理：:

```xml
<bean id="proxy" class="org.springframework.jmx.access.MBeanProxyFactoryBean">
    <property name="objectName" value="bean:name=testBean"/>
    <property name="proxyInterface" value="org.springframework.jmx.IJmxTestBean"/>
</bean>
```

在这里您可以看到一个代理是为在`ObjectName`下注册的MBean创建的`bean:name=testBean`。代理将实现的接口由`proxyInterfaces`属性控制，这些接口上的映射方法和属性的规则与MBean上的操作和属性是`InterfaceBasedMBeanInfoAssembler`使用的相同规则。

`MBeanProxyFactoryBean`可以为任何可通过`MBeanServerConnection`访问的MBean创建代理。默认情况下，本地`MBeanServer`位于和使用，但您可以重写此操作，并提供指向远程`MBeanServer`的`MBeanServerConnection`，以满足指向远程mbean的代理：

```xml
<bean id="clientConnector"
        class="org.springframework.jmx.support.MBeanServerConnectionFactoryBean">
    <property name="serviceUrl" value="service:jmx:rmi://remotehost:9875"/>
</bean>

<bean id="proxy" class="org.springframework.jmx.access.MBeanProxyFactoryBean">
    <property name="objectName" value="bean:name=testBean"/>
    <property name="proxyInterface" value="org.springframework.jmx.IJmxTestBean"/>
    <property name="server" ref="clientConnector"/>
</bean>
```

在这里您可以看到，我们创建一个`MBeanServerConnection`指向远程机器使用 `MBeanServerConnectionFactoryBean`。然后，通过`server`属性将此`MBeanServerConnection`传递到`MBeanProxyFactoryBean` 。创建的代理将通过此`MBeanServerConnection`将所有调用转发到`MBeanServer` 。

<a id="jmx-notifications"></a>

### [](#jmx-notifications)4.6. 通知

Spring JMX提供了JMX全面通知的支持

<a id="jmx-notifications-listeners"></a>

#### [](#jmx-notifications-listeners)4.6.1. 注册通知的监听器

Spring的JMX支持使您可以很容易地注册任何数量的`NotificationListeners`与任何数量的MBean(这包括Spring的`MBeanExporter` 公开的MBean和通过其他机制注册的MBean。例如，考虑每当目标MBean的属性发生变化时，希望（通过 `Notification`）通知的场景。 以下示例将通知写入控制台:

```java
package com.example;

import javax.management.AttributeChangeNotification;
import javax.management.Notification;
import javax.management.NotificationFilter;
import javax.management.NotificationListener;

public class ConsoleLoggingNotificationListener
        implements NotificationListener, NotificationFilter {

    public void handleNotification(Notification notification, Object handback) {
        System.out.println(notification);
        System.out.println(handback);
    }

    public boolean isNotificationEnabled(Notification notification) {
        return AttributeChangeNotification.class.isAssignableFrom(notification.getClass());
    }

}
```

以下示例将 `ConsoleLoggingNotificationListener`（在前面的示例中定义）添加到`notificationListenerMappings`：:

```xml
<beans>

    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="beans">
            <map>
                <entry key="bean:name=testBean1" value-ref="testBean"/>
            </map>
        </property>
        <property name="notificationListenerMappings">
            <map>
                <entry key="bean:name=testBean1">
                    <bean class="com.example.ConsoleLoggingNotificationListener"/>
                </entry>
            </map>
        </property>
    </bean>

    <bean id="testBean" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>

</beans>
```

使用上述配置，每次从目标MBean(`bean:name=testBean1`)广播JMX通知时，都将通知通过`notificationListenerMappings`属性注册为侦听器的`ConsoleLoggingNotificationListener` bean。然后， `ConsoleLoggingNotificationListener` bean可以采取它认为适当的任何行动，以响应`Notification`。

你也可以直接使用bean的名字作为公开的bean和监听器的桥接:

```xml
<beans>

    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="beans">
            <map>
                <entry key="bean:name=testBean1" value-ref="testBean"/>
            </map>
        </property>
        <property name="notificationListenerMappings">
            <map>
                <entry key="testBean">
                    <bean class="com.example.ConsoleLoggingNotificationListener"/>
                </entry>
            </map>
        </property>
    </bean>

    <bean id="testBean" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>

</beans>
```

如果要为封闭的`MBeanExporter`导出的所有bean注册单个`NotificationListener`实例，则可以使用特殊通配符（`*`）作为`notificationListenerMappings`属性映射中的条目的键，如以下示例所示：

```xml
<property name="notificationListenerMappings">
    <map>
        <entry key="*">
            <bean class="com.example.ConsoleLoggingNotificationListener"/>
        </entry>
    </map>
</property>
```

如果需要执行相反的流程(即，对一个MBean注册多个不同的监听器)，则必须改用`notificationListeners`属性(并且优先于`notificationListenerMappings`属性)。这一次， 不是简单地为单个MBean配置`NotificationListener`，而是配置`NotificationListenerBean`实例。`NotificationListenerBean` 封装了`NotificationListener`和`ObjectName`(或`ObjectNames`),它将在`MBeanServer`中注册 。`NotificationListenerBean`还封装了许多其他属性，如`NotificationFilter`和任意handback对象，可在高级JMX通知方案中使用。

使用`NotificationListenerBean`实例时的配置与以前的情况不同:

```xml
<beans>

    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="beans">
            <map>
                <entry key="bean:name=testBean1" value-ref="testBean"/>
            </map>
        </property>
        <property name="notificationListeners">
            <list>
                <bean class="org.springframework.jmx.export.NotificationListenerBean">
                    <constructor-arg>
                        <bean class="com.example.ConsoleLoggingNotificationListener"/>
                    </constructor-arg>
                    <property name="mappedObjectNames">
                        <list>
                            <value>bean:name=testBean1</value>
                        </list>
                    </property>
                </bean>
            </list>
        </property>
    </bean>

    <bean id="testBean" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>

</beans>
```

上面的示例等效于第一个通知示例，我们假设在每次引发`Notification`时都要给出一个handback对象，而且我们还希望通过提供一个`NotificationFilter`来过滤掉无关的`Notifications`。以下示例实现了以下目标：

```xml
<beans>

    <bean id="exporter" class="org.springframework.jmx.export.MBeanExporter">
        <property name="beans">
            <map>
                <entry key="bean:name=testBean1" value-ref="testBean1"/>
                <entry key="bean:name=testBean2" value-ref="testBean2"/>
            </map>
        </property>
        <property name="notificationListeners">
            <list>
                <bean class="org.springframework.jmx.export.NotificationListenerBean">
                    <constructor-arg ref="customerNotificationListener"/>
                    <property name="mappedObjectNames">
                        <list>
                            <!-- handles notifications from two distinct MBeans -->
                            <value>bean:name=testBean1</value>
                            <value>bean:name=testBean2</value>
                        </list>
                    </property>
                    <property name="handback">
                        <bean class="java.lang.String">
                            <constructor-arg value="This could be anything..."/>
                        </bean>
                    </property>
                    <property name="notificationFilter" ref="customerNotificationListener"/>
                </bean>
            </list>
        </property>
    </bean>

    <!-- implements both the NotificationListener and NotificationFilter interfaces -->
    <bean id="customerNotificationListener" class="com.example.ConsoleLoggingNotificationListener"/>

    <bean id="testBean1" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="TEST"/>
        <property name="age" value="100"/>
    </bean>

    <bean id="testBean2" class="org.springframework.jmx.JmxTestBean">
        <property name="name" value="ANOTHER TEST"/>
        <property name="age" value="200"/>
    </bean>

</beans>
```

（有关handback对象的完整讨论，实际上，`NotificationFilter`是什么，请参阅标题'JMX通知模型'的JMX规范（1.2）部分。

<a id="jmx-notifications-publishing"></a>

#### [](#jmx-notifications-publishing)4.6.2. 发布通知

Spring提供的支持不仅用于注册接收`Notifications` ， 而且还用于发布`Notifications` 。

请注意， 这一节讨论的是只与Spring管理的bean， 是真正通过`MBeanExporter`公开的MBean；任何现有的、用户定义的MBean都应使用标准JMX API进行通知发布。

Spring的JMX通知发布支持中的关键接口是`NotificationPublisher`接口（在`org.springframework.jmx.export.notification`包中定义）。任何将通过`MBeanExporter`实例导出为MBean的bean都 可以实现相关的`NotificationPublisherAware`接口以获取对`NotificationPublisher`实例的访问权限。 `NotificationPublisherAware`接口通过一个简单的setter方法向实现bean提供`NotificationPublisher`实例，然后bean可以使用它来发布`Notifications`。

如[`NotificationPublisher`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/jmx/export/notification/NotificationPublisher.html)类的javadocs中所述，通过NotificationPublisher机制发布事件的托管bean不负责任何通知侦听器的状态管理和类似。Spring的JMX支持将负责处理所有JMX基础问题， 应用程序开发人员需要做的事就是实现`NotificationPublisherAware`接口并使用提供的`NotificationPublisher`实例启动发布事件。请注意， `NotificationPublisher`将在托管bean注册到 `MBeanServer`后设置。

使用`NotificationPublisher`实例非常简单，一个简单地创建一个JMX`Notification`实例(或一个适当的`Notification` 子类的实例)，是一个用与要发布的事件相关的数据填充通知，然后一个调用`NotificationPublisher`实例上的`sendNotification(Notification)`，传递`Notification`。

在以下示例中， `JmxTestBean`公开的实例将在每次调用`add(int, int)`操作时发布`NotificationEvent` ：

```java
package org.springframework.jmx;

import org.springframework.jmx.export.notification.NotificationPublisherAware;
import org.springframework.jmx.export.notification.NotificationPublisher;
import javax.management.Notification;

public class JmxTestBean implements IJmxTestBean, NotificationPublisherAware {

    private String name;
    private int age;
    private boolean isSuperman;
    private NotificationPublisher publisher;

    // other getters and setters omitted for clarity

    public int add(int x, int y) {
        int answer = x + y;
        this.publisher.sendNotification(new Notification("add", this, 0));
        return answer;
    }

    public void dontExposeMe() {
        throw new RuntimeException();
    }

    public void setNotificationPublisher(NotificationPublisher notificationPublisher) {
        this.publisher = notificationPublisher;
    }

}
```

`NotificationPublisher`接口和工作机制的是Spring对JMX支持的一个更好的特性。然而它来与你的类的price标签耦合Spring和JMX；一如既往，，这里的建议是真诚的。 如果您需要`NotificationPublisher`提供的功能，并且您可以接受与Spring和JMX的耦合，那么就这样做。

<a id="jmx-resources"></a>

### [](#jmx-resources)4.7. 更多资源

This section contains links to further resources about JMX:

*   Oracle中 [JMX](http://www.oracle.com/technetwork/java/javase/tech/javamanagement-140525.html) 的主页
    
*   [JMX 规范](https://jcp.org/aboutJava/communityprocess/final/jsr003/index3.html) (JSR-000003).
    
*   [JMX Remote API 规范](https://jcp.org/aboutJava/communityprocess/final/jsr160/index.html) (JSR-000160).
    
*   [MX4J 主页](http://mx4j.sourceforge.net/). (MX4J是各种JMX规范的开源实现。)