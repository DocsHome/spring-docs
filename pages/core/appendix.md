<a id="appendix"></a>

[](#appendix)9\. 附录
-------------------

<a id="xsd-schemas"></a>

### [](#xsd-schemas)9.1. XML Schemas

附录的这一部分列出了与核心容器相关的XML Schemas。

<a id="xsd-schemas-util"></a>

#### [](#xsd-schemas-util)9.1.1. The `util` Schema

顾名思义，`util`标签处理常见的实用程序配置问题，例如配置集合，引用常量等等。要使用`util` schema中的标记，您需要在Spring XML配置文件的顶部有以下声明。 下面代码段中的文本引用了正确的schema，以便您可以使用`util`命名空间中的标记:

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:util="http://www.springframework.org/schema/util" xsi:schemaLocation="
            http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd">

            <!-- bean definitions here -->
    </beans>

<a id="xsd-schemas-util-constant"></a>

##### [](#xsd-schemas-util-constant)使用 `<util:constant/>`

考虑下面bean的定义:

    <bean id="..." class="...">
        <property name="isolation">
            <bean id="java.sql.Connection.TRANSACTION_SERIALIZABLE"
                    class="org.springframework.beans.factory.config.FieldRetrievingFactoryBean" />
        </property>
    </bean>

上述配置使用Spring `FactoryBean`实现（ `FieldRetrievingFactoryBean`），将bean上的隔离属性的值设置为 `java.sql.Connection.TRANSACTION_SERIALIZABLE`常量值。 这看起来很不错，但它是一个稍微冗长的，并且公开Spring的内部管道给最终用户（这是不必要的）。

以下基于XML Schema的版本更简洁，清楚地表达了开发人员的意图（“注入此常量值”），并且它读得更好：

    <bean id="..." class="...">
        <property name="isolation">
            <util:constant static-field="java.sql.Connection.TRANSACTION_SERIALIZABLE"/>
        </property>
    </bean>

<a id="xsd-schemas-util-frfb"></a>

###### [](#xsd-schemas-util-frfb)从字段值设置Bean属性或构造函数参数

[`FieldRetrievingFactoryBean`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/config/FieldRetrievingFactoryBean.html) 是一个`FactoryBean`，用于检索静态或非静态字段值。它通常用于检索`public` `static` `final`常量，然后可用于为另一个bean设置属性值或构造函数的参数。

以下示例显示了如何使用[`staticField`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/config/FieldRetrievingFactoryBean.html#setStaticField(java.lang.String))属性公开静态字段:

    <bean id="myField"
            class="org.springframework.beans.factory.config.FieldRetrievingFactoryBean">
        <property name="staticField" value="java.sql.Connection.TRANSACTION_SERIALIZABLE"/>
    </bean>

还有一个便捷使用形式，其中静态字段被指定为bean名称，如以下示例所示：:

    <bean id="java.sql.Connection.TRANSACTION_SERIALIZABLE"
            class="org.springframework.beans.factory.config.FieldRetrievingFactoryBean"/>

这确实意味着bean `id`不再有任何选择（因此引用它的任何其他bean也必须使用这个更长的名称）， 但这种形式定义非常简洁，非常方便用作内部 bean，因为不必为bean引用指定`id`，如下例所示:

    <bean id="..." class="...">
        <property name="isolation">
            <bean id="java.sql.Connection.TRANSACTION_SERIALIZABLE"
                    class="org.springframework.beans.factory.config.FieldRetrievingFactoryBean" />
        </property>
    </bean>

您还可以访问另一个bean的非静态（实例）字段，如[`FieldRetrievingFactoryBean`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/config/FieldRetrievingFactoryBean.html)类的API文档中所述

在Spring中，将枚举值作为属性或构造函数参数注入到bean中非常容易，因为您实际上并不需要做任何事情，也不知道Spring内部的任何内容(甚至包括`FieldRetrievingFactoryBean`相似的类）。 以下示例枚举显示了注入枚举值的容易程度:

    package javax.persistence;

    public enum PersistenceContextType {

        TRANSACTION,
        EXTENDED
    }

现在考虑下面的`PersistenceContextType` 类型的setter和相应的bean定义:

    package example;

    public class Client {

        private PersistenceContextType persistenceContextType;

        public void setPersistenceContextType(PersistenceContextType type) {
            this.persistenceContextType = type;
        }
    }

    <bean class="example.Client">
        <property name="persistenceContextType" value="TRANSACTION"/>
    </bean>

<a id="xsd-schemas-util-property-path"></a>

##### [](#xsd-schemas-util-property-path)使用 `<util:property-path/>`

请考虑以下示例:

    <!-- target bean to be referenced by name -->
    <bean id="testBean" class="org.springframework.beans.TestBean" scope="prototype">
        <property name="age" value="10"/>
        <property name="spouse">
            <bean class="org.springframework.beans.TestBean">
                <property name="age" value="11"/>
            </bean>
        </property>
    </bean>

    <!-- results in 10, which is the value of property 'age' of bean 'testBean' -->
    <bean id="testBean.age" class="org.springframework.beans.factory.config.PropertyPathFactoryBean"/>

上述配置使用Spring `FactoryBean`实现（`PropertyPathFactoryBean`）创建名为`testBean.age`的bean（类型为`int`），其值等于`testBean` bean的`age`属性。

现在考虑以下示例，它添加了一个`<util:property-path/>` 元素:

    <!-- target bean to be referenced by name -->
    <bean id="testBean" class="org.springframework.beans.TestBean" scope="prototype">
        <property name="age" value="10"/>
        <property name="spouse">
            <bean class="org.springframework.beans.TestBean">
                <property name="age" value="11"/>
            </bean>
        </property>
    </bean>

    <!-- results in 10, which is the value of property 'age' of bean 'testBean' -->
    <util:property-path id="name" path="testBean.age"/>

`<property-path/>` 元素的`path`属性的值遵循`beanName.beanProperty`的形式。 在这种情况下，它会获取名为`testBean`的bean的`age`属性。 该`age`属性值是`10`。

<a id="xsd-schemas-util-property-path-dependency"></a>

###### [](#xsd-schemas-util-property-path-dependency)使用 `<util:property-path/>`设置Bean属性或构造函数参数

`PropertyPathFactoryBean` 是一个用于计算给定目标对象的属性路径的 `FactoryBean` 。目标对象可以直接指定，也可以通过bean名称指定。 然后，您可以在另一个bean定义中将此值用作属性值或构造函数参数。

以下示例按名称显示了针对另一个bean使用的路径:

    // target bean to be referenced by name
    <bean id="person" class="org.springframework.beans.TestBean" scope="prototype">
        <property name="age" value="10"/>
        <property name="spouse">
            <bean class="org.springframework.beans.TestBean">
                <property name="age" value="11"/>
            </bean>
        </property>
    </bean>

    // results in 11, which is the value of property 'spouse.age' of bean 'person'
    <bean id="theAge"
            class="org.springframework.beans.factory.config.PropertyPathFactoryBean">
        <property name="targetBeanName" value="person"/>
        <property name="propertyPath" value="spouse.age"/>
    </bean>

在以下示例中，path被内部bean解析:

    <!-- results in 12, which is the value of property 'age' of the inner bean -->
    <bean id="theAge"
            class="org.springframework.beans.factory.config.PropertyPathFactoryBean">
        <property name="targetObject">
            <bean class="org.springframework.beans.TestBean">
                <property name="age" value="12"/>
            </bean>
        </property>
        <property name="propertyPath" value="age"/>
    </bean>

这也是一个快捷的形式，其中bean名称是属性的路径。

    <!-- results in 10, which is the value of property 'age' of bean 'person' -->
    <bean id="person.age"
            class="org.springframework.beans.factory.config.PropertyPathFactoryBean"/>

此形式表示bean的名称中是没得选择的，对它的任何引用也必须使用相同的`id`，即它的路径。当然，如果用作内部bean，则根本不需要引用它。如下所示：

    <bean id="..." class="...">
        <property name="age">
            <bean id="person.age"
                    class="org.springframework.beans.factory.config.PropertyPathFactoryBean"/>
        </property>
    </bean>

结果类型可以在实际定义中具体设置。对于大多数用例来说，这是不必要的，但对于某些用例来说是可以使用的。有关此功能的更多信息，请参阅javadoc。

<a id="xsd-schemas-util-properties"></a>

##### [](#xsd-schemas-util-properties)使用 `<util:properties/>`

请考虑以下示例:

    <!-- creates a java.util.Properties instance with values loaded from the supplied location -->
    <bean id="jdbcConfiguration" class="org.springframework.beans.factory.config.PropertiesFactoryBean">
        <property name="location" value="classpath:com/foo/jdbc-production.properties"/>
    </bean>

上述配置使用Spring `FactoryBean`实现（`PropertiesFactoryBean`）来实例化一个`java.util.Properties`实例，其中包含从提供的[`Resource`](#resources)位置加载的值。

以下示例使用`util:properties`元素来进行更简洁的表示:

    <!-- creates a java.util.Properties instance with values loaded from the supplied location -->
    <util:properties id="jdbcConfiguration" location="classpath:com/foo/jdbc-production.properties"/>

<a id="xsd-schemas-util-list"></a>

##### [](#xsd-schemas-util-list)使用 `<util:list/>`

请考虑以下示例:

    <!-- creates a java.util.List instance with values loaded from the supplied 'sourceList' -->
    <bean id="emails" class="org.springframework.beans.factory.config.ListFactoryBean">
        <property name="sourceList">
            <list>
                <value>[email protected]</value>
                <value>[email protected]</value>
                <value>[email protected]</value>
                <value>[email protected]</value>
            </list>
        </property>
    </bean>

上述配置使用Spring `FactoryBean`实现（`ListFactoryBean`）创建 `java.util.List`实例，并使用从提供的`sourceList`获取的值对其进行初始化。

以下示例使用`<util:list/>`元素进行更简洁的表示:

    <!-- creates a java.util.List instance with the supplied values -->
    <util:list id="emails">
        <value>[email protected]</value>
        <value>[email protected]</value>
        <value>[email protected]</value>
        <value>[email protected]</value>
    </util:list>

您还可以使用 `<util:list/>`元素上的`list-class`属性显式控制实例化和填充的`List`的确切类型。 例如，如果我们确实需要实例化 `java.util.LinkedList`，我们可以使用以下配置:

    <util:list id="emails" list-class="java.util.LinkedList">
        <value>[email protected]</value>
        <value>[email protected]</value>
        <value>[email protected]</value>
        <value>d'[email protected]</value>
    </util:list>

如果未提供`list-class`属性，则容器将选择`List` 实现。

<a id="xsd-schemas-util-map"></a>

##### [](#xsd-schemas-util-map)使用 `<util:map/>`

请考虑以下示例:

    <!-- creates a java.util.Map instance with values loaded from the supplied 'sourceMap' -->
    <bean id="emails" class="org.springframework.beans.factory.config.MapFactoryBean">
        <property name="sourceMap">
            <map>
                <entry key="pechorin" value="[email protected]"/>
                <entry key="raskolnikov" value="[email protected]"/>
                <entry key="stavrogin" value="[email protected]"/>
                <entry key="porfiry" value="[email protected]"/>
            </map>
        </property>
    </bean>

上述配置使用Spring `FactoryBean`实现（`MapFactoryBean`）创建一个`java.util.Map`实例，该实例使用从提供的`'sourceMap'`获取的键值对进行初始化。

以下示例使用`<util:map/>`元素进行更简洁的表示：

    <!-- creates a java.util.Map instance with the supplied key-value pairs -->
    <util:map id="emails">
        <entry key="pechorin" value="[email protected]"/>
        <entry key="raskolnikov" value="[email protected]"/>
        <entry key="stavrogin" value="[email protected]"/>
        <entry key="porfiry" value="[email protected]"/>
    </util:map>

您还可以使用`<util:map/>`元素上的`'map-class'`属性显式控制实例化和填充的`Map`的确切类型。 例如，如果我们真的需要实例化`java.util.TreeMap` ，我们可以使用以下配置：:

    <util:map id="emails" map-class="java.util.TreeMap">
        <entry key="pechorin" value="[email protected]"/>
        <entry key="raskolnikov" value="[email protected]"/>
        <entry key="stavrogin" value="[email protected]"/>
        <entry key="porfiry" value="[email protected]"/>
    </util:map>

如果未提供`'map-class'`属性，则容器将选择`Map` 实现。

<a id="xsd-schemas-util-set"></a>

##### [](#xsd-schemas-util-set)使用 `<util:set/>`

请考虑以下示例:

    <!-- creates a java.util.Set instance with values loaded from the supplied 'sourceSet' -->
    <bean id="emails" class="org.springframework.beans.factory.config.SetFactoryBean">
        <property name="sourceSet">
            <set>
                <value>[email protected]</value>
                <value>[email protected]</value>
                <value>[email protected]</value>
                <value>[email protected]</value>
            </set>
        </property>
    </bean>

上述配置使用Spring `FactoryBean`实现（ `SetFactoryBean`）创建一个`java.util.Set`实例，该实例使用从提供的 `sourceSet`获取的值进行初始化。

以下示例使用`<util:set/>`元素进行更简洁的表示：

    <!-- creates a java.util.Set instance with the supplied values -->
    <util:set id="emails">
        <value>[email protected]</value>
        <value>[email protected]</value>
        <value>[email protected]</value>
        <value>[email protected]</value>
    </util:set>

您还可以使用`<util:set/>` 元素上的`set-class`属性显式控制实例化和填充的`Set`的确切类型。 例如，如果我们确实需要实例化`java.util.TreeSet` ，我们可以使用以下配置:

    <util:set id="emails" set-class="java.util.TreeSet">
        <value>[email protected]</value>
        <value>[email protected]</value>
        <value>[email protected]</value>
        <value>[email protected]</value>
    </util:set>

如果未提供 `set-class`属性，则容器将选择`Set`实现。

<a id="xsd-schemas-aop"></a>

#### [](#xsd-schemas-aop)9.1.2. The `aop` Schema

`aop`标签用于配置Spring中的所有AOP，包括Spring自己的基于代理的AOP框架和Spring与AspectJ AOP框架的集成。 这些标签在为[面向切面的编程](#aop)一章中全面介绍。

为了完整性起见，要使用`aop` schema中的标签,您需要在Spring XML配置文件的顶部有以下xsd：以下代码段中的文本引用了正确的schema，以便您可以使用AOP命名空间中的标签。

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:aop="http://www.springframework.org/schema/aop" xsi:schemaLocation="
            http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

        <!-- bean definitions here -->
    </beans>

<a id="xsd-schemas-context"></a>

#### [](#xsd-schemas-context)9.1.3. The `context` Schema

`context`标签处理与管道(plumbing）)有关的`ApplicationContext`配置\- 也就是说，通常不是对最终用户很重要的bean，而是在Spring中执行大量“grunt”工作的bean。 例如`BeanfactoryPostProcessors`。以下代码段引用了正确的schema，以便您可以使用`context`命名空间中的元素：:

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:context="http://www.springframework.org/schema/context" xsi:schemaLocation="
            http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

        <!-- bean definitions here -->
    </beans>

<a id="xsd-schemas-context-pphc"></a>

##### [](#xsd-schemas-context-pphc)使用 `<property-placeholder/>`

这个元素用于替代`${…​}` 的占位符，这些占位符是针对指定的属性文件（作为[Spring资源位置](#resources)）解析的。 此元素是一种便捷机制，可为您设置 [`PropertyPlaceholderConfigurer`](#beans-factory-placeholderconfigurer)。 如果您需要更多地控制`PropertyPlaceholderConfigurer`，您可以自己明确定义一个。

<a id="xsd-schemas-context-ac"></a>

##### [](#xsd-schemas-context-ac)使用 `<annotation-config/>`

此元素激活Spring基础结构以检测bean类中的注解：:

*   Spring’s [`@Required`](#beans-required-annotation) and [`@Autowired`](#beans-annotation-config)

*   JSR 250’s `@PostConstruct`, `@PreDestroy` and `@Resource` (if available)

*   JPA’s `@PersistenceContext` and `@PersistenceUnit` (if available).


或者，您可以选择显式激活这些注解的各个`BeanPostProcessors`。

这个元素没有激活处理Spring的 [`@Transactional`](data-access.html#transaction-declarative-annotations)注解。 使用[`<tx:annotation-driven/>`](data-access.html#tx-decl-explained) 来激活Spring的`@Transactional`注解。

<a id="xsd-schemas-context-component-scan"></a>

##### [](#xsd-schemas-context-component-scan)使用 `<component-scan/>`

此元素在[基于容器配置](#beans-annotation-config)中进行了详细说明。

<a id="xsd-schemas-context-ltw"></a>

##### [](#xsd-schemas-context-ltw)使用 `<load-time-weaver/>`

此元素在[加载时使用AspectJ在spring框架中](#aop-aj-ltw)中进行了详细说明。

<a id="xsd-schemas-context-sc"></a>

##### [](#xsd-schemas-context-sc)使用 `<spring-configured/>`

此元素在 [使用AspectJ来独立注入domain的object使用spring](#aop-atconfigurable)中进行了详细说明。

<a id="sd-schemas-context-mbe"></a>

##### [](#xsd-schemas-context-mbe)使用 `<mbean-export/>`

此元素在 [配置基于注解的MBean的导出](integration.html#jmx-context-mbeanexport)中进行了详细说明。

<a id="xsd-schemas-beans"></a>

#### [](#xsd-schemas-beans)9.1.4. The Beans Schema

最后，但并非最不重要的是，`beans` schema标签。这些都是相同的标签，已经在Spring框架中崭露头角。此处不显示bean架构中各种标签的示例， 因为它们在 [详细信息的依赖性和配置](#beans-factory-properties-detailed)(甚至在整个[章节](#beans)）中有相当全面的介绍。

请注意，您可以向 `<bean/>`XML定义添加零个或多个键值对。 如果有的话，使用这些额外的元数据完成的工作完全取决于您自己的自定义逻辑 （因此，如果您按照[XML Schema Authoring](#xml-custom)的附录中所述编写自己的自定义元素，通常只能使用它。

以下示例显示了周围`<bean/>`上下文中的 `<meta/>` 元素（请注意，没有任何逻辑可以解释它，元数据实际上是无用的）。

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

        <bean id="foo" class="x.y.Foo">
            <meta key="cacheName" value="foo"/> (1)
            <property name="name" value="Rick"/>
        </bean>

    </beans>

**1、**这是示例`meta`元素

在上面的示例中，您将假定有一些逻辑将使用bean定义，并通过提供的元数据设置一些缓存基础结构。

<a id="xml-custom"></a>

### [](#xml-custom)9.2. XML Schema Authoring

从版本2.0开始，Spring就为定义和配置bean的基本Spring XML格式的可扩展性提供了一种机制。 本节介绍如何编写自己的自定义XML bean定义解析器并将这些解析器集成到Spring IoC容器中。

为了便于使用架构感知的XML编辑器编写配置文件，Spring的可扩展XML配置机制基于xml Schema。如果您对Spring当前的XML配置扩展不熟悉，则应首先阅读标题为[\[xsd-config\]](#xsd-config)的附录。

要创建新的XML配置扩展：:

1.  [Author](#xsd-custom-schema)：编写xml的schema来描述您的自定义元素。

2.  [Code](#xsd-custom-namespacehandler)：编写自定义`NamespaceHandler` 实现。

3.  [Code](#xsd-custom-parser)：编写一个或多个`BeanDefinitionParser`实现（这是完成实际工作的地方）。

4.  [注册](#xsd-custom-registration) ：使用Spring注册。


下面是对每个步骤的描述。对于本例，我们将创建一个XML扩展(一个自定义XML元素），它允许我们以一种简单的方式配置`SimpleDateFormat`类型的对象(在`java.text`包中）。 当我们完成后，我们将能够定义类型 `SimpleDateFormat` 定义如下：:

    <myns:dateformat id="dateFormat"
        pattern="yyyy-MM-dd HH:mm"
        lenient="true"/>

（不要担心这个例子过于简单，后面还有很多的案例。第一个简单的案例的目的是完成基本步骤的调用）

<a id="xsd-custom-schema"></a>

#### [](#xsd-custom-schema)9.2.1. 编写Schema

创建一个用于Spring的IoC容器的XML配置扩展，首先要创建一个XML Schema来描述扩展。 对于我们的示例，我们使用以下schema来配置`SimpleDateFormat`对象：

    <!-- myns.xsd (inside package org/springframework/samples/xml) -->

    <?xml version="1.0" encoding="UTF-8"?>
    <xsd:schema xmlns="http://www.mycompany.com/schema/myns"
            xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            xmlns:beans="http://www.springframework.org/schema/beans"
            targetNamespace="http://www.mycompany.com/schema/myns"
            elementFormDefault="qualified"
            attributeFormDefault="unqualified">

        <xsd:import namespace="http://www.springframework.org/schema/beans"/>

        <xsd:element name="dateformat">
            <xsd:complexType>
                <xsd:complexContent>
                    <xsd:extension base="beans:identifiedType"> (1)
                        <xsd:attribute name="lenient" type="xsd:boolean"/>
                        <xsd:attribute name="pattern" type="xsd:string" use="required"/>
                    </xsd:extension>
                </xsd:complexContent>
            </xsd:complexType>
        </xsd:element>
    </xsd:schema>

**1、**(强调的行包含可识别的所有标签的扩展库(意味着它们具有`id`属性，将用作容器中的bean标识符）。我们可以使用此属性，因为我们导入了Spring提供的`beans`命名空间。

前面的schema允许我们使用`<myns:dateformat/>` 元素直接在XML应用程序上下文文件中配置`SimpleDateFormat`对象，如以下示例所示:

    <myns:dateformat id="dateFormat"
        pattern="yyyy-MM-dd HH:mm"
        lenient="true"/>

请注意，在我们创建基础结构类之后，前面的XML代码段与以下XML代码段基本相同:

    <bean id="dateFormat" class="java.text.SimpleDateFormat">
        <constructor-arg value="yyyy-HH-dd HH:mm"/>
        <property name="lenient" value="true"/>
    </bean>

前两个片段中的第二个在容器中创建一个bean（由名称为`SimpleDateFormat`类型的`dateFormat`标识），并设置了几个属性。

基于schema创建的配置格式可以与带有schema感知的XML编辑器的IDE集成。使用正确的创建模式，可以让用户再几个配置选择之间进行自由切换（其实说的就是eclipse编辑XML的多种视图）。

<a id="xsd-custom-namespacehandler"></a>

#### [](#xsd-custom-namespacehandler)9.2.2. 编写`NamespaceHandler`

除了schema之外，我们需要一个`NamespaceHandler`来解析Spring在解析配置文件时遇到的这个特定命名空间的所有元素。 对于此示例， `NamespaceHandler` 应该负责解析`myns:dateformat`元素。

`NamespaceHandler` 接口有三个方法：:

*   `init()`: 允许初始化`NamespaceHandler` ，在使用处理程序之前此方法将被Spring调用。

*   `BeanDefinition parse(Element, ParserContext)`: 当Spring遇到top-level元素(不嵌套在bean定义或其他命名空间中)时调用。此方法可以注册bean定义本身和/或返回bean定义。

*   `BeanDefinitionHolder decorate(Node, BeanDefinitionHolder, ParserContext)`: 当Spring遇到不同命名空间的属性或嵌套元素时调用。一个或多个bean定义的装饰将被使用， （例如）与[Spring支持的范围](#beans-factory-scopes)一起使用。 我们将首先写一个简单的例子，不使用装饰器，之后我们在一个更高级的例子中展示装饰。


尽管完全可以为整个命名空间编写自己的`NamespaceHandler` (从而提供分析命名空间中每个元素的代码）。但通常情况下，Spring XML配置文件中的每个顶级XML元素都会生成一个bean 定义(在我们的例子中， 单个 `<myns:dateformat/>`元素导致单个`SimpleDateFormat`定义）。Spring具有许多支持此方案的便捷类。在本例中，我们将使用`NamespaceHandlerSupport`类:

    package org.springframework.samples.xml;

    import org.springframework.beans.factory.xml.NamespaceHandlerSupport;

    public class MyNamespaceHandler extends NamespaceHandlerSupport {

        public void init() {
            registerBeanDefinitionParser("dateformat", new SimpleDateFormatBeanDefinitionParser());
        }

    }

您可能会注意到此类中实际上并没有很多解析逻辑。实际上， `NamespaceHandlerSupport`类具有内置的委托概念。 它支持注册任何数量的`BeanDefinitionParser`实例，当它需要解析其命名空间中的元素时，它会委托给它们。 这种关注的清晰分离使`NamespaceHandler` 能够处理对其命名空间中所有自定义元素的解析的编排，同时委托`BeanDefinitionParsers` 执行XML解析的繁琐工作。 这意味着每个`BeanDefinitionParser` 只包含解析单个自定义元素的逻辑，我们可以在下一步中看到。

<a id="xsd-custom-parser"></a>

#### [](#xsd-custom-parser)9.2.3. 使用 `BeanDefinitionParser`

如果`NamespaceHandler`遇到了已映射到特定bean定义分析器(在本例中为`dateformat` )的类型的XML元素，则将使用`BeanDefinitionParser`。换言之， `BeanDefinitionParser`负责分析在架构中定义的一个不同的顶级XML元素。在解析器中，我们将可以访问XML元素(以及它的子组件）以便我们能够解析我们的自定义XML内容。如下面的示例所示:

    package org.springframework.samples.xml;

    import org.springframework.beans.factory.support.BeanDefinitionBuilder;
    import org.springframework.beans.factory.xml.AbstractSingleBeanDefinitionParser;
    import org.springframework.util.StringUtils;
    import org.w3c.dom.Element;

    import java.text.SimpleDateFormat;

    public class SimpleDateFormatBeanDefinitionParser extends AbstractSingleBeanDefinitionParser { (1)

        protected Class getBeanClass(Element element) {
            return SimpleDateFormat.class; (2)
        }

        protected void doParse(Element element, BeanDefinitionBuilder bean) {
            // this will never be null since the schema explicitly requires that a value be supplied
            String pattern = element.getAttribute("pattern");
            bean.addConstructorArg(pattern);

            // this however is an optional property
            String lenient = element.getAttribute("lenient");
            if (StringUtils.hasText(lenient)) {
                bean.addPropertyValue("lenient", Boolean.valueOf(lenient));
            }
        }

    }

**1、**我们使用Spring提供的`AbstractSingleBeanDefinitionParser`来处理创建单个`BeanDefinition`的许多基本工作。

**2、**我们提供 `AbstractSingleBeanDefinitionParser` 超类，其类型是我们的单个`BeanDefinition`所代表的类型。

在这个简单的例子中，这就是我们需要做的一切。 我们的单个`BeanDefinition`的创建由`AbstractSingleBeanDefinitionParser`超类处理，bean定义的唯一标识符的提取和设置也是如此。

<a id="xsd-custom-registration"></a>

#### [](#xsd-custom-registration)9.2.4. 注册处理器和schema

编码完成。 剩下要做的就是让Spring XML解析基础架构了解我们的自定义元素。 我们通过在两个专用属性文件中注册我们的自定义`namespaceHandler`和自定义XSD文件来实现。 这些属性文件都放在应用程序的`META-INF`目录中。例如，可以与JAR文件中的二进制类一起分发。 Spring XML解析基础结构将通过使用这些特殊的属性文件来自动获取新的扩展，其格式将在接下来的两节中详细介绍。

<a id="xsd-custom-registration-spring-handlers"></a>

##### [](#xsd-custom-registration-spring-handlers)Writing `META-INF/spring.handlers`

名为 `spring.handlers`的属性文件包含XML Schema URI到命名空间处理程序类的映射。 对于我们的示例，我们需要编写以下内容:

http\\://www.mycompany.com/schema/myns=org.springframework.samples.xml.MyNamespaceHandler

( `:` 字符是Java属性格式的有效分隔符，因此 `:`URI中的字符需要使用反斜杠进行转义。）

键值对的第一部分(key)是与自定义命名空间扩展关联的URI，需要与自定义XSD schema中指定的`targetNamespace`属性的值完全匹配

<a id="xsd-custom-registration-spring-schemas"></a>

##### [](#xsd-custom-registration-spring-schemas)Writing 'META-INF/spring.schemas'

称为`spring.schemas`的属性文件包含xml schema位置(与xml文件中的schema声明一起使用，将schema用作`xsi:schemaLocation`属性的一部分）到类路径资源的映射。 这个文件需要阻止Spring使用绝对的默认的`EntityResolver`及要求网络访问来接收schema文件。如果在此属性文件中指定映射，Spring将在类路径中搜索schema(在本例中为 `org.springframework.samples.xml`包中的`myns.xsd`）。以下代码段显示了我们需要为自定义schema添加的行：:

http\\://www.mycompany.com/schema/myns/myns.xsd=org/springframework/samples/xml/myns.xsd

(（请记住：必须转义`:` 字符。）)

建议您在类路径上的`NamespaceHandler`和`BeanDefinitionParser`类旁边部署XSD文件（或多个文件）。

<a id="xsd-custom-using"></a>

#### [](#xsd-custom-using)9.2.5. 在Spring XML配置中使用自定义扩展

使用您自己已经实现的自定义扩展，与使用Spring提供的“自定义”扩展是没有区别的。在下面的示例中，可以使用Spring XML配置文件，以前的步骤开发自定义的`<dateformat/>`元素:

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:myns="http://www.mycompany.com/schema/myns"
        xsi:schemaLocation="
            http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.mycompany.com/schema/myns http://www.mycompany.com/schema/myns/myns.xsd">

        <!-- as a top-level bean -->
        <myns:dateformat id="defaultDateFormat" pattern="yyyy-MM-dd HH:mm" lenient="true"/> (1)

        <bean id="jobDetailTemplate" abstract="true">
            <property name="dateFormat">
                <!-- as an inner bean -->
                <myns:dateformat pattern="HH:mm MM-dd-yyyy"/>
            </property>
        </bean>

    </beans>

**1** 我们自定义的bean

<a id="xsd-custom-meat"></a>

#### [](#xsd-custom-meat)9.2.6. 更详细的例子

本节介绍自定义XML扩展的一些更详细的示例。

<a id="xsd-custom-custom-nested"></a>

##### [](#xsd-custom-custom-nested)在自定义元素中嵌套自定义元素

本节中提供的示例显示了如何编写满足以下配置目标所需的各种部件:

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:foo="http://www.foo.com/schema/component"
        xsi:schemaLocation="
            http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.foo.com/schema/component http://www.foo.com/schema/component/component.xsd">

        <foo:component id="bionic-family" name="Bionic-1">
            <foo:component name="Mother-1">
                <foo:component name="Karate-1"/>
                <foo:component name="Sport-1"/>
            </foo:component>
            <foo:component name="Rock-1"/>
        </foo:component>

    </beans>

上述配置实际上嵌套了彼此之间的自定义扩展，由上面的`<foo:component/>`元素实际配置的类是组件类(直接显示在下面)。 请注意，`Component`类如何不公开`Component`属性的setter方法。这使得使用setter注入为`components`类配置bean定义变得困难(或者说是不可能的) 。以下清单显示了`Component`类:

    package com.foo;

    import java.util.ArrayList;
    import java.util.List;

    public class Component {

        private String name;
        private List<Component> components = new ArrayList<Component> ();

        // mmm, there is no setter method for the 'components'
        public void addComponent(Component component) {
            this.components.add(component);
        }

        public List<Component> getComponents() {
            return components;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

    }

此问题的典型解决方案是创建一个自定义`FactoryBean`，用于公开`components` 属性的setter属性。 以下清单显示了这样的自定义`FactoryBean`:

    package com.foo;

    import org.springframework.beans.factory.FactoryBean;

    import java.util.List;

    public class ComponentFactoryBean implements FactoryBean<Component> {

        private Component parent;
        private List<Component> children;

        public void setParent(Component parent) {
            this.parent = parent;
        }

        public void setChildren(List<Component> children) {
            this.children = children;
        }

        public Component getObject() throws Exception {
            if (this.children != null && this.children.size() > 0) {
                for (Component child : children) {
                    this.parent.addComponent(child);
                }
            }
            return this.parent;
        }

        public Class<Component> getObjectType() {
            return Component.class;
        }

        public boolean isSingleton() {
            return true;
        }

    }

这很好用，但它向最终用户公开了很多Spring管道。 我们要做的是编写一个隐藏所有Spring管道的自定义扩展。 如果我们坚持[前面描述的步骤](#xsd-custom-introduction)，我们首先创建XSD schema来定义自定义标记的结构，如下面的清单所示:

    <?xml version="1.0" encoding="UTF-8" standalone="no"?>

    <xsd:schema xmlns="http://www.foo.com/schema/component"
            xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            targetNamespace="http://www.foo.com/schema/component"
            elementFormDefault="qualified"
            attributeFormDefault="unqualified">

        <xsd:element name="component">
            <xsd:complexType>
                <xsd:choice minOccurs="0" maxOccurs="unbounded">
                    <xsd:element ref="component"/>
                </xsd:choice>
                <xsd:attribute name="id" type="xsd:ID"/>
                <xsd:attribute name="name" use="required" type="xsd:string"/>
            </xsd:complexType>
        </xsd:element>

    </xsd:schema>

再次按照[前面描述的过程](#xsd-custom-introduction)，我们再创建一个自定义`NamespaceHandler`:

    package com.foo;

    import org.springframework.beans.factory.xml.NamespaceHandlerSupport;

    public class ComponentNamespaceHandler extends NamespaceHandlerSupport {

        public void init() {
            registerBeanDefinitionParser("component", new ComponentBeanDefinitionParser());
        }

    }

接下来是自定义`BeanDefinitionParser`。 请记住，我们正在创建描述`ComponentFactoryBean`的`BeanDefinition`。 以下清单显示了我们的自定义`BeanDefinitionParser`：:

    package com.foo;

    import org.springframework.beans.factory.config.BeanDefinition;
    import org.springframework.beans.factory.support.AbstractBeanDefinition;
    import org.springframework.beans.factory.support.BeanDefinitionBuilder;
    import org.springframework.beans.factory.support.ManagedList;
    import org.springframework.beans.factory.xml.AbstractBeanDefinitionParser;
    import org.springframework.beans.factory.xml.ParserContext;
    import org.springframework.util.xml.DomUtils;
    import org.w3c.dom.Element;

    import java.util.List;

    public class ComponentBeanDefinitionParser extends AbstractBeanDefinitionParser {

        protected AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
            return parseComponentElement(element);
        }

        private static AbstractBeanDefinition parseComponentElement(Element element) {
            BeanDefinitionBuilder factory = BeanDefinitionBuilder.rootBeanDefinition(ComponentFactoryBean.class);
            factory.addPropertyValue("parent", parseComponent(element));

            List<Element> childElements = DomUtils.getChildElementsByTagName(element, "component");
            if (childElements != null && childElements.size() > 0) {
                parseChildComponents(childElements, factory);
            }

            return factory.getBeanDefinition();
        }

        private static BeanDefinition parseComponent(Element element) {
            BeanDefinitionBuilder component = BeanDefinitionBuilder.rootBeanDefinition(Component.class);
            component.addPropertyValue("name", element.getAttribute("name"));
            return component.getBeanDefinition();
        }

        private static void parseChildComponents(List<Element> childElements, BeanDefinitionBuilder factory) {
            ManagedList<BeanDefinition> children = new ManagedList<BeanDefinition>(childElements.size());
            for (Element element : childElements) {
                children.add(parseComponentElement(element));
            }
            factory.addPropertyValue("children", children);
        }

    }

最后，需要通过修改`META-INF/spring.handlers`和`META-INF/spring.schemas`文件，在Spring XML基础结构中注册各种部件，如下所示:

\# in 'META-INF/spring.handlers'
http\\://www.foo.com/schema/component=com.foo.ComponentNamespaceHandler

\# in 'META-INF/spring.schemas'
http\\://www.foo.com/schema/component/component.xsd=com/foo/component.xsd

<a id="xsd-custom-custom-just-attributes"></a>

##### [](#xsd-custom-custom-just-attributes)自定义“Normal”元素的属性

编写自己的自定义分析器和关联的部件并不难，但有时它不是正确的做法。请考虑您需要将元数据添加到已经存在的bean定义的情况。在这种情况下，你当然不想要去写你自己的整个自定义扩展， 相反，您只想向现有的bean定义元素添加一个附加属性。

另一个例子，假设您要为服务对象定义一个bean(它不知道它）正在访问群集[JCache](https://jcp.org/en/jsr/detail?id=107),，并且您希望确保命名的JCache 实例在周围的群集。 以下清单显示了这样一个定义:

    <bean id="checkingAccountService" class="com.foo.DefaultCheckingAccountService"
            jcache:cache-name="checking.account">
        <!-- other dependencies here... -->
    </bean>

然后，我们可以在解析 `'jcache:cache-name'`属性时创建另一个`BeanDefinition`。这个`BeanDefinition`将为我们初始化命名的JCache。 我们还可以修改 `'checkingAccountService'`的现有`BeanDefinition`，以便它依赖于这个新的JCache初始化`BeanDefinition`。 以下清单显示了我们的`JCacheInitializer`:

    package com.foo;

    public class JCacheInitializer {

        private String name;

        public JCacheInitializer(String name) {
            this.name = name;
        }

        public void initialize() {
            // lots of JCache API calls to initialize the named cache...
        }

    }

现在我们可以转到自定义扩展。 首先，我们需要编写描述自定义属性的XSD schema，如下所示:

    <?xml version="1.0" encoding="UTF-8" standalone="no"?>

    <xsd:schema xmlns="http://www.foo.com/schema/jcache"
            xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            targetNamespace="http://www.foo.com/schema/jcache"
            elementFormDefault="qualified">

        <xsd:attribute name="cache-name" type="xsd:string"/>

    </xsd:schema>

接下来，我们需要创建关联的`NamespaceHandler`，如下所示:

    package com.foo;

    import org.springframework.beans.factory.xml.NamespaceHandlerSupport;

    public class JCacheNamespaceHandler extends NamespaceHandlerSupport {

        public void init() {
            super.registerBeanDefinitionDecoratorForAttribute("cache-name",
                new JCacheInitializingBeanDefinitionDecorator());
        }

    }

接下来，我们需要创建解析器。请注意，在这种情况下，因为我们要解析XML属性，所以我们编写 `BeanDefinitionDecorator` 而不是 `BeanDefinitionParser`。 以下清单显示了我们的`BeanDefinitionDecorator`:

    package com.foo;

    import org.springframework.beans.factory.config.BeanDefinitionHolder;
    import org.springframework.beans.factory.support.AbstractBeanDefinition;
    import org.springframework.beans.factory.support.BeanDefinitionBuilder;
    import org.springframework.beans.factory.xml.BeanDefinitionDecorator;
    import org.springframework.beans.factory.xml.ParserContext;
    import org.w3c.dom.Attr;
    import org.w3c.dom.Node;

    import java.util.ArrayList;
    import java.util.Arrays;
    import java.util.List;

    public class JCacheInitializingBeanDefinitionDecorator implements BeanDefinitionDecorator {

        private static final String[] EMPTY_STRING_ARRAY = new String[0];

        public BeanDefinitionHolder decorate(Node source, BeanDefinitionHolder holder,
                ParserContext ctx) {
            String initializerBeanName = registerJCacheInitializer(source, ctx);
            createDependencyOnJCacheInitializer(holder, initializerBeanName);
            return holder;
        }

        private void createDependencyOnJCacheInitializer(BeanDefinitionHolder holder,
                String initializerBeanName) {
            AbstractBeanDefinition definition = ((AbstractBeanDefinition) holder.getBeanDefinition());
            String[] dependsOn = definition.getDependsOn();
            if (dependsOn == null) {
                dependsOn = new String[]{initializerBeanName};
            } else {
                List dependencies = new ArrayList(Arrays.asList(dependsOn));
                dependencies.add(initializerBeanName);
                dependsOn = (String[]) dependencies.toArray(EMPTY_STRING_ARRAY);
            }
            definition.setDependsOn(dependsOn);
        }

        private String registerJCacheInitializer(Node source, ParserContext ctx) {
            String cacheName = ((Attr) source).getValue();
            String beanName = cacheName + "-initializer";
            if (!ctx.getRegistry().containsBeanDefinition(beanName)) {
                BeanDefinitionBuilder initializer = BeanDefinitionBuilder.rootBeanDefinition(JCacheInitializer.class);
                initializer.addConstructorArg(cacheName);
                ctx.getRegistry().registerBeanDefinition(beanName, initializer.getBeanDefinition());
            }
            return beanName;
        }

    }

最后，我们需要通过修改`META-INF/spring.handlers` 和`META-INF/spring.schemas`文件来注册Spring XML基础结构中的各种工件，如下所示:

\# in 'META-INF/spring.handlers'
http\\://www.foo.com/schema/jcache=com.foo.JCacheNamespaceHandler

\# in 'META-INF/spring.schemas'
http\\://www.foo.com/schema/jcache/jcache.xsd=com/foo/jcache.xsd