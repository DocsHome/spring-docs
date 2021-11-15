<a id="beans"></a>

[](#beans)1\. IOC容器
-------------------

这章重点说明IOC容器.

<a id="beans-introduction"></a>

### [](#beans-introduction)1.1. Spring IoC容器和bean的介绍

本章介绍Spring框架中控制反转 [Inversion of Control](https://github.com/DocsHome/spring-docs/blob/master/pages/overview/overview.md#background-ioc) 的实现. IOC与大家熟知的依赖注入同理，指的是对象仅通过构造函数参数、工厂方法的参数或在对象实例构造以后或从工厂方法返回以后，在对象实例上设置的属性来定义它们的依赖关系（即它们使用的其他对象). 然后容器在创建bean时注入这些需要的依赖。 这个过程基本上是bean本身的逆过程（因此称为IOC），通过使用类的直接构造或服务定位器模式等机制来控制其依赖项的实例化或位置。

`org.springframework.beans` 和 `org.springframework.context` 是实现Spring IOC容器框架的基础. [`BeanFactory`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/BeanFactory.html) 接口提供了一种更先进的配置机制来管理任意类型的对象. [`ApplicationContext`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/ApplicationContext.html) 是`BeanFactory`的子接口. 他提供了:

*   更容易与Spring的AOP特性集成

*   消息资源处理(用于国际化)

*   事件发布

*   应用层特定的上下文，如用于web应用程序的`WebApplicationContext`.


简而言之，`BeanFactory`提供了配置框架的基本功能，`ApplicationContext`添加了更多特定于企业的功能。 `ApplicationContext`完全扩展了`BeanFactory`的功能，这些内容将在介绍Spring IoC容器的章节专门讲解。有关使用`BeanFactory`更多信息，请参见`BeanFactory`。

在Spring中，由Spring IOC容器管理的，构成程序的骨架的对象成为Bean。bean对象是指经过IoC容器实例化，组装和管理的对象。此外，bean就是应用程序中众多对象之一 。bean和bean的依赖由容器所使用的配置元数据反射而来。

<a id="beans-basics"></a>

### [](#beans-basics)1.2. 容器概述

`org.springframework.context.ApplicationContext`是Spring IoC容器实现的代表，它负责实例化，配置和组装Bean。容器通过读取配置元数据获取有关实例化、配置和组装哪些对象的说明 。配置元数据可以使用XML、Java注解或Java代码来呈现。它允许你处理应用程序的对象与其他对象之间的互相依赖关系。

Spring提供了`ApplicationContext`接口的几个实现。 在独立应用程序中，通常创建[`ClassPathXmlApplicationContext`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/support/ClassPathXmlApplicationContext.html)或[`FileSystemXmlApplicationContext`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/support/FileSystemXmlApplicationContext.html)的实例。虽然XML一直是定义配置元数据的传统格式， 但是您可以指定容器使用Java注解或编程的方式编写元数据格式，并通过提供少量的XML配置以声明对某些额外元数据的支持。

在大多数应用场景中，不需要用户显式的编写代码来实例化IOC容器的一个或者多个实例。例如，在Web应用场景中，只需要在web.xml中添加大概8行简单的web描述样板就行了。（ [便捷的ApplicationContext实例化Web应用程序](#context-create)） 如果你使用的是基于Eclipse的[Spring Tool Suite](https://spring.io/tools/sts)开发环境，该样板配置只需点击几下鼠标或按几下键盘就能创建了。

下图展示了Spring工作方式的高级视图，应用程序的类与元数据配置相互配合，这样，在`ApplicationContext`创建和初始化后，你立即拥有一个可配置的，可执行的系统或应用程序。

![container magic](https://github.com/DocsHome/spring-docs/blob/master/pages/images/container-magic.png)

图 1\. IOC容器

<a id="beans-factory-metadata"></a>

#### [](#beans-factory-metadata)1.2.1. 配置元数据

如上图所示，Spring IOC容器使用元数据配置这种形式，这个配置元数据表示了应用开发人员告诉Spring容器以何种方式实例化、配置和组装应用程序中的对象。

配置元数据通常以简单、直观的XML格式提供，本章的大部分内容都使用这种格式来说明Spring IoC容器的关键概念和特性。

XML并不是配置元数据的唯一方式，Spring IoC容器本身是完全与元数据配置的实际格式分离的。现在，许多开发人员选择[基于Java的配置](#beans-java)来开发应用程序。

更多其他格式的元数据见:

*   [基于注解的配置](#beans-annotation-config): Spring 2.5 支持基于注解的元数据配置.

*   [基于Java的配置](#beans-java): 从 Spring 3.0开始, 由Spring JavaConfig项目提供的功能已经成为Spring核心框架的一部分。因此，你可以使用Java配置来代替XML配置定义外部bean 。要使用这些新功能，请参阅 [`@Configuration`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Configuration.html), [`@Bean`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Bean.html), [`@Import`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Import.html), 和 [`@DependsOn`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/DependsOn.html) annotations.

Spring配置至少一个（通常不止一个）由容器来管理。基于XML的元数据配置将这些bean配置为`<bean/>`元素，并放置于`<bean/>`元素内部。 典型的Java配置是在使用`@Configuration`注解过的类中，在它的方法上使用`@Bean`注解。

这些bean定义会对应到构成应用程序的实际对象。通常你会定义服务层对象，数据访问对象（DAOs），表示对象(如Struts `Action`的实例)，基础对象（如Hibernate 的`SessionFactories`, JMS `Queues`）。通常不会在容器中配置细粒度的域对象，但是，因为它的创建和加载通常是DAO和业务逻辑的任务。 但是，你可以使用Spring与AspectJ 集成独立于 IoC 容器来创建的对象，请参阅[AspectJ在Spring中进行依赖关系注入域对象](#aop-atconfigurable)

下面的示例显示了基于XML元数据配置的基本结构:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="..." class="...">  (1) (2)
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <bean id="..." class="...">
        <!-- collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions go here -->

</beans>
```

**1**

`id` 属性是字符串 ，用来识别唯一的bean定义.

**2**

`class` 属性定义了bean的类型，使用全类名.

`id`属性的值是指引用协作对象（在这个例子没有显示用于引用协作对象的XML）。请参阅[依赖](#beans-dependencies)获取更多信息

<a id="beans-factory-instantiation"></a>

#### [](#beans-factory-instantiation)1.2.2. 实例化容器

提供给ApplicationContext构造函数的路径就是实际的资源字符串，使容器能从各种外部资源(如本地文件系统、Java类路径等)装载元数据配置。

    ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

当你了解Spring IoC容器，你可能想知道更多关于Spring的抽象资源（详细描述[资源](#resources)）它提供了一种方便的，由URI语法定义的位置读取InputStream描述的方式 ，资源路径被用于构建应用程序上下文[应用环境和资源路径](#resources-app-ctx)

下面的例子显示了服务层对象(`services.xml`)配置文件::

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- services -->

    <bean id="petStore" class="org.springframework.samples.jpetstore.services.PetStoreServiceImpl">
        <property name="accountDao" ref="accountDao"/>
        <property name="itemDao" ref="itemDao"/>
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for services go here -->

</beans>
```

下面的示例显示了数据访问对象（`daos.xml`）配置文件:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="accountDao"
        class="org.springframework.samples.jpetstore.dao.jpa.JpaAccountDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <bean id="itemDao" class="org.springframework.samples.jpetstore.dao.jpa.JpaItemDao">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for data access objects go here -->

</beans>
```

在上面的例子中，服务层由PetStoreServiceImpl类，两个数据访问对象JpaAccountDao类和JpaItemDao类（基于JPA对象/关系映射标准）组成 property name元素是指JavaBean属性的名称，而ref元素引用另一个bean定义的名称。id和ref元素之间的这种联系表达了组合对象之间的相互依赖关系。有关对象间的依赖关系，请参阅[依赖](#beans-dependencies) .

<a id="beans-factory-xml-import"></a>

##### [](#beans-factory-xml-import)组合基于XML的元数据配置

使用XML配置，可以让bean定义分布在多个XML文件上，这种方法直观优雅清晰明显。通常，每个单独的XML配置文件代表架构中的一个逻辑层或模块。

你可以使用应用程序上下文构造函数从所有这些XML片段加载bean定义，这个构造函数可以输入多个资源位置，[如上一节所示](#beans-factory-instantiation)。 或者，使用`<import/>`元素也可以从另一个（或多个）文件加载bean定义。例如：

    <beans>
        <import resource="services.xml"/>
        <import resource="resources/messageSource.xml"/>
        <import resource="/resources/themeSource.xml"/>
    
        <bean id="bean1" class="..."/>
        <bean id="bean2" class="..."/>
    </beans>

上面的例子中，使用了3个文件：`services.xml`, `messageSource.xml`, 和 `themeSource.xml`来加载外部Bean的定义。 导入文件采用的都是相对路径，因此`services.xml`必须和导入文件位于同一目录或类路径中，而`messageSource.xml` 和 `themeSource.xml` 必须在导入文件的资源位置中。正如你所看到的，前面的斜线将会被忽略，但考虑到这些路径是相对的，最佳的使用是不用斜线的。 这个XML文件的内容都会被导入，包括顶级的 `<beans/>`元素，但必须遵循Spring Schema定义XML bean定义的规则 。

这种相对路径的配置是可行的，但不推荐这样做，引用在使用相对于"../"路径的父目录文件中，这样做会对当前应用程序之外的文件产生依赖关系。 特别是对于`classpath:`: URLs(例如`classpath:../services.xml`)，不建议使用此引用，因为在该引用中，运行时解析过程选择“最近的”classpath根目录，然后查看其父目录。 类路径的变化或者选择了不正确的目录都会导致此配置不可用。

您可以使用完全限定的资源位置而不是相对路径:例如，`file:C:/config/services.xml`或者`classpath:/config/services.xml`.。 但是，请注意，您正在将应用程序的配置与特定的绝对位置耦合。通常会选取间接的方式应对这种绝对路径，例如使用占位符“${…}”来解决对JVM系统属性的引用。

import 是由bean命名空间本身提供的功能。在Spring提供的XML命名空间中，如“`context`”和“`util`”命名空间，可以用于对普通bean定义进行更高级的功能配置。

<a id="groovy-bean-definition-dsl"></a>

##### [](#groovy-bean-definition-dsl)DSL定义Groovy Bean

作为从外部配置元数据的另一个示例，bean定义也可以使用Spring的Groovy DSL来定义。Grails框架有此配置实例，通常， 可以在具有以下结构的".groovy"文件中配置bean定义。例如：

```groovy
beans {
    dataSource(BasicDataSource) {
        driverClassName = "org.hsqldb.jdbcDriver"
        url = "jdbc:hsqldb:mem:grailsDB"
        username = "sa"
        password = ""
        settings = [mynew:"setting"]
    }
    sessionFactory(SessionFactory) {
        dataSource = dataSource
    }
    myService(MyService) {
        nestedBean = { AnotherBean bean ->
            dataSource = dataSource
        }
    }
}
```

这种配置风格在很大程度上等价于XML bean定义，甚至支持Spring的XML配置名称空间。它还允许通过`importBeans`指令导入XML bean定义文件。


<a id="beans-factory-client"></a>

#### [](#beans-factory-client)1.2.3. 使用容器

`ApplicationContext`是能够创建bean定义以及处理相互依赖关系的高级工厂接口，使用方法`T getBean(String name, Class<T> requiredType)`获取容器实例。

`ApplicationContext`可以读取bean定义并访问它们 如下 :

```java
// create and configure beans
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

// retrieve configured instance
PetStoreService service = context.getBean("petStore", PetStoreService.class);

// use configured instance
List<String> userList = service.getUsernameList();
```

使用Groovy配置引导看起来非常相似，只是用到不同的上下文实现类：它是Groovy感知的（但也需理解XML bean定义） 如下:

```groovy
ApplicationContext context = new GenericGroovyApplicationContext("services.groovy", "daos.groovy");
```

最灵活的变体是`GenericApplicationContext`，例如读取XML文件的`XmlBeanDefinitionReader`。如下面的示例所示:

```java
GenericApplicationContext context = new GenericApplicationContext();
new XmlBeanDefinitionReader(context).loadBeanDefinitions("services.xml", "daos.xml");
context.refresh();
```

您还可以为Groovy文件使用`GroovyBeanDefinitionReader`，如下面的示例所示:

```groovy
GenericApplicationContext context = new GenericApplicationContext();
new GroovyBeanDefinitionReader(context).loadBeanDefinitions("services.groovy", "daos.groovy");
context.refresh();
```

这一类的读取可以在同一个 `ApplicationContext`上混合使用，也可以自动匹配，如果需要可以从不同的配置源读取bean定义。

您可以使用 `getBean`来获取bean实例， `ApplicationContext`接口也可以使用其他的方法来获取bean。但是在理想情况下，应用程序代码永远不应该使用它们。 事实上，你的应用程序代码也不应该调用的`getBean()`方法，因此对Spring API没有依赖。例如，Spring与Web框架的集成为各种Web框架组件(如控制器和JSF管理bean） 提供了依赖项注入功能，从而允许开发者通过元数据声明对特定bean的依赖(例如，自动注解）。

<a id="beans-definition"></a>

### [](#beans-definition)1.3. Bean 的概述

Spring IoC容器管理一个或多个bean。这些bean是由您提供给容器的元数据配置创建的(例如，XML `<bean/>`定义的形式)。

在容器内部，这些bean定义表示为`BeanDefinition`对象，其中包含（其他信息）以下元数据

*   限定包类名称: 通常，定义的bean的实际实现类。

*   bean行为配置元素, 定义Bean的行为约束(例如作用域，生命周期回调等等）

*   bean需要引用其他bean来完成工作. 这些引用也称为协作或依赖关系.

*   其他配置用于新对象的创建，例如使用bean的数量来管理连接池，或者限制池的大小。


以下是每个bean定义的属性:

Table 1. Bean的定义  

| Property                 | 对应的章节名                                                |
| ------------------------ | ----------------------------------------------------------- |
| Class                    | [实例化Bean](#beans-factory-class)                          |
| Name                     | [命名Bean](#beans-beanname)                                 |
| Scope                    | [Bean 的作用域](#beans-factory-scopes)                      |
| Constructor arguments    | [依赖注入](#beans-factory-collaborators)                    |
| Properties               | [依赖注入](#beans-factory-collaborators)                    |
| Autowiring mode          | [自动装配](#beans-factory-autowire)                         |
| Lazy initialization mode | [懒加载Bean](#beans-factory-lazy-init)                      |
| Initialization method    | [初始化方法回调](#beans-factory-lifecycle-initializingbean) |
| Destruction method       | [销毁方法回调](#beans-factory-lifecycle-disposablebean)     |

除了bean定义包含如何创建特定的bean的信息外，`ApplicationContext`实现还允许用户在容器中注册现有的、已创建的对象。 通过`getBeanFactory()`访问到返回值为`DefaultListableBeanFactory`的实现的ApplicationContext的BeanFactory `DefaultListableBeanFactory`支持通过`registerSingleton(..)`和`registerBeanDefinition(..)`方法来注册对象。 然而，典型的应用程序只能通过元数据配置来定义bean。

为了让容器正确推断它们在自动装配和其它内置步骤，需要尽早注册Bean的元数据和手动使用单例的实例。虽然覆盖现有的元数据和现有的单例实例在某种程度上是支持的， 但是新bean在运行时(同时访问动态工厂）注册官方并不支持，可能会导致并发访问异常、bean容器中的不一致状态，或者两者兼有。

<a id="beans-beanname"></a>

#### [](#beans-beanname)1.3.1. 命名bean

每个bean都有一个或多个标识符，这些标识符在容器托管时必须是唯一的。bean通常只有一个标识符，但如果需要到的标识不止一个时，可以考虑使用别名。

在基于XML的配置中，开发者可以使用`id`属性，name属性或两者都指定bean的标识符。 `id`属性允许您指定一个`id`，通常这些名字使用字母和数字的组合(例如'myBean', 'someService'，等等），但也可以包含特殊字符。 如果你想使用bean别名，您可以在name属性上定义，使用逗号(`,`），分号(`;`)，或白色空格进行分隔。由于历史因素， 请注意，在Spring 3.1之前的版本中，id属性被定义为`xsd:ID`类型，它会限制某些字符。从3.1开始，它被定义为`xsd:string`类型。请注意，由于bean `id`的唯一性，他仍然由容器执行，不再由XML解析器执行。

您也无需提供bean的`name`或 `id`，如果没有显式地提供`name`或 `id`，容器会给bean生成唯一的名称。 然而，如果你想引用bean的名字，可以使用ref元素或使用[Service Locator](#beans-servicelocator)（服务定位器）来进行查找（此时必须提供名称）。 不使用名称的情况有： [内部bean](#beans-inner-beans)和[自动装配的协作者](#beans-factory-autowire)

Bean 的命名约定

bean的命名是按照标准的Java字段名称命名来进行的。也就是说，bean名称开始需要以小写字母开头，后面采用“驼峰式”的方法。 例如`accountManager`,`accountService`, `userDao`, `loginController`等。

一致的beans命名能够让配置更方便阅读和理解，如果你正在使用Spring AOP，当你通过bean名称应用到通知时，这种命名方式会有很大的帮助。

在类路径中进行组件扫描时， Spring 会根据上面的规则为未命名的组件生成 bean 名称，规则是：采用简单的类名，并将其初始字符转化为小写字母。 然而，在特殊情况下，当有一个以上的字符，同时第一个和第二个字符都是大写时，原来的规则仍然应该保留。这些规则与Java中定义实例的相同。 例如Spring使用的`java.beans.Introspector.decapitalize` 类。

<a id="beans-beanname-alias"></a>

##### [](#beans-beanname-alias)为外部定义的bean起别名

在对bean定义时，除了使用`id`属性指定唯一的名称外，还可以提供多个别名，这需要通过`name`属性指定。 所有这个名称都会指向同一个bean，在某些情况下提供别名非常有用，例如为了让应用每一个组件都能更容易的对公共组件进行引用。

然而，在定义bean时就指定所有的别名并不是很恰当的。有时期望能够在当前位置为那些在别处定义的bean引入别名。在XML配置文件中， 可以通过`<alias/>`元素来定义bean别名，例如：

```xml
<alias name="fromName" alias="toName"/>
```

上面示例中，在同一个容器中名为`fromName` 的bean定义，在增加别名定义后，也可以使用`toName`来引用。

例如，在子系统A中通过名字`subsystemA-dataSource`配置的数据源。在子系统B中可能通过名字 `subsystemB-dataSource`来引用。 当两个子系统构成主应用的时候，主应用可能通过名字`myApp-dataSource`引用数据源，将全部三个名字引用同一个对象，你可以将下面的别名定义添加到应用配置中：

```xml
<alias name="subsystemA-dataSource" alias="subsystemB-dataSource"/>
<alias name="subsystemA-dataSource" alias="myApp-dataSource" />
```

现在，每个组件和主应用程序都可以通过一个唯一的名称引用dataSource，并保证不与任何其他定义冲突（有效地创建命名空间），但它们引用相同的bean。 .

Java配置

如果你是用java配置， `@Bean`可以用来提供别名，详情见[使用`@Bean` 注解。](#beans-java-bean-annotation)

<a id="beans-factory-class"></a>

#### [](#beans-factory-class)1.3.2. 实例化Bean

bean定义基本上就是用来创建一个或多个对象的配置，当需要bean的时候，容器会查找配置并且根据bean定义封装的元数据来创建（或获取）实际对象。

如果你使用基于XML的配置，那么可以在`<bean/>` 元素中通过 `class`属性来指定对象类型。 class属性实际上就是`BeanDefinition`实例中的`class`属性。他通常是必需的（一些例外情况，查看 [使用实例工厂方法实例化](#beans-factory-class-instance-factory-method) 和 [Bean 定义的继承](#beans-child-bean-definitions))。有两种方式使用Class属性

*   通常情况下，会直接通过反射调用构造方法来创建bean，这种方式与Java代码的`new`创建相似。

*   通过静态工厂方法创建，类中包含静态方法。通过调用静态方法返回对象的类型可能和Class一样，也可能完全不一样。


内部类名

如果你想配置静态内部类，那么必须使用内部类的二进制名称。

例如，在`com.example`有个`SomeThing`类，这个类里面有个静态内部类`OtherThing`，这种情况下bean定义的class属性应该写作 `com.example.SomeThing$OtherThing`.

使用$字符来分隔外部类和内部类的名称

<a id="beans-factory-class-ctor"></a>

##### [](#beans-factory-class-ctor)使用构造器实例化

当您通过构造方法创建bean时，所有普通类都可以使用并与Spring兼容。也就是说，正在开发的类不需要实现任何特定接口或以特定方式编码。 只要指定bean类就足够了。但是，根据您为该特定bean使用的IoC类型，您可能需要一个默认（空）构造函数。

Spring IoC容器几乎可以管理您希望它管理的任何类。它不仅限于管理真正的JavaBeans。大多数Spring用户更喜欢管理那些只有一个默认构造函数（无参数） 和有合适的setter和getter方法的真实的JavaBeans，还可以在容器中放置更多的外部非bean形式（non-bean-style)类，例如：如果需要使用一个绝对违反JavaBean规范的遗留连接池时 Spring也是可以管理它的。

使用基于XML的配置元数据，您可以按如下方式指定bean类：:

```xml
<bean id="exampleBean" class="examples.ExampleBean"/>

<bean name="anotherExample" class="examples.ExampleBeanTwo"/>
```

给构造方法指定参数以及为bean实例化设置属性将在后面的[依赖注入](#beans-factory-collaborators)中说明。

<a id="beans-factory-class-static-factory-method"></a>

##### [](#beans-factory-class-static-factory-method)使用静态工厂方法实例化

当采用静态工厂方法创建bean时，除了需要指定class属性外，还需要通过`factory-method` 属性来指定创建bean实例的工厂方法。 Spring将会调用此方法（其可选参数接下来会介绍）返回实例对象。从这样看来，它与通过普通构造器创建类实例没什么两样。

下面的bean定义展示了如何通过工厂方法来创建bean实例。注意，此定义并未指定对象的返回类型，只是指定了该类包含的工厂方法，在这个例中， `createInstance()`必须是一个静态（static）的方法。

```xml
<bean id="clientService"
    class="examples.ClientService"
    factory-method="createInstance"/>
```

以下示例显示了一个可以使用前面的bean定义的类:

```java
public class ClientService {
    private static ClientService clientService = new ClientService();
    private ClientService() {}

    public static ClientService createInstance() {
        return clientService;
    }
}
```

给工厂方法指定参数以及为bean实例设置属性的详细内容请查阅[依赖和配置详解](#beans-factory-properties-detailed) 。

<a id="beans-factory-class-instance-factory-method"></a>

##### [](#beans-factory-class-instance-factory-method)使用实例工厂方法实例化

通过调用工厂实例的非静态方法进行实例化与通过[静态工厂方法](#beans-factory-class-static-factory-method)实例化类似， 要使用此机制，请将class属性保留为空，并在`factory-bean`属性中指定当前（或父级或祖先）容器中bean的名称，该容器包含要调用以创建对象的实例方法。 使用`factory-method`属性设置工厂方法本身的名称。以下示例显示如何配置此类bean：

```xml
<!-- the factory bean, which contains a method called createInstance() -->
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
    <!-- inject any dependencies required by this locator bean -->
</bean>

<!-- the bean to be created via the factory bean -->
<bean id="clientService"
    factory-bean="serviceLocator"
    factory-method="createClientServiceInstance"/>
```

以下示例显示了相应的Java类:

```java
public class DefaultServiceLocator {

    private static ClientService clientService = new ClientServiceImpl();

    public ClientService createClientServiceInstance() {
        return clientService;
    }
}
```

一个工厂类也可以包含多个工厂方法，如以下示例所示:

```xml
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
    <!-- inject any dependencies required by this locator bean -->
</bean>

<bean id="clientService"
    factory-bean="serviceLocator"
    factory-method="createClientServiceInstance"/>

<bean id="accountService"
    factory-bean="serviceLocator"
    factory-method="createAccountServiceInstance"/>
```

以下示例显示了相应的Java类:

```java
public class DefaultServiceLocator {

    private static ClientService clientService = new ClientServiceImpl();

    private static AccountService accountService = new AccountServiceImpl();

    public ClientService createClientServiceInstance() {
        return clientService;
    }

    public AccountService createAccountServiceInstance() {
        return accountService;
    }
}
```

这种方法表明可以通过依赖注入（DI）来管理和配置工厂bean本身。请参阅详细信息中的[依赖和配置详解](#beans-factory-properties-detailed)。

在Spring文档中，“factory bean”是指在Spring容器中配置并通过[实例](#beans-factory-class-instance-factory-method) 或 [静态工厂方法](#beans-factory-class-static-factory-method) 创建对象的bean。相比之下，FactoryBean（注意大小写）是指Spring特定的[`FactoryBean`](#beans-factory-extension-factorybean)。

<a id="beans-dependencies"></a>

### [](#beans-dependencies)1.4. 依赖

一般情况下企业应用不会只有一个对象（Spring Bean），甚至最简单的应用都需要多个对象协同工作。下一节将解释如何从定义单个Bean到让多个Bean协同工作。

<a id="beans-factory-collaborators"></a>

#### [](#beans-factory-collaborators)1.4.1. 依赖注入

依赖注入是让对象只通过构造参数、工厂方法的参数或者配置的属性来定义他们的依赖的过程。这些依赖也是其他对象所需要协同工作的对象， 容器会在创建Bean的时候注入这些依赖。整个过程完全反转了由Bean自己控制实例化或者依赖引用，所以这个过程也称之为“控制反转”

当使用了依赖注入的特性以后，会让开发者更容易管理和解耦对象之间的依赖，使代码变得更加简单。对象之间不再关注依赖，也不需要知道依赖类的位置。如此一来，开发的类更易于测试 尤其是当开发者的依赖是接口或者抽象类的情况时，开发者可以轻易地在单元测试中mock对象。

依赖注入主要使用两种方式，一种是[基于构造函数的注入](#beans-constructor-injection)，另一种的[基于Setter方法的依赖注入](#beans-setter-injection)。

<a id="beans-constructor-injection"></a>

##### [](#beans-constructor-injection)基于构造函数的依赖注入

基于构造函数的依赖注入是由IoC容器来调用类的构造函数，构造函数的参数代表这个Bean所依赖的对象。构造函数的依赖注入与调用带参数的静态工厂方法基本一样。 调用具有特定参数的静态工厂方法来构造bean几乎是等效的，本讨论同样处理构造函数和静态工厂方法的参数。下面的例子展示了一个通过构造函数来实现依赖注入的类。:

```java
public class SimpleMovieLister {

    // the SimpleMovieLister has a dependency on a MovieFinder
    private MovieFinder movieFinder;

    // a constructor so that the Spring container can inject a MovieFinder
    public SimpleMovieLister(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // business logic that actually uses the injected MovieFinder is omitted...
}
```

请注意，这个类没有什么特别之处。 它是一个POJO，它不依赖于容器特定的接口，基类或注解。

<a id="beans-factory-ctor-arguments-resolution"></a>

###### [](#beans-factory-ctor-arguments-resolution)解析构造器参数

构造函数的参数解析是通过参数的类型来匹配的。如果在Bean的构造函数参数不存在歧义，那么构造器参数的顺序也就是就是这些参数实例化以及装载的顺序。参考如下代码：

```java
package x.y;

public class ThingOne {

    public ThingOne(ThingTwo thingTwo, ThingThree thingThree) {
        // ...
    }
}
```

假设`ThingTwo`和`ThingThree`类与继承无关，也没有什么歧义。下面的配置完全可以工作正常。开发者无需再到`<constructor-arg/>`元素中指定构造函数参数的index或type

```xml
<beans>
    <bean id="thingOne" class="x.y.ThingOne">
        <constructor-arg ref="thingTwo"/>
        <constructor-arg ref="thingThree"/>
    </bean>

    <bean id="thingTwo" class="x.y.ThingTwo"/>

    <bean id="thingThree" class="x.y.ThingThree"/>
</beans>
```

当引用另一个bean时，如果类型是已知的，匹配就会工作正常（与前面的示例一样）。当使用简单类型的时候（例如`<value>true</value>`）， Spring IoC容器无法判断值的类型，所以也是无法匹配的，考虑代码：

```java
package examples;

public class ExampleBean {

    // Number of years to calculate the Ultimate Answer
    private int years;

    // The Answer to Life, the Universe, and Everything
    private String ultimateAnswer;

    public ExampleBean(int years, String ultimateAnswer) {
        this.years = years;
        this.ultimateAnswer = ultimateAnswer;
    }
}
```

构造函数参数类型匹配

在前面的场景中，如果使用 type 属性显式指定构造函数参数的类型，则容器可以使用与简单类型的类型匹配。如下例所示：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg type="int" value="7500000"/>
    <constructor-arg type="java.lang.String" value="42"/>
</bean>
```

构造函数参数索引

您可以使用`index`属性显式指定构造函数参数的索引，如以下示例所示：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg index="0" value="7500000"/>
    <constructor-arg index="1" value="42"/>
</bean>
```

除了解决多个简单值的歧义之外，指定索引还可以解决构造函数具有相同类型的两个参数的歧义。

index 从0开始。

构造函数参数名称

您还可以使用构造函数参数名称消除歧义，如以下示例所示：:

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg name="years" value="7500000"/>
    <constructor-arg name="ultimateAnswer" value="42"/>
</bean>
```

需要注意的是，解析这个配置的代码必须启用了调试标记来编译，这样Spring才可以从构造函数查找参数名称。开发者也可以使用[@ConstructorProperties](https://download.oracle.com/javase/6/docs/api/java/beans/ConstructorProperties.html)注解来显式声明构造函数的名称。 例如下面代码:

```java
package examples;

public class ExampleBean {

    // Fields omitted

    @ConstructorProperties({"years", "ultimateAnswer"})
    public ExampleBean(int years, String ultimateAnswer) {
        this.years = years;
        this.ultimateAnswer = ultimateAnswer;
    }
}
```

<a id="beans-setter-injection"></a>

##### [](#beans-setter-injection)基于setter方法的依赖注入

基于setter函数的依赖注入是让容器调用拥有Bean的无参构造函数，或者`无参静态工厂方法`，然后再来调用setter方法来实现依赖注入。

下面的例子展示了使用setter方法进行的依赖注入的过程。其中类对象只是简单的POJO，它不依赖于容器特定的接口，基类或注解。

```java
public class SimpleMovieLister {

    // the SimpleMovieLister has a dependency on the MovieFinder
    private MovieFinder movieFinder;

    // a setter method so that the Spring container can inject a MovieFinder
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // business logic that actually uses the injected MovieFinder is omitted...
}
```

`ApplicationContext`所管理Bean同时支持基于构造函数和基于setter方法的依赖注入，同时也支持使用setter方法在通过构造函数注入依赖之后再次注入依赖。 开发者在`BeanDefinition`中可以使用`PropertyEditor`实例来自由选择注入方式。然而，大多数的开发者并不直接使用这些类，而是更喜欢使用XML配置来进行bean定义， 或者基于注解的组件（例如使用 `@Component`,`@Controller`等），或者在配置了`@Configuration`的类上面使用`@Bean`的方法。 然后，这些源在内部转换为 `BeanDefinition`的实例，并用于加载整个Spring IoC容器实例。

如何选择基于构造器和基于setter方法?

因为开发者可以混用两种依赖注入方式，两种方式用于处理不同的情况：必要的依赖通常通过构造函数注入，而可选的依赖则通过setter方法注入。其中，在setter方法上添加[@Required](#beans-required-annotation) 注解可用于构造必要的依赖。

Spring团队推荐使用基于构造函数的注入，因为这种方式会促使开发者将组件开发成不可变对象并且确保注入的依赖不为`null`。另外，基于构造函数的注入的组件被客户端调用的时候也已经是完全构造好的 。当然，从另一方面来说，过多的构造函数参数也是非常糟糕的代码方式，这种方式说明类附带了太多的功能，最好重构将不同职能分离。

基于setter的注入只用于可选的依赖，但是也最好配置一些合理的默认值。否则，只能对代码的依赖进行非null值检查了。基于setter方法的注入有一个便利之处是：对象可以重新配置和重新注入。 因此，使用setter注入管理 [JMX MBeans](https://github.com/DocsHome/spring-docs/blob/master/pages/integration/integration.md#jmx) 是很方便的

依赖注入的两种风格适合大多数的情况，但是在使用第三方库的时候，开发者可能并没有源码，那么就只能使用基于构造函数的依赖注入了。

<a id="beans-dependency-resolution"></a>

##### [](#beans-dependency-resolution)决定依赖的过程

容器解析Bean的过程如下:

*   创建并根据描述的元数据来实例化`ApplicationContext`，元数据配置可以是XML文件、Java代码或者注解。

*   每一个Bean的依赖都通过构造函数参数或属性，或者静态工厂方法的参数等等来表示。这些依赖会在Bean创建的时候装载和注入

*   每一个属性或者构造函数的参数都是真实定义的值或者引用容器其他的Bean.

*   每一个属性或者构造参数可以根据指定的类型转换为所需的类型。Spring也可以将String转成默认的Java内置类型。例如`int`, `long`, `String`, `boolean`等 。


Spring容器会在容器创建的时候针对每一个Bean进行校验。但是Bean的属性在Bean没有真正创建之前是不会进行配置的，单例类型的Bean是容器创建的时候配置成预实例状态的。 [Bean的作用域](#beans-factory-scopes)后面再说，其他的Bean都只有在请求的时候，才会创建，显然创建Bean对象会有一个依赖顺序图，这个图表示Bean之间的依赖关系。 容器根据此来决定创建和配置Bean的顺序。

循环依赖

如果开发者主要使用基于构造函数的依赖注入，那么很有可能出现循环依赖的情况。

例如：类A在构造函数中依赖于类B的实例，而类B的构造函数又依赖类A的实例。如果这样配置类A和类B相互注入的话，Spring IoC容器会发现这个运行时的循环依赖， 并且抛出`BeanCurrentlyInCreationException`。

开发者可以选择setter方法来配置依赖注入，这样就不会出现循环依赖的情况。或者根本就不使用基于构造函数的依赖注入，而仅仅使用基于setter方法的依赖注入。 换言之，但是开发者可以将循环依赖配置为基于Setter方法的依赖注入（尽管不推荐这样做）

不像典型的例子（没有循环依赖的情况），bean A和bean B之间的循环依赖关系在完全初始化之前就已经将其中一个bean注入到另一个中了（典型的鸡和鸡蛋的故事）

你可以信任Spring做正确的事。它在容器加载时检测配置问题，例如对不存在的bean和循环依赖的引用。 当实际创建bean时，Spring会尽可能晚地设置属性并解析依赖项。这也意味着Spring容器加载正确后会在bean注入依赖出错的时候抛出异常。例如，bean抛出缺少属性或者属性不合法的异常 ，这种延迟的解析也是`ApplicationContext` 的实现会令单例Bean处于预实例化状态的原因。这样，通过`ApplicationContext`创建bean，可以在真正使用bean之前消耗一些内存代价而发现配置的问题 。开发者也可以覆盖默认的行为让单例bean延迟加载，而不总是处于预实例化状态。

如果不存在循环依赖的话，bean所引用的依赖会预先全部构造。举例来说，如果bean A依赖于bean B，那么Spring IoC容器会先配置bean B，然后调用bean A的setter方法来构造bean A。 换言之，bean先会实例化，然后再注入依赖，最后才是相关生命周期方法的调用（就像配置文件的[初始化方法](#beans-factory-lifecycle-initializingbean) 或者[InitializingBean的回调函数](#beans-factory-lifecycle-initializingbean)那样）。

<a id="beans-some-examples"></a>

##### [](#beans-some-examples)依赖注入的例子

下面的例子使用基于XML的元数据配置，然后使用setter方式进行依赖注入。下面是Spring中使用XML文件声明bean定义的片段：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <!-- setter injection using the nested ref element -->
    <property name="beanOne">
        <ref bean="anotherExampleBean"/>
    </property>

    <!-- setter injection using the neater ref attribute -->
    <property name="beanTwo" ref="yetAnotherBean"/>
    <property name="integerProperty" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```

以下示例显示了相应的 `ExampleBean`类：

```java
public class ExampleBean {

    private AnotherBean beanOne;

    private YetAnotherBean beanTwo;

    private int i;

    public void setBeanOne(AnotherBean beanOne) {
        this.beanOne = beanOne;
    }

    public void setBeanTwo(YetAnotherBean beanTwo) {
        this.beanTwo = beanTwo;
    }

    public void setIntegerProperty(int i) {
        this.i = i;
    }
}
```

在前面的示例中，setter被声明为与XML文件中指定的属性匹配。以下示例使用基于构造函数的DI：

```xml
<bean id="exampleBean" class="examples.ExampleBean">
    <!-- constructor injection using the nested ref element -->
    <constructor-arg>
        <ref bean="anotherExampleBean"/>
    </constructor-arg>

    <!-- constructor injection using the neater ref attribute -->
    <constructor-arg ref="yetAnotherBean"/>

    <constructor-arg type="int" value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```

以下示例显示了相应的`ExampleBean`类：

```java
public class ExampleBean {

    private AnotherBean beanOne;

    private YetAnotherBean beanTwo;

    private int i;

    public ExampleBean(
        AnotherBean anotherBean, YetAnotherBean yetAnotherBean, int i) {
        this.beanOne = anotherBean;
        this.beanTwo = yetAnotherBean;
        this.i = i;
    }
}
```

bean定义中指定的构造函数参数用作 `ExampleBean`的构造函数的参数。

现在考虑这个示例的变体，其中，不使用构造函数，而是告诉Spring调用静态工厂方法来返回对象的实例：

```xml
<bean id="exampleBean" class="examples.ExampleBean" factory-method="createInstance">
    <constructor-arg ref="anotherExampleBean"/>
    <constructor-arg ref="yetAnotherBean"/>
    <constructor-arg value="1"/>
</bean>

<bean id="anotherExampleBean" class="examples.AnotherBean"/>
<bean id="yetAnotherBean" class="examples.YetAnotherBean"/>
```

以下示例显示了相应的`ExampleBean`类：

```java
public class ExampleBean {

    // a private constructor
    private ExampleBean(...) {
        ...
    }

    // a static factory method; the arguments to this method can be
    // considered the dependencies of the bean that is returned,
    // regardless of how those arguments are actually used.
    public static ExampleBean createInstance (
        AnotherBean anotherBean, YetAnotherBean yetAnotherBean, int i) {

        ExampleBean eb = new ExampleBean (...);
        // some other operations...
        return eb;
    }
}
```

`静态工厂方法`的参数由`<constructor-arg/>`元素提供，与实际使用的构造函数完全相同。工厂方法返回类的类型不必与包含`静态工厂方法` 的类完全相同， 尽管在本例中是这样。实例（非静态）工厂方法的使用方式也是相似的（除了使用`factory-bean`属性而不是`class`属性。因此此处不在展开讨论。

<a id="beans-factory-properties-detailed"></a>

#### [](#beans-factory-properties-detailed)1.4.2. 依赖和配置的细节

[如上一节所述](#beans-factory-collaborators)，您可以将bean的属性和构造函数参数定义为对其他bean的引用，或者作为其内联定义的值。Spring可以允许您在基于XML的配置元数据（定义Bean）中使用子元素`<property/>` and `<constructor-arg/>` 来达到这种目的。

<a id="beans-value-element"></a>

##### [](#beans-value-element)直接值（基本类型，String等等）

`<property/>` 元素的`value` 属性将属性或构造函数参数指定为人类可读的字符串表示形式， Spring的[conversion service](#core-convert-ConversionService-API)用于将这些值从`String` 转换为属性或参数的实际类型。 以下示例显示了要设置的各种值：

```xml
<bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <!-- results in a setDriverClassName(String) call -->
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/mydb"/>
    <property name="username" value="root"/>
    <property name="password" value="masterkaoli"/>
</bean>
```

以下示例使用[p命名空间](#beans-p-namespace)进行更简洁的XML配置：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource"
        destroy-method="close"
        p:driverClassName="com.mysql.jdbc.Driver"
        p:url="jdbc:mysql://localhost:3306/mydb"
        p:username="root"
        p:password="masterkaoli"/>

</beans>
```

前面的XML更简洁。 但是因为属性的类型是在运行时确定的，而非设计时确定的。所有有可能在运行时发现拼写错误。，除非您在创建bean定义时使用支持自动属性完成的IDE（例如[IntelliJ IDEA](http://www.jetbrains.com/idea/) or the [Spring Tool Suite](https://spring.io/tools/sts)）。 所以，强烈建议使用此类IDE帮助。

你也可以配置一个`java.util.Properties` 的实例，如下：

```xml
<bean id="mappings"
    class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">

    <!-- typed as a java.util.Properties -->
    <property name="properties">
        <value>
            jdbc.driver.className=com.mysql.jdbc.Driver
            jdbc.url=jdbc:mysql://localhost:3306/mydb
        </value>
    </property>
</bean>
```

Spring的容器会将`<value/>`里面的文本通过JavaBean的`PropertyEditor` 机制转换成`java.util.Properties` 实例， 这种嵌套`<value/>`元素的快捷方式也是Spring团队推荐使用的。

<a id="beans-idref-element"></a>

###### [](#beans-idref-element)`idref` 元素

`idref` 元素只是一种防错方法，可以将容器中另一个bean的`id`（字符串值 \- 而不是引用）传递给`<constructor-arg/>`或`<property/>`元素：

```xml
<bean id="theTargetBean" class="..."/>

<bean id="theClientBean" class="...">
    <property name="targetName">
        <idref bean="theTargetBean"/>
    </property>
</bean>
```

前面的bean定义代码段运行时与以下代码段完全等效：

```xml
<bean id="theTargetBean" class="..." />

<bean id="client" class="...">
    <property name="targetName" value="theTargetBean"/>
</bean>
```

Spring团队更推荐第一种方式，因为使用了`idref` 标签，它会让容器在部署阶段就对bean进行校验，以确保bean一定存在。而使用第二种方式的话，是没有任何校验的。 只有实际上引用了`client` bean的`targetName`属性，不对其值进行校验。在实例化client的时候才会被发现。如果client是[prototype](#beans-factory-scopes)类型的Bean的话，那么类似拼写之类的错误会在容器部署以后很久才能发现。

`idref`元素的`local`属性在Spring 4.0以后的xsd中已经不再支持了，而是使用了bean引用。如果更新了版本的话，只要将`idref local`引用都转换成`idref bean` 即可。

在`ProxyFactoryBean`定义中，元素所携带的值在[AOP拦截器a>的配置中很常见（至少在Spring 2.0之前的版本是这样）。在指定拦截器名称时使用`<idref/>` 元素可防止拦截器漏掉id或拼写错误。](#aop-pfb-1)

<a id="beans-ref-element"></a>

##### [](#aop-pfb-1)[](#beans-ref-element)引用其他的Bean（装配）

`ref` 元素是`<constructor-arg/>` 或 or `<property/>`定义元素中的最后一个元素。 你可以通过这个标签配置一个bean来引用另一个bean。当需要引用一个bean的时候，被引用的bean会先实例化，然后配置属性，也就是引用的依赖。如果该bean是单例bean的话 ，那么该bean会早由容器初始化。最终会引用另一个对象的所有引用，bean的范围以及校验取决于你是否有通过`bean`, `local,` or `parent` 这些属性来指定对象的id或者name属性。

通过指定 `bean`属性中的`<ref/>`元素来指定依赖是最常见的一种方式，可以引用容器或者父容器中的bean，不在同一个XML文件定义也可以引用。 其中`bean` 属性中的值可以和其他引用`bean` 中的`id`属性一致，或者和其中的某个`name` 属性一致，以下示例显示如何使用ref元素：：

    <ref bean="someBean"/>

通过指定bean的`parent`属性可以创建一个引用到当前容器的父容器之中。`parent`属性的值可以与目标bean的`id`属性一致，或者和目标bean的`name`属性中的某个一致，目标bean必须是当前引用目标bean容器的父容器 。开发者一般只有在存在层次化容器，并且希望通过代理来包裹父容器中一个存在的bean的时候才会用到这个属性。 以下一对列表显示了如何使用父属性:

```xml
<!-- in the parent context -->
<bean id="accountService" class="com.something.SimpleAccountService">
    <!-- insert dependencies as required as here -->
</bean>

<!-- in the child (descendant) context -->
<bean id="accountService" <!-- bean name is the same as the parent bean -->
    class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target">
        <ref parent="accountService"/> <!-- notice how we refer to the parent bean -->
    </property>
    <!-- insert other configuration and dependencies as required here -->
</bean>
```

与idref标签一样，`ref`元素中的`local` 标签在xsd 4.0，以后已经不再支持了，开发者可以通过将已存在的`ref local`改为`ref bean` 来完成Spring版本升级。

<a id="beans-inner-beans"></a>

##### [](#beans-inner-beans)内部bean

定义在`<bean/>`元素的`<property/>`或者`<constructor-arg/>` 元素之内的bean叫做内部bean，如下例所示：:

```xml
<bean id="outer" class="...">
    <!-- instead of using a reference to a target bean, simply define the target bean inline -->
    <property name="target">
        <bean class="com.example.Person"> <!-- this is the inner bean -->
            <property name="name" value="Fiona Apple"/>
            <property name="age" value="25"/>
        </bean>
    </property>
</bean>
```

内部bean定义不需要定义的ID或名称。如果指定，则容器不使用此类值作为标识符。容器还会在创建时忽略`scope` 标签，因为内部bean始终是匿名的，并且始终使用外部bean创建。 开发者是无法将内部bean注入到外部bean以外的其他bean中的。

作为一个极端情况，可以从自定义范围接收销毁回调 ， 例如：请求范围的内部bean包含了单例bean，那么内部bean实例会绑定到包含的bean，而包含的bean允许访问request的scope生命周期。 这种场景并不常见，内部bean通常只是供给它的外部bean使用。

<a id="beans-collection-elements"></a>

##### [](#beans-collection-elements)集合

在`<list/>`, `<set/>`, `<map/>`, 和 `<props/>`元素中，您可以分别配置Java集合类型 `List`, `Set`, `Map`, and `Properties`的属性和参数。 以下示例显示了如何使用它们:

```xml
<bean id="moreComplexObject" class="example.ComplexObject">
    <!-- results in a setAdminEmails(java.util.Properties) call -->
    <property name="adminEmails">
        <props>
            <prop key="administrator">administrator@example.org</prop>
            <prop key="support">support@example.org</prop>
            <prop key="development">development@example.org</prop>
        </props>
    </property>
    <!-- results in a setSomeList(java.util.List) call -->
    <property name="someList">
        <list>
            <value>a list element followed by a reference</value>
            <ref bean="myDataSource" />
        </list>
    </property>
    <!-- results in a setSomeMap(java.util.Map) call -->
    <property name="someMap">
        <map>
            <entry key="an entry" value="just some string"/>
            <entry key ="a ref" value-ref="myDataSource"/>
        </map>
    </property>
    <!-- results in a setSomeSet(java.util.Set) call -->
    <property name="someSet">
        <set>
            <value>just some string</value>
            <ref bean="myDataSource" />
        </set>
    </property>
</bean>
```

当然，map的key或者value，或者集合的value都可以配置为下列元素之一:

    bean | ref | idref | list | set | map | props | value | null

<a id="beans-collection-elements-merging"></a>

###### [](#beans-collection-elements-merging)集合的合并

Spring的容器也支持集合合并，开发者可以定义父样式的`<list/>`, `<map/>`, `<set/>` 或 `<props/>`元素， 同时有子样式的`<list/>`, `<map/>`, `<set/>` 或 `<props/>`元素。也就是说，子集合的值是父元素和子元素集合的合并值。

有关合并的这一节讨论父子bean机制，不熟悉父和子bean定义的读者可能希望在继续之前阅读[相关部分](#beans-child-bean-definitions)。

以下示例演示了集合合并：:

```xml
<beans>
    <bean id="parent" abstract="true" class="example.ComplexObject">
        <property name="adminEmails">
            <props>
                <prop key="administrator">administrator@example.com</prop>
                <prop key="support">support@example.com</prop>
            </props>
        </property>
    </bean>
    <bean id="child" parent="parent">
        <property name="adminEmails">
            <!-- the merge is specified on the child collection definition -->
            <props merge="true">
                <prop key="sales">sales@example.com</prop>
                <prop key="support">support@example.co.uk</prop>
            </props>
        </property>
    </bean>
<beans>
```

请注意，在`child` bean 定义的`adminEmails`中的`<props/>`使用`merge=true` 属性。 当容器解析并实例化`child` bean时，生成的实例有一个`adminEmails`属性集合， 其实例中包含的`adminEmails`集合就是child的`adminEmails`以及parent的`adminEmails`集合。以下清单显示了结果： :

```
administrator=administrator@example.com
sales=sales@example.com
support=support@example.co.uk
```

子`属性`集合的值集继承父`<props/>`的所有属性元素，子值的`支持`值覆盖父集合中的值。

这个合并的行为和`<list/>`, `<map/>`, 和 `<set/>`之类的集合类型的行为是类似的。`List` 在特定例子中，与List集合类型类似， 有着隐含的 `ordered`概念。所有的父元素里面的值，是在所有子元素的值之前配置的。但是像`Map`, `Set`, 和 `Properties`的集合类型，是不存在顺序的。

<a id="beans-collection-merge-limitations"></a>

###### [](#beans-collection-merge-limitations)集合合并的限制

您不能合并不同类型的集合（例如要将`Map` 和`List`合并是不可能的）。如果开发者硬要这样做就会抛出异常， `merge`的属性是必须特指到更低级或者继承的子节点定义上， 特指merge属性到父集合的定义上是冗余的，而且在合并上也没有任何效果。

<a id="beans-collection-elements-strongly-typed"></a>

###### [](#beans-collection-elements-strongly-typed)强类型的集合

在Java 5以后，开发者可以使用强类型的集合了。也就是，开发者可以声明`Collection`类型，然后这个集合只包含`String`元素（举例来说）。 如果开发者通过Spring来注入强类型的Collection到bean中，开发者就可以利用Spring的类型转换支持来做到 以下Java类和bean定义显示了如何执行此操作:

```xml
public class SomeClass {

    private Map<String, Float> accounts;

    public void setAccounts(Map<String, Float> accounts) {
        this.accounts = accounts;
    }
}

<beans>
    <bean id="something" class="x.y.SomeClass">
        <property name="accounts">
            <map>
                <entry key="one" value="9.99"/>
                <entry key="two" value="2.75"/>
                <entry key="six" value="3.99"/>
            </map>
        </property>
    </bean>
</beans>
```

当`something`的属性`accounts`准备注入的时候，accounts的泛型信息Map`Map<String, Float>` 就会通过反射拿到。 这样，Spring的类型转换系统能够识别不同的类型，如上面的例子`Float`然后会将字符串的值`9.99, 2.75`, 和`3.99`转换成对应的`Float`类型。

<a id="beans-null-element"></a>

##### [](#beans-null-element)Null and 空字符串

`Strings`将属性的空参数视为空字符串。下面基于XML的元数据配置就会将`email` 属性配置为String的值。

```xml
<bean class="ExampleBean">
    <property name="email" value=""/>
</bean>
```

上面的示例等效于以下Java代码:

```java
exampleBean.setEmail("");
```

`<null/>`将被处理为null值。以下清单显示了一个示例:

```xml
<bean class="ExampleBean">
    <property name="email">
        <null/>
    </property>
</bean>
```

上述配置等同于以下Java代码：

```java
exampleBean.setEmail(null);
```

<a id="beans-p-namespace"></a>

##### [](#beans-p-namespace)使用p命名空间简化XML配置

p命名空间让开发者可以使用`bean`的属性，而不必使用嵌套的`<property/>`元素。

Spring是支持基于XML的格式化[命名空间](#xsd-schemas)扩展的。本节讨论的`beans` 配置都是基于XML的，p命名空间是定义在Spring Core中的（不是在XSD文件）。

以下示例显示了两个XML片段（第一个使用标准XML格式，第二个使用p命名空间），它们解析为相同的结果：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean name="classic" class="com.example.ExampleBean">
        <property name="email" value="someone@somewhere.com"/>
    </bean>

    <bean name="p-namespace" class="com.example.ExampleBean"
        p:email="someone@somewhere.com"/>
</beans>
```

上面的例子在bean中定义了`email`的属性。这种定义告知Spring这是一个属性声明。如前面所描述的，p命名空间并没有标准的定义模式，所以开发者可以将属性的名称配置为依赖名称。

下一个示例包括另外两个bean定义，它们都引用了另一个bean:

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean name="john-classic" class="com.example.Person">
        <property name="name" value="John Doe"/>
        <property name="spouse" ref="jane"/>
    </bean>

    <bean name="john-modern"
        class="com.example.Person"
        p:name="John Doe"
        p:spouse-ref="jane"/>

    <bean name="jane" class="com.example.Person">
        <property name="name" value="Jane Doe"/>
    </bean>
</beans>
```

此示例不仅包含使用p命名空间的属性值，还使用特殊格式来声明属性引用。第一个bean定义使用`<property name="spouse" ref="jane"/>`来创建从bean `john`到bean `jane`的引用， 而第二个bean定义使用`p:spouse-ref="jane"`来作为指向bean的引用。在这个例子中 `spouse`是属性的名字，而`-ref`部分表名这个依赖不是直接的类型，而是引用另一个bean。

p命名空间并不如标准XML格式灵活。例如，声明属性的引用可能和一些以`Ref`结尾的属性相冲突，而标准的XML格式就不会。Spring团队推荐开发者能够和团队商量一下，协商使用哪一种方式，而不要同时使用三种方法。

<a id="beans-c-namespace"></a>

##### [](#beans-c-namespace)使用c命名空间简化XML

与 [p命名空间](#beans-p-namespace)类似，c命名空间是在Spring 3.1首次引入的，c命名空间允许使用内联的属性来配置构造参数而不必使用`constructor-arg` 。

以下示例使用`c:`命名空间的例子来执行与[基于Constructor的依赖注入](#beans-constructor-injection)相同的操作：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:c="http://www.springframework.org/schema/c"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="thingOne" class="x.y.ThingTwo"/>
    <bean id="thingTwo" class="x.y.ThingThree"/>

    <!-- traditional declaration -->
    <bean id="thingOne" class="x.y.ThingOne">
        <constructor-arg ref="thingTwo"/>
        <constructor-arg ref="thingThree"/>
        <constructor-arg value="something@somewhere.com"/>
    </bean>

    <!-- c-namespace declaration -->
    <bean id="thingOne" class="x.y.ThingOne" c:thingTwo-ref="thingTwo" c:thingThree-ref="thingThree" c:email="something@somewhere.com"/>

</beans>
```

`c:`:命名空间使用了和`p:` :命名空间相类似的方式（使用了`-ref` 来配置引用).而且,同样的,c命名空间也是定义在Spring Core中的（不是XSD模式).

在少数的例子之中,构造函数的参数名字并不可用（通常,如果字节码没有debug信息的编译),你可以使用回调参数的索引，如下面的例子:

    <!-- c-namespace index declaration -->
    <bean id="thingOne" class="x.y.ThingOne" c:_0-ref="thingTwo" c:_1-ref="thingThree"/>

由于XML语法，索引表示法需要使用`_`作为属性名字的前缀，因为XML属性名称不能以数字开头（即使某些IDE允许它）。

实际上，[构造函数解析机制](#beans-factory-ctor-arguments-resolution)在匹配参数方面非常有效，因此除非您确实需要，否则我们建议在整个配置中使用名称表示法。

<a id="beans-compound-property-names"></a>

##### [](#beans-compound-property-names)组合属性名

开发者可以配置混合的属性，只需所有的组件路径（除了最后一个属性名字）不能为`null`即可。参考如下定义：

```xml
<bean id="something" class="things.ThingOne">
    <property name="fred.bob.sammy" value="123" />
</bean>
```

`something`有 `fred`的属性，而其中`fred`属性有`bob`属性，而`bob`属性之中有`sammy`属性，那么最后这个`sammy`属性会配置为123。 想要上述的配置能够生效，`fred`属性需要有`bob`属性而且在`fred`构造之后不为null即可。

<a id="beans-factory-dependson"></a>

#### [](#beans-factory-dependson)1.4.3. 使用 `depends-on`属性

如果一个bean是另一个bean的依赖，通常这个bean也就是另一个bean的属性之一。多数情况下，开发者可以在配置XML元数据的时候使用[`<ref/>`](#beans-ref-element)标签。 然而，有时bean之间的依赖不是直接关联的。例如：需要调用类的静态实例化器来触发依赖，类似数据库驱动注册。`depends-on`属性可以显式强制初始化一个或多个bean。 以下示例使用depends-on属性表示对单个bean的依赖关系:

```xml
<bean id="beanOne" class="ExampleBean" depends-on="manager"/>
<bean id="manager" class="ManagerBean" />
```

如果想要依赖多个bean，可以提供多个名字作为`depends-on`的值。以逗号、空格或者分号分割:

```xml
<bean id="beanOne" class="ExampleBean" depends-on="manager,accountDao">
    <property name="manager" ref="manager" />
</bean>

<bean id="manager" class="ManagerBean" />
<bean id="accountDao" class="x.y.jdbc.JdbcAccountDao" />
```

`depends-on`属性既可以指定初始化时间依赖性，也可以指定单独的bean，相应的销毁时间的依赖。独立定义了`depends-on`属性的bean会优先销毁 （相对于`depends-on`的bean销毁，这样`depends-on`可以控制销毁的顺序。

<a id="beans-factory-lazy-init"></a>

#### [](#beans-factory-lazy-init)1.4.4. 懒加载Bean

默认情况下， `ApplicationContext` 会在实例化的过程中创建和配置所有的单例[singleton](#beans-factory-scopes-singleton) bean。总的来说， 这个预初始化是很不错的。因为这样能及时发现环境上的一些配置错误，而不是系统运行了很久之后才发现。如果这个行为不是迫切需要的，开发者可以通过将Bean标记为延迟加载就能阻止这个预初始化 懒加载bean会通知IoC不要让bean预初始化而是在被引用的时候才会实例化。

在XML中，此行为由`<bean/>`元素上的`lazy-init` 属性控制，如以下示例所示：

```xml
<bean id="lazy" class="com.something.ExpensiveToCreateBean" lazy-init="true"/>
<bean name="not.lazy" class="com.something.AnotherBean"/>
```

当将bean配置为上述XML的时候， `ApplicationContext`之中的`lazy` bean是不会随着 `ApplicationContext`的启动而进入到预初始化状态的。 只有那些非延迟加载的bean是处于预初始化的状态的。

然而，如果延迟加载的类是作为单例非延迟加载的bean的依赖而存在的话，`ApplicationContext`仍然会在`ApplicationContext`启动的时候加载。 因为作为单例bean的依赖，会随着单例bean的实例化而实例化。

您还可以使用`<beans/>`元素上的`default-lazy-init`属性在容器级别控制延迟初始化，以下示例显示：

```xml
<beans default-lazy-init="true">
    <!-- no beans will be pre-instantiated... -->
</beans>
```

<a id="beans-factory-autowire"></a>

#### [](#beans-factory-autowire)1.4.5. 自动装配

Spring容器可以根据bean之间的依赖关系自动装配，开发者可以让Spring通过`ApplicationContext`来自动解析这些关联，自动装载有很多优点:

*   自动装载能够明显的减少指定的属性或者是构造参数。（在本章[其他地方讨论](#beans-child-bean-definitions)的其他机制，如bean模板，在这方面也很有价值。）

*   自动装载可以扩展开发者的对象，比如说，如果开发者需要加一个依赖，只需关心如何更改配置即可自动满足依赖关联。这样，自动装载在开发过程中是极其高效的，无需明确选择装载的依赖会使系统更加稳定


使用基于XML的配置元数据（请参阅[依赖注入](#beans-factory-collaborators)）时，可以使用`<bean/>` 元素的`autowire`属性为bean定义指定autowire模式。 自动装配功能有四种方式。开发者可以指定每个bean的装配方式，这样bean就知道如何加载自己的依赖。下表描述了四种自动装配模式：

Table 2. 装配模式

| 模式          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| `no`          | (默认) 不自动装配。Bean引用必须由 `ref` 元素定义，对于比较大的项目的部署，不建议修改默认的配置 ，因为明确指定协作者可以提供更好的控制和清晰度。在某种程度上，它记录了系统的结构。 |
| `byName`      | 按属性名称自动装配。 Spring查找与需要自动装配的属性同名的bean。 例如，如果bean配置为根据名字装配，他包含 的属性名字为`master`（即，它具有`setMaster(..)`方法），则Spring会查找名为 `master` 的bean定义并使用它来设置属性。 |
| `byType`      | 如果需要自动装配的属性的类型在容器中只存在一个的话，他允许自动装配。如果存在多个，则抛出致命异常，这表示您不能对该bean使用`byType`自动装配。 如果没有匹配的bean，则不会发生任何事情（未设置该属性）。 |
| `constructor` | 类似于`byType`，但应用于构造函数参数。 如果容器中没有一个Bean的类型和构造函数参数类型一致的话，则会引发致命错误。 |

通过`byType`或者`constructor`的自动装配方式，开发者可以装载数组和强类型集合。在这样的例子中，所有容器中的匹配了指定类型的bean都会自动装配到bean上来完成依赖注入。 开发者可以自动装配key为`String`.的强类型的 `Map` 。自动装配的Map值会包含所有的bean实例值来匹配指定的类型，`Map`的key会包含关联的bean的名字。

<a id="beans-autowired-exceptions"></a>

##### [](#beans-autowired-exceptions)自动装配的局限和缺点

自动装配在项目中一致使用时效果最佳。如果一般不使用自动装配，那么开发人员使用它来装配一个或两个bean定义可能会让人感到困惑。

考虑自动装配的局限和缺点:

*   `property`和`constructor-arg`设置中的显式依赖项始终覆盖自动装配。开发者不能自动装配一些简单属性，您不能自动装配简单属性，例如基本类型 ，字符串和类（以及此类简单属性的数组）。这种限制是按设计的。

*   自动装配比显式的配置更容易歧义，尽管上表表明了不同自动配置的特点，Spring也会尽可能避免不必要的装配错误。但是Spring管理的对象关系仍然不如显式配置那样明确。

*   从Spring容器生成文档的工具可能无法有效的提供装配信息。

*   容器中的多个bean定义可能与setter方法或构造函数参数所指定的类型相匹配， 这有利于自动装配。对于arrays, collections, or `Map`实例来说这不是问题。但是如果是对只有一个依赖项的值是有歧义的话，那么这个项是无法解析的。如果没有唯一的bean，则会抛出异常。


在后面的场景，你可有如下的选择：

*   放弃自动装配，改用显式的配置。
*   通过将`autowire-candidate` 属性设置为`false`，避免对bean定义进行自动装配，[如下一节所述](#beans-factory-autowire-candidate)。
*   通过将其`<bean/>` 元素的`primary`属性设置为 `true`，将单个bean定义指定为主要候选项。
*   使用基于注解的配置实现更细粒度的控制，如[基于注解的容器配置](#beans-annotation-config)中所述。

<a id="beans-factory-autowire-candidate"></a>

##### [](#beans-factory-autowire-candidate)将bean从自动装配中排除

在每个bean的基础上，您可以从自动装配中排除bean。 在Spring的XML格式中，将`<bean/>` 元素的`autowire-candidate` 属性设置为`false`。 容器使特定的bean定义对自动装配基础结构不可用（包括注解样式配置，如[`@Autowired`](#beans-autowired-annotation)）。

`autowire-candidate` 属性旨在仅影响基于类型的自动装配。 它不会影响名称的显式引用，即使指定的bean未标记为autowire候选，也会解析它。 因此，如果名称匹配，则按名称自动装配会注入bean。

开发者可以通过模式匹配而不是Bean的名字来限制自动装配的候选者。最上层的`<beans/>` 元素会在`default-autowire-candidates` 属性中来配置多种模式。 例如，限制自动装配候选者的名字以`Repository`结尾，可以配置成`*Repository`。如果需要配置多种模式，只需要用逗号分隔开即可。 bean定义的`autowire-candidate`属性的显式值`true` 或`false`始终优先。 对于此类bean，模式匹配规则不适用。

上面的这些技术在配置那些无需自动装配的bean是相当有效的，当然这并不是说这类bean本身无法自动装配其他的bean。而是说这些bean不再作为自动装配的依赖候选者。

<a id="beans-factory-method-injection"></a>

#### [](#beans-factory-method-injection)1.4.6.方法注入

在大多数的应用场景下，多数的bean都是[单例](#beans-factory-scopes-singleton)的。当这个单例的bean需要和另一个单例的或者非单例的bean协作使用的时候，开发者只需要配置依赖bean为这个bean的属性即可。 但是有时会因为bean具有不同的生命周期而产生问题。假设单例的bean A在每个方法调用中使用了非单例的bean B。容器只会创建bean A一次，而只有一个机会来配置属性。 那么容器就无法为每一次创建bean A时都提供新的bean B实例。

一种解决方案就是放弃IoC，开发者可以通过实现`ApplicationContextAware`接口让[bean A对ApplicationContext](#beans-factory-aware)可见。 从而通过调用[`getBean("B")`](#beans-factory-client)来在bean A 需要新的实例的时候来获取到新的B实例。参考下面例子。

```java
// a class that uses a stateful Command-style class to perform some processing
package fiona.apple;

// Spring-API imports
import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;

public class CommandManager implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    public Object process(Map commandState) {
        // grab a new instance of the appropriate Command
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    protected Command createCommand() {
        // notice the Spring API dependency!
        return this.applicationContext.getBean("command", Command.class);
    }

    public void setApplicationContext(
            ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

上面的代码并不让人十分满意，因为业务的代码已经与Spring框架耦合在一起。方法注入是Spring IoC容器的一个高级功能，可以让您处理这种问题。 Spring提供了一个稍微高级的注入方式来处理这种问题

您可以在此[博客条目](https://spring.io/blog/2004/08/06/method-injection/)中阅读有关方法注入的更多信息。

<a id="beans-factory-lookup-method-injection"></a>

##### [](#beans-factory-lookup-method-injection)查找方法注入

查找方法注入是容器覆盖管理bean上的方法的能力，以便返回容器中另一个命名bean的查找结果。查找方法通常涉及原型bean，[如前面描述的场景](#beans-factory-method-injection)。 Spring框架通过使用CGLIB库生成的字节码来生成动态子类重写的方法实现此注入。

*   如果想让这个动态子类正常工作，那么Spring容器所继承的Bean不能是`final`的，而覆盖的方法也不能是`final`的。

*   对具有`抽象`方法的类进行单元测试时，需要开发者对类进行子类化，并提供`抽象`方法的具体实现。

*   组件扫描也需要具体的方法，因为它需要获取具体的类。

*   另一个关键限制是查找方法不适用于工厂方法，特别是在配置类中不使用`@Bean`的方法。因为在这种情况下，容器不负责创建实例，因此不能在运行时创建运行时生成的子类。


对于前面代码片段中的`CommandManager`类，Spring容器动态地覆盖`createCommand()`方法的实现。 `CommandManager`类不再拥有任何的Spring依赖，如下：

```java
package fiona.apple;

// no more Spring imports!

public abstract class CommandManager {

    public Object process(Object commandState) {
        // grab a new instance of the appropriate Command interface
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    // okay... but where is the implementation of this method?
    protected abstract Command createCommand();
}
```

在包含需要注入方法的客户端类中 (在本例中为`CommandManager`）注入方法的签名需要如下形式：

```javajava
<public|protected> [abstract] <return-type> theMethodName(no-arguments);
```

如果方法是`abstract`的， 那么动态生成的子类会实现该方法。否则，动态生成的子类将覆盖原始类定义的具体方法。例如：

```xml
<!-- a stateful bean deployed as a prototype (non-singleton) -->
<bean id="myCommand" class="fiona.apple.AsyncCommand" scope="prototype">
    <!-- inject dependencies here as required -->
</bean>

<!-- commandProcessor uses statefulCommandHelper -->
<bean id="commandManager" class="fiona.apple.CommandManager">
    <lookup-method name="createCommand" bean="myCommand"/>
</bean>
```

当需要新的myCommand bean实例时，标识为`commandManager`的bean会调用自身的`createCommand()`方法.开发者必须小心部署`myCommand` bean为原型bean. 如果所需的bean是[单例](#beans-factory-scopes-singleton)的,那么每次都会返回相同的`myCommand` bean实例.

另外,如果是基于注解的配置模式,你可以在查找方法上定义`@Lookup`注解,如下:

```java
public abstract class CommandManager {

    public Object process(Object commandState) {
        Command command = createCommand();
        command.setState(commandState);
        return command.execute();
    }

    @Lookup("myCommand")
    protected abstract Command createCommand();
}
```

或者，更常见的是，开发者也可以根据查找方法的返回类型来查找匹配的bean，如下

```java
public abstract class CommandManager {

    public Object process(Object commandState) {
        MyCommand command = createCommand();
        command.setState(commandState);
        return command.execute();
    }

    @Lookup
    protected abstract MyCommand createCommand();
}
```

注意开发者可以通过创建子类实现lookup方法，以便使它们与Spring的组件扫描规则兼容，同时抽象类会在默认情况下被忽略。这种限制不适用于显式注册bean或明确导入bean的情况。

另一种可以访问不同生命周期的方法是`ObjectFactory`/`Provider`注入，具体参看 [作用域的bean依赖](#beans-factory-scopes-other-injection)

您可能还会发现`ServiceLocatorFactoryBean`（在`org.springframework.beans.factory.config`包中）很有用。

<a id="beans-factory-arbitrary-method-replacement"></a>

##### [](#beans-factory-arbitrary-method-replacement)替换任意方法

从前面的描述中，我们知道查找方法是有能力来覆盖任何由容器管理的bean方法的。开发者最好跳过这一部分，除非一定需要用到这个功能。

通过基于XML的元数据配置，开发者可以使用`replaced-method`元素来替换已存在方法的实现。考虑以下类，它有一个我们想要覆盖的名为`computeValue` 的方法：

```java
public class MyValueCalculator {

    public String computeValue(String input) {
        // some real code...
    }

    // some other methods...
}
```

实现`org.springframework.beans.factory.support.MethodReplacer`接口的类提供了新的方法定义，如以下示例所示：

```java
/**
 * meant to be used to override the existing computeValue(String)
 * implementation in MyValueCalculator
 */
public class ReplacementComputeValue implements MethodReplacer {

    public Object reimplement(Object o, Method m, Object[] args) throws Throwable {
        // get the input value, work with it, and return a computed result
        String input = (String) args[0];
        ...
        return ...;
    }
}
```

如果需要覆盖bean方法的XML配置如下类似于以下示例：

```xml
<bean id="myValueCalculator" class="x.y.z.MyValueCalculator">
    <!-- arbitrary method replacement -->
    <replaced-method name="computeValue" replacer="replacementComputeValue">
        <arg-type>String</arg-type>
    </replaced-method>
</bean>

<bean id="replacementComputeValue" class="a.b.c.ReplacementComputeValue"/>
```

您可以在`<replaced-method/>`元素中使用一个或多个 `<arg-type/>`元素来指示被覆盖的方法的方法。当需要覆盖的方法存在重载方法时，必须指定所需参数。 为了方便起见，字符串的类型会匹配以下类型，它完全等同于`java.lang.String`

    java.lang.String
    String
    Str

因为，通常来说参数的个数已经足够区别不同的方法，这种快捷的写法可以省去很多的代码。

<a id="beans-factory-scopes"></a>

### [](#beans-factory-scopes)1.5. bean的作用域

创建bean定义时，同时也会定义该如何创建Bean实例。 这些具体创建的过程是很重要的，因为只有通过对这些配置过程，您才能创建实例对象。

您不仅可以将不同的依赖注入到bean中，还可以配置bean的作用域。这种方法是非常强大而且也非常灵活，开发者可以通过配置来指定对象的作用域，无需在Java类的层次上配置。 bean可以配置多种作用域，Spring框架支持五种作用域，有三种作用域是当开发者使用基于Web的`ApplicationContext`的时候才有效的。您还可以创建[自定义范围.](#beans-factory-scopes-custom)。

下表描述了支持的范围:

Table 3. Bean 的作用域

| 作用域                                                | 描述                                                         |
| ----------------------------------------------------- | ------------------------------------------------------------ |
| [singleton](#beans-factory-scopes-singleton)          | (默认) 每一Spring IOC容器都拥有唯一的实例对象。              |
| [prototype](#beans-factory-scopes-prototype)          | 一个Bean定义可以创建任意多个实例对象.                        |
| [request](#beans-factory-scopes-request)              | 将单个bean定义范围限定为单个HTTP请求的生命周期。 也就是说，每个HTTP请求都有自己的bean实例，它是在单个bean定义的后面创建的。 只有基于Web的Spring `ApplicationContext`的才可用。 |
| [session](#beans-factory-scopes-session)              | 将单个bean定义范围限定为HTTP `Session`的生命周期。 只有基于Web的Spring `ApplicationContext`的才可用。 |
| [application](#beans-factory-scopes-application)      | 将单个bean定义范围限定为`ServletContext`的生命周期。 只有基于Web的Spring `ApplicationContext`的才可用。 |
| [websocket](https://github.com/DocsHome/spring-docs/blob/master/pages/web/web.md#websocket-stomp-websocket-scope) | 将单个bean定义范围限定为 `WebSocket`的生命周期。 只有基于Web的Spring `ApplicationContext`的才可用。 |

从Spring 3.0开始，线程作用域默认是可用的，但默认情况下未注册。 有关更多信息，请参阅[`SimpleThreadScope`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/support/SimpleThreadScope.html)的文档。 有关如何注册此范围或任何其他自定义范围的说明，请参阅使用[自定义范围](#beans-factory-scopes-custom-using)。

<a id="beans-factory-scopes-singleton"></a>

#### [](#beans-factory-scopes-singleton)1.5.1. 单例作用域

单例bean在全局只有一个共享的实例，所有依赖单例bean的场景中，容器返回的都是同一个实例。

换句话说，当您定义一个bean并且它的范围是一个单例时，Spring IoC容器只会根据bean的定义来创建该bean的唯一实例。 这些唯一的实例会缓存到容器中，后续针对单例bean的请求和引用，都会从这个缓存中拿到这个唯一实例。 下图显示了单例范围的工作原理：

![singleton](https://github.com/DocsHome/spring-docs/blob/master/pages/images/singleton.png)

Spring的单例bean概念不同于设计模式（GoF）之中所定义的单例模式。设计模式中的单例模式是将一个对象的作用域硬编码的，一个ClassLoader只能有唯一的一个实例。 而Spring的单例作用域是以容器为前提的，每个容器每个bean只能有一个实例。 这意味着，如果在单个Spring容器中为特定类定义一个bean，则Spring容器会根据bean定义创建唯一的bean实例。 单例作用域是Spring的默认作用域。 下面的例子是在XML中配置单例模式Bean的例子：

```xml
<bean id="accountService" class="com.something.DefaultAccountService"/>

<!-- the following is equivalent, though redundant (singleton scope is the default) -->
<bean id="accountService" class="com.something.DefaultAccountService" scope="singleton"/>
```

<a id="beans-factory-scopes-prototype"></a>

#### [](#beans-factory-scopes-prototype)1.5.2. 原型作用域

非单例的、原型bean指的是每次请求bean实例时,返回的都是新的对象实例。也就是说，每次注入到另外的bean或者通过调用 `getBean()`方法来获得的bean都是全新的实例。 基于线程安全性的考虑，当bean对象有状态时使用原型作用域，而无状态时则使用单例作用域。

下图说明了Spring原型作用域：:

![prototype](https://github.com/DocsHome/spring-docs/blob/master/pages/images/prototype.png)

（数据访问对象（DAO）通常不配置为原型，因为典型的DAO不具有任何会话状态。我们可以更容易重用单例图的核心。）

用下面的例子来说明Spring的原型作用域:

```xml
<bean id="accountService" class="com.something.DefaultAccountService" scope="prototype"/>
```

与其他作用域相比，Spring不会完整地管理原型bean的生命周期。 Spring容器只会初始化、配置和装载这些bean，然后传递给Client。但是之后就不会再有该原型实例的进一步记录。 也就是说，初始化生命周期回调方法在所有作用域的bean是都会调用的，但是销毁生命周期回调方法在原型bean是不会调用的。所以，客户端代码必须注意清理原型bean以及释放原型bean所持有的资源。 可以通过使用自定义的[bean post-processor](#beans-factory-extension-bpp)（Bean的后置处理器）来让Spring释放掉原型bean所持有的资源。

在某些方面，Spring容器关于原型作用域的bean就是取代了Java的`new` 操作符。 所有的生命周期的控制都由客户端来处理（有关Spring容器中bean的生命周期的详细信息，请参阅[Bean的生命周期](#beans-factory-lifecycle)）。

<a id="beans-factory-scopes-sing-prot-interaction"></a>

#### [](#beans-factory-scopes-sing-prot-interaction)1.5.3. 依赖原型bean的单例bean

当您使用具有依赖于原型bean的单例作用域bean时，请注意在实例化时解析依赖项。 因此，如果将原型bean注入到单例的bean中，则会实例化一个新的原型bean，然后将依赖注入到单例bean中。 这个依赖的原型bean仍然是同一个实例。

但是，假设您希望单例作用域的bean在运行时重复获取原型作用域的bean的新实例。 您不能将原型作用域的bean依赖注入到您的单例bean中， 因为当Spring容器实例化单例bean并解析注入其依赖项时，该注入只发生一次。 如果您需要在运行时多次使用原型bean的新实例，请参阅[方法注入](#beans-factory-method-injection)。

<a id="beans-factory-scopes-other"></a>

#### [](#beans-factory-scopes-other)1.5.4. 请求，会话，应用程序和WebSocket作用域

`request`, `session`, `application`, 和 `websocket`作用域只有在Web中使用Spring的`ApplicationContext`（例如`ClassPathXmlApplicationContext`）的情况下才用得上。 如果在普通的Spring IoC容器，例如ClassPathXmlApplicationContext中使用这些作用域，将会抛出IllegalStateException异常来说明使用了未知的作用域。

<a id="beans-factory-scopes-other-web-configuration"></a>

##### [](#beans-factory-scopes-other-web-configuration)初始化Web的配置

为了能够使用请求 `request`, `session`, `application`, 和`websocket`（Web范围的bean），需要在配置bean之前作一些基础配置。 而对于标准的作用域，例如单例和原型作用域，这种基础配置是不需要的。

如何完成此初始设置取决于您的特定Servlet环境.

例如，如果开发者使用了Spring Web MVC框架，那么每一个请求都会通过Spring的`DispatcherServlet`来处理，那么也无需特殊的设置了。DispatcherServlet和DispatcherPortlet已经包含了相应状态。

如果您使用Servlet 2.5 Web容器，并且在Spring的`DispatcherServlet`之外处理请求（例如，使用JSF或Struts时），则需要注册`org.springframework.web.context.request.RequestContextListener`或者`ServletRequestListener`。 对于Servlet 3.0+，可以使用`WebApplicationInitializer`接口以编程方式完成此操作。 如果需要兼容旧版本容器的话，将以下声明添加到Web应用程序的`web.xml` 文件中:

```xml
<web-app>
    ...
    <listener>
        <listener-class>
            org.springframework.web.context.request.RequestContextListener
        </listener-class>
    </listener>
    ...
</web-app>
```

或者，如果对Listener不是很熟悉，请考虑使用Spring的`RequestContextFilter`。 Filter映射取决于Web应用的配置，因此您必须根据需要进行更改。 以下清单显示了Web应用程序的过滤器部分:

```xml
<web-app>
    ...
    <filter>
        <filter-name>requestContextFilter</filter-name>
        <filter-class>org.springframework.web.filter.RequestContextFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>requestContextFilter</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
    ...
</web-app>
```

`DispatcherServlet`, `RequestContextListener`, and `RequestContextFilter`所做的工作实际上是一样的，都是将request对象请求绑定到服务的 `Thread`上。 这才使bean在之后的调用链上对请求和会话作用域可见。

<a id="beans-factory-scopes-request"></a>

##### [](#beans-factory-scopes-request)Request作用域

参考下面这个XML配置的bean定义:

```xml
<bean id="loginAction" class="com.something.LoginAction" scope="request"/>
```

Spring容器会在每次使用`LoginAction`来处理每个HTTP请求时都会创建新的`LoginAction`实例。也就是说，`LoginAction` bean的作用域是HTTP Request级别的。 开发者可以随意改变实例的状态，因为其他通过`loginAction`请求来创建的实例根本看不到开发者改变的实例状态，所有创建的Bean实例都是根据独立的请求创建的。当请求处理完毕，这个bean也将会销毁。

当使用注解配置或Java配置时，使用`@RequestScope`注解修饰的bean会被设置成request作用域。 以下示例显示了如何执行此操作：

```java
@RequestScope
@Component
public class LoginAction {
    // ...
}
```

<a id="beans-factory-scopes-session"></a>

##### [](#beans-factory-scopes-session)Session作用域

参考下面XML配置的bean的定义:

```xml
<bean id="userPreferences" class="com.something.UserPreferences" scope="session"/>
```

Spring容器通过在单个HTTP会话的生命周期中使用`UserPreferences` bean定义来创建`UserPreferences` bean的新实例。换言之，`UserPreferences` Bean的作用域是HTTP S`Session`级别的，在request-scoped作用域的bean上， 开发者可以随意的更改实例的状态，同样，其他HTTP `Session`的基本实例在每个`Session`中都会请求userPreferences来创建新的实例，所以，开发者更改bean的状态， 对于其他的Bean仍然是不可见的。当HTTP `Session`被销毁时，根据这个`Session`来创建的bean也将会被销毁。

使用注解配置和Java配置时，使用`@SessionScope` 注解修饰的 bean 会被设置成`session`作用域。

```java
@SessionScope
@Component
public class UserPreferences {
    // ...
}
```

<a id="beans-factory-scopes-application"></a>

##### [](#beans-factory-scopes-application)Application作用域

参考下面用XML配置的bean的定义:

```xml
<bean id="appPreferences" class="com.something.AppPreferences" scope="application"/>
```

Spring容器会在整个Web应用内使用到`appPreferences`的时候创建一个新的`AppPreferences`的实例。也就是说，`appPreferences` bean是在`ServletContext` 级别的， 就像普通的`ServletContext` 属性一样。这种作用域在一些程度上来说和Spring的单例作用域是极为相似，但是也有如下不同之处，应用作用域是每个`ServletContext` 中包含一个 而不是每个Spring ApplicationContext中只有一个（某些应用可能包含多个ApplicationContext）。应用作用域仅仅对`ServletContext` 可见，单例bean是对ApplicationContext可见。

当使用注解配置或Java配置时，使用`@ApplicationScope` 注解修饰的bean会被设置成`application`作用域 。以下示例显示了如何执行此操作:

```java
@ApplicationScope
@Component
public class AppPreferences {
    // ...
}
```

<a id="beans-factory-scopes-other-injection"></a>

##### [](#beans-factory-scopes-other-injection)有作用域bean的依赖

Spring IoC容器不仅仅管理对象（bean）的实例化，同时也负责装配依赖。如果开发者要将一个bean装配到比它作用域更广的bean时（例如HTTP请求返回的bean），那么开发者应当选择注入AOP代理而不是使用带作用域的bean。 也就是说，开发者需要注入代理对象，而这个代理对象既可以找到实际的bean，还能够创建全新的bean。

您还可以在作为单例的作用域的bean之间使用`<aop:scoped-proxy/>`，然后引用通过可序列化的中间代理，从而能够在反序列化时重新获取目标`单例`bean。

当针对原型作用域的bean声明`<aop:scoped-proxy/>`时，每个通过代理的调用都会产生新的目标实例。

此外，作用域代理并不是取得作用域bean的唯一安全方式。 开发者也可以通过简单的声明注入（即构造函数或setter参数或自动装配字段）`ObjectFactory<MyTargetBean>`， 然后允许通过类似`getObject()`的方法调用来获取一些指定的依赖，而不是直接储存依赖的实例。

作为扩展变体，您可以声明`ObjectProvider<MyTargetBean>`，它提供了几个额外的访问变体，包括`getIfAvailable` 和 `getIfUnique`。

JSR-330将这样的变种称为Provider，它使用`Provider<MyTargetBean>` 声明以及相关的 `get()` 方法来尝试获取每一个配置。 有关JSR-330整体的更多详细信息，[请参看此处](#beans-standard-annotations)。

以下示例中的配置只有一行，但了解“为什么”以及它背后的“如何”非常重要：:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- an HTTP Session-scoped bean exposed as a proxy -->
    <bean id="userPreferences" class="com.something.UserPreferences" scope="session">
        <!-- instructs the container to proxy the surrounding bean -->
        <aop:scoped-proxy/> (1)
    </bean>

    <!-- a singleton-scoped bean injected with a proxy to the above bean -->
    <bean id="userService" class="com.something.SimpleUserService">
        <!-- a reference to the proxied userPreferences bean -->
        <property name="userPreferences" ref="userPreferences"/>
    </bean>
</beans>
```

**1**、定义代理的行。

要创建这样的一个代理，只需要在带作用域的bean定义中添加子节点`<aop:scoped-proxy/>`即可（具体查看[选择创建代理的类型](#beans-factory-scopes-other-injection-proxies)和 [基于XML Schema的配置](#xsd-schemas)）。为什么在`request`, `session`和自定义作用域级别范围内的bean定义需要`<aop:scoped-proxy/>`， 考虑以下单例bean定义，并将其与您需要为上述范围定义的内容进行对比（请注意，以下`userPreferences`bean定义不完整）:

```xml
<bean id="userPreferences" class="com.something.UserPreferences" scope="session"/>

<bean id="userManager" class="com.something.UserManager">
    <property name="userPreferences" ref="userPreferences"/>
</bean>
```

在上面的例子中，单例bean（`userManager`）注入了注入了HTTP `Session`级别的`userPreferences`依赖。 显然， 问题就是`userPreferences`在Spring容器中只会实例化一次。它的依赖项（在这种情况下只有一个，`userPreferences`）也只注入一次。 这意味着`userManager` 每次使用的是完全相同的`userPreferences`对象（即最初注入它的对象）进行操作。

这不是将短周期作用域bean注入到长周期作用域bean时所需的行为，例如将HTTP `Session`级别的作用域bean作为依赖注入到单例bean中。相反，开发者需要一个 `userManager`对象， 而在HTTP `Session`的生命周期中，开发者需要一个特定于HTTP Session的 userPreferences 对象。因此，容器创建一个对象，该对象公开与UserPreferences类（理想情况下为`UserPreferences`实例的对象） 完全相同的公共接口，该对象可以从作用域机制（HTTP Request、`Session`等）中获取真实的`UserPreferences`对象。容器将这个代理对象注入到`userManager`中， 而不知道这个`UserPreferences`引用是一个代理。在这个例子中，当一个UserManager实例在依赖注入的UserPreferences对象上调用一个方法时， 它实际上是在调用代理的方法，再由代理从HTTP `Session`（本例）获取真实的`UserPreferences`对象，并将方法调用委托给检索到的实际`UserPreferences`对象。

因此，在将`request-` and `session-scoped`的bean来作为依赖时，您需要以下（正确和完整）配置，如以下示例所示： 所以当开发者希望能够正确的使用配置请求、会话或者全局会话级别的bean来作为依赖时，需要进行如下类似的配置。

```xml
<bean id="userPreferences" class="com.something.UserPreferences" scope="session">
    <aop:scoped-proxy/>
</bean>

<bean id="userManager" class="com.something.UserManager">
    <property name="userPreferences" ref="userPreferences"/>
</bean>
```

<a id="beans-factory-scopes-other-injection-proxies"></a>

###### [](#beans-factory-scopes-other-injection-proxies)选择要创建的代理类型

默认情况下，当Spring容器为使用`<aop:scoped-proxy/>` 元素标记的bean创建代理时，将创建基于CGLIB的类代理。

CGLIB代理只拦截公共方法调用！ 不要在这样的代理上调用非公共方法。 它们不会委托给实际的作用域目标对象。

或者，您可以通过为`<aop:scoped-proxy/>`元素的`proxy-target-class` 属性的值指定`false`来配置Spring容器， 以便为此类作用域bean创建基于JDK接口的标准代理。 使用基于接口的JDK代理意味着开发者无需引入第三方库即可完成代理。 但是，这也意味着带作用域的bean需要额外实现一个接口，而依赖是从这些接口来获取的。 以下示例显示基于接口的代理：

```xml
<!-- DefaultUserPreferences implements the UserPreferences interface -->
<bean id="userPreferences" class="com.stuff.DefaultUserPreferences" scope="session">
    <aop:scoped-proxy proxy-target-class="false"/>
</bean>

<bean id="userManager" class="com.stuff.UserManager">
    <property name="userPreferences" ref="userPreferences"/>
</bean>
```

有关选择基于类或基于接口的代理的更多详细信息，请参阅[代理机制](#aop-proxying)。

<a id="beans-factory-scopes-custom"></a>

#### [](#beans-factory-scopes-custom)1.5.5. 自定义作用域

bean的作用域机制是可扩展的，开发者可以自定义作用域，甚至重新定义已经存在的作用域，但是Spring团队不推荐这样做，而且开发者也不能重写`singleton` 和 `prototype`作用域。

<a id="beans-factory-scopes-custom-creating"></a>

##### [](#beans-factory-scopes-custom-creating)创建自定义作用域

为了能够使Spring可以管理开发者定义的作用域，开发者需要实现`org.springframework.beans.factory.config.Scope`。如何实现自定义的作用域， 可以参考Spring框架的一些实现或者有关[`Scope`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/config/Scope.html) 的javadoc

`Scope`接口有四个方法用于操作对象,例如获取、移除或销毁等操作。

例如，传入Session作用域该方法将会返回一个 session-scoped的bean（如果它不存在，那么将会返回绑定session作用域的新实例）。下面的方法返回相应作用域的对象：

```java
Object get(String name, ObjectFactory objectFactory)
```

下面的方法将从相应的作用域中移除对象。同样，以会话为例，该函数会删除会话作用域的Bean。删除的对象会作为返回值返回，当无法找到对象时将返回null。 以下方法从相应作用域中删除对象：:

```java
Object remove(String name)
```

以下方法注册范围在销毁时或在Scope中的指定对象被销毁时应该执行的回调:

```java
void registerDestructionCallback(String name, Runnable destructionCallback)
```

有关销毁回调的更多信息，请参看[javadoc](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/config/Scope.html#registerDestructionCallback)或Spring的Scope实现部分。

下面的方法获取相应作用域的区分标识符:

```java
String getConversationId()
```

这个标识符在不同的作用域中是不同的。例如对于会话作用域，这个标识符就是会话的标识符。.

<a id="beans-factory-scopes-custom-using"></a>

##### [](#beans-factory-scopes-custom-using)使用自定义作用域

在实现了自定义`作用域`后，开发者还需要让Spring容器能够识别发现所创建的新`作用域`。下面的方法就是在Spring容器中用来注册新`Scope`的:

```java
void registerScope(String scopeName, Scope scope);
```

这个方法是在`ConfigurableBeanFactory`的接口中声明的，可以用在多数的`ApplicationContext`实现，也可以通过 `BeanFactory`属性来调用。

`registerScope(..)`方法的第一个参数是相关`作用域`的唯一名称。举例来说，Spring容器中的单例和原型就以它本身来命名。 第二个参数就是开发者希望注册和使用的自定义`Scope`实现的具有对象 T

假定开发者实现了自定义`Scope`，然后可以按如下步骤来注册。

下一个示例使用SimpleThreadScope，这个例子在Spring中是有实现的，但没有默认注册。 您自定义的作用域也可以通过如下的方式来注册。

```java
Scope threadScope = new SimpleThreadScope();
beanFactory.registerScope("thread", threadScope);
```

然后，您可以创建符合自定义Scope的作用域规则的bean定义，如下所示：

```xml
<bean id="..." class="..." scope="thread">
```

在自定义作用域中，开发者也不限于仅仅通过编程的方式来注册作用域，还可以通过配置`CustomScopeConfigurer` 类来实现。如以下示例所示：:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean class="org.springframework.beans.factory.config.CustomScopeConfigurer">
        <property name="scopes">
            <map>
                <entry key="thread">
                    <bean class="org.springframework.context.support.SimpleThreadScope"/>
                </entry>
            </map>
        </property>
    </bean>

    <bean id="thing2" class="x.y.Thing2" scope="thread">
        <property name="name" value="Rick"/>
        <aop:scoped-proxy/>
    </bean>

    <bean id="thing1" class="x.y.Thing1">
        <property name="thing2" ref="thing2"/>
    </bean>

</beans>
```

在`FactoryBean`实现中添加了`<aop:scoped-proxy/>`元素时，它是工厂bean本身的作用域，而不是从`getObject()`方法返回的对象。

<a id="beans-factory-nature"></a>

### [](#beans-factory-nature)1.6. 自定义Bean的特性

Spring Framework提供了许多可用于自定义bean特性的接口。 本节将它们分组如下：

*   [Lifecycle Callbacks(生命周期回调)](#beans-factory-lifecycle)

*   [`ApplicationContextAware` and `BeanNameAware`](#beans-factory-aware)

*   [其他 `Aware` 接口](#aware-list)

<a id="beans-factory-lifecycle"></a>

#### [](#beans-factory-lifecycle)1.6.1. 生命周期回调

你可以实现`InitializingBean` 和 `DisposableBean`接口，让容器里管理Bean的生命周期。容器会在调用`afterPropertiesSet()` 之后和`destroy()`之前会允许bean在初始化和销毁bean时执行某些操作。

JSR-250 `@PostConstruct` 和 `@PreDestroy`注解通常被认为是在现代Spring应用程序中接收生命周期回调的最佳实践。 使用这些注解意味着您的bean不会耦合到特定于Spring的接口。 有关详细信息，请参阅[使用 `@PostConstruct` 和 `@PreDestroy`](#beans-postconstruct-and-predestroy-annotations).

如果您不想使用JSR-250注解但仍想删除耦合，请考虑使用`init-method` 和 `destroy-method`定义对象元数据。

在内部，Spring 框架使用`BeanPostProcessor` 实现来处理任何回调接口并调用适当的方法。 如果您需要Spring默认提供的自定义功能或其他生命周期行为，您可以自己实现`BeanPostProcessor`。 有关更多信息，请参阅[容器扩展点](#beans-factory-extension)。

除了初始化和销毁方法的回调，Spring管理的对象也实现了Lifecycle接口来让管理的对象在容器的`生命周期`内启动和关闭。

本节描述了生命周期回调接口。.

<a id="beans-factory-lifecycle-initializingbean"></a>

##### [](#beans-factory-lifecycle-initializingbean)初始化方法回调

`org.springframework.beans.factory.InitializingBean`接口允许bean在所有的必要的依赖配置完成后执行bean的初始化， `InitializingBean` 接口中指定使用如下方法:

```java
void afterPropertiesSet() throws Exception;
```

Spring团队是不建议开发者使用`InitializingBean`接口，因为这样会将代码耦合到Spring的特殊接口上。他们建议使用[`@PostConstruct`](#beans-postconstruct-and-predestroy-annotations) 注解或者指定一个POJO的实现方法， 这会比实现接口更好。在基于XML的元数据配置上，开发者可以使用`init-method` 属性来指定一个没有参数的方法，使用Java配置的开发者可以在`@Bean`上添加 `initMethod` 属性。 请参阅 [接收生命周期回调](#beans-java-lifecycle-callbacks)接收生命周期回调：

```xml
<bean id="exampleInitBean" class="examples.ExampleBean" init-method="init"/>

public class ExampleBean {

    public void init() {
        // do some initialization work
    }
}
```

前面的示例与以下示例（由两个列表组成）具有几乎完全相同的效果：

```xml
<bean id="exampleInitBean" class="examples.AnotherExampleBean"/>

public class AnotherExampleBean implements InitializingBean {

    public void afterPropertiesSet() {
        // do some initialization work
    }
}
```

但是，前面两个示例中的第一个没有将代码耦合到Spring。

<a id="beans-factory-lifecycle-disposablebean"></a>

##### [](#beans-factory-lifecycle-disposablebean)销毁方法的回调

实现`org.springframework.beans.factory.DisposableBean` 接口的Bean就能让容器通过回调来销毁bean所引用的资源。 `DisposableBean` 接口指定一个方法：

```java
void destroy() throws Exception;
```

我们建议您不要使用 `DisposableBean` 回调接口，因为它会不必要地将代码耦合到Spring。或者，我们建议使用[`@PreDestroy`](#beans-postconstruct-and-predestroy-annotations)注解 或指定bean定义支持的泛型方法。 在基于XML的元数据配置中，您可以在`<bean/>`上使用`destroy-method`属性。 使用Java配置，您可以使用`@Bean`的 `destroyMethod` 属性。 请参阅[接收生命周期回调](#beans-java-lifecycle-callbacks)。 考虑以下定义：

```xml
<bean id="exampleInitBean" class="examples.ExampleBean" destroy-method="cleanup"/>

public class ExampleBean {

    public void cleanup() {
        // do some destruction work (like releasing pooled connections)
    }
}
```

前面的定义与以下定义几乎完全相同：:

```xml
<bean id="exampleInitBean" class="examples.AnotherExampleBean"/>

public class AnotherExampleBean implements DisposableBean {

    public void destroy() {
        // do some destruction work (like releasing pooled connections)
    }
}
```

但是，前面两个定义中的第一个没有将代码耦合到Spring。.

您可以为`<bean>` 元素的`destroy-method`属性分配一个特殊的（推断的）值，该值指示Spring自动检测特定bean类的`close`或者`shutdown`方法。 （因此，任何实现`java.lang.AutoCloseable`或`java.io.Closeable`的类都将匹配。） 您还可以在`<bean>` 元素的`default-destroy-method`属性上设置此特殊（推断）值，用于让所有的bean都实现这个行为（[参见默认初始化和销毁方法](#beans-factory-lifecycle-default-init-destroy-methods)）。 请注意，这是Java配置的默认行为。

<a id="beans-factory-lifecycle-default-init-destroy-methods"></a>

##### [](#beans-factory-lifecycle-default-init-destroy-methods)默认初始化和销毁方法

当您不使用Spring特有的`InitializingBean`和 `DisposableBean`回调接口来实现初始化和销毁方法时，您定义方法的名称最好类似于`init()`, `initialize()`, `dispose()`。 这样可以在项目中标准化类方法，并让所有开发者都使用一样的名字来确保一致性。

您可以配置Spring容器来针对每一个Bean都查找这种名字的初始化和销毁回调方法。也就是说， 任意的开发者都会在应用的类中使用一个叫 `init()`的初始化回调。而不需要在每个bean中都定义`init-method="init"` 这种属性， Spring IoC容器会在bean创建的时候调用那个回调方法（[如前面描述](#beans-factory-lifecycle)的标准生命周期一样）。这个特性也将强制开发者为其他的初始化以及销毁回调方法使用同样的名字。

假设您的初始化回调方法名为`init()`，而您的destroy回调方法名为`destroy()`。 然后，您的类类似于以下示例中的类：

```java
public class DefaultBlogService implements BlogService {

    private BlogDao blogDao;

    public void setBlogDao(BlogDao blogDao) {
        this.blogDao = blogDao;
    }

    // this is (unsurprisingly) the initialization callback method
    public void init() {
        if (this.blogDao == null) {
            throw new IllegalStateException("The [blogDao] property must be set.");
        }
    }
}
```

然后，您可以在类似于以下内容的bean中使用该类:

```xml
<beans default-init-method="init">

    <bean id="blogService" class="com.something.DefaultBlogService">
        <property name="blogDao" ref="blogDao" />
    </bean>

</beans>
```

顶级`<beans/>`元素属性上存在`default-init-method`属性会导致Spring IoC容器将bean类上的`init`方法识别为初始化方法回调。 当bean被创建和组装时，如果bean拥有同名方法的话，则在适当的时候调用它。

您可以使用 `<beans/>`元素上的`default-destroy-method`属性，以类似方式（在XML中）配置destroy方法回调。

当某些bean已有的回调方法与配置的默认回调方法不相同时，开发者可以通过特指的方式来覆盖掉默认的回调方法。以XML为例，可以通过使用元素的`init-method` 和`destroy-method`属性来覆盖掉`<bean/>`中的配置。

Spring容器会做出如下保证，bean会在装载了所有的依赖以后，立刻就开始执行初始化回调。这样的话，初始化回调只会在直接的bean引用装载好后调用， 而此时AOP拦截器还没有应用到bean上。首先目标的bean会先完全初始化，然后AOP代理和拦截链才能应用。如果目标bean和代理是分开定义的，那么开发者的代码甚至可以跳过AOP而直接和引用的bean交互。 因此，在初始化方法中应用拦截器会前后矛盾，因为这样做耦合了目标bean的生命周期和代理/拦截器，还会因为与bean产生了直接交互进而引发不可思议的现象。

<a id="beans-factory-lifecycle-combined-effects"></a>

##### [](#beans-factory-lifecycle-combined-effects)组合生命周期策略

从Spring 2.5开始，您有三种选择用于控制bean生命周期行为:

*   [`InitializingBean`](#beans-factory-lifecycle-initializingbean) 和 [`DisposableBean`](#beans-factory-lifecycle-disposablebean) 回调接口

*   自定义 `init()` 和 `destroy()` 方法

*   [`@PostConstruct` 和 `@PreDestroy` 注解](#beans-postconstruct-and-predestroy-annotations). 你也可以在bean上同时使用这些机制.


如果bean配置了多个生命周期机制，而且每个机制都配置了不同的方法名字时，每个配置的方法会按照以下描述的顺序来执行。但是，如果配置了相同的名字， 例如初始化回调为`init()`，在不止一个生命周期机制配置为这个方法的情况下，这个方法只会执行一次。如[上一节中所述](#beans-factory-lifecycle-default-init-destroy-methods)。

为同一个bean配置的多个生命周期机制具有不同的初始化方法，如下所示:

1.  包含`@PostConstruct`注解的方法

2.  在`InitializingBean` 接口中的`afterPropertiesSet()` 方法

3.  自定义的`init()` 方法


Destroy方法以相同的顺序调用:

1.  包含`@PreDestroy`注解的方法

2.  在`DisposableBean`接口中的`destroy()` 方法

3.  自定义的`destroy()` 方法

<a id="beans-factory-lifecycle-processor"></a>

##### [](#beans-factory-lifecycle-processor)开始和关闭回调

`Lifecycle`接口中为所有具有自定义生命周期需求的对象定义了一些基本方法（例如启动或停止一些后台进程）:

```java
public interface Lifecycle {

    void start();

    void stop();

    boolean isRunning();
}
```

任何Spring管理的对象都可以实现`Lifecycle` 接口。然后，当`ApplicationContext`接收到启动和停止信号时（例如，对于运行时的停止/重启场景），ApplicationContext会通知到所有上下文中包含的生命周期对象。 它通过委托 `LifecycleProcessor`完成此操作，如下面的清单所示：

```java
public interface LifecycleProcessor extends Lifecycle {

    void onRefresh();

    void onClose();
}
```

请注意，`LifecycleProcessor`是 `Lifecycle`接口的扩展。 它还添加了另外两种方法来响应刷新和关闭的上下文。

注意，常规的`org.springframework.context.Lifecycle`接口只是为明确的开始/停止通知提供一个约束，而并不表示在上下文刷新就会自动开始。 要对特定bean的自动启动（包括启动阶段）进行细粒度控制，请考虑实现`org.springframework.context.SmartLifecycle`接口。

同时，停止通知并不能保证在销毁之前出现。在正常的关闭情况下，所有的`Lifecycle`都会在销毁回调准备好之前收到停止通知，然而， 在上下文生命周期中的热刷新或者停止尝试刷新时，则只会调用销毁方法。

启动和关闭调用的顺序非常重要。如果任何两个对象之间存在“依赖”关系，则依赖方在其依赖之后开始，并且在其依赖之前停止。但是，有时，直接依赖性是未知的。 您可能只知道某种类型的对象应该在另一种类型的对象之前开始。 在这些情况下， `SmartLifecycle`接口定义了另一个选项，即在其超级接口`Phased` 上定义的 `getPhase()` 方法。 以下清单显示了`Phased`接口的定义

```java
public interface Phased {

    int getPhase();
}
```

以下清单显示了`SmartLifecycle`接口的定义:

```java
public interface SmartLifecycle extends Lifecycle, Phased {

    boolean isAutoStartup();

    void stop(Runnable callback);
}
```

当启动时，拥有最低phased的对象会优先启动，而当关闭时，会相反的顺序执行。因此，如果一个对象实现了`SmartLifecycle`，然后令其`getPhase()`方法返回`Integer.MIN_VALUE`值的话， 就会让该对象最早启动，而最晚销毁。显然，如果`getPhase()`方法返回了`Integer.MAX_VALUE`值则表明该对象会最晚启动，而最早销毁。 当考虑到使用phased值时，也同时需要了解正常没有实现`SmartLifecycle`的`Lifecycle`对象的默认值，这个值是0。因此，配置任意的负值都将表明将对象会在标准组件启动之前启动 ，而在标准组件销毁以后再进行销毁。

`SmartLifecycle`接口也定义了一个名为stop的回调方法，任何实现了`SmartLifecycle`接口的方法都必须在关闭流程完成之后调用回调中的`run()`方法。 这样做可以进行异步关闭，而`lifecycleProcessor`的默认实现`DefaultLifecycleProcessor`会等到配置的超时时间之后再调用回调。默认的每一阶段的超时时间为30秒。 您可以通过在上下文中定义名为 `lifecycleProcessor` 的bean来覆盖默认生命周期处理器实例。 如果您只想修改超时，则定义以下内容就足够了：

```xml
<bean id="lifecycleProcessor" class="org.springframework.context.support.DefaultLifecycleProcessor">
    <!-- timeout value in milliseconds -->
    <property name="timeoutPerShutdownPhase" value="10000"/>
</bean>
```

如前所述，`LifecycleProcessor` 接口还定义了用于刷新和关闭上下文的回调方法。在关闭过程中，如果`stop()`方法已经被调用，则就会执行关闭流程。 但是如果上下文正在关闭中则不会在进行此流程，而刷新的回调会使用到`SmartLifecycle`的另一个特性。当上下文刷新完毕（所有的对象已经实例化并初始化）后， 就会调用刷新回调，默认的生命周期处理器会检查每一个`SmartLifecycle` 对象的`isAutoStartup()`方法返回的Boolean值.如果为真，对象将会自动启动而不是等待明确的上下文调用， 或者调用自己的`start()`方法(不同于上下文刷新，标准的上下文实现是不会自动启动的）。`phase`的值以及“depends-on”关系会决定对象启动和销毁的顺序。

<a id="beans-factory-shutdown"></a>

##### [](#beans-factory-shutdown)在非Web应用中优雅地关闭Spring IoC容器

本节仅适用于非Web应用程序。 Spring的基于Web的`ApplicationContext` 实现已经具有代码，可以在关闭相关Web应用程序时正常关闭Spring IoC容器。

如果开发者在非Web应用环境使用Spring IoC容器的话（例如，在桌面客户端的环境下）开发者需要在JVM上注册一个关闭的钩子，来确保在关闭Spring IoC容器的时候能够调用相关的销毁方法来释放掉引用的资源。 当然，开发者也必须正确配置和实现那些销毁回调。

要注册关闭钩子，请调用`ConfigurableApplicationContext`接口上声明的`registerShutdownHook()` 方法，如以下示例所示：

```java
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public final class Boot {

    public static void main(final String[] args) throws Exception {
        ConfigurableApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

        // add a shutdown hook for the above context...
        ctx.registerShutdownHook();

        // app runs here...

        // main method exits, hook is called prior to the app shutting down...
    }
}
```

<a id="beans-factory-aware"></a>

#### [](#beans-factory-aware)1.6.2. `ApplicationContextAware` 和 `BeanNameAware`

当`ApplicationContext` 创建实现`org.springframework.context.ApplicationContextAware`接口的对象实例时，将为该实例提供对该 `ApplicationContext`.的引用。 以下清单显示了`ApplicationContextAware`接口的定义：

```java
public interface ApplicationContextAware {

    void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
}
```

这样bean就能够通过编程的方式创建和操作`ApplicationContext` 了。通过`ApplicationContext` 接口，或者通过将引用转换成已知的接口的子类， 例如`ConfigurableApplicationContext`就能够提供一些额外的功能。其中的一个用法就是可以通过编程的方式来获取其他的bean。 有时候这个能力非常有用。当然，Spring团队并不推荐这样做，因为这样会使代码与Spring框架耦合，同时也没有遵循IoC的风格。 `ApplicationContext` 中其它的方法可以提供一些诸如资源的访问、发布应用事件或者添加`MessageSource`之类的功能。[`ApplicationContext`的附加功能](#context-introduction)中描述了这些附加功能。

从Spring 2.5开始， 自动装配是另一种获取`ApplicationContext`引用的替代方法。传统的的`构造函数` 和 `byType`的装载方式自动装配模式（如[自动装配](#beans-factory-autowire)中所述） 可以通过构造函数或setter方法的方式注入，开发者也可以通过注解注入的方式。为了更为方便，包括可以注入的字段和多个参数方法，请使用新的基于注解的自动装配功能。 这样，`ApplicationContext`将自动装配字段、构造函数参数或方法参数，如果相关的字段，构造函数或方法带有 `@Autowired`注解，则该参数需要`ApplicationContext`类型。 有关更多信息，请参阅[使用 `@Autowired`](#beans-autowired-annotation)@Autowired。

当`ApplicationContext`创建实现了`org.springframework.beans.factory.BeanNameAware`接口的类，那么这个类就可以针对其名字进行配置。以下清单显示了BeanNameAware接口的定义：:

```java
public interface BeanNameAware {

    void setBeanName(String name) throws BeansException;
}
```

这个回调的调用在属性配置完成之后，但是在初始化回调之前。例如`InitializingBean`, `afterPropertiesSet`方法以及自定义的初始化方法等。

<a id="aware-list"></a>

#### [](#aware-list)1.6.3. 其他的 `Aware`接口

除了 `ApplicationContextAware`和`BeanNameAware`（前面已讨论过）之外，Spring还提供了一系列`Aware`接口，让bean告诉容器，它们需要一些具体的基础配置信息。。 一些重要的`Aware`接口参看下表：

Table 4. Aware 接口

| 名称                             | 注入的依赖                                                   | 所对应的章节                                                 |
| -------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `ApplicationContextAware`        | 声明 `ApplicationContext`                                    | [`ApplicationContextAware` 和 `BeanNameAware`](#beans-factory-aware) |
| `ApplicationEventPublisherAware` | `ApplicationContext`的事件发布者                             | [`ApplicationContext`的其他功能](#context-introduction)      |
| `BeanClassLoaderAware`           | 用于加载bean类的类加载器                                     | [实例化Bean](#beans-factory-class)                           |
| `BeanFactoryAware`               | 声明 `BeanFactory`.                                          | [`ApplicationContextAware` 和 `BeanNameAware`](#beans-factory-aware) |
| `BeanNameAware`                  | 声明bean的名称.                                              | [`ApplicationContextAware` 和 `BeanNameAware`](#beans-factory-aware) |
| `BootstrapContextAware`          | 容器运行的资源适配器`BootstrapContext`。通常仅在JCA感知的 `ApplicationContext` 实例中可用 | [JCA CCI](https://github.com/DocsHome/spring-docs/blob/master/pages/integration/integration.md#cci)                              |
| `LoadTimeWeaverAware`            | 定义的weaver用于在加载时处理类定义.                          | [在Spring框架中使用AspectJ进行加载时织入](#aop-aj-ltw)       |
| `MessageSourceAware`             | 用于解析消息的已配置策略（支持参数化和国际化）               | [`ApplicationContext`的其他作用](#context-introduction)      |
| `NotificationPublisherAware`     | Spring JMX通知发布者                                         | [通知](https://github.com/DocsHome/spring-docs/blob/master/pages/integration/integration.md#jmx-notifications)                   |
| `ResourceLoaderAware`            | 配置的资源加载器                                             | [资源](#resources)                                           |
| `ServletConfigAware`             | 当前`ServletConfig`容器运行。仅在Web下的Spring `ApplicationContext`中有效 | [Spring MVC](https://github.com/DocsHome/spring-docs/blob/master/pages/web/web.md#mvc)                                   |
| `ServletContextAware`            | 容器运行的当前ServletContext。仅在Web下的Spring `ApplicationContext`中有效。 | [Spring MVC](https://github.com/DocsHome/spring-docs/blob/master/pages/web/web.md#mvc)                                     |

请再次注意，使用这些接口会将您的代码绑定到Spring API，而不会遵循IoC原则。 因此，我们建议将它们用于需要以编程方式访问容器的基础架构bean。

<a id="beans-child-bean-definitions"></a>

### [](#beans-child-bean-definitions)1.7. bean定义的继承

bean定义可以包含许多配置信息，包括构造函数参数，属性值和特定于容器的信息，例如初始化方法，静态工厂方法名称等。 子bean定义从父定义继承配置数据。 子定义可以覆盖某些值或根据需要添加其他值。 使用父子bean定义可以节省很多配置输入。 实际上，这是一种模板形式。

如果开发者编程式地使用`ApplicationContext`接口，子bean定义可以通过`ChildBeanDefinition`类来表示。很多开发者不会使用这个级别的方法， 而是会在类似于`ClassPathXmlApplicationContext`中声明式地配置bean定义。当你使用基于XML的配置时，你可以在子bean中使用parent属性，该属性的值用来识别父bean。 以下示例显示了如何执行此操作：

```xml
<bean id="inheritedTestBean" abstract="true"
        class="org.springframework.beans.TestBean">
    <property name="name" value="parent"/>
    <property name="age" value="1"/>
</bean>

<bean id="inheritsWithDifferentClass"
        class="org.springframework.beans.DerivedTestBean"
        parent="inheritedTestBean" init-method="initialize">  (1)
    <property name="name" value="override"/>
    <!-- the age property value of 1 will be inherited from parent -->
</bean>
```

**1**。请注意`parent`属性。

子bean如果没有指定class，它将使用父bean定义的class。但也可以进行重写。在后一种情况中，子bean必须与父bean兼容，也就是说，它必须接受父bean的属性值。

子bean定义从父类继承作用域、构造器参数、属性值和可重写的方法，除此之外，还可以增加新值。开发者指定任何作用域、初始化方法、销毁方法和/或者静态工厂方法设置都会覆盖相应的父bean设置。

剩下的设置会取子bean定义：依赖、自动注入模式、依赖检查、单例、延迟加载。

前面的示例通过使用`abstract`属性将父bean定义显式标记为`abstract`。 如果父定义未指定类，则需要将父bean定义显式标记为`abstract`，如以下示例所示：

```xml
<bean id="inheritedTestBeanWithoutClass" abstract="true">
    <property name="name" value="parent"/>
    <property name="age" value="1"/>
</bean>

<bean id="inheritsWithClass" class="org.springframework.beans.DerivedTestBean"
        parent="inheritedTestBeanWithoutClass" init-method="initialize">
    <property name="name" value="override"/>
    <!-- age will inherit the value of 1 from the parent bean definition-->
</bean>
```

父bean不能单独实例化，因为它不完整，并且也明确标记为`abstract`。当定义是`abstract`的时，它只能用作纯模板bean定义，用作子定义的父定义。如果试图单独地使用声明了`abstract`的父bean， 通过引用它作为另一个bean的ref属性，或者使用父bean id进行显式的`getBean()`调用，都将返回一个错误。同样，容器内部的 `preInstantiateSingletons()`方法也会忽略定义为`abstract`的bean。

`ApplicationContext` 默认会预先实例化所有的单例bean。因此，如果开发者打算把(父）bean定义仅仅作为模板来使用，同时为它指定了class属性， 那么必须确保设置_abstract_的属性值为true。否则，应用程序上下文会(尝试）预实例化这个`abstract` bean。

<a id="beans-factory-extension"></a>

### [](#beans-factory-extension)1.8. 容器的扩展点

通常，应用程序开发者无需继承`ApplicationContext`的实现类。相反，Spring IoC容器可以通过插入特殊的集成接口实现进行扩展。接下来的几节将介绍这些集成接口。

<a id="beans-factory-extension-bpp"></a>

#### [](#beans-factory-extension-bpp)1.8.1. 使用`BeanPostProcessor`自定义Bean

`BeanPostProcessor`接口定义了可以实现的回调方法，以提供您自己的（或覆盖容器的默认）实例化逻辑，依赖关系解析逻辑等。 如果要在Spring容器完成实例化，配置和初始化bean之后实现某些自定义逻辑，则可以插入一个或多个`BeanPostProcessor`实现。

您可以配置多个`BeanPostProcessor` 实例，并且可以通过设置`order`属性来控制这些 `BeanPostProcessor` 实例的执行顺序。 仅当`BeanPostProcessor`实现 `Ordered`接口时，才能设置此属性。如果编写自己的`BeanPostProcessor`，则应考虑实现Ordered接口。 有关更多详细信息， 请参阅[`BeanPostProcessor`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/config/BeanPostProcessor.html) 和 [`Ordered`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/core/Ordered.html)的javadoc。 另请参阅有关[`BeanPostProcessor` 实例](#beans-factory-programmatically-registering-beanpostprocessors)的编程注册的说明。

`BeanPostProcessor`实例在bean（或对象）实例上运行。 也就是说，Spring IoC容器实例化一个bean实例，然后才能用`BeanPostProcessor` 对这个实例进行处理。

`BeanPostProcessor`会在整个容器内起作用，所有它仅仅与正在使用的容器相关。如果在一个容器中定义了`BeanPostProcessor`，那么它只会处理那个容器中的bean。 换句话说，在一个容器中定义的bean不会被另一个容器定义的`BeanPostProcessor`处理，即使这两个容器都是同一层次结构的一部分。

要更改实际的bean定义（即定义bean的蓝图），您需要使用`BeanFactoryPostProcessor`，使用BeanFactoryPostProcessor自定义配置元数据。 [使用 `BeanFactoryPostProcessor`自定义配置元数据](#beans-factory-extension-factory-postprocessors).

`org.springframework.beans.factory.config.BeanPostProcessor`接口由两个回调方法组成，当一个类被注册为容器的后置处理器时，对于容器创建的每个bean实例， 后置处理器都会在容器初始化方法（如`InitializingBean.afterPropertiesSet()`之前和容器声明的`init`方法）以及任何bean初始化回调之后被调用。后置处理器可以对bean实例执行任何操作， 包括完全忽略回调。bean后置处理器，通常会检查回调接口或者使用代理包装bean。一些Spring AOP基础架构类为了提供包装好的代理逻辑，会被实现为bean后置处理器。

`ApplicationContext`会自动地检测所有定义在配置元文件中，并实现了`BeanPostProcessor` 接口的bean。`ApplicationContext`会注册这些beans为后置处理器， 使他们可以在bean创建完成之后被调用。bean后置处理器可以像其他bean一样部署到容器中。

当在配置类上使用 `@Bean` 工厂方法声明`BeanPostProcessor`时，工厂方法返回的类型应该是实现类自身。，或至少也是`org.springframework.beans.factory.config.BeanPostProcessor`接口， 要清楚地表明这个bean的后置处理器的本质特点。否则，在它完全创建之前，`ApplicationContext`将不能通过类型自动探测它。由于`BeanPostProcessor`在早期就需要被实例化， 以适应上下文中其他bean的实例化，因此这个早期的类型检查是至关重要的。

以编程方式注册`BeanPostProcessor`实例，虽然`BeanPostProcessor`注册的推荐方法是通过`ApplicationContext`自动检测（如前所述），但您可以以编程的方式使用`ConfigurableBeanFactory`的`addBeanPostProcessor`方法进行注册。 这对于在注册之前需要对条件逻辑进行评估，或者是在继承层次的上下文之间复制bean的后置处理器中是有很有用的。 但请注意，以编程方式添加的`BeanPostProcessor`实例不遵循`Ordered`接口。这里，注册顺序决定了执行的顺序。 另请注意，以编程方式注册的`BeanPostProcessor`实例始终在通过自动检测注册的实例之前处理，而不管任何显式排序。

`BeanPostProcessor` 实例 and AOP 自动代理

实现`BeanPostProcessor` 接口的类是特殊的，容器会对它们进行不同的处理。所有`BeanPostProcessor` 和他们直接引用的beans都会在容器启动的时候被实例化， 并作为`ApplicationContext`特殊启动阶段的一部分。接着，所有的`BeanPostProcessor` 都会以一个有序的方式进行注册，并应用于容器中的所有bean。 因为AOP自动代理本身被实现为`BeanPostProcessor`，这个`BeanPostProcessor`和它直接应用的beans都不适合进行自动代理，因此也就无法在它们中织入切面。

对于所有这样的bean，您应该看到一条信息性日志消息: `Bean someBean is not eligible for getting processed by all BeanPostProcessor interfaces (for example: not eligible for auto-proxying)`.

如果你使用自动装配或 `@Resource`（可能会回退到自动装配）将Bean连接到`BeanPostProcessor`中，Spring可能会在搜索类型匹配的依赖关系候选时访问到意外类型的bean； 因此，对它们不适合进行自动代理，或者对其他类型的bean进行后置处理。例如，如果有一个使用 `@Resource` 注解的依赖项，其中字段或setter名称不直接对应于bean的声明名称而且没有使用name属性， 则Spring会访问其他bean以按类型匹配它们。

以下示例显示如何在`ApplicationContext`中编写，注册和使用`BeanPostProcessor`实例。

<a id="beans-factory-extension-bpp-examples-hw"></a>

##### [](#beans-factory-extension-bpp-examples-hw)示例: Hello World, `BeanPostProcessor`-style

第一个例子说明了基本用法。 该示例显示了一个自定义`BeanPostProcessor`实现，该实现在容器创建时调用每个bean的 `toString()` 方法，并将生成的字符串输出到系统控制台。

以下清单显示了自定义`BeanPostProcessor` 实现类定义:

```java
package scripting;

import org.springframework.beans.factory.config.BeanPostProcessor;

public class InstantiationTracingBeanPostProcessor implements BeanPostProcessor {

    // simply return the instantiated bean as-is
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean; // we could potentially return any object reference here...
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("Bean '" + beanName + "' created : " + bean.toString());
        return bean;
    }
}
```

以下beans元素使用`InstantiationTracingBeanPostProcessor`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:lang="http://www.springframework.org/schema/lang"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/lang
        http://www.springframework.org/schema/lang/spring-lang.xsd">

    <lang:groovy id="messenger"
            script-source="classpath:org/springframework/scripting/groovy/Messenger.groovy">
        <lang:property name="message" value="Fiona Apple Is Just So Dreamy."/>
    </lang:groovy>

    <!--
    when the above bean (messenger) is instantiated, this custom
    BeanPostProcessor implementation will output the fact to the system console
    -->
    <bean class="scripting.InstantiationTracingBeanPostProcessor"/>

</beans>
```

注意`InstantiationTracingBeanPostProcessor`是如何定义的，它甚至没有名字，因为它是一个bean，所以它可以像任何其他bean一样进行依赖注入 （前面的配置还定义了一个由Groovy脚本支持的bean。在[动态语言支持](https://github.com/DocsHome/spring-docs/blob/master/pages/languages/languages.md#dynamic-language)一章中详细介绍了Spring动态语言支持）。

下面简单的Java应用执行了前面代码和配置:

```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.scripting.Messenger;

public final class Boot {

    public static void main(final String[] args) throws Exception {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("scripting/beans.xml");
        Messenger messenger = (Messenger) ctx.getBean("messenger");
        System.out.println(messenger);
    }

}
```

上述应用程序的输出类似于以下内容:

Bean 'messenger' created :  org.springframework.scripting.groovy.GroovyMessenger@272961
org.springframework.scripting.groovy.GroovyMessenger@272961

<a id="beans-factory-extension-bpp-examples-rabpp"></a>

##### [](#beans-factory-extension-bpp-examples-rabpp)示例: The `RequiredAnnotationBeanPostProcessor`

自定义`BeanPostProcessor`实现与回调接口或注解配合使用，是一种常见的扩展Spring IoC容器手段，一个例子就是`RequiredAnnotationBeanPostProcessor`，这是`BeanPostProcessor`实现。 它确保用（任意）注解标记的bean上的JavaBean属性实际上（配置为）依赖注入值。

<a id="beans-factory-extension-factory-postprocessors"></a>

#### [](#beans-factory-extension-factory-postprocessors)1.8.2. 使用`BeanFactoryPostProcessor`自定义元数据配置

下一个我们要关注的扩展点是`org.springframework.beans.factory.config.BeanFactoryPostProcessor`。这个接口的语义与`BeanPostProcessor`类似， 但有一处不同，`BeanFactoryPostProcessor`操作bean的元数据配置。也就是说，也就是说，Spring IoC容器允许`BeanFactoryPostProcessor`读取配置元数据， 并可能在容器实例化除`BeanFactoryPostProcessor`实例之外的任何bean之前更改它。

您可以配置多个`BeanFactoryPostProcessor`实例，并且可以通过设置`order`属性来控制这些`BeanFactoryPostProcessor`实例的运行顺序（`BeanFactoryPostProcessor`必须实现了`Ordered`接口才能设置这个属性）。 如果编写自己的BeanFactoryPostProcessor，则应考虑实现Ordered接口。 有关更多详细信息， 请参阅[`BeanFactoryPostProcessor`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/config/BeanFactoryPostProcessor.html) 和 [`Ordered`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/core/Ordered.html) 接口的javadoc.

如果想修改实际的bean实例（也就是说，从元数据配置中创建的对象）那么需要使用`BeanPostProcessor`（前面在使用[`BeanPostProcessor`自定义Bean](#beans-factory-extension-bpp)中进行了描述）来替代。 在`BeanFactoryPostProcessor`（例如使用`BeanFactory.getBean()`）中使用这些bean的实例虽然在技术上是可行的，但这么来做会将bean过早实例化， 这违反了标准的容器生命周期。同时也会引发一些副作用，例如绕过bean的后置处理。

`BeanFactoryPostProcessor`会在整个容器内起作用，所有它仅仅与正在使用的容器相关。如果在一个容器中定义了`BeanFactoryPostProcessor`， 那么它只会处理那个容器中的bean。 换句话说，在一个容器中定义的bean不会被另一个容器定义的`BeanFactoryPostProcessor`处理，即使这两个容器都是同一层次结构的一部分。

bean工厂后置处理器在`ApplicationContext`中声明时自动执行，这样就可以对定义在容器中的元数据配置进行修改。 Spring包含许多预定义的bean工厂后处理器， 例如`PropertyOverrideConfigurer` 和`PropertyPlaceholderConfigurer`。 您还可以使用自定义`BeanFactoryPostProcessor`。 例如，注册自定义属性编辑器。 .

`ApplicationContext` 自动检测部署到其中的任何实现`BeanFactoryPostProcessor`接口的bean。 它在适当的时候使用这些bean作为bean工厂后置处理器。 你可以部署这些后置处理器为你想用的任意其它bean。

注意，和`BeanPostProcessor`一样，通常不应该配置`BeanFactoryPostProcessor`来进行延迟初始化。如果没有其它bean引用`Bean(Factory)PostProcessor`， 那么后置处理器就不会被初始化。因此，标记它为延迟初始化就会被忽略，，即便你在`<beans />`元素声明中设置`default-lazy-init`=true属性，`Bean(Factory)PostProcessor`也会提前初始化bean。

<a id="beans-factory-placeholderconfigurer"></a>

##### [](#beans-factory-placeholderconfigurer)示例: 类名替换`PropertyPlaceholderConfigurer`

您可以使用`PropertyPlaceholderConfigurer`通过使用标准Java `Properties`格式从单独文件中的bean定义外部化属性值。 这样做可以使部署应用程序的人能够定制特定于环境的属性，如数据库URL和密码，而无需修改容器的主XML定义文件或文件的复杂性或风险。

考虑以下这个基于XML的元数据配置代码片段，这里的DataSource使用了占位符来定义:

```xml
<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations" value="classpath:com/something/jdbc.properties"/>
</bean>

<bean id="dataSource" destroy-method="close"
        class="org.apache.commons.dbcp.BasicDataSource">
    <property name="driverClassName" value="${jdbc.driverClassName}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
</bean>
```

该示例显示了从外部属性文件配置的属性。在运行时，`PropertyPlaceholderConfigurer`应用于替换DataSource的某些属性的元数据。 要替换的值被指定为$ {property-name}形式的占位符，它遵循Ant和log4j以及JSP EL样式。

而真正的值是来自于标准的Java `Properties`格式的文件:

```
jdbc.driverClassName=org.hsqldb.jdbcDriver
jdbc.url=jdbc:hsqldb:hsql://production:9002
jdbc.username=sa
jdbc.password=root
```

在上面的例子中，`${jdbc.username}` 字符串在运行时将替换为值'sa'，并且同样适用于与属性文件中的键匹配的其他占位符值。 `PropertyPlaceholderConfigurer`检查bean定义的大多数属性和属性中的占位符。 此外，您可以自定义占位符前缀和后缀。

使用Spring 2.5中引入的`context` 命名空间，您可以使用专用配置元素配置属性占位符。 您可以在`location`属性中以逗号分隔列表的形式提供一个或多个位置，如以下示例所示：

```xml
<context:property-placeholder location="classpath:com/something/jdbc.properties"/>
```

`PropertyPlaceholderConfigurer`不仅在您指定的属性文件中查找属性。 默认情况下，如果它在指定的属性文件中找不到属性，它还会检查Java `System`属性。 开发者可以通过设置`systemPropertiesMode`属性，使用下面三个整数的某一个来自定义这种行为：

*   `never` (0): 从不检查系统属性。

*   `fallback` (1): 如果没有在指定的属性文件中解析到属性，那么就检查系统属性（默认）。

*   `override` (2): 在检查指定的属性文件之前，首先去检查系统属性，允许系统属性覆盖其它任意的属性资源。


有关更多信息，请参见[`PropertyPlaceholderConfigurer`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/config/PropertyPlaceholderConfigurer.html) javadoc

你可以使用`PropertyPlaceholderConfigurer`来替换类名，当开发者在运行时需要选择某个特定的实现类时，这是很有用的。例如

```xml
<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations">
        <value>classpath:com/something/strategy.properties</value>
    </property>
    <property name="properties">
        <value>custom.strategy.class=com.something.DefaultStrategy</value>
    </property>
</bean>

<bean id="serviceStrategy" class="${custom.strategy.class}"/>
```

如果在运行时无法将类解析为有效类，则在即将创建bean时，bean的解析将失败，这是 `ApplicationContext`在对非延迟初始化bean的`preInstantiateSingletons()`阶段发生的事。

<a id="beans-factory-overrideconfigurer"></a>

##### [](#beans-factory-overrideconfigurer)示例: `PropertyOverrideConfigurer`

`PropertyOverrideConfigurer`, 另外一种bean工厂后置处理器，类似于`PropertyPlaceholderConfigurer`，但与后者不同的是：对于所有的bean属性，原始定义可以有默认值或也可能没有值。 如果一个`Properties`覆盖文件没有配置特定的bean属性，则就会使用默认的上下文定义

注意，bean定义是不知道是否被覆盖的，所以从XML定义文件中不能马上看到那个配置正在被使用。在拥有多个`PropertyOverrideConfigurer` 实例的情况下，为相同bean的属性定义不同的值时，基于覆盖机制只会有最后一个生效。

属性文件配置行采用以下格式:

```
beanName.property=value
```

例如:

```
dataSource.driverClassName=com.mysql.jdbc.Driver
dataSource.url=jdbc:mysql:mydb
```

这个示例文件可以和容器定义一起使用，该容器定义包含一个名为`dataSource`的bean，该bean具有 `driver`和`url`属性

复合属性名称也是被支持的，只要被重写的最后一个属性以外的路径中每个组件都已经是非空时（假设由构造方法初始化）。 在下面的示例中，`tom` bean的`fred`属性的 `bob`属性的`sammy`属性设置值为`123`：

```
tom.fred.bob.sammy=123
```

指定的覆盖值通常是文字值，它们不会被转换成bean的引用。这个约定也适用于当XML中的bean定义的原始值指定了bean引用时。

使用Spring 2.5中引入的`context`命名空间，可以使用专用配置元素配置属性覆盖，如以下示例所示：

    <context:property-override location="classpath:override.properties"/>

<a id="beans-factory-extension-factorybean"></a>

#### [](#beans-factory-extension-factorybean)1.8.3. 使用`FactoryBean`自定义初始化逻辑

为自己工厂的对象实现`org.springframework.beans.factory.FactoryBean`接口。

`FactoryBean`接口就是Spring IoC容器实例化逻辑的可插拔点，如果你的初始化代码非常复杂，那么相对于（潜在地）大量详细的XML而言，最好是使用Java语言来表达。 你可以创建自定义的`FactoryBean`，在该类中编写复杂的初始化代码。然后将自定义的`FactoryBean`插入到容器中。

`FactoryBean`接口提供下面三个方法

*   `Object getObject()`: 返回这个工厂创建的对象实例。这个实例可能是共享的，这取决于这个工厂返回的是单例还是原型实例。

*   `boolean isSingleton()`: 如果`FactoryBean`返回单例，那么这个方法就返回`true`，否则返回`false`。

*   `Class getObjectType()`: 返回由`getObject()`方法返回的对象类型，如果事先不知道的类型则会返回null。


Spring框架大量地使用了`FactoryBean` 的概念和接口，`FactoryBean` 接口的50多个实现都随着Spring一同提供。

当开发者需要向容器请求一个真实的`FactoryBean`实例（而不是它生产的bean）时，调用 `ApplicationContext`的`getBean()`方法时在bean的id之前需要添加连字符（&） 所以对于一个给定id为myBean的`FactoryBean`，调用容器的`getBean("myBean")`方法返回的是FactoryBean的代理，而调用`getBean("&myBean")`方法则返回FactoryBean实例本身

<a id="beans-annotation-config"></a>

### [](#beans-annotation-config)1.9. 基于注解的容器配置

注解是否比配置Spring的XML更好?

在引入基于注解的配置之后,引发了这种方法是否比XML更优秀的问题.简短的答案是得看情况,每种方法都有其优缺点。通常由开发人员决定使用更适合他们的策略。 首先看看两种定义方式,注解在它们的声明中提供了很多上下文信息，使得配置变得更短、更简洁；但是，XML擅长于在不接触源码或者无需反编译的情况下装配组件，一些开发人员更喜欢在源码上使用注解配置。 而另一些人认为注解类不再是POJO，同时认为注解配置会很分散，最终难以控制。

无论选择如何，Spring都可以兼顾两种风格，甚至可以将它们混合在一起。Spring通过其[JavaConfig](#beans-java) 选项，允许注解以无侵入的方式使用，即无需接触目标组件源代码。 而且在工具应用方面， [Spring Tool Suite](https://spring.io/tools/sts)支持所有配置形式。

XML设置的替代方法是基于注解的配置，它依赖于字节码元数据来连接组件进而替代XML声明。开发人员通过使用相关类、方法或字段声明上的注解来将配置移动到组件类本身。而不是使用XML bean来配置。 如示例中所述，[`RequiredAnnotationBeanPostProcessor`](#beans-factory-extension-bpp-examples-rabpp),将`BeanPostProcessor` 与注解混合使用是扩展Spring IoC容器的常用方法。 例如，Spring 2.0引入了使用[`@Required`](#beans-required-annotation)注解强制属性必须在配置的时候被填充， Spring 2.5使用同样的方式来驱动Spring的依赖注入。本质上，`@Autowired`注解提供的功能与[自动装配协作](#beans-factory-autowire)中描述的相同，但具有更细粒度的控制和更广泛的适用性。 Spring 2.5还增加了对JSR-250注解的支持，例如`@PostConstruct`和`@PreDestroy`。 Spring 3.0增加了对`javax.inject`包中包含的JSR-330（Java的依赖注入）注解的支持， 例如`@Inject`和`@Named`。有关这些注解的详细信息，请参阅[相关章节](#beans-standard-annotations)。

注解注入在XML注入之前执行，因此同时使用这两种方式进行注入时，XML配置会覆盖注解配置。

与之前一样，你可以将它们注册为单独的bean定义，但也可以通过在基于XML的Spring配置中包含以下标记来隐式注册它们（请注意包含`context`命名空间）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

</beans>
```

（隐式注册的后处理器包括 [`AutowiredAnnotationBeanPostProcessor`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.html), [`CommonAnnotationBeanPostProcessor`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/annotation/CommonAnnotationBeanPostProcessor.html), [`PersistenceAnnotationBeanPostProcessor`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/orm/jpa/support/PersistenceAnnotationBeanPostProcessor.html), 和前面提到的 [`RequiredAnnotationBeanPostProcessor`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/annotation/RequiredAnnotationBeanPostProcessor.html).)

`<context:annotation-config/>`只有在定义bean的相同应用程序上下文中查找bean上的注解。 这意味着，如果将 `<context:annotation-config/>` 放在`DispatcherServlet`的`WebApplicationContext`中， 它只检查控制器中的`@Autowired` bean，而不检查您的服务。 有关更多信息，请参阅 [DispatcherServlet](https://github.com/DocsHome/spring-docs/blob/master/pages/web/web.md#mvc-servlet)。

<a id="beans-required-annotation"></a>

#### [](#beans-required-annotation)1.9.1. @Required

`@Required`注解适用于bean属性setter方法，如下例所示：

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Required
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

此注解仅表示受影响的bean属性必须在配置时通过bean定义中的显式赋值或自动注入值。如果受影响的bean属性尚未指定值，容器将抛出异常；这导致及时的、明确的失败，避免在运行后再抛出`NullPointerException`或类似的异常。 在这里，建议开发者将断言放入bean类本身，例如放入init方法。这样做强制执行那些必需的引用和值，即使是在容器外使用这个类。

<a id="beans-autowired-annotation"></a>

#### [](#beans-autowired-annotation)1.9.2. `@Autowired`

可以使用JSR 330的 `@Inject`注解代替本节中包含的示例中的Spring的`@Autowired`注解。 有关详细信息，[请参见此处](#beans-standard-annotations)

开发者可以在构造器上使用`@Autowired`注解:

```java
public class MovieRecommender {

    private final CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

从Spring Framework 4.3开始，如果目标bean仅定义一个构造函数，则不再需要`@Autowired`构造函数。如果有多个构造函数可用，则至少有一个必须注解`@Autowired`以让容器知道它使用的是哪个

您还可以将`@Autowired`注解应用于“传统”setter方法，如以下示例所示：

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Autowired
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

您还可以将注解应用于具有任意名称和多个参数的方法，如以下示例所示：:

```java
public class MovieRecommender {

    private MovieCatalog movieCatalog;

    private CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public void prepare(MovieCatalog movieCatalog,
            CustomerPreferenceDao customerPreferenceDao) {
        this.movieCatalog = movieCatalog;
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

还可以将`@Autowired`应用于字段，甚至可以和构造函数混合使用:

```java
public class MovieRecommender {

    private final CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    private MovieCatalog movieCatalog;

    @Autowired
    public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

确保您的组件（例如，`MovieCatalog`或`CustomerPreferenceDao`）始终按照用于@Autowired注入点的类型声明。 否则，由于在运行时未找到类型匹配，注入可能会失败。

对于通过类路径扫描找到的XML定义的bean或组件类，容器通常预先知道具体类型。 但是，对于`@Bean`工厂方法，您需要确保其声明的具体返回类型。 对于实现多个接口的组件或可能由其实现类型引用的组件，请考虑在工厂方法上声明最具体的返回类型（至少与引用bean的注入点所需的特定类型一致）。 .

也可以用在数组上，注解用于标注属性或方法，数组的类型是`ApplicationContext`中定义的bean类型。如以下示例所示：

```java
public class MovieRecommender {

    @Autowired
    private MovieCatalog[] movieCatalogs;

    // ...
}
```

也可以应用于集合类型，如以下示例所示:

```java
public class MovieRecommender {

    private Set<MovieCatalog> movieCatalogs;

    @Autowired
    public void setMovieCatalogs(Set<MovieCatalog> movieCatalogs) {
        this.movieCatalogs = movieCatalogs;
    }

    // ...
}
```

如果想让数组元素或集合元素按特定顺序排列，应用的bean可以实现`org.springframework.core.Ordered`， 或者使用`@Order`或标准的@`@Priority` 注解，否则，它们的顺序遵循容器中相应目标bean定义的注册顺序。

您可以在类级别和`@Bean`方法上声明 `@Order`注解，可能是通过单个bean定义（在多个定义使用相同bean类的情况下）。 `@Order`值可能会影响注入点的优先级，但要注意它们不会影响单例启动顺序，这是由依赖关系和`@DependsOn`声明确定的。

请注意，标准的`javax.annotation.Priority`注解在`@Bean`级别不可用，因为它无法在方法上声明。 它的语义可以通过`@Order`值与`@Primary`定义每个类型的单个bean上。

只要键类型是`String`，`Map`类型就可以自动注入。 Map值将包含所有类型的bean，并且键将包含相应的bean名称。如以下示例所示：

```java
public class MovieRecommender {

    private Map<String, MovieCatalog> movieCatalogs;

    @Autowired
    public void setMovieCatalogs(Map<String, MovieCatalog> movieCatalogs) {
        this.movieCatalogs = movieCatalogs;
    }

    // ...
}
```

默认情况下，当没有候选的bean可用时，自动注入将会失败；默认的处理方式是将带有注解的方法，、构造函数和字段标明为required=false属性。这种设置不是必须的，如下：

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Autowired(required = false)
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

只有一个带注解的构造函数per-class 可以标记为required，但是可以注解多个非必需的构造函数。在这种情况下，每个项都会是候选者，而Spring使用的是最贪婪的构造函数。 这个构造函数的依赖关系可以得到满足，那就是具有最多参数的构造函数。

推荐使用`@Required`注解来代替`@Autowired`的required属性，required属性表示该属性不是自动装配必需的，如果该属性不能被自动装配。 则该属性会被忽略。 另一方面， `@Required`会强调通过容器支持的任何方式来设置属性。 如果没有值被注入的话，会引发相应的异常。

或者，您可以通过Java 8的`java.util.Optional`表达特定依赖项的非必需特性，如以下示例所示：

```java
public class SimpleMovieLister {

    @Autowired
    public void setMovieFinder(Optional<MovieFinder> movieFinder) {
        ...
    }
}
```

从Spring Framework 5.0开始，您还可以使用`@Nullable` 注解（任何包中的任何类型，例如，来自JSR-305的 `javax.annotation.Nullable`）：

```java
public class SimpleMovieLister {

    @Autowired
    public void setMovieFinder(@Nullable MovieFinder movieFinder) {
        ...
    }
}
```

您也可以使用`@Autowired`作为常见的可解析依赖关系的接口，`BeanFactory`, `ApplicationContext`, `Environment`, `ResourceLoader`, `ApplicationEventPublisher`, 和 `MessageSource` 这些接口及其扩展接口（例如`ConfigurableApplicationContext` 或 `ResourcePatternResolver`） 会自动解析，无需特殊设置。 以下示例自动装配`ApplicationContext`对象：

```java
public class MovieRecommender {

    @Autowired
    private ApplicationContext context;

    public MovieRecommender() {
    }

    // ...
}
```

`@Autowired`, `@Inject`, `@Resource`, 和 `@Value` 注解 由 Spring `BeanPostProcessor` 实现.也就是说开发者不能使用自定义的`BeanPostProcessor`或者自定义`BeanFactoryPostProcessor`r来使用这些注解 必须使用XML或Spring @Bean方法显式地“连接”这些类型。

<a id="beans-autowired-annotation-primary"></a>

#### [](#beans-autowired-annotation-primary)1.9.3. `@Primary`

由于按类型的自动注入可能匹配到多个候选者，所以通常需要对选择过程添加更多的约束。使用Spring的`@Primary`注解是实现这个约束的一种方法。 它表示如果存在多个候选者且另一个bean只需要一个特定类型的bean依赖时，就明确使用标记有`@Primary`注解的那个依赖。如果候选中只有一个"Primary" bean，那么它就是自动注入的值

请考虑以下配置，将`firstMovieCatalog`定义为主要`MovieCatalog`：

```java
@Configuration
public class MovieConfiguration {

    @Bean
    @Primary
    public MovieCatalog firstMovieCatalog() { ... }

    @Bean
    public MovieCatalog secondMovieCatalog() { ... }

    // ...
}
```

使用上述配置，以下 `MovieRecommender`将与`firstMovieCatalog`一起自动装配：

```java
public class MovieRecommender {

    @Autowired
    private MovieCatalog movieCatalog;

    // ...
}
```

相应的bean定义如下:

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog" primary="true">
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

<a id="beans-autowired-annotation-qualifiers"></a>

#### [](#beans-autowired-annotation-qualifiers)1.9.4. 使用qualifiers微调基于注解的自动装配

`@Primary` 是一种用于解决自动装配多个值的注入的有效的方法，当需要对选择过程做更多的约束时，可以使用Spring的`@Qualifier`注解，可以为指定的参数绑定限定的值。 缩小类型匹配集，以便为每个参数选择特定的bean。 在最简单的情况下，这可以是一个简单的描述性值，如以下示例所示：

```java
public class MovieRecommender {

    @Autowired
    @Qualifier("main")
    private MovieCatalog movieCatalog;

    // ...
}
```

您还可以在各个构造函数参数或方法参数上指定`@Qualifier`注解，如以下示例所示：

```java
public class MovieRecommender {

    private MovieCatalog movieCatalog;

    private CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    public void prepare(@Qualifier("main")MovieCatalog movieCatalog,
            CustomerPreferenceDao customerPreferenceDao) {
        this.movieCatalog = movieCatalog;
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

以下示例显示了相应的bean定义。.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="main"/> (1)

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier value="action"/> (2)

        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

**1**、带有限定符"`main`"的bean会被装配到拥有相同值的构造方法参数上.

**2**、带有限定符"`action`"的bean会被装配到拥有相同值的构造方法参数上.

bean的name会作为备用的qualifier值,因此可以定义bean的`id`为 main 替代内嵌的qualifier元素.这种匹配方式同样有效。但是，虽然可以使用这个约定来按名称引用特定的bean， 但是`@Autowired`默认是由带限定符的类型驱动注入的。这就意味着qualifier值，甚至是bean的name作为备选项，只是为了缩小类型匹配的范围。它们在语义上不表示对唯一bean id的引用。 良好的限定符值是像`main` 或 `EMEA` 或 `persistent`这样的，能表示与bean id无关的特定组件的特征，在匿名bean定义的情况下可以自动生成。

Qualifiers也可以用于集合类型，如上所述，例如 `Set<MovieCatalog>`。在这种情况下，根据声明的限定符，所有匹配的bean都作为集合注入。 这意味着限定符不必是唯一的。 相反，它们构成过滤标准。 例如，您可以使用相同的限定符值“action”定义多个`MovieCatalog` bean，所有这些bean都注入到使用`@Qualifier("action")`注解的`Set<MovieCatalog>`中。

在类型匹配候选项中，根据目标bean名称选择限定符值，在注入点不需要`@Qualifier`注解。 如果没有其他解析指示符（例如限定符或主标记）， 则对于非唯一依赖性情况，Spring会将注入点名称（即字段名称或参数名称）与目标bean名称进行匹配，然后选择同名的候选者，如果有的话。

如果打算by name来驱动注解注入，那么就不要使用`@Autowired`（多数情况），即使在技术上能够通过@Qualifier值引用bean名字。相反，应该使用JSR-250 `@Resource` 注解，该注解在语义上定义为通过其唯一名称标识特定目标组件，其中声明的类型与匹配进程无关。`@Autowired`具有多种不同的语义，在by type选择候选bean之后，指定的`String`限定的值只会考虑这些被选择的候选者。 例如将`account` 限定符与标有相同限定符标签的bean相匹配。

对于自身定义为 collection, `Map`, 或者 array type的bean， `@Resource`是一个很好的解决方案，通过唯一名称引用特定的集合或数组bean。 也就是说，从Spring4.3开始，只要元素类型信息保存在 `@Bean` 返回类型签名或集合（或其子类）中，您就可以通过Spring的 `@Autowired`类型匹配算法匹配Map和数组类型。 在这种情况下，可以使用限定的值来选择相同类型的集合，如上一段所述。

从Spring4.3开始，`@Autowired`也考虑了注入的自引用，即引用当前注入的bean。自引用只是一种后备选项，还是优先使用正常的依赖注入操作其它bean。 在这个意义上，自引用不参与到正常的候选者选择中，并且总是次要的，，相反，它们总是拥有最低的优先级。在实践中，自引用通常被用作最后的手段。例如，通过bean的事务代理在同一实例上调用其他方法 在这种情况下，考虑将受影响的方法分解为单独委托的bean，或者使用 `@Resource`,，它可以通过其唯一名称获取代理返回到当前的bean上。

`@Autowired`可以应用在字段、构造函数和多参数方法上，允许在参数上使用qualifier限定符注解缩小取值范围。相比之下，`@Resource`仅支持具有单个参数的字段和bean属性setter方法。 因此，如果注入目标是构造函数或多参数方法，请使用qualifiers限定符。

开发者也可以创建自定义的限定符注解，只需定义一个注解，在其上提供了@Qualifier注解即可。如以下示例所示：

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Genre {

    String value();
}
```

然后，您可以在自动装配的字段和参数上提供自定义限定符，如以下示例所示：

```java
public class MovieRecommender {

    @Autowired
    @Genre("Action")
    private MovieCatalog actionCatalog;

    private MovieCatalog comedyCatalog;

    @Autowired
    public void setComedyCatalog(@Genre("Comedy") MovieCatalog comedyCatalog) {
        this.comedyCatalog = comedyCatalog;
    }

    // ...
}
```

接下来，提供候选bean定义的信息。开发者可以添加`<qualifier/>`标签作为`<bean/>`标签的子元素，然后指定 `type`类型和`value`值来匹配自定义的qualifier注解。 type是自定义注解的权限定类名(包路径+类名）。如果没有重名的注解，那么可以使用类名(不含包路径）。 以下示例演示了两种方法：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="Genre" value="Action"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="example.Genre" value="Comedy"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean id="movieRecommender" class="example.MovieRecommender"/>

</beans>
```

在 [类路径扫描和组件管理](#beans-classpath-scanning),将展示一个基于注解的替代方法，可以在XML中提供qualifier元数据, 请参阅[使用注解提供限定符元数据。](#beans-scanning-qualifiers)

在某些情况下，使用没有值的注解可能就足够了。当注解用于更通用的目的并且可以应用在多种不同类型的依赖上时，这是很有用的。 例如，您可以提供可在没有Internet连接时搜索的Offline目录。 首先，定义简单注解，如以下示例所示：

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Offline {

}
```

然后将注解添加到需要自动注入的字段或属性中:

```java
public class MovieRecommender {

    @Autowired
    @Offline (1)
    private MovieCatalog offlineCatalog;

    // ...
}
```

**1**、此行添加`@Offline`注解

现在bean定义只需要一个限定符类型，如下例所示：:

```xml
<bean class="example.SimpleMovieCatalog">
    <qualifier type="Offline"/> (1)
    <!-- inject any dependencies required by this bean -->
</bean>
```

**1**、此元素指定限定符。

开发者还可以为自定义限定名qualifier注解增加属性，用于替代简单的`value`属性。如果在要自动注入的字段或参数上指定了多个属性值，则bean的定义必须全部匹配这些属性值才能被视为自动注入候选者。 例如，请考虑以下注解定义：

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface MovieQualifier {

    String genre();

    Format format();
}
```

在这种情况下， `Format`是一个枚举类型，定义如下:

```java
public enum Format {
    VHS, DVD, BLURAY
}
```

要自动装配的字段使用自定义限定符进行注解，并包含两个属性的值：`genre` 和 `format`，如以下示例所示:

```java
public class MovieRecommender {

    @Autowired
    @MovieQualifier(format=Format.VHS, genre="Action")
    private MovieCatalog actionVhsCatalog;

    @Autowired
    @MovieQualifier(format=Format.VHS, genre="Comedy")
    private MovieCatalog comedyVhsCatalog;

    @Autowired
    @MovieQualifier(format=Format.DVD, genre="Action")
    private MovieCatalog actionDvdCatalog;

    @Autowired
    @MovieQualifier(format=Format.BLURAY, genre="Comedy")
    private MovieCatalog comedyBluRayCatalog;

    // ...
}
```

最后，bean定义应包含匹配的限定符值。此示例还演示了可以使用bean meta属性而不是使用`<qualifier/>`子元素。如果可行，`<qualifier/>`元素及其属性优先， 但如果不存在此类限定符，那么自动注入机制会使用 `<meta/>` 标签中提供的值，如以下示例中的最后两个bean定义：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="MovieQualifier">
            <attribute key="format" value="VHS"/>
            <attribute key="genre" value="Action"/>
        </qualifier>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <qualifier type="MovieQualifier">
            <attribute key="format" value="VHS"/>
            <attribute key="genre" value="Comedy"/>
        </qualifier>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <meta key="format" value="DVD"/>
        <meta key="genre" value="Action"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

    <bean class="example.SimpleMovieCatalog">
        <meta key="format" value="BLURAY"/>
        <meta key="genre" value="Comedy"/>
        <!-- inject any dependencies required by this bean -->
    </bean>

</beans>
```

<a id="beans-generics-as-qualifiers"></a>

#### [](#beans-generics-as-qualifiers)1.9.5. 使用泛型作为自动装配限定符

除了`@Qualifier` 注解之外，您还可以使用Java泛型类型作为隐式的限定形式。 例如，假设您具有以下配置：

```java
@Configuration
public class MyConfiguration {

    @Bean
    public StringStore stringStore() {
        return new StringStore();
    }

    @Bean
    public IntegerStore integerStore() {
        return new IntegerStore();
    }
}
```

假设上面的bean都实现了泛型接口,即 `Store<String>`和`Store<Integer>`,那么可以用`@Autowire`来注解`Store` 接口, 并将泛型用作限定符，如下例所示：

```java
@Autowired
private Store<String> s1; // <String> qualifier, injects the stringStore bean

@Autowired
private Store<Integer> s2; // <Integer> qualifier, injects the integerStore bean
```

通用限定符也适用于自动装配列表，`Map`实例和数组。 以下示例自动装配通用`List`：

```java
// Inject all Store beans as long as they have an <Integer> generic
// Store<String> beans will not appear in this list
@Autowired
private List<Store<Integer>> s;
```

<a id="beans-custom-autowire-configurer"></a>

#### [](#beans-custom-autowire-configurer)1.9.6. `CustomAutowireConfigurer`

[`CustomAutowireConfigurer`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/annotation/CustomAutowireConfigurer.html) 是一个`BeanFactoryPostProcessor`，它允许开发者注册自定义的qualifier注解类型，而无需指定`@Qualifier`注解，以下示例显示如何使用`CustomAutowireConfigurer`:

```xml
<bean id="customAutowireConfigurer"
        class="org.springframework.beans.factory.annotation.CustomAutowireConfigurer">
    <property name="customQualifierTypes">
        <set>
            <value>example.CustomQualifier</value>
        </set>
    </property>
</bean>
```

`AutowireCandidateResolver` 通过以下方式确定自动注入的候选者:

*   每个bean定义的`autowire-candidate`值

*   在`<beans/>`元素上使用任何可用的 `default-autowire-candidates` 模式

*   存在 `@Qualifier` 注解以及使用`CustomAutowireConfigurer`注册的任何自定义注解


当多个bean有资格作为自动注入的候选项时，“primary”的确定如下：如果候选者中只有一个bean定义的 `primary`属性设置为`true`，则选择它。

<a id="beans-resource-annotation"></a>

#### [](#beans-resource-annotation)1.9.7. `@Resource`

Spring还通过在字段或bean属性setter方法上使用JSR-250 `@Resource`注解来支持注入。 这是Java EE 5和6中的常见模式（例如，在JSF 1.2托管bean或JAX-WS 2.0端点中）。 Spring也为Spring管理对象提供这种模式。

`@Resource` 接受一个name属性.。默认情况下，Spring将该值解释为要注入的bean名称。 换句话说，它遵循按名称语义，如以下示例所示:

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Resource(name="myMovieFinder") (1)
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

**1**、这行注入一个`@Resource`.

如果未明确指定名称，则默认名称是从字段名称或setter方法派生的。 如果是字段，则采用字段名称。 在setter方法的情况下，它采用bean属性名称。 下面的例子将把名为`movieFinder`的bean注入其setter方法：

```java
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Resource
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

`ApplicationContext`若使用了`CommonAnnotationBeanPostProcessor`，注解提供的name名字将被解析为bean的name名字。 如果配置了Spring的 [`SimpleJndiBeanFactory`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/jndi/support/SimpleJndiBeanFactory.html)，这些name名称就可以通过JNDI解析。但是，推荐使用默认的配置，简单地使用Spring的JNDI，这样可以保持逻辑引用。而不是直接引用。

@Resource在没有明确指定name时，其行为类似于`@Autowired`，对于特定bean(Spring API内的bean）， `@Resource` 找到主要类型匹配而不是特定的命名bean， 并解析众所周知的可解析依赖项：`ApplicationContext`, `ResourceLoader`, `ApplicationEventPublisher`, 和 `MessageSource`接口。

因此，在以下示例中，`customerPreferenceDao`字段首先查找名为customerPreferenceDao的bean，如果未找到，则会使用类型匹配`CustomerPreferenceDao`类的实例：

```java
public class MovieRecommender {

    @Resource
    private CustomerPreferenceDao customerPreferenceDao;

    @Resource
    private ApplicationContext context; (1)

    public MovieRecommender() {
    }

    // ...
}
```

**1**、`context`域将会注入`ApplicationContext`

<a id="beans-postconstruct-and-predestroy-annotations"></a>

#### [](#beans-postconstruct-and-predestroy-annotations)1.9.8. `@PostConstruct` 和 `@PreDestroy`

`CommonAnnotationBeanPostProcessor` 不仅仅识别`@Resource` 注解，还识别JSR-250生命周期注解。，在Spring 2.5中引入了这些注解， 它们提供了另一个替代[初始化回调](#beans-factory-lifecycle-initializingbean)和[销毁回调](#beans-factory-lifecycle-disposablebean)。 如果`CommonAnnotationBeanPostProcessor`在Spring `ApplicationContext`中注册，它会在相应的Spring bean生命周期中调用相应的方法，就像是Spring生命周期接口方法，或者是明确声明的回调函数那样。 在以下示例中，缓存在初始化时预先填充并在销毁时清除：

```java
public class CachingMovieLister {

    @PostConstruct
    public void populateMovieCache() {
        // populates the movie cache upon initialization...
    }

    @PreDestroy
    public void clearMovieCache() {
        // clears the movie cache upon destruction...
    }
}
```

有关组合各种生命周期机制的影响的详细信息，请参阅组合[生命周期机制](#beans-factory-lifecycle-combined-effects)。

<a id="beans-classpath-scanning"></a>

### [](#beans-classpath-scanning)1.10. 类路径扫描和管理组件

本章中的大多数示例会使用XML配置指定在Spring容器中生成每个`BeanDefinition`的元数据，上一节（[基于注解的容器配置](#beans-annotation-config)）演示了如何通过源代码注解提供大量的元数据配置。 然而，即使在这些示例中，注解也仅仅用于驱动依赖注入。 “base” bean依然会显式地在XML文件中定义。本节介绍通过扫描类路径隐式检测候选组件的选项。候选者组件是class类， 这些类经过过滤匹配，由Spring容器注册的bean定义会成为Spring bean。这消除了使用XML执行bean注册的需要(也就是没有XML什么事儿了),可以使用注解(例如`@Component`)， AspectJ类型表达式或开发者自定义过滤条件来选择哪些类将在容器中注册bean定义。

从Spring 3.0开始，Spring JavaConfig项目提供的许多功能都是核心Spring 框架的一部分。这允许开发者使用Java而不是使用传统的XML文件来定义bean。 有关如何使用这些新功能的示例，请查看`@Configuration`, `@Bean`,`@Import`, 和 `@DependsOn`注解。

<a id="beans-stereotype-annotations"></a>

#### [](#beans-stereotype-annotations)1.10.1. `@Component`注解和更多模板注解

`@Repository`注解用于满足存储库(也称为数据访问对象或DAO)的情况,这个注解的用途是自动转换异常。如[异常转换](https://github.com/DocsHome/spring-docs/blob/master/pages/dataaccess/data-access.md#orm-exception-translation)中所述。

Spring提供了更多的构造型注解：`@Component`, `@Service`, 和`@Controller`. `@Component` 可用于管理任何Spring的组件。 `@Repository`, `@Service`, 或 `@Controller`是`@Component`的特殊化。用于更具体的用例（分别在持久性，服务和表示层中）。 因此，您可以使用`@Component`注解组件类，但是，通过使用`@Repository`, `@Service`, 和 `@Controller`注解它们，能够让你的类更易于被合适的工具处理或与相应的切面关联。 例如，这些注解可以使目标组件变成切入点。在Spring框架的未来版本中，`@Repository`, `@Service`, 和 `@Controller`也可能带有附加的语义。 因此，如果在使用`@Component` 或 `@Service` 来选择服务层时，@Service显然是更好的选择。同理，在持久化层要选择`@Repository`，它能自动转换异常。

<a id="beans-meta-annotations"></a>

#### [](#beans-meta-annotations)1.10.2. 使用元注解和组合注解

Spring提供的许多注解都可以在您自己的代码中用作元注解。 元注解是可以应用于另一个注解的注解。 例如，[前面提到的](#beans-stereotype-annotations) `@Service`注解是使用`@Component`进行元注解的，如下例所示：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component (1)
public @interface Service {

    // ....
}
```

**1**、`Component`使`@Service`以与@`@Component`相同的方式处理。

元注解也可以进行组合，进而创建组合注解。例如，来自Spring MVC的`@RestController`注解是由`@Controller`和`@ResponseBody`组成的

此外，组合注解也可以重新定义来自元注解的属性。这在只想公开元注解的部分属性时非常有用。例如，Spring的`@SessionScope`注解将它的作用域硬编码为`session`，但仍允许自定义`proxyMode`。 以下清单显示了`SessionScope`注解的定义：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Scope(WebApplicationContext.SCOPE_SESSION)
public @interface SessionScope {

    /**
     * Alias for {@link Scope#proxyMode}.
     * <p>Defaults to {@link ScopedProxyMode#TARGET_CLASS}.
     */
    @AliasFor(annotation = Scope.class)
    ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;

}
```

然后，您可以使用`@SessionScope`而不声明`proxyMode`，如下所示：

```java
@Service
@SessionScope
public class SessionScopedService {
    // ...
}
```

您还可以覆盖`proxyMode`的值，如以下示例所示:

```java
@Service
@SessionScope(proxyMode = ScopedProxyMode.INTERFACES)
public class SessionScopedUserService implements UserService {
    // ...
}
```

有关更多详细信息，请参阅[Spring注解编程模型](https://github.com/spring-projects/spring-framework/wiki/Spring-Annotation-Programming-Model)wiki页面.

<a id="beans-scanning-autodetection"></a>

#### [](#beans-scanning-autodetection)1.10.3. 自动探测类并注册bean定义

Spring可以自动检测各代码层中被注解的类，并使用`ApplicationContext`内注册相应的`BeanDefinition`。例如，以下两个类就可以被自动探测：

```java
@Service
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Autowired
    public SimpleMovieLister(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}

@Repository
public class JpaMovieFinder implements MovieFinder {
    // implementation elided for clarity
}
```

想要自动检测这些类并注册相应的bean，需要在`@Configuration`配置中添加`@ComponentScan`注解，其中`basePackages`属性是两个类的父包路径。 （或者，您可以指定以逗号或分号或空格分隔的列表，其中包含每个类的父包）。

```java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
    ...
}
```

为简洁起见，前面的示例可能使用了注解的`value`属性（即`@ComponentScan("org.example")`）。

或者使用XML配置代替扫描:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="org.example"/>

</beans>
```

使用`<context:component-scan>`隐式启用`<context:annotation-config>`的功能。 使用`<context:component-scan>`时，通常不需要包含`<context:annotation-config>`元素。

类路径扫描的包必须保证这些包出现在类路径中。当使用Ant构建JAR时，请确保你没有激活JAR任务的纯文件开关。此外在某些环境装由于安全策略，类路径目录可能不能访问。 JDK 1.7.0_45及更高版本上的独立应用程序（需要在清单中设置“Trusted-Library”） - 请参阅 [http://stackoverflow.com/questions/19394570/java-jre-7u45-breaks-classloader-getresources](https://stackoverflow.com/questions/19394570/java-jre-7u45-breaks-classloader-getresources)）。

在JDK 9的模块路径（Jigsaw）上，Spring的类路径扫描通常按预期工作。，但是，请确保在模块信息描述符中导出组件类。 如果您希望Spring调用类的非公共成员，请确保它们已“打开”（即，它们在`module-info`描述符中使用`opens` 声明而不是`exports`声明）。

在使用component-scan元素时， `AutowiredAnnotationBeanPostProcessor` 和 `CommonAnnotationBeanPostProcessor`都会隐式包含。意味着这两个组件也是自动探测和注入的。 所有这些都无需XML配置。

您可以通过annotation-config=false属性来禁用`AutowiredAnnotationBeanPostProcessor` 和`CommonAnnotationBeanPostProcessor`的注册。

<a id="beans-scanning-filters"></a>

#### [](#beans-scanning-filters)1.10.4. 在自定义扫描中使用过滤器

默认情况下，使用`@Component`, `@Repository`, `@Service`,`@Controller`注解的类或者注解为`@Component`的自定义注解类才能被检测为候选组件。 但是，开发者可以通过应用自定义过滤器来修改和扩展此行为。将它们添加为`@ComponentScan`注解的`includeFilters`或`excludeFilters`参数(或作为`component-scan` 元素。元素的`include-filter`或`exclude-filter`子元素。每个filter元素都需要包含`type`和`expression`属性。下表介绍了筛选选项：

Table 5.过滤类型

| 过滤类型             | 表达式例子                   | 描述                                                         |
| -------------------- | ---------------------------- | ------------------------------------------------------------ |
| annotation (default) | `org.example.SomeAnnotation` | 要在目标组件中的类级别出现的注解。                           |
| assignable           | `org.example.SomeClass`      | 目标组件可分配给（继承或实现）的类（或接口）。               |
| aspectj              | `org.example..*Service+`     | 要由目标组件匹配的AspectJ类型表达式。                        |
| regex                | `org\.example\.Default.*`    | 要由目标组件类名匹配的正则表达式。                           |
| custom               | `org.example.MyTypeFilter`   | `org.springframework.core.type .TypeFilter`接口的自定义实现。 |

以下示例显示忽略所有`@Repository` 注解并使用“stub”存储库的配置：

```java
@Configuration
@ComponentScan(basePackages = "org.example",
        includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
        excludeFilters = @Filter(Repository.class))
public class AppConfig {
    ...
}
```

以下清单显示了等效的XML：:

```java
<beans>
    <context:component-scan base-package="org.example">
        <context:include-filter type="regex"
                expression=".*Stub.*Repository"/>
        <context:exclude-filter type="annotation"
                expression="org.springframework.stereotype.Repository"/>
    </context:component-scan>
</beans>
```

你还可以通过在注解上设置`useDefaultFilters=false`或通过`use-default-filters="false"`作为<`<component-scan/>` 元素的属性来禁用默认过滤器。这样将不会自动检测带有`@Component`, `@Repository`,`@Service`, `@Controller`, 或 `@Configuration`.

<a id="beans-factorybeans-annotations"></a>

#### [](#beans-factorybeans-annotations)1.10.5.在组件中定义bean的元数据

Spring组件也可以向容器提供bean定义元数据，。在`@Configuration`注解的类中使用`@Bean`注解定义bean元数据(也就是Spring bean),以下示例显示了如何执行此操作：

```java
@Component
public class FactoryMethodComponent {

    @Bean
    @Qualifier("public")
    public TestBean publicInstance() {
        return new TestBean("publicInstance");
    }

    public void doWork() {
        // Component method implementation omitted
    }
}
```

这个类是一个Spring组件，它有个 `doWork()`方法。然而，它还有一个工厂方法 `publicInstance()`用于产生bean定义。`@Bean`注解了工厂方法， 还设置了其他bean定义的属性，例如通过`@Qualifier`注解的qualifier值。可以指定的其他方法级别的注解是 `@Scope`, `@Lazy`以及自定义的qualifier注解。

除了用于组件初始化的角色之外，`@Lazy`注解也可以在`@Autowired`或者code>@Inject注解上，在这种情况下，该注入将会变成延迟注入代理lazy-resolution proxy（也就是懒加载）。

自动注入的字段和方法也可以像前面讨论的一样被支持，也支持`@Bean`方法的自动注入。以下示例显示了如何执行此操作：:

```java
@Component
public class FactoryMethodComponent {

    private static int i;

    @Bean
    @Qualifier("public")
    public TestBean publicInstance() {
        return new TestBean("publicInstance");
    }

    // use of a custom qualifier and autowiring of method parameters
    @Bean
    protected TestBean protectedInstance(
            @Qualifier("public") TestBean spouse,
            @Value("#{privateInstance.age}") String country) {
        TestBean tb = new TestBean("protectedInstance", 1);
        tb.setSpouse(spouse);
        tb.setCountry(country);
        return tb;
    }

    @Bean
    private TestBean privateInstance() {
        return new TestBean("privateInstance", i++);
    }

    @Bean
    @RequestScope
    public TestBean requestScopedInstance() {
        return new TestBean("requestScopedInstance", 3);
    }
}
```

该示例将方法参数为`String`，名称为`country`的bean自动装配为另一个名为`privateInstance`的bean的`age`属性值。 Spring表达式语言元素通过记号`#{ <expression> }`来定义属性的值。对于 `@Value`注解，表达式解析器在解析表达式后，会查找bean的名字并设置value值。

从Spring4.3开始，您还可以声明一个类型为`InjectionPoint`的工厂方法参数（或其更具体的子类：`DependencyDescriptor`）以访问触发创建当前bean的请求注入点。 请注意，这仅适用于真实创建的bean实例，而不适用于注入现有实例。因此，这个特性对prototype scope的bean最有意义。对于其他作用域，工厂方法将只能看到触发在给定scope中创建新bean实例的注入点。 例如，触发创建一个延迟单例bean的依赖。在这种情况下，使用提供的注入点元数据拥有优雅的语义。 以下示例显示了如何使用`InjectionPoint`:

```java
@Component
public class FactoryMethodComponent {

    @Bean @Scope("prototype")
    public TestBean prototypeInstance(InjectionPoint injectionPoint) {
        return new TestBean("prototypeInstance for " + injectionPoint.getMember());
    }
}
```

在Spring组件中处理`@Bean`和在code>@Configuration中处理是不一样的，区别在于，在`@Component`中，不会使用CGLIB增强去拦截方法和属性的调用。在`@Configuration`注解的类中， `@Bean`注解创建的bean对象会使用CGLIB代理对方法和属性进行调用。方法的调用不是常规的Java语法，而是通过容器来提供通用的生命周期管理和代理Spring bean， 甚至在通过编程的方式调用`@Bean`方法时也会产生对其它bean的引用。相比之下，在一个简单的`@Component`类中调用`@Bean`方法中的方法或字段具有标准Java语义，这里没有用到特殊的CGLIB处理或其他约束。

开发者可以将`@Bean`方法声明为`static`的，并允许在不将其包含的配置类作为实例的情况下调用它们。这在定义后置处理器bean时是特别有意义的。 例如`BeanFactoryPostProcessor` 或`BeanPostProcessor`),，因为这类bean会在容器的生命周期前期被初始化，而不会触发其它部分的配置。

对静态`@Bean`方法的调用永远不会被容器拦截，即使在`@Configuration`类内部。这是用为CGLIB的子类代理限制了只会重写非静态方法。因此， 对另一个`@Bean`方法的直接调用只能使用标准的Java语法。也只能从工厂方法本身直接返回一个独立的实例。

由于Java语言的可见性，`@Bean`方法并不一定会对容器中的bean有效。开发者可能很随意的在非`@Configuration`类中定义或定义为静态方法。然而， 在`@Configuration`类中的正常`@Bean`方法都会被重写，因此，它们不应该定义为`private`或`final`。

`@Bean`方法也可以用在父类中，同样适用于Java 8接口中的默认方法。这使得组建复杂的配置时能具有更好的灵活性，甚至可能通过Java 8的默认方法实现多重继承。 这种特性在Spring 4.2开始支持。

最后，请注意，单个类可以为同一个bean保存多个`@Bean`方法，例如根据运行时可用的依赖关系选择合适的工厂方法。使用算法会选择 “最贪婪“的构造方法， 一些场景可能会按如下方法选择相应的工厂方法：满足最多依赖的会被选择，这与使用`@Autowired` 时选择多个构造方法时类似。

<a id="beans-scanning-name-generator"></a>

#### [](#beans-scanning-name-generator)1.10.6. 命名自动注册组件

扫描处理过程，其中一步就是自动探测组件，扫描器使用`BeanNameGenerator`对探测到的组件命名。默认情况下，各代码层注解(`@Component`, `@Repository`, `@Service`, 和 `@Controller`)所包含的name值，将会作为相应的bean定义的名字。

如果这些注解没有name值，或者是其他一些被探测到的组件（比如使用自定义过滤器探测到的)，默认会又bean name生成器生成，使用小写类名作为bean名字。 例如，如果检测到以下组件类，则名称为`myMovieLister`和`movieFinderImpl`:

```java
@Service("myMovieLister")
public class SimpleMovieLister {
    // ...
}

@Repository
public class MovieFinderImpl implements MovieFinder {
    // ...
}
```

如果您不想依赖默认的bean命名策略，则可以提供自定义bean命名策略。首先，实现 [`BeanNameGenerator`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/support/BeanNameGenerator.html)接口，并确保包括一个默认的无参构造函数。 然后，在配置扫描程序时提供完全限定的类名，如以下示例注解和bean定义所示：

    @Configuration
    @ComponentScan(basePackages = "org.example", nameGenerator = MyNameGenerator.class)
    public class AppConfig {
        ...
    }

    <beans>
        <context:component-scan base-package="org.example"
            name-generator="org.example.MyNameGenerator" />
    </beans>

作为一般规则，考虑在其他组件可能对其进行显式引用时使用注解指定名称。 另一方面，只要容器负责装配时，自动生成的名称就足够了。

<a id="beans-scanning-scope-resolver"></a>

#### [](#beans-scanning-scope-resolver)1.10.7.为自动检测组件提供范围

与一般的Spring管理组件一样，自动检测组件的默认和最常见的作用域是`singleton`。但是，有时您需要一个可由`@Scope`注解指定的不同作用域。 您可以在注解中提供作用域的名称，如以下示例所示：

```java
@Scope("prototype")
@Repository
public class MovieFinderImpl implements MovieFinder {
    // ...
}
```

`@Scope`注解仅在具体bean类（用于带注解的组件）或工厂方法（用于`@Bean`方法）上进行关联。 与XML bean定义相比，没有bean继承的概念，并且 类级别的继承结构与元数据无关。

有关特定于Web的范围（如Spring上下文中的“request” or “session”）的详细信息，请参阅[请求，会话，应用程序和WebSocket作用域](#beans-factory-scopes-other)。 这些作用域与构建注解一样，您也可以使用Spring的元注解方法编写自己的作用域注解：例如，使用`@Scope("prototype")`进行元注解的自定义注解，可能还会声明自定义作用域代理模式。

想要提供自定义作用域的解析策略，而不是依赖于基于注解的方法，那么需要实现[`ScopeMetadataResolver`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/annotation/ScopeMetadataResolver.html)接口，并确保包含一个默认的无参数构造函数。 然后，在配置扫描程序时提供完全限定类名。以下注解和bean定义示例显示：

```java
@Configuration
@ComponentScan(basePackages = "org.example", scopeResolver = MyScopeResolver.class)
public class AppConfig {
    ...
}

<beans>
    <context:component-scan base-package="org.example" scope-resolver="org.example.MyScopeResolver"/>
</beans>
```

当使用某个非单例作用域时，为作用域对象生成代理可能非常必要，原因参看 [作为依赖关系的作用域bean](#beans-factory-scopes-other-injection)。 为此，组件扫描元素上提供了scoped-proxy属性。 三个可能的值是：`no`, `interfaces`, 和 `targetClass`。 例如，以下配置导致标准JDK动态代理：

```java
@Configuration
@ComponentScan(basePackages = "org.example", scopedProxy = ScopedProxyMode.INTERFACES)
public class AppConfig {
    ...
}

<beans>
    <context:component-scan base-package="org.example" scoped-proxy="interfaces"/>
</beans>
```

<a id="beans-scanning-qualifiers"></a>

#### [](#beans-scanning-qualifiers)1.10.8. 为注解提供Qualifier元数据

在前面[使用qualifiers微调基于注解自动装配](#beans-autowired-annotation-qualifiers)讨论过`@Qualifier` 注解。该部分中的示例演示了在解析自动注入候选者时使用 `@Qualifier`注解和自定义限定符注解以提供细粒度控制。 因为这些示例基于XML bean定义，所以使用XML中的`bean`元素的 `qualifier` 或 `meta`子元素在候选bean定义上提供了限定符元数据。当依靠类路径扫描并自动检测组件时， 可以在候选类上提供具有类型级别注解的限定符元数据。以下三个示例演示了此技术：

```java
@Component
@Qualifier("Action")
public class ActionMovieCatalog implements MovieCatalog {
    // ...
}

@Component
@Genre("Action")
public class ActionMovieCatalog implements MovieCatalog {
    // ...
}

@Component
@Offline
public class CachingMovieCatalog implements MovieCatalog {
    // ...
}
```

与大多数基于注解的替代方法一样，注解元数据绑定到类定义本身，而使用在XML配置时，允许同一类型的beans在qualifier元数据中提供变量， 因为元数据是依据实例而不是类来提供的。

<a id="beans-scanning-index"></a>

#### [](#beans-scanning-index)1.10.9. 生成候选组件的索引

虽然类路径扫描非常快，但通过在编译时创建候选的静态列表。可以提高大型应用程序的启动性能。在此模式下，应用程序的所有模块都必须使用此机制， 当 `ApplicationContext`检测到此类索引时，它将自动使用它，而不是扫描类路径。

若要生成索引， 只需向包含组件扫描指令目标组件的每个模块添加一个附加依赖项。以下示例显示了如何使用Maven执行此操作：

```java
<dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context-indexer</artifactId>
        <version>5.1.3.BUILD-SNAPSHOT</version>
        <optional>true</optional>
    </dependency>
</dependencies>
```

以下示例显示了如何使用Gradle执行此操作:

```groovy
dependencies {
    compileOnly("org.springframework:spring-context-indexer:5.1.3.BUILD-SNAPSHOT")
}
```

这个过程将产生一个名为`META-INF/spring.components`的文件，并将包含在jar包中。

在IDE中使用此模式时，必须将`spring-context-indexer`注册为注解处理器， 以确保更新候选组件时索引是最新的。

如果在类路径中找到 `META-INF/spring.components` 时，将自动启用索引。如果某个索引对于某些库(或用例)是不可用的， 但不能为整个应用程序构建，则可以将`spring.index.ignore`设置为`true`，从而将其回退到常规类路径的排列(即根本不存在索引)， 或者作为系统属性或在`spring.properties`文件位于类路径的根目录中。

<a id="beans-standard-annotations"></a>

### [](#beans-standard-annotations)1.11. 使用JSR 330标准注解

从Spring 3.0开始，Spring提供对JSR-330标准注解（依赖注入）的支持。 这些注解的扫描方式与Spring注解相同。 要使用它们，您需要在类路径中包含相关的jar。

如果使用Maven工具，那么`@javax.inject.Inject`可以在Maven中央仓库中找到( [http://repo1.maven.org/maven2/javax/inject/javax.inject/1/](https://repo1.maven.org/maven2/javax/inject/javax.inject/1/)). 您可以将以下依赖项添加到文件pom.xml：:

```xml
<dependency>
    <groupId>javax.inject</groupId>
    <artifactId>javax.inject</artifactId>
    <version>1</version>
</dependency>
```

<a id="beans-inject-named"></a>

#### [](#beans-inject-named)1.11.1. 使用`@Inject` 和 `@Named`注解实现依赖注入

`@javax.inject.Inject`可以使用以下的方式来替代`@Autowired`注解:

```java
import javax.inject.Inject;

public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    public void listMovies() {
        this.movieFinder.findMovies(...);
        ...
    }
}
```

与 `@Autowired`一样，您可以在字段，方法和构造函数参数级别使用`@Inject`注解。此外，还可以将注入点声明为`Provider`。 它允许按需访问作用域较小的bean或通过`Provider.get()`调用对其他bean进行延迟访问。以下示例提供了前面示例的变体：

```java
import javax.inject.Inject;
import javax.inject.Provider;

public class SimpleMovieLister {

    private Provider<MovieFinder> movieFinder;

    @Inject
    public void setMovieFinder(Provider<MovieFinder> movieFinder) {
        this.movieFinder = movieFinder;
    }

    public void listMovies() {
        this.movieFinder.get().findMovies(...);
        ...
    }
}
```

如果想要为注入的依赖项使用限定名称，则应该使用`@Named`注解。如下所示：

```java
import javax.inject.Inject;
import javax.inject.Named;

public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(@Named("main") MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

与`@Autowired`一样，`@Inject` 也可以与`java.util.Optional`或`@Nullable`一起使用。 这在这里用更适用，因为`@Inject`没有`required`的属性。 以下一对示例显示了如何使用`@Inject`和`@Nullable`:

```java
public class SimpleMovieLister {

    @Inject
    public void setMovieFinder(Optional<MovieFinder> movieFinder) {
        ...
    }
}

public class SimpleMovieLister {

    @Inject
    public void setMovieFinder(@Nullable MovieFinder movieFinder) {
        ...
    }
}
```

<a id="beans-named"></a>

#### [](#beans-named)1.11.2. `@Named` 和 `@ManagedBean`注解: 标准与 `@Component` 注解相同

`@javax.inject.Named` 或 `javax.annotation.ManagedBean`可以使用下面的方式来替代`@Component`注解：

```java
import javax.inject.Inject;
import javax.inject.Named;

@Named("movieListener")  // @ManagedBean("movieListener") could be used as well
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

在不指定组件名称的情况下使用`@Component`是很常见的。 `@Named`可以以类似的方式使用，如下例所示：

```java
import javax.inject.Inject;
import javax.inject.Named;

@Named
public class SimpleMovieLister {

    private MovieFinder movieFinder;

    @Inject
    public void setMovieFinder(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }

    // ...
}
```

当使用`@Named` 或 `@ManagedBean`时，可以与Spring注解完全相同的方式使用component-scanning组件扫描。 如以下示例所示:

```java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
    ...
}
```

与`@Component`相反，JSR-330 `@Named` 和 JSR-250 `ManagedBean`注解不可组合。 请使用Spring的原型模型（stereotype mode)来构建自定义组件注解。

<a id="beans-standard-annotations-limitations"></a>

#### [](#beans-standard-annotations-limitations)1.11.3. 使用 JSR-330标准注解的限制

使用标准注解时，需要知道哪些重要功能是不可用的。如下表所示：

Table 6. Spring的组件模型元素 vs JSR-330 变量

| Spring              | javax.inject.*        | javax.inject restrictions / comments                         |
| ------------------- | --------------------- | ------------------------------------------------------------ |
| @Autowired          | @Inject               | `@Inject` 没有'required'属性。 可以与Java 8的 `Optional`一起使用。 |
| @Component          | @Named / @ManagedBean | JSR-330不提供可组合模型，只是一种识别命名组件的方法。        |
| @Scope("singleton") | @Singleton            | JSR-330的默认作用域就像Spring的`prototype`。 但是，为了使其与Spring的一般默认值保持一致，默认情况下，Spring容器中声明的JSR-330 bean是一个 `singleton`。 为了使用除 `singleton`之外的范围，您应该使用Spring的`@Scope`注解。 `javax.inject`还提供了[@Scope](https://download.oracle.com/javaee/6/api/javax/inject/Scope.html)注解。 然而，这个仅用于创建自己的注解。 |
| @Qualifier          | @Qualifier / @Named   | `javax.inject.Qualifier` 只是用于构建自定义限定符的元注解。 可以通过`javax.inject.Named`创建与Spring中`@Qualifier`一样的限定符。 |
| @Value              | -                     | 无                                                           |
| @Required           | -                     | 无                                                           |
| @Lazy               | -                     | 无                                                           |
| ObjectFactory       | Provider              | `javax.inject.Provider` avax.inject.Provider是Spring的`ObjectFactory`的直接替代品， 仅仅使用简短的`get()`方法即可。 它也可以与Spring的`@Autowired`结合使用，也可以与非注解的构造函数和setter方法结合使用。 |

<a id="beans-java"></a>

### [](#beans-java)1.12. 基于Java的容器配置

本节介绍如何在Java代码中使用注解来配置Spring容器。 它包括以下主题：:

*   [基本概念: `@Bean` 和 `@Configuration`](#beans-java-basic-concepts)

*   [使用`AnnotationConfigApplicationContext`实例化Spring容器](#beans-java-instantiating-container)

*   [使用`@Bean`注解](#beans-java-bean-annotation)

*   [使用`@Configuration`注解](#beans-java-configuration-annotation)

*   [编写基于Java的配置](#beans-java-composing-configuration-classes)

*   [定义Bean配置文件](#beans-definition-profiles)

*   [`PropertySource` 抽象](#beans-property-source-abstraction)

*   [使用 `@PropertySource`](#beans-using-propertysource)

*   [声明中的占位符](#beans-placeholder-resolution-in-statements)

<a id="beans-java-basic-concepts"></a>

#### [](#beans-java-basic-concepts)1.12.1. 基本概念: `@Bean` 和 `@Configuration`

Spring新的基于Java配置的核心内容是`@Configuration`注解的类和`@Bean`注解的方法。

`@Bean`注解用于表明方法的实例化，、配置和初始化都是由Spring IoC容器管理的新对象，对于那些熟悉Spring的`<beans/>`XML配置的人来说， `@Bean`注解扮演的角色与`<beans/>`元素相同。开发者可以在任意的Spring `@Component`中使用`@Bean`注解方法 ，但大多数情况下，`@Bean`是配合`@Configuration`使用的。

使用`@Configuration`注解类时，这个类的目的就是作为bean定义的地方。此外，`@Configuration`类允许通过调用同一个类中的其他`@Bean`方法来定义bean间依赖关系。 最简单的`@Configuration`类如下所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public MyService myService() {
        return new MyServiceImpl();
    }
}
```

前面的`AppConfig`类等效于以下Spring `<beans/>`XML：

```xml
<beans>
    <bean id="myService" class="com.acme.services.MyServiceImpl"/>
</beans>
```

完整的@Configuration模式对比“lite”模式的@Bean?

当`@Bean`方法在没有用 `@Configuration`注解的类中声明时，它们将会被称为“lite”的模式处理。例如，`@Component`中声明的bean方法或者一个普通的旧类中的bean方法将被视为 “lite”的。包含类的主要目的不同，而`@Bean`方法在这里是一种额外的好处。。例如，服务组件可以通过在每个适用的组件类上使用额外的 `@Bean`方法将管理视图公开给容器。 在这种情况下，`@Bean`方法是一种通用的工厂方法机制。

与完整的 `@Configuration`不同，lite的`@Bean`方法不能声明bean之间的依赖关系。 相反，它们对其包含组件的内部状态进行操作，并且可以有选择的对它们可能声明的参数进行操作。因此，这样的`@Bean`注解的方法不应该调用其他`@Bean`注解的方法。 每个这样的方法实际上只是特定bean引用的工厂方法，没有任何特殊的运行时语义。不经过CGLIB处理，所以在类设计方面没有限制（即，包含类可能是最终的）。

在常见的场景中，`@Bean`方法将在`@Configuration`类中声明，确保始终使用“full”模式，这将防止相同的`@Bean`方法被意外地多次调用，这有助于减少在 “lite”模式下操作时难以跟踪的细微错误。

`@Bean`和`@Configuration`注解将在下面的章节深入讨论，首先，我们将介绍使用基于Java代码的配置来创建Spring容器的各种方法。

<a id="beans-java-instantiating-container"></a>

#### [](#beans-java-instantiating-container)1.12.2. 使用`AnnotationConfigApplicationContext`初始化Spring容器

以下部分介绍了Spring的`AnnotationConfigApplicationContext`，它是在Spring 3.0中引入的。这是一个强大的(versatile)`ApplicationContext` 实现,它不仅能解析`@Configuration`注解类 ,也能解析 `@Component`注解的类和使用JSR-330注解的类.

当使用`@Configuration`类作为输入时,`@Configuration`类本身被注册为一个bean定义,类中所有声明的`@Bean`方法也被注册为bean定义.

当提供 `@Component`和JSR-330类时，它们被注册为bean定义，并且假定在必要时在这些类中使用DI元数据，例如`@Autowired` 或 `@Inject`。

<a id="beans-java-instantiating-container-contstructor"></a>

##### [](#beans-java-instantiating-container-contstructor)简单结构

与实例化`ClassPathXmlApplicationContext`时Spring XML文件用作输入的方式大致相同， 在实例化`AnnotationConfigApplicationContext`时可以使用`@Configuration` 类作为输入。 这允许完全无XML使用Spring容器，如以下示例所示：

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

如前所述，`AnnotationConfigApplicationContext`不仅限于使用`@Configuration`类。 任何`@Component`或JSR-330带注解的类都可以作为输入提供给构造函数，如以下示例所示：

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(MyServiceImpl.class, Dependency1.class, Dependency2.class);
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

上面假设`MyServiceImpl`, `Dependency1`, 和 `Dependency2`使用Spring依赖注入注解，例如`@Autowired`。

<a id="beans-java-instantiating-container-register"></a>

##### [](#beans-java-instantiating-container-register)使用`register(Class<?>…)`编程构建容器

`AnnotationConfigApplicationContext`可以通过无参构造函数实例化，然后调用`register()` 方法进行配置。 这种方法在以编程的方式构建 `AnnotationConfigApplicationContext`时特别有用。下列示例显示了如何执行此操作

```java
public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.register(AppConfig.class, OtherConfig.class);
    ctx.register(AdditionalConfig.class);
    ctx.refresh();
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

<a id="beans-java-instantiating-container-scan"></a>

##### [](#beans-java-instantiating-container-scan)3 使用`scan(String…)`扫描组件

要启用组件扫描，可以按如下方式注解`@Configuration`类:

```java
@Configuration
@ComponentScan(basePackages = "com.acme") (1)
public class AppConfig  {
    ...
}
```

**1**、此注解可启用组件扫描。

有经验的用户可能更熟悉使用XML的等价配置形式，如下例所示：

    <beans>
        <context:component-scan base-package="com.acme"/>
    </beans>

上面的例子中，`com.acme`包会被扫描，只要是使用了`@Component`注解的类，都会被注册进容器中。同样地，`AnnotationConfigApplicationContext`公开的`scan(String…)` 方法也允许扫描类完成同样的功能 如以下示例所示：

```java
public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.scan("com.acme");
    ctx.refresh();
    MyService myService = ctx.getBean(MyService.class);
}
```

请记住`@Configuration`类是使用`@Component`进行[元注解](#beans-meta-annotations)的，因此它们是组件扫描的候选者。 在前面的示例中， 假设AppConfig在com.acme包（或下面的任何包）中声明，它在`scan()`调用期间被拾取。 在`refresh()`之后，它的所有`@Bean`方法都被处理并在容器中注册为bean定义。

<a id="beans-java-instantiating-container-web"></a>

##### [](#beans-java-instantiating-container-web)使用`AnnotationConfigWebApplicationContext`支持Web应用程序

`WebApplicationContext`与`AnnotationConfigApplicationContext`的变种是 `AnnotationConfigWebApplicationContext`配置。这个实现可以用于配置Spring `ContextLoaderListener` servlet监听器 ，Spring MVC的 `DispatcherServlet`等等。以下web.xml代码段配置典型的Spring MVC Web应用程序（请注意`contextClass` context-param和init-param的使用）：

```xml
<web-app>
    <!-- Configure ContextLoaderListener to use AnnotationConfigWebApplicationContext
        instead of the default XmlWebApplicationContext -->
    <context-param>
        <param-name>contextClass</param-name>
        <param-value>
            org.springframework.web.context.support.AnnotationConfigWebApplicationContext
        </param-value>
    </context-param>

    <!-- Configuration locations must consist of one or more comma- or space-delimited
        fully-qualified @Configuration classes. Fully-qualified packages may also be
        specified for component-scanning -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>com.acme.AppConfig</param-value>
    </context-param>

    <!-- Bootstrap the root application context as usual using ContextLoaderListener -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!-- Declare a Spring MVC DispatcherServlet as usual -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <!-- Configure DispatcherServlet to use AnnotationConfigWebApplicationContext
            instead of the default XmlWebApplicationContext -->
        <init-param>
            <param-name>contextClass</param-name>
            <param-value>
                org.springframework.web.context.support.AnnotationConfigWebApplicationContext
            </param-value>
        </init-param>
        <!-- Again, config locations must consist of one or more comma- or space-delimited
            and fully-qualified @Configuration classes -->
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>com.acme.web.MvcConfig</param-value>
        </init-param>
    </servlet>

    <!-- map all requests for /app/* to the dispatcher servlet -->
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>
</web-app>
```

<a id="beans-java-bean-annotation"></a>

#### [](#beans-java-bean-annotation)1.12.3. 使用`@Bean` 注解

`@Bean` @Bean是一个方法级别的注解，它与XML中的 `<bean/>`元素类似。注解支持 `<bean/>`提供的一些属性，例如 \* [init-method](#beans-factory-lifecycle-initializingbean) \* [destroy-method](#beans-factory-lifecycle-disposablebean) \* [autowiring](#beans-factory-autowire) \* `name`

开发者可以在`@Configuration`类或`@Component`类中使用`@Bean`注解。

<a id="beans-java-declaring-a-bean"></a>

##### [](#beans-java-declaring-a-bean)声明一个Bean

要声明一个bean，只需使用`@Bean`注解方法即可。使用此方法，将会在`ApplicationContext`内注册一个bean，bean的类型是方法的返回值类型。默认情况下， bean名称将与方法名称相同。以下示例显示了`@Bean`方法声明：

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferServiceImpl transferService() {
        return new TransferServiceImpl();
    }
}
```

前面的配置完全等同于以下Spring XML:

```xml
<beans>
    <bean id="transferService" class="com.acme.TransferServiceImpl"/>
</beans>
```

这两个声明都在`ApplicationContext`中创建一个名为`transferService`的bean，并且绑定了`TransferServiceImpl`的实例。如下图所示：

transferService -> com.acme.TransferServiceImpl

您还可以使用接口（或基类）返回类型声明`@Bean`方法，如以下示例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl();
    }
}
```

但是，这会将预先类型预测的可见性限制为指定的接口类型(`TransferService`),然后在实例化受影响的单一bean时,只知道容器的完整类型(`TransferServiceImpl`）。 。非延迟的单例bean根据它们的声明顺序进行实例化，因此开发者可能会看到不同类型的匹配结果，这具体取决于另一个组件尝试按未类型匹配的时间(如`@Autowired TransferServiceImpl`， 一旦`transferService` bean已被实例化,这个问题就被解决了).

如果通过声明的服务接口都是引用类型,那么`@Bean` 返回类型可以安全地加入该设计决策.但是,对于实现多个接口的组件或可能由其实现类型引用的组件, 更安全的方法是声明可能的最具体的返回类型(至少按照注入点所要求的特定你的bean）。

<a id="beans-java-dependencies"></a>

##### [](#beans-java-dependencies)Bean之间的依赖

一个使用`@Bean`注解的方法可以具有任意数量的参数描述构建该bean所需的依赖，例如，如果我们的`TransferService`需要`AccountRepository`， 我们可以使用方法参数来实现该依赖关系，如以下示例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferService transferService(AccountRepository accountRepository) {
        return new TransferServiceImpl(accountRepository);
    }
}
```

这个解析机制与基于构造函数的依赖注入非常相似。有关详细信息，请参阅[相关部分](#beans-constructor-injection)。

<a id="beans-java-lifecycle-callbacks"></a>

##### [](#beans-java-lifecycle-callbacks)接收生命周期回调

使用`@Bean`注解定义的任何类都支持常规的生命周期回调，并且可以使用JSR-的`@PostConstruct`和`@PreDestroy`注解。 有关更多详细信息，请参阅 [JSR-250](#beans-postconstruct-and-predestroy-annotations) 注解。

完全支持常规的Spring[生命周期](#beans-factory-nature)回调。 如果bean实现`InitializingBean`, `DisposableBean`, 或 `Lifecycle`，则它们各自的方法由容器调用。

同样地，还完全支持标准的`*Aware`，如[BeanFactoryAware](#beans-beanfactory), [BeanNameAware](#beans-factory-aware), [MessageSourceAware](#context-functionality-messagesource), [ApplicationContextAware](#beans-factory-aware)。

`@Bean`注解支持指定任意初始化和销毁回调方法，就像`bean`元素上的Spring XML的 `init-method`和`destroy-method` 属性一样，如下例所示：

```java
public class BeanOne {

    public void init() {
        // initialization logic
    }
}

public class BeanTwo {

    public void cleanup() {
        // destruction logic
    }
}

@Configuration
public class AppConfig {

    @Bean(initMethod = "init")
    public BeanOne beanOne() {
        return new BeanOne();
    }

    @Bean(destroyMethod = "cleanup")
    public BeanTwo beanTwo() {
        return new BeanTwo();
    }
}
```

默认情况下，使用Java Config定义的bean中`close`方法或者`shutdown`方法，会作为销毁回调而自动调用。若bean中有`close` 或 `shutdown` 方法，并且您不希望在容器关闭时调用它，则可以将`@Bean(destroyMethod="")` 添加到bean定义中以禁用默认`(inferred)` 模式。

开发者可能希望对通过JNDI获取的资源执行此操作，因为它的生命周期是在应用程序外部管理的。更进一步，使用 `DataSource`时一定要关闭它，不关闭将会出问题。

以下示例说明如何防止`DataSource`的自动销毁回调：

```java
@Bean(destroyMethod="")
public DataSource dataSource() throws NamingException {
    return (DataSource) jndiTemplate.lookup("MyDS");
}
```

同样地，使用`@Bean`方法，通常会选择使用程序化的JNDI查找：使用Spring的`JndiTemplate` / `JndiLocatorDelegate`帮助类或直接使用JNDI的`InitialContext` ，但是不要使用`JndiObjectFactoryBean`的变体，因为它会强制开发者声明一个返回类型作为`FactoryBean`的类型用于代替实际的目标类型，这会使得交叉引用变得很困难。

对于前面注解中上面示例中的`BeanOne`，在构造期间直接调用`init()`方法同样有效，如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public BeanOne beanOne() {
        BeanOne beanOne = new BeanOne();
        beanOne.init();
        return beanOne;
    }

    // ...
}
```

当您直接使用Java（new对象那种）工作时，您可以使用对象执行任何您喜欢的操作，并且不必总是依赖于容器生命周期。

<a id="beans-java-specifying-bean-scope"></a>

##### [](#beans-java-specifying-bean-scope)指定Bean范围

Spring包含`@Scope`注解，以便您可以指定bean的范围。

<a id="beans-java-available-scopes"></a>

###### [](#beans-java-available-scopes)使用 `@Scope` 注解

可以使用任意标准的方式为 `@Bean`注解的bean指定一个作用域，你可以使用[Bean Scopes](#beans-factory-scopes)中的任意标准作用域

默认范围是`singleton`的，但是可以使用 `@Scope`注解来覆盖。如下例所示：

```java
@Configuration
public class MyConfiguration {

    @Bean
    @Scope("prototype")
    public Encryptor encryptor() {
        // ...
    }
}
```

<a id="beans-java-scoped-proxy"></a>

###### [](#beans-java-scoped-proxy)`@Scope` and `scoped-proxy`

Spring提供了一种通过[scoped proxies](#beans-factory-scopes-other-injection)处理作用域依赖项的便捷方法。使用XML配置时创建此类代理的最简单方法是`<aop:scoped-proxy/>`元素。 使用`@Scope`注解在Java中配置bean提供了与`proxyMode`属性的等效支持。 默认值为无代理（`ScopedProxyMode.NO`），但您可以指定`ScopedProxyMode.TARGET_CLASS` 或 `ScopedProxyMode.INTERFACES`。

如果使用Java将XML参考文档（请参阅[scoped proxies](#beans-factory-scopes-other-injection)）的作用域代理示例移植到我们的 `@Bean`，它类似于以下内容：

```java
// an HTTP Session-scoped bean exposed as a proxy
@Bean
@SessionScope
public UserPreferences userPreferences() {
    return new UserPreferences();
}

@Bean
public Service userService() {
    UserService service = new SimpleUserService();
    // a reference to the proxied userPreferences bean
    service.setUserPreferences(userPreferences());
    return service;
}
```

<a id="beans-java-customizing-bean-naming"></a>

##### [](#beans-java-customizing-bean-naming)自定义Bean命名

默认情况下，配置类使用`@Bean`方法的名称作为结果bean的名称。 但是，可以使用`name`属性覆盖此功能，如以下示例所示：

```java
@Configuration
public class AppConfig {

    @Bean(name = "myThing")
    public Thing thing() {
        return new Thing();
    }
}
```

<a id="beans-java-bean-aliasing"></a>

##### [](#beans-java-bean-aliasing)bean别名

正如[Bean的 命名](#beans-beanname)中所讨论的，有时需要为单个bean提供多个名称，也称为bean别名。 `@Bean`注解的 `name`属性为此接受String数组。 以下示例显示如何为bean设置多个别名：

```java
@Configuration
public class AppConfig {

    @Bean(name = { "dataSource", "subsystemA-dataSource", "subsystemB-dataSource" })
    public DataSource dataSource() {
        // instantiate, configure and return DataSource bean...
    }
}
```

<a id="beans-java-bean-description"></a>

##### [](#beans-java-bean-description)Bean 的描述

有时，提供更详细的bean文本描述会很有帮助。 当bean被暴露（可能通过JMX）用于监视目的时，这可能特别有用。

要向@Bean添加描述，可以使用[`@Description`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/annotation/Description.html)注解，如以下示例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    @Description("Provides a basic example of a bean")
    public Thing thing() {
        return new Thing();
    }
}
```

<a id="beans-java-configuration-annotation"></a>

#### [](#beans-java-configuration-annotation)1.12.4. 使用 `@Configuration` 注解

`@Configuration`是一个类级别的注解,表明该类将作为bean定义的元数据配置. `@Configuration`类会将有`@Bean`注解的公开方法声明为bean, .在 `@Configuration`类上调用`@Bean`方法也可以用于定义bean间依赖关系, 有关一般介绍，请参阅 [基本概念: `@Bean` 和 `@Configuration`](#beans-java-basic-concepts)

<a id="beans-java-injecting-dependencies"></a>

##### [](#beans-java-injecting-dependencies)注入内部bean依赖

当Bean彼此有依赖关系时,表示依赖关系就像调用另一个bean方法一样简单.如下例所示：

```java
@Configuration
public class AppConfig {

    @Bean
    public BeanOne beanOne() {
        return new BeanOne(beanTwo());
    }

    @Bean
    public BeanTwo beanTwo() {
        return new BeanTwo();
    }
}
```

在前面的示例中，`beanOne`通过构造函数注入接收对`beanTwo`的引用。

这种声明bean间依赖关系的方法只有在 `@Configuration` 类中声明`@Bean`方法时才有效。 您不能使用普通的`@Component`类声明bean间依赖关系。

<a id="beans-java-method-injection"></a>

##### [](#beans-java-method-injection)查找方法注入

如前所述，[查找方法注入](#beans-factory-method-injection)是一项很少使用的高级功能。 在单例范围的bean依赖于原型范围的bean的情况下，它很有用。 Java提供了很友好的API来实现这种模式。以下示例显示了如何使用查找方法注入：

```java
public abstract class CommandManager {
    public Object process(Object commandState) {
        // grab a new instance of the appropriate Command interface
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    // okay... but where is the implementation of this method?
    protected abstract Command createCommand();
}
```

通过使用Java配置，您可以创建 `CommandManager`的子类，其中抽象的 `createCommand()` 方法被覆盖，以便查找新的（原型）对象。 以下示例显示了如何执行此操作：

```java
@Bean
@Scope("prototype")
public AsyncCommand asyncCommand() {
    AsyncCommand command = new AsyncCommand();
    // inject dependencies here as required
    return command;
}

@Bean
public CommandManager commandManager() {
    // return new anonymous implementation of CommandManager with command() overridden
    // to return a new prototype Command object
    return new CommandManager() {
        protected Command createCommand() {
            return asyncCommand();
        }
    }
}
```

<a id="beans-java-further-information-java-config"></a>

##### [](#beans-java-further-information-java-config)有关基于Java的配置如何在内部工作的更多信息

请考虑以下示例，该示例显示了被调用两次的`@Bean`注解方法:

```java
@Configuration
public class AppConfig {

    @Bean
    public ClientService clientService1() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientService clientService2() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientDao clientDao() {
        return new ClientDaoImpl();
    }
}
```

`clientDao()`在`clientService1()`中调用一次，在`clientService2()`中调用一次。由于此方法创建了`ClientDaoImpl`的新实例并将其返回，因此通常希望有两个实例（每个服务一个）。 这肯定会有问题：在Spring中，实例化的bean默认具有`singleton`范围。这就是它的神奇之处:所有`@Configuration`类在启动时都使用 `CGLIB`进行子类化。 在子类中，子方法在调用父方法并创建新实例之前，首先检查容器是否有任何缓存（作用域）bean。

这种行为可以根据bean的作用域而变化,我们这里只是讨论单例.

从Spring 3.2开始，不再需要将CGLIB添加到类路径中，因为CGLIB类已经在`org.springframework.cglib`下重新打包并直接包含在spring-core JAR中。

由于CGLIB在启动时动态添加功能，因此存在一些限制。 特别是，配置类不能是 final的。 但是，从4.3开始，配置类允许使用任何构造函数，包括使用`@Autowired`或单个非默认构造函数声明进行默认注入。

如果想避免因CGLIB带来的限制,请考虑声明非`@Configuration`类的`@Bean`方法，例如在纯的`@Component`类 .这样在`@Bean`方法之间的交叉方法调用将不会被拦截,此时必须在构造函数或方法级别上进行依赖注入。

<a id="beans-java-composing-configuration-classes"></a>

#### [](#beans-java-composing-configuration-classes)1.12.5. 编写基于Java的配置

Spring的基于Java的配置功能允许您撰写注解，这可以降低配置的复杂性。

<a id="beans-java-using-import"></a>

##### [](#beans-java-using-import)使用`@Import` 注解

就像在Spring XML文件中使用`<import/>`元素来帮助模块化配置一样，`@Import` 注解允许从另一个配置类加载`@Bean`定义，如下例所示：

```java
@Configuration
public class ConfigA {

    @Bean
    public A a() {
        return new A();
    }
}

@Configuration
@Import(ConfigA.class)
public class ConfigB {

    @Bean
    public B b() {
        return new B();
    }
}
```

现在，在实例化上下文时，不需要同时指定`ConfigA.class`和 `ConfigB.class`，只需要显式提供`ConfigB`，如下例所示：

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(ConfigB.class);

    // now both beans A and B will be available...
    A a = ctx.getBean(A.class);
    B b = ctx.getBean(B.class);
}
```

这种方法简化了容器实例化，因为只需要处理一个类，而不是要求您在构造期间记住可能大量的`@Configuration`类。

从Spring Framework 4.2开始，`@Import`还支持引用常规组件类，类似于`AnnotationConfigApplicationContext.register`方法。 如果要避免组件扫描，这一点特别有用，可以使用一些配置类作为明确定义所有组件的入口点。

<a id="beans-java-injecting-imported-beans"></a>

###### [](#beans-java-injecting-imported-beans)在导入的`@Bean`定义上注入依赖项

上面的例子可以运行,,但是太简单了。在大多数实际情况下，bean将在配置类之间相互依赖.在使用XML时,这本身不是问题,因为没有涉及到编译器. 可以简单地声明 `ref="someBean"`,并且相信Spring将在容器初始化期间可以很好地处理它。当然，当使用`@Configuration`类时，Java编译器会有一些限制 ，即需符合Java的语法。

幸运的是，解决这个问题很简单。正如我们[已经讨论过](#beans-java-dependencies)的，`@Bean`方法可以有任意数量的参数来描述bean的依赖关系。 考虑以下更多真实场景，其中包含几个 `@Configuration`类，每个类都取决于其他类中声明的bean：

```java
@Configuration
public class ServiceConfig {

    @Bean
    public TransferService transferService(AccountRepository accountRepository) {
        return new TransferServiceImpl(accountRepository);
    }
}

@Configuration
public class RepositoryConfig {

    @Bean
    public AccountRepository accountRepository(DataSource dataSource) {
        return new JdbcAccountRepository(dataSource);
    }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return new DataSource
    }
}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    // everything wires up across configuration classes...
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

还有另一种方法可以达到相同的效果。请记住，`@Configuration`类最终只是容器中的另一个bean： 这意味着它们可以利用`@Autowired` 和 `@Value` 注入以及与任何其他bean相同的其他功能。

确保以这种方式注入的依赖项只是最简单的。`@Configuration`类在上下文初始化期间很早就被处理，并且强制以这种方式注入依赖项可能会导致意外的早期初始化。 尽可能采用基于参数的注入，如前面的示例所示。

另外，要特别注意通过`@Bean`的`BeanPostProcessor` 和 `BeanFactoryPostProcessor`定义。 这些通常应该声明为静态`@Bean`方法，而不是触发其包含配置类的实例化。否则，`@Autowired` 和 `@Value`不能在配置类本身上工作，因为它过早地被创建为bean实例。

以下示例显示了如何将一个bean自动连接到另一个bean:

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private AccountRepository accountRepository;

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl(accountRepository);
    }
}

@Configuration
public class RepositoryConfig {

    private final DataSource dataSource;

    @Autowired
    public RepositoryConfig(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(dataSource);
    }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return new DataSource
    }
}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    // everything wires up across configuration classes...
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

仅在Spring Framework 4.3中支持`@Configuration`类中的构造函数注入。 另请注意，如果目标bean仅定义了一个构造函数，则无需指定`@Autowired`。 在前面的示例中，`RepositoryConfig`构造函数中不需要`@Autowired`。

完全导入bean便于查找

在上面的场景中,`@Autowired`可以很好的工作,使设计更具模块化,但是自动注入哪个bean依然有些模糊不清.例如, 作为一个开发者查看`ServiceConfig`类时,你怎么知道`@Autowired AccountRepository`在哪定义的呢?代码中并未明确指出, 还好, [Spring Tool Suite](https://spring.io/tools/sts)提供的工具可以呈现图表，显示所有内容的连线方式，这可能就是您所需要的。 此外，您的Java IDE可以轻松找到`AccountRepository`类型的所有声明和用法，并快速显示返回该类型的`@Bean`方法的位置。

万一需求不允许这种模糊的装配,并且您希望从IDE中从一个`@Configuration`类直接导航到另一个`@Configuration`类，请考虑自动装配配置类本身。 以下示例显示了如何执行此操作：

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private RepositoryConfig repositoryConfig;

    @Bean
    public TransferService transferService() {
        // navigate 'through' the config class to the @Bean method!
        return new TransferServiceImpl(repositoryConfig.accountRepository());
    }
}
```

在前面的情况中，定义`AccountRepository` 是完全明确的。但是，`ServiceConfig`现在与`RepositoryConfig`紧密耦合。这是一种权衡的方法。 通过使用基于接口的或基于类的抽象`@Configuration` 类，可以在某种程度上减轻这种紧密耦合。请考虑以下示例：

```java
@Configuration
public class ServiceConfig {

    @Autowired
    private RepositoryConfig repositoryConfig;

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl(repositoryConfig.accountRepository());
    }
}

@Configuration
public interface RepositoryConfig {

    @Bean
    AccountRepository accountRepository();
}

@Configuration
public class DefaultRepositoryConfig implements RepositoryConfig {

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(...);
    }
}

@Configuration
@Import({ServiceConfig.class, DefaultRepositoryConfig.class})  // import the concrete config!
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return DataSource
    }

}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
```

现在，`ServiceConfig`与具体的`DefaultRepositoryConfig`松散耦合，内置的IDE工具仍然很有用：您可以很容易获取`RepositoryConfig`实现类的继承体系。 以这种方式,操作`@Configuration`类及其依赖关系与操作基于接口的代码的过程没有什么区别

如果要影响某些bean的启动创建顺序，可以考虑将其中一些声明为`@Lazy` （用于在首次访问时创建而不是在启动时）或`@DependsOn`某些其他bean（确保在创建之前创建特定的其他bean（当前的bean，超出后者的直接依赖性所暗示的））。

<a id="beans-java-conditional"></a>

##### [](#beans-java-conditional)有条件地包含`@Configuration`类或`@Bean`方法

基于某些任意系统状态，有条件地启用或禁用完整的`@Configuration`类甚至单独的 `@Bean` 方法通常很有用。 一个常见的例子是， 只有在Spring环境中启用了特定的配置文件时才使用`@Profile` 注解来激活bean（有关详细信息，请参阅Bean[定义配置文件](#beans-definition-profiles)）。

`@Profile`注解实际上是通过使用更灵活的注解`@Conditional`实现的。[`@Conditional`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/annotation/Conditional.html)注解表示特定的`org.springframework.context.annotation.Condition`实现。 它表明`@Bean`被注册之前会先"询问"`@Conditional`注解。

`Condition`接口的实现提供了一个返回`true` 或 `false`的`matches(…)`方法。例如，以下清单显示了用于 `@Profile`的实际`Condition`实现：

```java
@Override
public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
    if (context.getEnvironment() != null) {
        // Read the @Profile annotation attributes
        MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
        if (attrs != null) {
            for (Object value : attrs.get("value")) {
                if (context.getEnvironment().acceptsProfiles(((String[]) value))) {
                    return true;
                }
            }
            return false;
        }
    }
    return true;
}
```

有关更多详细信息，请参阅[`@Conditional`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/annotation/Conditional.html)javadoc。

<a id="beans-java-combining"></a>

##### [](#beans-java-combining)结合Java和XML配置

Spring的`@Configuration`类支持但不一定成为Spring XML的100％完全替代品。 某些工具（如Spring XML命名空间）仍然是配置容器的理想方法。在XML方便或必要的情况下，您可以选择：通过使用例如`ClassPathXmlApplicationContext`以“以XML为中心”的方式实例化容器， 或者通过使用`AnnotationConfigApplicationContext`以“以Java为中心”的方式实例化它。`@ImportResource`注解，根据需要导入XML。

<a id="beans-java-combining-xml-centric"></a>

###### [](#beans-java-combining-xml-centric)以XML为中心使用`@Configuration`类

更受人喜爱的方法是从包含`@Configuration`类的XML启动容器.例如，在使用Spring的现有系统中,大量使用的是Spring XML配置,所以很容易根据需要创建`@Configuration`类 ,并将他们到包含XML文件中。我们将介绍在这种“以XML为中心”的情况下使用`@Configuration`类的选项。

将`@Configuration`类声明为普通的Spring`<bean/>` 元素

请记住,`@Configuration`类最终也只是容器中的bean定义。在本系列示例中，我们创建一个名为AppConfig的`@Configuration`类，并将其作为`<bean/>`定义包含在`system-test-config.xml`中。 由于 `<context:annotation-config/>`已打开，容器会识别`@Configuration` 注解并正确处理`AppConfig`中声明的`@Bean` 方法。

以下示例显示了Java中的普通配置类:

```java
@Configuration
public class AppConfig {

    @Autowired
    private DataSource dataSource;

    @Bean
    public AccountRepository accountRepository() {
        return new JdbcAccountRepository(dataSource);
    }

    @Bean
    public TransferService transferService() {
        return new TransferService(accountRepository());
    }
}
```

以下示例显示了示例`system-test-config.xml`文件的一部分：

    <beans>
        <!-- enable processing of annotations such as @Autowired and @Configuration -->
        <context:annotation-config/>
        <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>

        <bean class="com.acme.AppConfig"/>

        <bean class="org.springframework.jdbc.datasource.DriverManagerDataSource">
            <property name="url" value="${jdbc.url}"/>
            <property name="username" value="${jdbc.username}"/>
            <property name="password" value="${jdbc.password}"/>
        </bean>
    </beans>

以下示例显示了可能的`jdbc.properties`文件:

```
jdbc.url=jdbc:hsqldb:hsql://localhost/xdb
jdbc.username=sa
jdbc.password=
```



    public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath:/com/acme/system-test-config.xml");
        TransferService transferService = ctx.getBean(TransferService.class);
        // ...
    }

在 `system-test-config.xml`文件中， `AppConfig` `<bean/>`不声明`id`元素。虽然这样做是可以的，但是没有必要，因为没有其他bean引用它，并且不太可能通过名称从容器中明确地获取它。 类似地，`DataSource` bean只是按类型自动装配，因此不严格要求显式的bean`id`。

使用<context:component-scan/> 来获取`@Configuration` 类

因为`@Configuration`是`@Component`注解的元注解,所以`@Configuration`注解的类也可以被自动扫描。使用与上面相同的场景，可以重新定义`system-test-config.xml` 以使用组件扫描。 请注意，在这种情况下，我们不需要显式声明 `<context:annotation-config/>`,，因为`<context:component-scan/>` 启用相同的功能。

以下示例显示了已修改的`system-test-config.xml`文件:

```xml
<beans>
    <!-- picks up and registers AppConfig as a bean definition -->
    <context:component-scan base-package="com.acme"/>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>

    <bean class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.username}"/>
        <property name="password" value="${jdbc.password}"/>
    </bean>
</beans>
```

<a id="beans-java-combining-java-centric"></a>

###### [](#beans-java-combining-java-centric)基于`@Configuration`混合XML的`@ImportResource`

在 `@Configuration`类为配置容器的主要方式的应用程序中,也需要使用一些XML配置。在这些情况下,只需使用`@ImportResource` ,并只定义所需的XML。这样做可以实现“以Java为中心”的方法来配置容器并尽可能少的使用XML。 以下示例（包括配置类，定义bean的XML文件，属性文件和主类）显示了如何使用`@ImportResource` 注解来实现根据需要使用XML的“以Java为中心”的配置：

```java
@Configuration
@ImportResource("classpath:/com/acme/properties-config.xml")
public class AppConfig {

    @Value("${jdbc.url}")
    private String url;

    @Value("${jdbc.username}")
    private String username;

    @Value("${jdbc.password}")
    private String password;

    @Bean
    public DataSource dataSource() {
        return new DriverManagerDataSource(url, username, password);
    }
}

properties-config.xml
<beans>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>
</beans>
```

```
jdbc.properties
jdbc.url=jdbc:hsqldb:hsql://localhost/xdb
jdbc.username=sa
jdbc.password=
```



    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
        TransferService transferService = ctx.getBean(TransferService.class);
        // ...
    }

<a id="beans-environment"></a>

### [](#beans-environment)1.13. 抽象环境

[`Environment`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/core/env/Environment.html)接口是集成在容器中的抽象，它模拟了应用程序环境的两个关键方面：[profiles](#beans-definition-profiles) 和 [properties](#beans-property-source-abstraction)。

profile配置是一个被命名的,bean定义的逻辑组,这些bean只有在给定的profile配置激活时才会注册到容器.无论是以XML还是通过注解定义,Bean都可以分配给配置文件 。`Environment`对象在profile中的角色是判断哪一个profile应该在当前激活和哪一个profile应该在默认情况下激活。

属性在几乎所有应用程序中都发挥着重要作用，可能源自各种源：属性文件，JVM系统属性，系统环境变量，JNDI，servlet上下文参数，ad-hoc属性对象，Map对象等。 与属性相关的`Environment`对象的作用是为用户提供方便的服务接口，用于配置属性源和从中解析属性。

<a id="beans-definition-profiles"></a>

#### [](#beans-definition-profiles)1.13.1. Bean定义Profiles

bean定义profiles是核心容器内的一种机制，该机制能在不同环境中注册不同的bean。“环境”这个词对不同的用户来说意味着不同的东西，这个功能可以帮助解决许多用例，包括：

*   在QA或生产环境中，针对开发中的内存数据源而不是从JNDI查找相同的数据源。

*   开发期使用监控组件，当部署以后则关闭监控组件，使应用更高效

*   为用户各自注册自定义bean实现


考虑`DataSource`的实际应用程序中的第一个用例。 在测试环境中，配置可能类似于以下内容：

```java
@Bean
public DataSource dataSource() {
    return new EmbeddedDatabaseBuilder()
        .setType(EmbeddedDatabaseType.HSQL)
        .addScript("my-schema.sql")
        .addScript("my-test-data.sql")
        .build();
}
```

现在考虑如何将此应用程序部署到QA或生产环境中，假设应用程序的数据源已注册到生产应用程序服务器的JNDI目录。 我们的`dataSource` bean现在看起来如下：

```java
@Bean(destroyMethod="")
public DataSource dataSource() throws Exception {
    Context ctx = new InitialContext();
    return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
}
```

问题是如何根据当前环境在使用这两种变体之间切换。随着时间的推移，Spring用户已经设计了许多方法来完成这项工作，通常依赖于系统环境变量和包含`${placeholder}`标记的XML`<import/>`语句的组合， 这些标记根据值解析为正确的配置文件路径一个环境变量。 Bean定义profiles是核心容器功能，可为此问题提供解决方案。

概括一下上面的场景，环境决定bean定义,最后发现,我们需要在某些上下文环境中使用某些bean,在其他环境中则不用这些bean.或者说, 在场景A中注册一组bean定义,而在场景B中注册另外一组。先看看如何通过修改配置来完成此需求：

<a id="beans-definition-profiles-java"></a>

##### [](#beans-definition-profiles-java)使用 `@Profile`

[`@Profile`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/annotation/Profile.html)注解用于当一个或多个配置文件激活的时候,用来指定组件是否有资格注册。使用前面的示例，我们可以重写`dataSource`配置，如下所示：

```java
@Configuration
@Profile("development")
public class StandaloneDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }
}

@Configuration
@Profile("production")
public class JndiDataConfig {

    @Bean(destroyMethod="")
    public DataSource dataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```

如前所述，使用`@Bean`方法，您通常选择使用Spring的`JndiTemplate`/`JndiLocatorDelegate`帮助程序或前面显示的 直接JNDI `InitialContext`用法但不使用`JndiObjectFactoryBean`变量来使用编程JNDI查找，这会强制您将返回类型声明为 `FactoryBean`类型。 As mentioned earlier, with `@Bean` methods, you typically choose to use programmatic JNDI lookups, by using either Spring’s `JndiTemplate`/`JndiLocatorDelegate` helpers or the straight JNDI `InitialContext` usage shown earlier but not the `JndiObjectFactoryBean` variant, which would force you to declare the return type as the `FactoryBean` type.

profile字符串可以包含简单的profile名称（例如，`production`）或profile表达式。 profile表达式允许表达更复杂的概要逻辑（例如，`production & us-east`）。 profile表达式支持以下运算符：

*   `!`: A logical “not” of the profile

*   `&`: A logical “and” of the profiles

*   `|`: A logical “or” of the profiles


你不能不使用括号而混合 `&` 和 `|` 。 例如，`production & us-east | eu-central`不是一个有效的表达。 它必须表示为 `production & (us-east | eu-central)`.。

您可以将`@Profile`用作[元注解](#beans-meta-annotations)，以创建自定义组合注解。 以下示例定义了一个自定义`@Production`注解，您可以将其用作`@Profile("production")`的替代品：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Profile("production")
public @interface Production {
}
```

如果`@Configuration`类标有 `@Profile`,类中所有`@Bean`和`@Import`注解相关的类都将被忽略,除非该profile被激活。 如果一个`@Component`或`@Configuration`类被标记为`@Profile({"p1", "p2"})`。那么除非profile 'p1' or 'p2' 已被激活。 否则该类将不会注册/处理。如果给定的配置文件以NOT运算符(`!`)为前缀，如果配置文件为not active，则注册的元素将被注册。 例如，给定`@Profile({"p1", "!p2"})`，如果配置文件“p1”处于活动状态或配置文件“p2”未激活，则会进行注册。

`@Profile`也能注解方法，用于配置一个配置类中的指定bean。如以下示例所示：

```java
@Configuration
public class AppConfig {

    @Bean("dataSource")
    @Profile("development") (1)
    public DataSource standaloneDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .addScript("classpath:com/bank/config/sql/test-data.sql")
            .build();
    }

    @Bean("dataSource")
    @Profile("production") (2)
    public DataSource jndiDataSource() throws Exception {
        Context ctx = new InitialContext();
        return (DataSource) ctx.lookup("java:comp/env/jdbc/datasource");
    }
}
```

**1**、`standaloneDataSource` 方法仅在 `development` 环境可用.

**2**、`jndiDataSource`方法仅在 `production` 环境可用.

在`@Bean` 方法上还添加有`@Profile`注解,可能会应用在特殊情况。在相同Java方法名称的重载`@Bean`方法(类似于构造函数重载）的情况下， 需要在所有重载方法上一致声明`@Profile`条件，如果条件不一致，则只有重载方法中第一个声明的条件才重要。因此，`@Profile`不能用于选择具有特定参数签名的重载方法， 所有工厂方法对相同的bean在Spring构造器中的解析算法在创建时是相同的。

如果想定义具有不同配置文件条件的备用bean，请使用不同的Java方法名称，通过`@Bean`名称属性指向相同的bean名称。如上例所示。 如果参数签名都是相同的（例如，所有的变体都是无参的工厂方法），这是安排有效Java类放在首要位置的唯一方法（因为只有一个 特定名称和参数签名的方法）。

<a id="beans-definition-profiles-xml"></a>

##### [](#beans-definition-profiles-xml)XML bean定义profiles

XML中的`<beans>` 元素有一个`profile` 属性,我们之前的示例配置可以在两个XML文件中重写，如下所示：

```xml
<beans profile="development"
    xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    xsi:schemaLocation="...">

    <jdbc:embedded-database id="dataSource">
        <jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
        <jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
    </jdbc:embedded-database>
</beans>

<beans profile="production"
    xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="...">

    <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
</beans>
```

也可以不用分开2个文件，在同一个XML中配置2个`<beans/>`，`<beans/>`元素也有profile属性。如以下示例所示：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="...">

    <!-- other bean definitions -->

    <beans profile="development">
        <jdbc:embedded-database id="dataSource">
            <jdbc:script location="classpath:com/bank/config/sql/schema.sql"/>
            <jdbc:script location="classpath:com/bank/config/sql/test-data.sql"/>
        </jdbc:embedded-database>
    </beans>

    <beans profile="production">
        <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
    </beans>
</beans>
```

`spring-bean.xsd` 强制允许将profile元素定义在文件的最后面，这有助于在XML文件中提供灵活的方式而又不引起混乱。

对应XML不支持前面描述的profile表达式。 但是，有可能通过使用`!` 来否定一个profile表达式。 也可以通过嵌套profiles来应用“and”，如以下示例所示：

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jdbc="http://www.springframework.org/schema/jdbc"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="...">

    <!-- other bean definitions -->

    <beans profile="production">
        <beans profile="us-east">
            <jee:jndi-lookup id="dataSource" jndi-name="java:comp/env/jdbc/datasource"/>
        </beans>
    </beans>
</beans>
```

在前面的示例中，如果`production` 和`us-east` profiles都处于活动状态，则会暴露`dataSource` bean。

<a id="beans-definition-profiles-enable"></a>

##### [](#beans-definition-profiles-enable)启用profile

现在已经更新了配置,但仍然需要指定要激活哪个配置文件, 如果我们现在开始我们的示例应用程序， 我们会看到抛出`NoSuchBeanDefinitionException`，因为容器找不到名为`dataSource`的Spring bean。

激活配置文件可以通过多种方式完成，但最直接的方法是以编程方式对可通过`ApplicationContext`提供的`Environment` API进行操作。 以下示例显示了如何执行此操作：

```java
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
ctx.getEnvironment().setActiveProfiles("development");
ctx.register(SomeConfig.class, StandaloneDataConfig.class, JndiDataConfig.class);
ctx.refresh();
```

此外,配置文件也可以通过`spring.profiles.active`属性声明式性地激活,可以通过系统环境变量，JVM系统属性，`web.xml`中的Servlet上下文参数指定， 甚至作为JNDI中的一个条目设置（[`PropertySource` 抽象](#beans-property-source-abstraction)）。在集成测试中，可以通过 `spring-test`模块中的`@ActiveProfiles`注解来声明活动配置文件(参见使用[环境配置文件的上下文配置](https://github.com/DocsHome/spring-docs/blob/master/pages/test/testing.mdl#testcontext-ctx-management-env-profiles))

配置文件不是“二选一”的。开发者可以一次激活多个配置文件。使用编程方式，您可以为`setActiveProfiles()`方法提供多个配置文件名称，该方法接受 `String…`varargs。 以下示例激活多个配置文件：

```java
ctx.getEnvironment().setActiveProfiles("profile1", "profile2");
```

声明性地，`spring.profiles.active`可以接受以逗号分隔的profile名列表，如以下示例所示：

```java
-Dspring.profiles.active="profile1,profile2"
```

<a id="beans-definition-profiles-default"></a>

##### [](#beans-definition-profiles-default)默认 Profile

default配置文件表示默认开启的profile配置。考虑以下配置:

```java
@Configuration
@Profile("default")
public class DefaultDataConfig {

    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.HSQL)
            .addScript("classpath:com/bank/config/sql/schema.sql")
            .build();
    }
}
```

如果没有配置文件激活，上面的`dataSource`就会被创建。这提供了一种默认的方式，如果有任何一个配置文件启用，default配置就不会生效。

默认配置文件的名字(default）可以通过`Environment`的`setDefaultProfiles()`方法或者`spring.profiles.default`属性修改。

<a id="beans-property-source-abstraction"></a>

#### [](#beans-property-source-abstraction)1.13.2. `PropertySource` 抽象

Spring的`Environment`抽象提供用于一系列的propertysources属性配置文件的搜索操作.请考虑以下列表：

```java
ApplicationContext ctx = new GenericApplicationContext();
Environment env = ctx.getEnvironment();
boolean containsMyProperty = env.containsProperty("my-property");
System.out.println("Does my environment contain the 'my-property' property? " + containsMyProperty);
```

在上面的代码段中,一个高级别的方法用于访问Spring是否为当前环境定义了`my-property` 属性。为了回答这个问题，`Environment`对象对一组PropertySource对象进行搜索。 [`PropertySource`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/core/env/PropertySource.html)是对任何键值对的简单抽象，Spring的[`StandardEnvironment`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/core/env/StandardEnvironment.html)配置有两个`PropertySource`对象 ，一个表示JVM系统属性(`System.getProperties()`),一个表示系统环境变量(`System.getenv()`)。

这些默认property源位于`StandardEnvironment`中,用于独立应用程序。[`StandardServletEnvironment`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/web/context/support/StandardServletEnvironment.html)用默认的property配置源填充。 默认配置源包括Servlet配置和Servlet上下文参数，它可以选择启用[`JndiPropertySource`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/jndi/JndiPropertySource.html)。有关详细信息，请参阅它的javadocs

具体地说，当您使用`StandardEnvironment`时，如果在运行时存在`my-property`系统属性或`my-propertyi`环境变量，则对 `env.containsProperty("my-property")`的调用将返回true。

执行的搜索是分层的。默认情况下，系统属性优先于环境变量，因此如果在调用`env.getProperty("my-property")`期间碰巧在两个位置都设置了`my-property`属性， 系统属性值返回优先于环境变量。 请注意，属性值未合并，而是由前面的条目完全覆盖。

对于常见的 `StandardServletEnvironment`，完整层次结构如下，最高优先级条目位于顶部：

1.  ServletConfig参数（如果适用 - 例如，在DispatcherServlet上下文的情况下）

2.  ServletContext参数（web.xml context-param条目）

3.  JNDI环境变量（`java:comp/env/`entries）

4.  JVM系统属性（`-D`命令行参数）

5.  JVM系统环境（操作系统环境变量）


最重要的是,整个机制都是可配置的。也许开发者需要一个自定义的properties源，并将该源整合到这个检索层级中。为此，请实现并实例化您自己的`PropertySource`，并将其添加到当前`Environment`的`PropertySource`集合中。 以下示例显示了如何执行此操作：

```java
ConfigurableApplicationContext ctx = new GenericApplicationContext();
MutablePropertySources sources = ctx.getEnvironment().getPropertySources();
sources.addFirst(new MyPropertySource());
```

在上面的代码中， `MyPropertySource`在搜索中添加了最高优先级。如果它包含`my-property`属性，则会检测并返回该属性， 优先于其他 `PropertySource`中的任何`my-property`属性。 [`MutablePropertySources`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/core/env/MutablePropertySources.html) API公开了许多方法，允许你显式操作property属性源。

<a id="beans-using-propertysource"></a>

#### [](#beans-using-propertysource)1.13.3. 使用 `@PropertySource`

[`@PropertySource`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/annotation/PropertySource.html) 注解提供了便捷的方式，用于增加`PropertySource`到Spring的 `Environment`中。

给定一个名为`app.properties`的文件，其中包含键值对`testbean.name=myTestBean`， 以下`@Configuration`类使用`@PropertySource`，以便调用`testBean.getName()` 返回`myTestBean`：

```java
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
public class AppConfig {

    @Autowired
    Environment env;

    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
```

任何的存在于`@PropertySource`中的`${…}`占位符，将会被解析为定义在环境中的属性配置文件中的属性值。 如以下示例所示：

```java
@Configuration
@PropertySource("classpath:/com/${my.placeholder:default/path}/app.properties")
public class AppConfig {

    @Autowired
    Environment env;

    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
```

假设`my.placeholder`存在于已注册的其中一个属性源中（例如，系统属性或环境变量），则占位符将解析为相应的值。 如果不是，则`default/path`用作默认值。 如果未指定默认值且无法解析属性，则抛出`IllegalArgumentException`。

根据Java 8惯例，`@PropertySource`注解是可重复的。 但是，所有这些`@PropertySource`注解都需要在同一级别声明，可以直接在配置类上声明， 也可以在同一自定义注解中作为元注解声明。 不建议混合直接注解和元注解，因为直接注解有效地覆盖了元注解。

<a id="beans-placeholder-resolution-in-statements"></a>

#### [](#beans-placeholder-resolution-in-statements)1.13.4. 在声明中的占位符

之前，元素中占位符的值只能针对JVM系统属性或环境变量进行解析。现在已经打破了这种情况。因为环境抽象集成在整个容器中，所以很容易通过它来对占位符进行解析. 这意味着开发者可以以任何喜欢的方式来配置这个解析过程，可以改变是优先查找系统properties或者是有限查找环境变量，或者删除它们；增加自定义property源，使之成为更合适的配置

具体而言，只要在`Environment`中可用，无论`customer`属性在何处定义，以下语句都可以工作：

```xml
<beans>
    <import resource="com/bank/service/${customer}-config.xml"/>
</beans>
```

<a id="context-load-time-weaver"></a>

### [](#context-load-time-weaver)1.14. 注册`LoadTimeWeaver`

`LoadTimeWeaver`被Spring用来在将类加载到Java虚拟机(JVM)中时动态地转换类

若要开启加载时织入,要在`@Configuration`类中增加`@EnableLoadTimeWeaving`注解，如以下示例所示：

```java
@Configuration
@EnableLoadTimeWeaving
public class AppConfig {
}
```

或者，对于XML配置，您可以使用`context:load-time-weaver` 元素:

```xml
<beans>
    <context:load-time-weaver/>
</beans>
```

一旦配置为 `ApplicationContext`,该 `ApplicationContext`中的任何bean都可以实现`LoadTimeWeaverAware`,从而接收对load-time weaver实例的引用。 这特别适用于[Spring的JPA支持](https://github.com/DocsHome/spring-docs/blob/master/pages/dataaccess/data-access.md#orm-jpa)。其中JPA类转换可能需要加载时织入。 有关更多详细信息，请参阅[`LocalContainerEntityManagerFactoryBean`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/orm/jpa/LocalContainerEntityManagerFactoryBean.html).。 有关AspectJ加载时编织的更多信息，请参阅[Spring Framework中使用AspectJ的加载时织入](#aop-aj-ltw)。

<a id="context-introduction"></a>

### [](#context-introduction)1.15.`ApplicationContext`的附加功能

正如 [前面章节](#beans)中讨论的，`org.springframework.beans.factory`包提供了管理和操作bean的基本功能，包括以编程方式。 `org.springframework.context`包添加了[`ApplicationContext`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/ApplicationContext.html)接口，该接口扩展了`BeanFactory`接口，此外还扩展了其他接口，以更面向应用程序框架的方式提供其他功能。 许多人以完全声明的方式使用`ApplicationContext`， 甚至不以编程方式创建它，而是依赖于诸如`ContextLoader`之类的支持类来自动实例化`ApplicationContext`，作为Java EE Web应用程序的正常启动过程的一部分。

为了以更加面向框架的方式增强`BeanFactory`的功能,上下文包还提供了以下功能。

*   通过`MessageSource`接口访问i18n风格的消息。

*   通过`ResourceLoader`接口访问URL和文件等资源。

*   事件发布，即通过使用`ApplicationEventPublisher`接口实现`ApplicationListener`接口的bean。

*   通过`HierarchicalBeanFactory`接口，加载多级contexts，允许关注某一层级context，比如应用的Web层。

<a id="context-functionality-messagesource"></a>

#### [](#context-functionality-messagesource)1.15.1. 使用`MessageSource`实现国际化

`ApplicationContext` 接口扩展了一个名为`MessageSource`的接口，因此提供了国际化(“i18n”)功能。 Spring还提供了`HierarchicalMessageSource`接口，该接口可以分层次地解析消息。 这些接口共同提供了Spring影响消息解析的基础。 这些接口上定义的方法包括：

*   `String getMessage(String code, Object[] args, String default, Locale loc)`: 用于从`MessageSource`检索消息的基本方法。 如果未找到指定区域设置的消息，则使用默认消息。 传入的任何参数都使用标准库提供的`MessageFormat`功能成为替换值。

*   `String getMessage(String code, Object[] args, Locale loc)`: 基本上与前一个方法相同，但有一个区别：不能指定默认消息。 如果找不到该消息，则抛出`NoSuchMessageException` 。

*   `String getMessage(MessageSourceResolvable resolvable, Locale locale)`: 前面方法中使用的所有属性也包装在名为 `MessageSourceResolvable`的类中，您可以将此方法用于此类。


当一个`ApplicationContext`被加载时,它会自动搜索在上下文中定义的一个`MessageSource`,bean必须包含名称`messageSource`,如果找到这样的bean 则将对前面方法的所有调用委派给消息源。如果没有找到消息源，`ApplicationContext`会尝试找到一个包含同名bean的父对象。如果有，它使用那个bean作为`MessageSource`。 如果`ApplicationContext`找不到消息的任何源，则会实例化空的`DelegatingMessageSource`，以便能够接受对上面定义的方法的调用。

Spring提供了两个MessageSource实现， `ResourceBundleMessageSource`和`StaticMessageSource`，为了做嵌套消息两者都实现了`HierarchicalMessageSource`。 `StaticMessageSource`很少使用，但提供了以编程方式向源添加消息。 以下示例显示了`ResourceBundleMessageSource`：

```xml
<beans>
    <bean id="messageSource"
            class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basenames">
            <list>
                <value>format</value>
                <value>exceptions</value>
                <value>windows</value>
            </list>
        </property>
    </bean>
</beans>
```

该示例假定您在类路径中定义了三个资源包，`format`, `exceptions` and `windows`。任何解析消息的请求都将以JDK标准方式处理, 通过`ResourceBundle`解析消息。出于示例的目的，假设上述两个资源包文件的内容如下：

```java
# in format.properties
message=Alligators rock!

# in exceptions.properties
argument.required=The {0} argument is required.
```

下一个示例显示了执行`MessageSource` 功能的程序。 请记住，所有`ApplicationContext`实现也都是`MessageSource` 实现，因此可以强制转换为`MessageSource` 接口。

```java
public static void main(String[] args) {
    MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
    String message = resources.getMessage("message", null, "Default", null);
    System.out.println(message);
}
```

上述程序产生的结果如下:

Alligators rock!

总而言之，`MessageSource`在名为`beans.xml`的文件中定义，该文件存在于类路径的根目录中。`messageSource`bean定义通过其basenames属性引用许多资源包。 在列表中传递给`basenames`属性的三个文件作为类路径根目录下的文件存在，分别称为`format.properties`, `exceptions.properties`和`windows.properties`。

下一个示例显示传递给消息查询的参数，这些参数将被转换为字符串并插入查找消息中的占位符。

```java
<beans>

    <!-- this MessageSource is being used in a web application -->
    <bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
        <property name="basename" value="exceptions"/>
    </bean>

    <!-- lets inject the above MessageSource into this POJO -->
    <bean id="example" class="com.something.Example">
        <property name="messages" ref="messageSource"/>
    </bean>

</beans>

public class Example {

    private MessageSource messages;

    public void setMessages(MessageSource messages) {
        this.messages = messages;
    }

    public void execute() {
        String message = this.messages.getMessage("argument.required",
            new Object [] {"userDao"}, "Required", null);
        System.out.println(message);
    }
}
```

调用 `execute()`方法得到的结果如下:

The userDao argument is required.

关于国际化(“i18n”)，Spring的各种 `MessageSource`实现遵循与标准JDK `ResourceBundle`相同的区域设置解析和回退规则。 简而言之，继续前面定义的示例`messageSource`，如果要根据British(`en-GB`)语言环境解析消息，则应分别创建名为`format_en_GB.properties`，`exceptions_en_GB.properties`和`windows_en_GB.properties`的文件。

通常，区域设置解析由应用程序的环境配置管理。在以下示例中，手动指定解析（英国）消息的区域设置：

\# in exceptions\_en\_GB.properties
argument.required=Ebagum lad, the {0} argument is required, I say, required.

```java
public static void main(final String[] args) {
    MessageSource resources = new ClassPathXmlApplicationContext("beans.xml");
    String message = resources.getMessage("argument.required",
        new Object [] {"userDao"}, "Required", Locale.UK);
    System.out.println(message);
}
```

运行上述程序产生的结果如下:

Ebagum lad, the 'userDao' argument is required, I say, required.

您还可以使用`MessageSourceAware`接口获取对已定义的任何`MessageSource`的引用。 在创建和配置bean时，应用程序上下文的`MessageSource`会注入实现`MessageSourceAware`接口的`ApplicationContext`中定义的任何bean。

作为`ResourceBundleMessageSource`的替代，Spring提供了一个`ReloadableResourceBundleMessageSource`类。 此变体支持相同的bundle文件格式，但比基于标准JDK的`ResourceBundleMessageSource`实现更灵活。特别是，它允许从任何Spring资源位置（不仅从类路径）读取文件，并支持bundle属性文件的热重新加载（同时在其间有效地缓存它们）。 有关详细信息，请参阅[`ReloadableResourceBundleMessageSource`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/support/ReloadableResourceBundleMessageSource.html) javadoc。

<a id="context-functionality-events"></a>

#### [](#context-functionality-events)1.15.2. 标准和自定义事件

`ApplicationContext`中的事件处理是通过`ApplicationEvent`类和`ApplicationListener`接口提供的。如果将实现`ApplicationListener`接口的bean部署到上下文中，则每次将`ApplicationEvent`发布到`ApplicationContext`时，都会通知该bean。 从本质上讲，这是标准的Observer设计模式。

从Spring 4.2开始，事件架构已经得到显着改进，并提供了一个[基于注解的模型](#context-functionality-events-annotation)使其有发布任意事件的能力（即，不一定从`ApplicationEvent`扩展的对象） 。当发布这样的对象时，我们将它包装在一个事件中。

下表描述了Spring提供的标准事件：:

Table 7. Built-in Events

| 事件                    | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| `ContextRefreshedEvent` | 初始化或刷新`ApplicationContext`时发布（例如，通过使用`ConfigurableApplicationContext` 接口上的`refresh()` 方法）。 这里，“initialized”意味着加载所有bean，检测并激活bean的后置处理器，预先实例化单例，并且可以使用`ApplicationContext`对象。 只要上下文尚未关闭，只要所选的`ApplicationContext`实际支持这种“热”刷新，就可以多次触发刷新。 例如，`XmlWebApplicationContext`支持热刷新，但`GenericApplicationContext` 不支持。 |
| `ContextStartedEvent`   | 通过使用`ConfigurableApplicationContext`接口上的`start()`方法启动`ApplicationContext` 时发布。 通常，此信号用于在显式停止后重新启动Bean，但它也可用于启动尚未为自动启动配置的组件（例如，在初始化时尚未启动的组件）。 |
| `ContextStoppedEvent`   | 通过使用`ConfigurableApplicationContext`接口上的`close()` 方法停止`ApplicationContext`时发布。 这里，“已停止”表示所有生命周期bean都会收到明确的停止信号。 可以通过`start()`调用重新启动已停止的上下文。 |
| `ContextClosedEvent`    | 通过使用`ConfigurableApplicationContext`接口上的`close()`方法关闭`ApplicationContext`时发布。 这里， “关闭” 意味着所有单例bean都被销毁。 封闭的环境达到其寿命终结。 它无法刷新或重新启动。 |
| `RequestHandledEvent`   | 一个特定于Web的事件，告诉所有bean已经为HTTP请求提供服务。 请求完成后发布此事件。 此事件仅适用于使用Spring的DispatcherServlet的Web应用程序。 |


您还可以创建和发布自己的自定义事件。 以下示例显示了一个扩展Spring的`ApplicationEvent`基类的简单类：

```java
public class BlackListEvent extends ApplicationEvent {

    private final String address;
    private final String content;

    public BlackListEvent(Object source, String address, String content) {
        super(source);
        this.address = address;
        this.content = content;
    }

    // accessor and other methods...
}
```

要发布自定义`ApplicationEvent`，请在`ApplicationEventPublisher`上调用`publishEvent()`方法。 通常，这是通过创建一个实现 `ApplicationEventPublisherAware`并将其注册为Spring bean的类来完成的。 以下示例显示了这样一个类：

```java
public class EmailService implements ApplicationEventPublisherAware {

    private List<String> blackList;
    private ApplicationEventPublisher publisher;

    public void setBlackList(List<String> blackList) {
        this.blackList = blackList;
    }

    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void sendEmail(String address, String content) {
        if (blackList.contains(address)) {
            publisher.publishEvent(new BlackListEvent(this, address, content));
            return;
        }
        // send email...
    }
}
```

在配置时，Spring容器检测到`EmailService`实现`ApplicationEventPublisherAware`并自动调用`setApplicationEventPublisher()`。 实际上，传入的参数是Spring容器本身。 您正在通过其`ApplicationEventPublisher`接口与应用程序上下文进行交互。

要接收自定义 `ApplicationEvent`，您可以创建一个实现`ApplicationListener`的类并将其注册为Spring bean。 以下示例显示了这样一个类：

```java
public class BlackListNotifier implements ApplicationListener<BlackListEvent> {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    public void onApplicationEvent(BlackListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```

请注意，`ApplicationListener`通常使用自定义事件的类型进行参数化（前面示例中为`BlackListEvent`）。这意味着`onApplicationEvent()`方法可以保持类型安全，从而避免任何向下转换的需要。 您可以根据需要注册任意数量的事件侦听器，但请注意，默认情况下，事件侦听器会同步接收事件。这意味着`publishEvent()`方法将阻塞，直到所有侦听器都已完成对事件的处理。 这种同步和单线程方法的一个优点是，当侦听器接收到事件时，如果事务上下文可用，它将在发布者的事务上下文内运行。如果需要另一个事件发布策略，请参阅Spring的[`ApplicationEventMulticaster`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/context/event/ApplicationEventMulticaster.html)接口的javadoc。

以下示例显示了用于注册和配置上述每个类的bean定义:

```xml
<bean id="emailService" class="example.EmailService">
    <property name="blackList">
        <list>
            <value>[email protected]</value>
            <value>[email protected]</value>
            <value>[email protected]</value>
        </list>
    </property>
</bean>

<bean id="blackListNotifier" class="example.BlackListNotifier">
    <property name="notificationAddress" value="[email protected]"/>
</bean>
```

总而言之，当调用`emailService` bean的`sendEmail()`方法时，如果有任何应列入黑名单的电子邮件消息，则会发布`BlackListEvent`类型的自定义事件。 `blackListNotifier` bean注册为`ApplicationListener` 并接收`BlackListEvent` ，此时它可以通知相关方。

Spring的事件机制是为在同一应用程序上下文中的Spring bean之间的简单通信而设计的。但是，对于更复杂的企业集成需求，单独维护的[Spring Integration](https://projects.spring.io/spring-integration/) 项目提供了完整的支持并可用于构建轻量级，[pattern-oriented](http://www.enterpriseintegrationpatterns.com)(面向模式），依赖Spring编程模型的事件驱动架构。

<a id="context-functionality-events-annotation"></a>

##### [](#context-functionality-events-annotation)基于注解的事件监听器

从Spring 4.2开始，您可以使用`EventListener`注解在托管bean的任何公共方法上注册事件监听器。 `BlackListNotifier`可以重写如下：

```java
public class BlackListNotifier {

    private String notificationAddress;

    public void setNotificationAddress(String notificationAddress) {
        this.notificationAddress = notificationAddress;
    }

    @EventListener
    public void processBlackListEvent(BlackListEvent event) {
        // notify appropriate parties via notificationAddress...
    }
}
```

方法签名再次声明它侦听的事件类型，但这次使用灵活的名称并且没有实现特定的侦听器接口。只要实际事件类型在其实现层次结构中解析通用参数，也可以通过泛型缩小事件类型。

如果您的方法应该监听多个事件，或者您想要根据任何参数进行定义，那么也可以在注解本身上指定事件类型。 以下示例显示了如何执行此操作：:

    @EventListener({ContextStartedEvent.class, ContextRefreshedEvent.class})
    public void handleContextStart() {
        ...
    }

还可以通过使用定义[`SpEL` 表达式](#expressions)的注解的`condition`属性来添加额外的运行时过滤，该表达式应匹配以实际调用特定事件的方法。

以下示例显示了仅当事件的`content`属性等于`my-event`时才能重写我们的通知程序以进行调用：

```java
@EventListener(condition = "#blEvent.content == 'my-event'")
public void processBlackListEvent(BlackListEvent blEvent) {
    // notify appropriate parties via notificationAddress...
}
```

每个`SpEL`表达式都针对专用上下文进行评估。 下表列出了可用于上下文的项目，以便您可以将它们用于条件事件处理：

Table 8. 事件SpEL可用的元数据

| 名字            | 位置               | 描述                                                         | 例子                                                         |
| --------------- | ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Event           | root               | 实际`ApplicationEvent`                                       | `#root.event`                                                |
| Arguments array | root object        | 用于调用目标的参数（作为数组）                               | `#root.args[0]`                                              |
| _Argument name_ | evaluation context | 任何方法参数的名称。 如果由于某种原因，名称不可用（例如，因为没有调试信息），参数名称也可以在#a `#a<#arg>`下获得，其中`#arg`代表参数索引（从0开始）。 | `#blEvent` or `#a0` (you can also use `#p0` or `#p<#arg>` notation as an alias) |

请注意，即使您的方法签名实际引用已发布的任意对象，`#root.event`也允许您访问基础事件。

如果需要发布一个事件作为处理另一个事件的结果，则可以更改方法签名以返回应发布的事件，如以下示例所示：

```java
@EventListener
public ListUpdateEvent handleBlackListEvent(BlackListEvent event) {
    // notify appropriate parties via notificationAddress and
    // then publish a ListUpdateEvent...
}
```

[异步侦听器](#context-functionality-events-async)不支持此功能。

这将通过上述方法处理每个`BlackListEvent`并发布一个新的`ListUpdateEvent`，如果需要发布多个事件，则可以返回事件 `集合`。

<a id="context-functionality-events-async"></a>

##### [](#context-functionality-events-async)异步的监听器

如果希望特定侦听器异步处理事件，则可以重用[常规 `@Async`支持](https://github.com/DocsHome/spring-docs/blob/master/pages/integration/integration.md#scheduling-annotation-support-async)。 以下示例显示了如何执行此操作：

```java
@EventListener
@Async
public void processBlackListEvent(BlackListEvent event) {
    // BlackListEvent is processed in a separate thread
}
```

使用异步事件时请注意以下限制:

* 如果事件侦听器抛出`Exception`，则不会将其传播给调用者。有关更多详细信息，请参阅 `AsyncUncaughtExceptionHandler`。

* 此类事件监听器无法发送回复。 如果您需要作为处理结果发送另一个事件，请注入[`ApplicationEventPublisher`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/aop/interceptor/AsyncUncaughtExceptionHandler.html)以手动发送事件。

<a id="context-functionality-events-order"></a>


##### [](#context-functionality-events-order)监听器的排序

如果需要在另一个侦听器之前调用一个侦听器，则可以将`@Order`注解添加到方法声明中，如以下示例所示：

```java
@EventListener
@Order(42)
public void processBlackListEvent(BlackListEvent event) {
    // notify appropriate parties via notificationAddress...
}
```

<a id="context-functionality-events-generics"></a>

##### [](#context-functionality-events-generics)泛型的事件

您还可以使用泛型来进一步定义事件的结构。 考虑使用`EntityCreatedEvent<T>`，其中`T`是创建的实际实体的类型。 例如，您可以创建以下侦听器定义以仅接收`Person`的`EntityCreatedEvent`：

```java
@EventListener
public void onPersonCreated(EntityCreatedEvent<Person> event) {
    ...
}
```

由于泛型擦除，只有此事件符合事件监听器所过滤的通用参数条件那么才会触发相应的处理事件(有点类似于`class PersonCreatedEvent extends EntityCreatedEvent<Person> { … }`）

在某些情况下，如果所有事件遵循相同的结构(如上述事件的情况），这可能变得相当乏味。在这种情况下，开发者可以实现`ResolvableTypeProvider`来引导框架超出所提供的运行时环境范围。

```java
public class EntityCreatedEvent<T> extends ApplicationEvent implements ResolvableTypeProvider {

    public EntityCreatedEvent(T entity) {
        super(entity);
    }

    @Override
    public ResolvableType getResolvableType() {
        return ResolvableType.forClassWithGenerics(getClass(), ResolvableType.forInstance(getSource()));
    }
}
```

这不仅适用于`ApplicationEvent` ，也适用于作为事件发送的任意对象。

<a id="context-functionality-resources"></a>

#### [](#context-functionality-resources)1.15.3.通过便捷的方式访问底层资源

为了最佳地使用和理解应用程序上下文，您应该熟悉Spring的资源抽象，如参考资料中所述[资源](#resources)。

应用程序上下文是`ResourceLoader`，可用于加载`Resource`对象。 `Resource`本质上是JDK`java.net.URL`类的功能更丰富的版本。实际上Resource的实现类中大多含有`java.net.URL`的实例。 `Resource`几乎能从任何地方透明的获取底层资源，可以是classpath类路径、文件系统、标准的URL资源及变种URL资源。如果资源定位字串是简单的路径， 没有任何特殊前缀，就适合于实际应用上下文类型。

可以配置一个bean部署到应用上下文中，用以实现特殊的回调接口，`ResourceLoaderAware`它会在初始化期间自动回调。应用程序上下文本身作为`ResourceLoader`传入。可以公开`Resource`的type属性，这样就可以访问静态资源 静态资源可以像其他properties那样被注入`Resource`。可以使用简单的字串路径指定资源，这需要依赖于特殊的JavaBean `PropertyEditor`（由上下文自动注册），当bean部署时候它将转换资源中的字串为实际的资源对象。

提供给`ApplicationContext`构造函数的一个或多个位置路径实际上是资源字符串，并且以简单形式对特定上下文实现进行适当处理。 `ClassPathXmlApplicationContext`将一个简单的定位路径视为类路径位置。开发者还可以使用带有特殊前缀的定位路径，这样就可以强制从classpath或者URL定义加载路径， 而不用考虑实际的上下文类型。

<a id="context-create"></a>

#### [](#context-create)1.15.4. 快速对Web应用的ApplicationContext实例化

开发者可以通过使用`ContextLoader`来声明性地创建`ApplicationContext`实例，当然也可以通过使用`ApplicationContext`的实现来编程实现`ApplicationContext`。

您可以使用`ContextLoaderListener`注册`ApplicationContext` ，如以下示例所示：

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/daoContext.xml /WEB-INF/applicationContext.xml</param-value>
</context-param>

<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

侦听器检查`contextConfigLocation`参数。 如果参数不存在，则侦听器将`/WEB-INF/applicationContext.xml`用作默认值。 当参数确实存在时，侦听器使用预定义的分隔符（逗号，分号和空格）分隔String，并将值用作搜索应用程序上下文的位置。 还支持Ant样式的路径模式。 示例是`/WEB-INF/*Context.xml`（对于所有名称以`Context.xml`结尾且位于`WEB-INF` 目录中的文件）和`/WEB-INF/**/*Context.xml`（对于所有这样的文件） `WEB-INF`的任何子目录中的文件。

<a id="context-deploy-rar"></a>

#### [](#context-deploy-rar)1.15.5. 使用Java EE RAR文件部署Spring的`ApplicationContext`

可以将Spring `ApplicationContext`部署为RAR文件，将上下文及其所有必需的bean类和库JAR封装在Java EE RAR部署单元中，这相当于引导一个独立的`ApplicationContext`。 只是托管在Java EE环境中，能够访问Java EE服务器设施。在部署无头WAR文件(实际上，没有任何HTTP入口点，仅用于在Java EE环境中引导Spring `ApplicationContext`的WAR文件）的情况下RAR部署是更自然的替代方案

RAR部署非常适合不需要HTTP入口点但仅由消息端点和调度作业组成的应用程序上下文。在这种情况下，Bean可以使用应用程序服务器资源， 例如JTA事务管理器和JNDI绑定的JDBC `DataSource`和JMS `ConnectionFactory`实例，并且还可以通过Spring的标准事务管理和JNDI和JMX支持设施向平台的JMX服务器注册。 应用程序组件还可以通过Spring的`TaskExecutor`抽象实现与应用程序服务器的JCA `WorkManager` 交互

有关RAR部署中涉及的配置详细信息，请参阅 [`SpringContextResourceAdapter`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/jca/context/SpringContextResourceAdapter.html)类的javadoc

对于将Spring ApplicationContext简单部署为Java EE RAR文件：

1.  将所有应用程序类打包到一个RAR文件（这是一个具有不同文件扩展名的标准JAR文件）。将所有必需的库JAR添加到RAR存档的根目录中。 。添加`META-INF/ra.xml`部署描述符 （如[`SpringContextResourceAdapter`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/jca/context/SpringContextResourceAdapter.html)的javadoc所示） 和相应的Spring XML bean定义文件（通常是 META-INF/applicationContext.xml）。

2.  将生成的RAR文件放入应用程序服务器的部署目录中。


这种RAR部署单元通常是独立的。它们不会将组件暴露给外部世界，甚至不会暴露给同一应用程序的其他模块。 与基于RAR的`ApplicationContext`的交互通常通过与其他模块共享的JMS目标进行。 例如，基于RAR的`ApplicationContext`还可以调度一些作业或对文件系统（等等）中的新文件作出反应。 如果它需要允许来自外部的同步访问，它可以（例如）导出RMI端点，这可以由同一台机器上的其他应用程序模块使用。

<a id="beans-beanfactory"></a>

### [](#beans-beanfactory)1.16.`BeanFactory`

`BeanFactory` API为Spring的IoC功能提供了基础。 它的特定契约主要用于与Spring的其他部分和相关的第三方框架集成其`DefaultListableBeanFactory`实现是更高级别`GenericApplicationContext`容器中的密钥委托。

`BeanFactory`和相关接口（例如`BeanFactoryAware`, `InitializingBean`，`DisposableBean`）是其他框架组件的重要集成点。 通过不需要任何注解或甚至反射，它们允许容器与其组件之间的非常有效的交互。 应用程序级bean可以使用相同的回调接口，但通常更喜欢通过注解或通过编程配置进行声明性依赖注入。

请注意，核心`BeanFactory` API级别及其 `DefaultListableBeanFactory`实现不会对配置格式或要使用的任何组件注解做出假设。 所有这些风格都通过扩展（例如`XmlBeanDefinitionReader`和`AutowiredAnnotationBeanPostProcessor`）进行，并作为核心元数据表示在共享`BeanDefinition`对象上运行。 这是使Spring的容器如此灵活和可扩展的本质。

<a id="context-introduction-ctx-vs-beanfactory"></a>

#### [](#context-introduction-ctx-vs-beanfactory)1.16.1. 选择`BeanFactory`还是`ApplicationContext`?

本节介绍`BeanFactory`和`ApplicationContext`容器级别之间的差异以及影响。

您应该使用`ApplicationContext`，除非您有充分的理由不这样做，使用`GenericApplicationContext`及其子类`AnnotationConfigApplicationContext`作为自定义引导的常见实现。 这些是Spring用于所有常见目的的核心容器的主要入口点：加载配置文件，触发类路径扫描，以编程方式注册bean定义和带注解的类，以及（从5.0开始）注册功能bean定义。

因为`ApplicationContext`包括`BeanFactory`的所有功能，和`BeanFactory`相比更值得推荐，除了一些特定的场景，例如在资源受限的设备上运行的内嵌的应用。 在`ApplicationContext`（例如`GenericApplicationContext`实现）中，按照约定（即通过bean名称或bean类型 - 特别是后处理器）检测到几种bean， 而普通的`DefaultListableBeanFactory`对任何特殊bean都是不可知的。

对于许多扩展容器功能，例如注解处理和AOP代理， [`BeanPostProcessor`扩展点是必不可少的。如果仅使用普通的`DefaultListableBeanFactory`，则默认情况下不会检测到并激活此类后置处理器。 这种情况可能令人困惑，因为您的bean配置实际上没有任何问题。 相反，在这种情况下，容器需要至少得多一些额外的处理。](#beans-factory-extension-bpp)

下表列出了`BeanFactory`和`ApplicationContext`接口和实现提供的功能。

Table 9.特性矩阵  

| Feature                              | `BeanFactory` | `ApplicationContext` |
| ------------------------------------ | ------------- | -------------------- |
| Bean Bean实例化/装配                 | Yes           | Yes                  |
| 集成的生命周期管理                   | No            | Yes                  |
| 自动注册 `BeanPostProcessor`         | No            | Yes                  |
| 自动注册 `BeanFactoryPostProcessor`  | No            | Yes                  |
| 便利的 `MessageSource` 访问 (国际化) | No            | Yes                  |
| 内置`ApplicationEvent` 发布机制      | No            | Yes                  |


要使用 `DefaultListableBeanFactory`显式注册bean的后置处理器，您需要以编程方式调用 `addBeanPostProcessor`，如以下示例所示：

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
// populate the factory with bean definitions

// now register any needed BeanPostProcessor instances
factory.addBeanPostProcessor(new AutowiredAnnotationBeanPostProcessor());
factory.addBeanPostProcessor(new MyBeanPostProcessor());

// now start using the factory
```

要将`BeanFactoryPostProcessor` 应用于普通的`DefaultListableBeanFactory`，需要调用其`postProcessBeanFactory`方法，如以下示例所示：

```java
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(new FileSystemResource("beans.xml"));

// bring in some property values from a Properties file
PropertyPlaceholderConfigurer cfg = new PropertyPlaceholderConfigurer();
cfg.setLocation(new FileSystemResource("jdbc.properties"));

// now actually do the replacement
cfg.postProcessBeanFactory(factory);
```

在这两种情况下，显示注册步骤都不方便，这就是为什么各种`ApplicationContext`变体优先于Spring支持的应用程序中的普通`DefaultListableBeanFactory`， 尤其是在典型企业设置中依赖`BeanFactoryPostProcessor` 和 `BeanPostProcessor`实例来扩展容器功能时。

`AnnotationConfigApplicationContext`具有注册的所有常见注解后置处理器，并且可以通过配置注解（例如`@EnableTransactionManagement`）在封面下引入其他处理器。 在Spring的基于注解的配置模型的抽象级别，bean的后置处理器的概念变成仅仅是内部容器细节。
