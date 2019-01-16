<a id="aop"></a>

[](#aop)5\. 使用Spring面向切面编程
--------------------------

面向切面编程(Aspect-oriented Programming 简称AOPAOP) ,是相对面向对象编程(Object-oriented Programming 简称OOP)的框架,作为OOP的一种功能补充. OOP主要的模块单元是类(class)。而AOP则是切面（aspect）。切面会将诸如事务管理这样跨越多个类型和对象的关注点模块化（在AOP的语义中，这类关注点被称为横切关注点（crosscutting））。

AOP是Spring框架重要的组件，虽然Spring IoC容器没有依赖AOP，因此Spring不会强迫开发者使用AOP。但AOP提供了非常棒的功能，用做对Spring IoC的补充。

Spring 2.0+ AOP

Spring 2.0 引入了一种更简单、更强大的方式用来自定义切面，开发者可以选择使用基于模式 [schema-based approach](#aop-schema) 的方式或使用[@AspectJ注解风格](#aop-ataspectj)方式来定义。这两种方式都完全支持通知（Advice）类型和AspectJ的切点语义，虽然实际上仍然是使用Spring AOP织入（weaving）的。

本章主要讨论Spring 2.0+ 框架对基于模式和基于@AspectJ的AOP支持。[下一章](#aop-api)，将讨论底层的AOP支持，如Spring 1.2应用程序中常见的那样。 The lower-level AOP support, as commonly exposed in Spring 1.2 applications, is discussed in .

AOP在Spring Framework中用于:

*   提供声明式企业服务，特别是用于替代EJB的声明式服务。最重要的服务是[声明式事务管理](#transaction-declarative)（declarative transaction management），这个服务建立在Spring的抽象事务管理（transaction abstraction）之上。

*   允许开发者实现自定义切面，使用AOP来完善OOP的功能。


如果只打算使用通用的声明式服务或者已有的声明式中间件服务，例如缓冲池（pooling）那么可以不直接使用AOP，也可以忽略本章大部分内容。

<a id="aop-introduction-defn"></a>

### [](#aop-introduction-defn)5.1. AOP 概念

让我们从定义一些核心AOP概念和术语开始。 这些术语不是特定于Spring的。 不幸的是，AOP术语不是特别直观。 但是，如果Spring使用自己的术语，那将更加令人困惑。

*   切面（Aspect）: 指关注点模块化，这个关注点可能会横切多个对象。事务管理是企业级Java应用中有关横切关注点的例子。 在Spring AOP中，切面可以使用通用类基于模式的方式（[schema-based approach](#aop-schema)）或者在普通类中以`@Aspect`注解（[@AspectJ 注解方式](#aop-ataspectj)）来实现。

*   连接点（Join point）: 在程序执行过程中某个特定的点，例如某个方法调用的时间点或者处理异常的时间点。在Spring AOP中，一个连接点总是代表一个方法的执行。

*   通知（Advice）: 在切面的某个特定的连接点上执行的动作。通知有多种类型，包括“around”, “before” and “after”等等。通知的类型将在后面的章节进行讨论。 许多AOP框架，包括Spring在内，都是以拦截器做通知模型的，并维护着一个以连接点为中心的拦截器链。

*   切点（Pointcut）: 匹配连接点的断言。通知和切点表达式相关联，并在满足这个切点的连接点上运行（例如，当执行某个特定名称的方法时）。切点表达式如何和连接点匹配是AOP的核心：Spring默认使用AspectJ切点语义。

*   引入（Introduction）: 声明额外的方法或者某个类型的字段。Spring允许引入新的接口（以及一个对应的实现）到任何被通知的对象上。例如，可以使用引入来使bean实现 `IsModified`接口， 以便简化缓存机制（在AspectJ社区，引入也被称为内部类型声明（inter））。

*   目标对象（Target object）: 被一个或者多个切面所通知的对象。也被称作被通知（advised）对象。既然Spring AOP是通过运行时代理实现的，那么这个对象永远是一个被代理（proxied）的对象。

*   AOP代理（AOP proxy）:AOP框架创建的对象，用来实现切面契约（aspect contract）（包括通知方法执行等功能）。在Spring中，AOP代理可以是JDK动态代理或CGLIB代理。

*   织入（Weaving）: 把切面连接到其它的应用程序类型或者对象上，并创建一个被被通知的对象的过程。这个过程可以在编译时（例如使用AspectJ编译器）、类加载时或运行时中完成。 Spring和其他纯Java AOP框架一样，是在运行时完成织入的。


Spring AOP包含以下类型的通知:

*   前置通知（Before advice）: 在连接点之前运行但无法阻止执行流程进入连接点的通知（除非它引发异常）。

*   后置返回通知（After returning advice）:在连接点正常完成后执行的通知（例如，当方法没有抛出任何异常并正常返回时）。

*   后置异常通知（After throwing advice）: 在方法抛出异常退出时执行的通知。

*   后置通知（总会执行）（After (finally) advice）: 当连接点退出的时候执行的通知（无论是正常返回还是异常退出）。

*   环绕通知（Around Advice）:环绕连接点的通知，例如方法调用。这是最强大的一种通知类型，。环绕通知可以在方法调用前后完成自定义的行为。它可以选择是否继续执行连接点或直接返回自定义的返回值又或抛出异常将执行结束。


环绕通知是最常用的一种通知类型。与AspectJ一样，在选择Spring提供的通知类型时，团队推荐开发者尽量使用简单的通知类型来实现需要的功能。例如， 如果只是需要使用方法的返回值来作缓存更新，虽然使用环绕通知也能完成同样的事情，但是仍然推荐使用后置返回通知来代替。使用最合适的通知类型可以让编程模型变得简单， 还能避免很多潜在的错误。例如，开发者无需调用于环绕通知（用`JoinPoint`）的 `proceed()`方法，也就不会产生调用的问题。

在Spring 2.0中，所有通知参数都是静态类型的，因此您可以使用相应类型的通知参数（例如，方法执行的返回值的类型）而不是`Object`数组。

切点和连接点匹配是AOP的关键概念，这个概念让AOP不同于其它仅仅提供拦截功能的旧技术。切入点使得通知可以独立于面向对象的层次结构进行定向。 例如，您可以将一个提供声明式事务管理的通知应用于跨多个对象（例如服务层中的所有业务操作）的一组方法。

<a id="aop-introduction-spring-defn"></a>

### [](#aop-introduction-spring-defn)5.2. Spring AOP的功能和目标

Spring AOP是用纯Java实现的。 不需要特殊的编译过程。 Spring AOP不需要控制类加载器层次结构，因此适合在servlet容器或应用程序服务器中使用。

Spring目前仅支持方法调用的方式作为连接点（在Spring bean上通知方法的执行）。虽然可以在不影响到Spring AOP核心API的情况下加入对成员变量拦截器支持， 但Spring并没有实现成员变量拦截器。如果需要通知对成员变量的访问和更新连接点，可以考虑其它语言，例如AspectJ。

Spring实现AOP的方法与其他的框架不同，Spring并不是要尝试提供最完整的AOP实现（尽管Spring AOP有这个能力），相反地，它其实侧重于提供一种AOP与Spring IoC容器的整合的实现，用于帮助解决在企业级开发中的常见问题。

因此，例如，Spring Framework的AOP功能通常与Spring IoC容器一起使用。通过使用普通bean定义语法来配置切面（尽管Spring提供了强大的“自动代理”功能）。 这是与其他AOP实现的重要区别。 使用Spring AOP无法轻松或高效地完成某些操作，例如建议非常细粒度的对象（通常是域对象）。 在这种情况下，AspectJ是最佳选择。 但是，我们的经验是，Spring AOP为适合AOP的企业Java应用程序中的大多数问题提供了出色的解决方案。

Spring AOP从来没有打算通过提供一种全面的AOP解决方案用于取代AspectJ，我们相信，基于代理的框架（如Spring AOP）和完整的框架（如AspectJ）都很有价值，而且它们是互补的，而不是竞争。 Spring将Spring AOP和IoC与AspectJ无缝集成，使得所有的AOP功能完全融入基于Spring的应用体系。这样的集成不会影响Spring AOP API或者AOP Alliance API。 Spring AOP仍然向后兼容。 有关Spring AOP API的讨论，请参阅[以下章节](#aop-api)。

Spring框架的一个核心原则是非侵入性。这意味着开发者无需在自身的业务/域模型上被迫引入框架特定的类和接口。然而，有些时候，Spring框架可以让开发者选择引入Spring框架特定的依赖关系到业务代码。 给予这种选择的理由是因为在某些情况下它可能是更易读或易于编写某些特定功能。Spring框架（几乎）总能给出这样的选择，开发者可以自由地做出明智的决定，选择最适合的特定用例或场景。

与本章相关的一个选择是选择哪种AOP框架（以及哪种AOP样式）。您可以选择AspectJ，Spring AOP或两者。也可以选择@AspectJ注解式的方法或Spring的XML配置方式。 事实上，本章以介绍@AspectJ方式为先不应该被视为Spring团队倾向于@AspectJ的方式胜过Spring的XML配置方式。

请参阅[选择要使用的AOP声明样式](#aop-choosing)，以更全面地讨论每种样式的“为什么和如何进行”。

<a id="aop-introduction-proxies"></a>

### [](#aop-introduction-proxies)5.3. AOP 代理

Spring默认使用标准的JDK动态代理来作为AOP的代理。这样任何接口（或者接口的set）都可以被代理。

Spring也支持使用CGLIB代理。对于需要代理类而不是代理接口的时候CGLIB代理是很有必要的。如果业务对象并没有实现接口，默认就会使用CGLIB代理 。此外，面向接口编程也是最佳实践，业务对象通常都会实现一个或多个接口。此外，还可以[强制的使用CGLIB代理](#aop-proxying)， 在那些（希望是罕见的）需要通知没有在接口中声明的方法时，或者当需要传递一个代理对象作为一种具体类型到方法的情况下。

掌握Spring AOP是基于代理的这一事实非常重要。 请参阅 [AOP代理](#aop-understanding-aop-proxies)，以全面了解此实现细节的实际含义。.

<a id="aop-ataspectj"></a>

### [](#aop-ataspectj)5.4.@AspectJ注解支持

@AspectJ会将切面声明为常规Java类的注解类型。 [AspectJ项目](https://www.eclipse.org/aspectj)引入了@AspectJ风格，并作为AspectJ 5发行版的一部分。Spring使用的注解类似于AspectJ 5， 使用AspectJ提供的库用来解析和匹配切点。AOP运行时仍然是纯粹的Spring AOP，并不依赖AspectJ编译器或编织器。

使用AspectJ编译器和织入并允许使用全部基于AspectJ语言，并在[Using AspectJ with Spring Applications](#aop-using-aspectj)进行了讨论。

<a id="aop-aspectj-support"></a>

#### [](#aop-aspectj-support)5.4.1. 允许@AspectJ的支持

要在Spring配置中使用@AspectJ切面，需要启用Spring支持，用于根据@AspectJ切面配置Spring AOP，并根据这些切面自动代理bean（事先判断是否在通知的范围内）。 通过自动代理的意思是：如果Spring确定一个bean是由一个或多个切面处理的，将据此为bean自动生成代理bean，并以拦截方法调用并确保需要执行的通知。

可以使用XML或Java配置的方式启用@AspectJ支持。不管哪一种方式，您还需要确保AspectJ的`aspectjweaver.jar`库位于应用程序的类路径中（版本1.8或更高版本）。此库可在AspectJ分发的`lib` 目录中或Maven Central存储库中找到。

<a id="aop-enable-aspectj-java"></a>

##### [](#aop-enable-aspectj-java)使用Java配置启用@AspectJ支持

要使用Java `@Configuration`启用@AspectJ支持，请添加 `@EnableAspectJAutoProxy`注解，如以下示例所示：

    @Configuration
    @EnableAspectJAutoProxy
    public class AppConfig {

    }

<a id="aop-enable-aspectj-xml"></a>

##### [](#aop-enable-aspectj-xml)使用XML配置启用@AspectJ支持

要使用基于XML的配置启用@AspectJ支持，请使用 `aop:aspectj-autoproxy`元素，如以下示例所示：

    <aop:aspectj-autoproxy/>

这假设您使用[XML Schema-based configuration](#xsd-schemas)中描述的schema支持。 有关如何在aop命名空间中导入标记，请参阅 [在`aop`aop命名空间中导入标记](#xsd-schemas-aop)。

<a id="aop-at-aspectj"></a>

#### [](#aop-at-aspectj)5.4.2. 声明切面

启用了@AspectJ支持后，在应用程序上下文中定义的任意bean（有`@Aspect`注解）的类都将被Spring自动检测，并用于配置Spring AOP。 接下来的两个示例显示了非常有用的方面所需的最小定义。

这两个示例中的第一个示例在应用程序上下文中显示了一个常规bean定义，该定义指向具有`@Aspect`注解的bean类：

    <bean id="myAspect" class="org.xyz.NotVeryUsefulAspect">
        <!-- configure properties of the aspect here -->
    </bean>

这两个示例中的第二个显示了`NotVeryUsefulAspect`类定义，该定义使用 `org.aspectj.lang.annotation.Aspect` 注解进行注解;

    package org.xyz;
    import org.aspectj.lang.annotation.Aspect;

    @Aspect
    public class NotVeryUsefulAspect {

    }

切面（使用 `@Aspect`的类）可以拥有方法和属性，与其他类并无不同。也可以包括切点、通知和内置类型（即引入）声明。

通过组件扫描自动检测切面

您可以在Spring XML配置中将切面类注册为常规bean，或者通过类路径扫描自动检测它们 - 与任何其他Spring管理的bean相同。然而注意到`@Aspect`注解对于类的自动探测是不够的， 为此，需要单独添加`@Component` ，注解（或自定义注解声明，用作Spring组件扫描器的规则之一）。

是否可以作为其他切面的切面通知?

在Spring AOP中，不可能将切面本身被作为其他切面的目标。类上的`@Aspect`注解表明他是一个切面并且排除在自动代理的范围之外。

<a id="aop-pointcuts"></a>

#### [](#aop-pointcuts)5.4.3. 声明切点

切点决定了匹配的连接点，从而使我们能够控制通知何时执行。Spring AOP只支持使用Spring bean的方法执行连接点，所以可以将切点看出是匹配Spring bean上方法的执行。 切点的声明包含两个部分：包含名称和任意参数的签名，以及明确需要匹配的方式执行的切点表达式。在@AspectJ注解方式的AOP中，一个切点的签名由常规方法定义来提供， 并且切点表达式使用 `@Pointcut`注解指定（方法作为切点签名必须有类型为`void`的返回）。

使用例子有助于更好地区分切点签名和切点表达式之间的关系。以下示例定义名为named `anyOldTransfer`的切点，该切点与名为`transfer`的任何方法的执行相匹配：

    @Pointcut("execution(* transfer(..))")// the pointcut expression
    private void anyOldTransfer() {}// the pointcut signature

切点表达式由`@Pointcut` 注解的值是常规的AspectJ 5切点表达式。关于AspectJ切点语言的描述，见 [AspectJ的编程指南](https://www.eclipse.org/aspectj/doc/released/progguide/index.html) （作为扩展， 请参考[AspectJ 5 Developer’s Notebook](https://www.eclipse.org/aspectj/doc/released/adk15notebook/index.html)）或者Colyer著的关于AspectJ的书籍。 例如，_Eclipse AspectJ_，或者参看Ramnivas Laddad的_AspectJ in Action_。

<a id="aop-pointcuts-designators"></a>

##### [](#aop-pointcuts-designators)支持切点标识符

Spring AOP支持使用以下AspectJ切点标识符(PCD),用于切点表达式：

*   `execution`: 用于匹配方法执行连接点。 这是使用Spring AOP时使用的主要切点标识符。

*   `within`: 限制匹配特定类型中的连接点（在使用Spring AOP时，只需执行在匹配类型中声明的方法）。

*   `this`: 在bean引用（Spring AOP代理）是给定类型的实例的情况下，限制匹配连接点（使用Spring AOP时方法的执行）。

*   `target`: 限制匹配到连接点（使用Spring AOP时方法的执行），其中目标对象（正在代理的应用程序对象）是给定类型的实例。

*   `args`: 限制与连接点的匹配（使用Spring AOP时方法的执行），其中变量是给定类型的实例。 AOP) where the arguments are instances of the given types.

*   `@target`: 限制与连接点的匹配（使用Spring AOP时方法的执行），其中执行对象的类具有给定类型的注解。

*   `@args`: 限制匹配连接点（使用Spring AOP时方法的执行），其中传递的实际参数的运行时类型具有给定类型的注解。

*   `@within`: 限制与具有给定注解的类型中的连接点匹配（使用Spring AOP时在具有给定注解的类型中声明的方法的执行）。

*   `@annotation`:限制匹配连接点（在Spring AOP中执行的方法具有给定的注解）。


其他切点类型

Spring并没有完全地支持AspectJ切点语言声明的切点标识符，包括 `call`, `get`, `set`, `preinitialization`, `staticinitialization`, `initialization`, `handler`, `adviceexecution`, `withincode`, `cflow`, `cflowbelow`, `if`, `@this`, 和 `@withincode`。在由Spring AOP解释的切点表达式中，使用这些切点标识符将导致`IllegalArgumentException`异常。

Spring AOP支持的切点标识符可以在将来的版本中扩展，以支持更多的AspectJ切点标识符。

因为Spring AOP限制了只匹配方法的连接点执行，所以上面的切点标识符的讨论比在AspectJ编程指南中找到的定义要窄。另外，AspectJ本身具有基于类型的语义， 并且在执行连接点上，`this`和`target`都指向同一个对象-即执行方法的对象。Spring AOP是一个基于代理的系统，区分代理对象本身（绑定到`this`）和代理（绑定到`target`）后的目标对象。

由于Spring AOP框架是基于代理的特性，定义的protected方法将不会被处理，不管是JDK的代理（做不到）还是CGLIB代理（有技术可以实现但是不建议）。 因此，任何给定的切点将只能与public方法匹配。

请注意，切点定义通常与任何截获的方法匹配。 如果切点严格意义上是公开的，即使在通过代理进行潜在非公共交互的CGLIB代理方案中，也需要相应地定义切点。

如果需要拦截包括protected和private方法甚至是构造函数，请考虑使用基于Spring驱动的[本地AspectJ编织](#aop-aj-ltw)而不是Spring的基于代理的AOP框架。 这构成了不同特性的AOP使用模式，所以在做出决定之前一定要先熟悉一下编织。

Spring AOP支持更多的PCD命名`bean`。PCD允许将连接点的匹配限制为特定的Spring bean或一系列Spring bean。 `bean` PCD具有以下形式：:

    bean(idOrNameOfBean)

`idOrNameOfBean`标识可以是任意符合Spring bean的名字， 提供了使用`*`字符的有限通配符支持，因此，如果为Spring bean建立了一些命名约定，则可以编写`bean` PCD表达式来选择它们。 与其他切点标识符的情况一样，PCD bean可以是`&&` (and), `||` (or), and `!`。

`bean` PCD仅在Spring AOP中受支持，而在本机AspectJ编织中不受支持。 它是AspectJ定义的标准PCD的Spring特定扩展，因此不适用于 `@Aspect` 模型中声明的切面。

PCD `bean`运行在实例级别上（基于Spring bean名称概念构建），而不是仅在类型级别（这是基于编织的AOP所限制的）。 基于实例的切点标识符是Spring基于代理的AOP框架的特殊功能，它与Spring bean工厂紧密集成，通过名称识别特定的bean是自然而直接的。

<a id="aop-pointcuts-combining"></a>

##### [](#aop-pointcuts-combining)合并切点表达式

您可以使用 `&&,` `||` 和 `!`等符号进行合并操作。也可以通过名字来指向切点表达式。 以下示例显示了三个切入点表达式：

    @Pointcut("execution(public * *(..))")
    private void anyPublicOperation() {} (1)

    @Pointcut("within(com.xyz.someapp.trading..*)")
    private void inTrading() {} (2)

    @Pointcut("anyPublicOperation() && inTrading()")
    private void tradingOperation() {} (3)

**(1)。**`anyPublicOperation`：如果方法执行连接点表示任何公共方法的执行，则匹配

**(2)。**`inTrading` ：如果方法执行在trading中，则匹配.

**(3)。**`tradingOperation` ：如果方法执行表示trading中的任何公共方法，则匹配。

如上所示，用更小的命名组件构建更复杂的切入点表达式是最佳实践。当按名称引用切点时，将应用普通的Java可见性规则（可以看到相同类型的私有切点，层次结构中受保护的切点，任何位置的公共切点等）。可见性并不影响切点匹配。

<a id="aop-common-pointcuts"></a>

##### [](#aop-common-pointcuts)共享通用的切点定义

在处理企业应用程序时，通常需要从几个切面来引用应用程序的模块和特定的操作集。建议定义一个“SystemArchitecture” 切面，以此为目的捕获通用的切点表达式。这样的切面通常类似于以下示例：

    package com.xyz.someapp;

    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.Pointcut;

    @Aspect
    public class SystemArchitecture {

        /**
         * A join point is in the web layer if the method is defined
         * in a type in the com.xyz.someapp.web package or any sub-package
         * under that.
         */
        @Pointcut("within(com.xyz.someapp.web..*)")
        public void inWebLayer() {}

        /**
         * A join point is in the service layer if the method is defined
         * in a type in the com.xyz.someapp.service package or any sub-package
         * under that.
         */
        @Pointcut("within(com.xyz.someapp.service..*)")
        public void inServiceLayer() {}

        /**
         * A join point is in the data access layer if the method is defined
         * in a type in the com.xyz.someapp.dao package or any sub-package
         * under that.
         */
        @Pointcut("within(com.xyz.someapp.dao..*)")
        public void inDataAccessLayer() {}

        /**
         * A business service is the execution of any method defined on a service
         * interface. This definition assumes that interfaces are placed in the
         * "service" package, and that implementation types are in sub-packages.
         *
         * If you group service interfaces by functional area (for example,
         * in packages com.xyz.someapp.abc.service and com.xyz.someapp.def.service) then
         * the pointcut expression "execution(* com.xyz.someapp..service.*.*(..))"
         * could be used instead.
         *
         * Alternatively, you can write the expression using the 'bean'
         * PCD, like so "bean(*Service)". (This assumes that you have
         * named your Spring service beans in a consistent fashion.)
         */
        @Pointcut("execution(* com.xyz.someapp..service.*.*(..))")
        public void businessService() {}

        /**
         * A data access operation is the execution of any method defined on a
         * dao interface. This definition assumes that interfaces are placed in the
         * "dao" package, and that implementation types are in sub-packages.
         */
        @Pointcut("execution(* com.xyz.someapp.dao.*.*(..))")
        public void dataAccessOperation() {}

    }

像这样定义的切点可以用在任何需要切点表达式的地方， 例如，要使服务层具有事务性，您可以编写以下内容：

    <aop:config>
        <aop:advisor
            pointcut="com.xyz.someapp.SystemArchitecture.businessService()"
            advice-ref="tx-advice"/>
    </aop:config>

    <tx:advice id="tx-advice">
        <tx:attributes>
            <tx:method name="*" propagation="REQUIRED"/>
        </tx:attributes>
    </tx:advice>

`<aop:config>` and `<aop:advisor>`元素在 [基于Schema的AOP 支持](#aop-schema)中进行了讨论。 [事务管理](data-access.html#transaction)中讨论了事务元素。

<a id="aop-pointcuts-examples"></a>

##### [](#aop-pointcuts-examples)Examples

Spring AOP用户可能最常使用`execution`切点标识符 ，执行表达式的格式为：

    execution(modifiers-pattern? ret-type-pattern declaring-type-pattern?name-pattern(param-pattern)
                throws-pattern?)

除返回类型模式（上面片段中的`ret-type-pattern` ）以外的所有部件、名称模式和参数模式都是可选的。返回类型模式确定要匹配的连接点的方法的返回类型必须是什么。 通常，可以使用`*`作为返回类型模式，它匹配任何返回类型。只有当方法返回给定类型时，完全限定的类型名称才会匹配。名称模式与方法名称匹配，可以将`*`通配符用作名称模式的全部或部分。 如果指定声明类型模式，则需要有后缀 `.`将其加入到名称模式组件中。参数模式稍微复杂一点。`()`匹配没有参数的方法。`(..)`匹配任意个数的参数（0个或多个）。 `(*)`匹配任何类型的单个参数。`(*,String)`匹配有两个参数而且第一个参数是任意类型，第二个必须是`String`的方法。有关更多信息，请参阅AspectJ编程指南的[语言语义](https://www.eclipse.org/aspectj/doc/released/progguide/semantics-pointcuts.html)部分。

以下示例显示了一些常见的切点表达式：

*   匹配任意公共方法的执行:

        execution(public * *(..))

*   匹配任意以`set`开始的方法:

        execution(* set*(..))

*   匹配定义了`AccountService`接口的任意方法:

        execution(* com.xyz.service.AccountService.*(..))

*   匹配定义在`service` 包中的任意方法:

        execution(* com.xyz.service.*.*(..))

*   匹配定义在service包和其子包中的任意方法:

        execution(* com.xyz.service..*.*(..))

*   匹配在service包中的任意连接点（只在Spring AOP中的方法执行）:

        within(com.xyz.service.*)

*   匹配在service包及其子包中的任意连接点（只在Spring AOP中的方法执行）:

        within(com.xyz.service..*)

*   匹配代理实现了`AccountService` 接口的任意连接点（只在Spring AOP中的方法执行）：

        this(com.xyz.service.AccountService)

    'this' 常常以捆绑的形式出现. 见后续的章节讨论如何在[声明通知](#aop-advice)中使用代理对象。

*   匹配当目标对象实现了`AccountService`接口的任意连接点（只在Spring AOP中的方法执行）:

        target(com.xyz.service.AccountService)

    'target' 常常以捆绑的形式出现. 见后续的章节讨论如何在[声明通知](#aop-advice)中使用目标对象。

*   匹配使用了单一的参数，并且参数在运行时被传递时可以序列化的任意连接点（只在Spring的AOP中的方法执行）。:

        args(java.io.Serializable)

    'args' 常常以捆绑的形式出现.见后续的章节讨论如何在[声明通知](#aop-advice)中使用方法参数。

    注意在这个例子中给定的切点不同于`execution(* *(java.io.Serializable))`. 如果在运行时传递的参数是可序列化的，则与execution匹配，如果方法签名声明单个参数类型可序列化，则与args匹配。

*   匹配当目标对象有`@Transactional`注解时的任意连接点（只在Spring AOP中的方法执行）。

        @target(org.springframework.transaction.annotation.Transactional)

    '@target' 也可以以捆绑的形式使用.见后续的章节讨论如何在[声明通知](#aop-advice)中使用注解对象。

*   匹配当目标对象的定义类型有`@Transactional`注解时的任意连接点（只在Spring的AOP中的方法执行）:

        @within(org.springframework.transaction.annotation.Transactional)

    '@within' 也可以以捆绑的形式使用.见后续的章节讨论如何在[声明通知](#aop-advice)中使用注解对象。

*   匹配当执行的方法有`@Transactional`注解的任意连接点（只在Spring AOP中的方法执行）:

        @annotation(org.springframework.transaction.annotation.Transactional)

    '@annotation' 也可以以捆绑的形式使用.见后续的章节讨论如何在[声明通知](#aop-advice)中使用注解对象。

*   匹配有单一的参数并且在运行时传入的参数类型有`@Classified`注解的任意连接点（只在Spring AOP中的方法执行）:

        @args(com.xyz.security.Classified)

    '@args' 也可以以捆绑的形式使用.见后续的章节讨论如何在[声明通知](#aop-advice)中使用注解对象。

*   匹配在名为`tradeService`的Spring bean上的任意连接点（只在Spring AOP中的方法执行）:

        bean(tradeService)

*   匹配以`Service`结尾的Spring bean上的任意连接点（只在Spring AOP中方法执行） :

        bean(*Service)


<a id="writing-good-pointcuts"></a>

##### [](#writing-good-pointcuts)编写好的切点

在编译过程中，AspectJ会尝试和优化匹配性能来处理切点。检查代码并确定每个连接点是否匹配（静态或动态）给定切点是一个代价高昂的过程。（动态匹配意味着无法从静态分析中完全确定匹配， 并且将在代码中放置测试，以确定在运行代码时是否存在实际匹配）。在第一次遇到切点声明时，AspectJ会将它重写为匹配过程的最佳形式。这是什么意思？基本上，切点是在DNF（析取范式）中重写的 ，切点的组成部分会被排序，以便先检查那些比较明确的组件。这意味着开发者不必担心各种切点标识符的性能，并且可以在切点声明中以任何顺序编写。

但是，AspectJ只能与被它指定的内容协同工作，并且为了获得最佳的匹配性能，开发者应该考虑它们试图实现的目标，并在定义中尽可能缩小匹配的搜索空间。 现有的标识符会自动选择下面三个中的一个 kinded, scoping, 和 contextual:

*   Kinded选择特定类型的连接点的标识符: `execution`, `get`, `set`, `call`, 和 `handler`.

*   Scoping选择一组连接点的匹配 （可能是许多种类）: `within` and `withincode`

*   Contextual基于上下文匹配 （或可选绑定）的标识符: `this`, `target`, and `@annotation`


一个写得很好的切入点应该至少包括前两种类型（kinded和scoping）。同时contextual标识符或许会被包括如果希望匹配基于连接点上下文或绑定在通知中使用的上下文。 只是提供kinded标识符或只提供contextual标识符器也能够工作，但是可能影响处理性能（时间和内存的使用），浪费了额外的处理和分析时间或空间。scoping标识符可以快速匹配并且使用AspectJ可以快速排除不会被处理的连接点组， 这也说明编写好的切点表达式是很重要的（因为没有明确指定时，它就会Loop Lookup循环匹配）。

<a id="aop-advice"></a>

#### [](#aop-advice)5.4.4. 声明通知

通知是与切点表达式相关联的概念，可以在切点匹配的方法之前、之后或之间执行。切点表达式可以是对命名切点的简单引用，也可以是即时声明的切点表达式。

<a id="aop-advice-before"></a>

##### [](#aop-advice-before)前置通知

您可以使用`@Before`注解在切面中的通知之前声明：

    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.Before;

    @Aspect
    public class BeforeExample {

        @Before("com.xyz.myapp.SystemArchitecture.dataAccessOperation()")
        public void doAccessCheck() {
            // ...
        }

    }

如果使用内置切点表达式，我们可以重写前面的示例，如下例所示：

    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.Before;

    @Aspect
    public class BeforeExample {

        @Before("execution(* com.xyz.myapp.dao.*.*(..))")
        public void doAccessCheck() {
            // ...
        }

    }

<a id="aop-advice-after-returning"></a>

##### [](#aop-advice-after-returning)后置返回通知

要想用后置返回通知可以在切面上添加`@AfterReturning`注解:

    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.AfterReturning;

    @Aspect
    public class AfterReturningExample {

        @AfterReturning("com.xyz.myapp.SystemArchitecture.dataAccessOperation()")
        public void doAccessCheck() {
            // ...
        }

    }

在同一切面中当然可以声明多个通知。在此只是为了迎合讨论的主题而只涉及单个通知。

有些时候需要在通知中获取实际的返回值。可以使用`@AfterReturning`，并指定returning字段如下:

    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.AfterReturning;

    @Aspect
    public class AfterReturningExample {

        @AfterReturning(
            pointcut="com.xyz.myapp.SystemArchitecture.dataAccessOperation()",
            returning="retVal")
        public void doAccessCheck(Object retVal) {
            // ...
        }

    }

在`returning`属性中使用的名字必须和通知方法中的参数名相关，方法执行返回时，返回值作为相应的参数值传递给advice方法。`returning`子句还限制只匹配那些返回指定类型的值的方法执行（在本例中为`Object`，它匹配任何返回值对象）。

请注意，当使用after-returning的通知时。不能返回不同的引用。

<a id="aop-advice-after-throwing"></a>

##### [](#aop-advice-after-throwing)后置异常通知

当方法执行并抛出异常时后置异常通知会被执行，需要使用`@AfterThrowing`注解来定义。如以下示例所示：

    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.AfterThrowing;

    @Aspect
    public class AfterThrowingExample {

        @AfterThrowing("com.xyz.myapp.SystemArchitecture.dataAccessOperation()")
        public void doRecoveryActions() {
            // ...
        }

    }

开发者常常希望当给定类型的异常被抛出时执行通知，并且也需要在通知中访问抛出的异常。使用`throwing`属性来限制匹配（如果需要，使用 `Throwable`作为异常类型），并将引发的异常绑定到通知参数。以下示例显示了如何执行此操作：

    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.AfterThrowing;

    @Aspect
    public class AfterThrowingExample {

        @AfterThrowing(
            pointcut="com.xyz.myapp.SystemArchitecture.dataAccessOperation()",
            throwing="ex")
        public void doRecoveryActions(DataAccessException ex) {
            // ...
        }

    }

`throwing`属性中使用的名字必须和通知方法中的参数名相关。当方法执行并抛出异常时，异常将会传递给通知方法作为相关的参数值。 抛出子句还限制与只引发指定类型的异常（在本例中为`DataAccessException`）的方法执行的匹配。

<a id="aop-advice-after-finally"></a>

##### [](#aop-advice-after-finally)后置通知(总会执行)

当匹配方法执行之后后置通知（总会执行）会被执行。这种情况使用`@After`注解来定义。后置通知必须被准备来处理正常或异常的返回条件。通常用于释放资源等等:

    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.After;

    @Aspect
    public class AfterFinallyExample {

        @After("com.xyz.myapp.SystemArchitecture.dataAccessOperation()")
        public void doReleaseLock() {
            // ...
        }

    }

<a id="aop-ataspectj-around-advice"></a>

##### [](#aop-ataspectj-around-advice)环绕通知

最后一种通知是环绕通知，环绕通知围绕方法执行。可以在方法执行之前和执行之后执行，并且定义何时做什么，甚至是否真正得到执行。如果需要在方法执行之前和之后以线程安全的方式 （例如启动和停止计时器） 共享状态， 则通常会使用环绕通知。总是建议使用最适合要求的通知（即可以用前置通知解决的就不要用环绕通知了）。

使用`@Around`注解来定义环绕通知，第一个参数必须是`ProceedingJoinPoint`类型的。在通知中调用 `ProceedingJoinPoint`中的 `proceed()`方法来引用执行的方法。`proceed`方法也可以被调用传递数组对象\- 数组的值将会被当作参数在方法执行时被使用。proceed方法也可以传入 `Object[]`。 数组中的值在进行时用作方法执行的参数。

在使用 `Object[]` 调用时 `proceed` 的行为与在AspectJ编译器编译的环绕通知进行的行为略有不同。对于使用传统AspectJ语言编写的通知， 传递给`proceed`的参数数必须与传递给环绕通知的参数数量（不是被连接点处理的参数的数目）匹配，并且传递的值将`proceed` 在给定的参数位置取代该值绑定到的实体的连接点的原始值（如果现在无法理解 ，请不要担心）。Spring处理的方式是简单的并且基于代理的，会生成更好的匹配语义。现在只需意识到这两种是有这么一点的不同的即可。有一种方法可以编写出100%兼容Spring AOP和AspectJ的匹配， [在后续的章节中](#aop-ataspectj-advice-params)将会讨论通知的参数。

以下示例显示如何使用around通知：

    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.Around;
    import org.aspectj.lang.ProceedingJoinPoint;

    @Aspect
    public class AroundExample {

        @Around("com.xyz.myapp.SystemArchitecture.businessService()")
        public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
            // start stopwatch
            Object retVal = pjp.proceed();
            // stop stopwatch
            return retVal;
        }

    }

环绕通知返回的值将会被调用的方法看到，例如，一个简单的缓存切面可以从缓存中返回一个值（如果有的话），如果没有则调用`proceed()`。 请注意，可以在around通知的主体内调用一次，多次或根本不调用。 所有这些都是合法的。

<a id="aop-ataspectj-advice-params"></a>

##### [](#aop-ataspectj-advice-params)通知的参数

Spring提供了全部类型的通知，这意味着需在通知签名中声明所需的参数（正如上面返回和异常的示例），而不是一直使用 `Object[]`数组。接着将会看到怎么声明参数以及上下文的值是如何在通知实体中被使用的。 首先，来看看如何编写一般的通知，找出编写通知的法子。

<a id="aop-ataspectj-advice-params-the-joinpoint"></a>

###### [](#aop-ataspectj-advice-params-the-joinpoint)访问当前的`连接点`

任何通知方法都可以声明一个类型为 `org.aspectj.lang.JoinPoint`的参数作为其第一个参数（注意，需要使用around advice来声明一个类型为`ProceedingJoinPoint`的第一个参数， 它是`JoinPoint`的一个子类。`JoinPoint`接口提供很多有用的方法：:

*   `getArgs()`: 返回方法参数.

*   `getThis()`: 返回代理对象.

*   `getTarget()`: 返回目标对象.

*   `getSignature()`:返回正在通知的方法的描述.

*   `toString()`: 打印方法被通知的有用描述.


See the [javadoc](https://www.eclipse.org/aspectj/doc/released/runtime-api/org/aspectj/lang/JoinPoint.html) for more detail.

<a id="aop-ataspectj-advice-params-passing"></a>

###### [](#aop-ataspectj-advice-params-passing)传递参数给通知

我们已经看到了如何绑定返回的值或异常值（在返回之后和抛出通知之后使用）。为了在通知代码段中使用参数值，可以使用绑定`args`的形式。如果在参数表达式中使用参数名代替类型名称， 则在调用通知时，要将相关的参数值当作参数传递。例如，假如在dao操作时将Account对象作为第一个参数传递给通知，并且需要在通知代码段内访问`Account`，可以这样写:

    @Before("com.xyz.myapp.SystemArchitecture.dataAccessOperation() && args(account,..)")
    public void validateAccount(Account account) {
        // ...
    }

切点表达式的`args(account,..)` 部分有两个目的。p它严格匹配了至少带一个参数的执行方法，并且传递给传递的参数是`Account`实例。 第二，它使得实际的`Account`对象通过`account`参数提供给通知。 parameter, and the argument passed to that parameter is an instance of . Second, it makes the actual object available to the advice through the parameter.

另一个方法写法就是先定义切点，然后，“provides”`Account`对象给匹配的连接点，有了连接点，那么引用连接点作为切点的通知就能获得Account对象的值。这看起来如下：

    @Pointcut("com.xyz.myapp.SystemArchitecture.dataAccessOperation() && args(account,..)")
    private void accountDataAccessOperation(Account account) {}

    @Before("accountDataAccessOperation(account)")
    public void validateAccount(Account account) {
        // ...
    }

有关更多详细信息，请参阅AspectJ编程指南。

代理对象( `this`)，目标对象 ( `target`)和注解 ( `@within`, `@target`, `@annotation`, and `@args`)都可以以类似的方式绑定。接下来的两个示例显示如何匹配带有`@Auditable`注解的注解方法的执行并获取audit代码代码:

首先是`@Auditable`注解的定义:

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    public @interface Auditable {
        AuditCode value();
    }

然后是匹配`@Auditable`方法通知的执行

    @Before("com.xyz.lib.Pointcuts.anyPublicMethod() && @annotation(auditable)")
    public void audit(Auditable auditable) {
        AuditCode code = auditable.value();
        // ...
    }

<a id="aop-ataspectj-advice-params-generics"></a>

###### [](#aop-ataspectj-advice-params-generics)通知参数和泛型

Spring AOP可以处理类声明和方法参数中使用的泛型。假设如下泛型类型:

    public interface Sample<T> {
        void sampleGenericMethod(T param);
        void sampleGenericCollectionMethod(Collection<T> param);
    }

只需将通知参数键入要拦截方法的参数类型，就可以将方法类型的检测限制为某些参数类型:

    @Before("execution(* ..Sample+.sampleGenericMethod(*)) && args(param)")
    public void beforeSampleMethod(MyType param) {
        // Advice implementation
    }

此方法不适用于泛型集合。 因此，您无法按如下方式定义切点:

    @Before("execution(* ..Sample+.sampleGenericCollectionMethod(*)) && args(param)")
    public void beforeSampleMethod(Collection<MyType> param) {
        // Advice implementation
    }

为了使这项工作，我们必须检查集合的每个元素，这是不合理的，因为我们也无法决定如何处理`null`值。 要实现与此类似的操作，您必须将参数键入`Collection<?>` 并手动检查元素的类型。

<a id="aop-ataspectj-advice-params-names"></a>

###### [](#aop-ataspectj-advice-params-names)声明参数的名字

参数在通知中的绑定依赖于名字匹配，重点在切点表达式中定义的参数名的方法签名上（通知和切点）。参数名称不能通过Java反射获得，因此Spring AOP使用以下策略来确定参数名称：

*   如果用户已明确指定参数名称，则使用指定的参数名称。通知和切点注解都有一个可选的`argNames`属性，您可以使用该属性指定带注解的方法的参数名称。 这些参数名称在运行时可用。 以下示例显示如何使用`argNames`属性:

        @Before(value="com.xyz.lib.Pointcuts.anyPublicMethod() && target(bean) && @annotation(auditable)",
                argNames="bean,auditable")
        public void audit(Object bean, Auditable auditable) {
            AuditCode code = auditable.value();
            // ... use code and bean
        }

    如果第一个参数是`JoinPoint`, `ProceedingJoinPoint`, 或 `JoinPoint.StaticPart` 类型，则可以从`argNames`属性的值中省略参数的名称。 例如，如果修改前面的通知以接收连接点对象，则`argNames`属性不需要包含它:

        @Before(value="com.xyz.lib.Pointcuts.anyPublicMethod() && target(bean) && @annotation(auditable)",
                argNames="bean,auditable")
        public void audit(JoinPoint jp, Object bean, Auditable auditable) {
            AuditCode code = auditable.value();
            // ... use code, bean, and jp
        }

    对`JoinPoint`,`ProceedingJoinPoint`, and `JoinPoint.StaticPart`类型的第一个参数的特殊处理方便不收集任何其他连接点上下文的通知。 在这种情况下，可以简单地省略`argNames`属性。例如，以下建议无需声明`argNames`属性:

        @Before("com.xyz.lib.Pointcuts.anyPublicMethod()")
        public void audit(JoinPoint jp) {
            // ... use jp
        }

*   使用`'argNames'`属性有点笨拙，所以如果没有指定`'argNames'`属性，Spring AOP会查看该类的调试信息，并尝试从局部变量表中确定参数名称。只要使用调试信息(`'-g:vars'`）编译类， 就会出现此信息。使用此标志进行编译的后果是：(1).您的代码将容易被理解(逆向工程。(2). 类文件的大小将会有些大(通常不是什么事)。(3). 对非使用本地变量的优化将不会应用于你的编译器。 换句话说，通过使用此标志构建，您应该不会遇到任何困难。

    如果即使没有调试信息，AspectJ编译器（ajc）也编译了@AspectJ方面，则无需添加 `argNames`属性，因为编译器会保留所需的信息。

*   如果代码是在没有必要的调试信息的情况下编译的，那么Spring AOP将尝试推断绑定变量与参数的配对（例如，如果在切点表达式中只绑定了一个变量，并且该通知方法只需要一个参数，此时两者匹配是明显的）。 如果给定了可用信息，变量的绑定是不明确的话，则会引发`AmbiguousBindingException`异常。

*   如果上述所有策略都失败，则抛出`IllegalArgumentException` 异常。


<a id="aop-ataspectj-advice-proceeding-with-the-call"></a>

###### [](#aop-ataspectj-advice-proceeding-with-the-call)处理参数

前面说过。将描述如何用在Spring AOP和AspectJ中一致的参数中编写`proceed`处理函数。解决方案是确保建议签名按顺序绑定每个方法参数。 以下示例显示了如何执行此操作：:

    @Around("execution(List<Account> find*(..)) && " +
            "com.xyz.myapp.SystemArchitecture.inDataAccessLayer() && " +
            "args(accountHolderNamePattern)")
    public Object preProcessQueryPattern(ProceedingJoinPoint pjp,
            String accountHolderNamePattern) throws Throwable {
        String newPattern = preProcess(accountHolderNamePattern);
        return pjp.proceed(new Object[] {newPattern});
    }

在许多情况下，无论如何都要执行此绑定（如前面的示例所示）。

<a id="aop-ataspectj-advice-ordering"></a>

##### [](#aop-ataspectj-advice-ordering)通知的顺序

当多个通知都希望在同一连接点上运行时会发生什么情况？Spring AOP遵循与AspectJ相同的优先级规则来确定通知执行的顺序。拥有最高优先权的通知会途中先"进入"（因此，给定两条前置通知，优先级最高的通知首先运行）。 从连接点"退出"，拥有最高优先级的通知最后才运行（退出）（（因此，如果有两个后置通知，那么拥有最高优先级的将在最后运行（退出））。

如果在不同切面定义的两个通知都需要在同一个连接点运行，那么除非开发者指定运行的先后，否则执行的顺序是未定义的。 可以通过指定优先级来控制执行顺序。这也是Spring推荐的方式，通过在切面类实现`org.springframework.core.Ordered`接口或使用`Order`对其进行注解即可。 如果有两个切面，从`Ordered.getValue()`（或注解值）返回较低值的方面具有较高的优先级。

当在同一切面定义的两条通知都需要在同一个连接点上运行时，排序也是未定义的（因为没有办法通过反射检索Javac编译的类的声明顺序） 。考虑将通知方法与一个通知方法合并，根据每个连接点在每个切面类或将通知切分为切面类，可以在切面级别指定顺序。

<a id="aop-introductions"></a>

#### [](#aop-introductions)5.4.5. 引入

引入（作为AspectJ中内部类型的声明）允许切面定义通知的对象实现给定的接口,并代表这些对象提供该接口的实现.

引入使用`@DeclareParents`注解来定义,这个注解用于声明匹配拥有新的父类的类型（因此得名）。例如， 给定名为`UsageTracked` 的接口和名为`DefaultUsageTracked`的接口的实现，以下切面声明服务接口的所有实现者也实现`UsageTracked`接口（例如，通过JMX公开统计信息）:

    @Aspect
    public class UsageTracking {

        @DeclareParents(value="com.xzy.myapp.service.*+", defaultImpl=DefaultUsageTracked.class)
        public static UsageTracked mixin;

        @Before("com.xyz.myapp.SystemArchitecture.businessService() && this(usageTracked)")
        public void recordUsage(UsageTracked usageTracked) {
            usageTracked.incrementUseCount();
        }

    }

要实现的接口由注解属性的类型来确定。 `@DeclareParents`注解的`value`值是AspectJ类型模式引过来的。注意上面例子中的前置通知， 服务bean可以直接作为`UsageTracked`接口的实现，如果以编程方式访问bean，您将编写以下内容：:

    UsageTracked usageTracked = (UsageTracked) context.getBean("myService");

<a id="aop-instantiation-models"></a>

#### [](#aop-instantiation-models)5.4.6. 切面实例化模型

这是一个高级主题。 如果您刚刚开始使用AOP，您可以跳过它直到稍后再了解。

默认情况下，应用程序上下文中的每个切面都有一个实例。AspectJ将其称为单例实例化模型。 可以使用交替生命周期定义切面。 Spring支持AspectJ的`perthis`和 `pertarget`实例化模型（目前不支持`percflow, percflowbelow,` 和 `pertypewithin`）。

您可以通过在`@Aspect`注解中指定`perthis`子句来声明相关方面。 请考虑以下示例:

    @Aspect("perthis(com.xyz.myapp.SystemArchitecture.businessService())")
    public class MyAspect {

        private int someState;

        @Before(com.xyz.myapp.SystemArchitecture.businessService())
        public void recordServiceUsage() {
            // ...
        }

    }

在前面的示例中，`'perthis'`子句的作用是为执行业务服务的每个唯一服务对象创建一个切面实例（每个唯一对象在由切点表达式匹配的连接点处绑定到'this'）。 方法实例是在第一次在服务对象上调用方法时创建的。当服务对象超出范围时，该切面也将超出范围。在创建切面实例之前，它包含的任意通知都不会执行。在创建了切面实例后， 其中声明的通知将在匹配的连接点中执行，但仅当服务对象是此切面关联的通知时才会运行。有关`per` 子句的更多信息，请参阅AspectJ编程指南。

`pertarget` 实例化模型的工作方式与`perthis`完全相同，但它为匹配的连接点处的每个唯一目标对象创建一个切面实例。

<a id="aop-ataspectj-example"></a>

#### [](#aop-ataspectj-example)5.4.7. AOP 例子

现在您已经了解了所有组成部分的工作原理，我们可以将它们放在一起做一些有用的事情.

由于并发问题（例如，死锁失败者），业务服务的执行有时会失败。如果重试该操作，则可能在下次尝试时成功。对于适合在这种情况下重试的业务服务（不需要返回给用户来解决冲突的幂等操作）。 希望透明地重试该操作，以避免客户端看到`PessimisticLockingFailureException`异常。这个需求很明显，它跨越了服务层中的多个服务，因此非常适合通过切面来实现。

因为我们想要重试操作，所以我们需要使用环绕通知，以便我们可以多次调用`proceed`。 以下清单显示了基本方面的实现:

    @Aspect
    public class ConcurrentOperationExecutor implements Ordered {

        private static final int DEFAULT_MAX_RETRIES = 2;

        private int maxRetries = DEFAULT_MAX_RETRIES;
        private int order = 1;

        public void setMaxRetries(int maxRetries) {
            this.maxRetries = maxRetries;
        }

        public int getOrder() {
            return this.order;
        }

        public void setOrder(int order) {
            this.order = order;
        }

        @Around("com.xyz.myapp.SystemArchitecture.businessService()")
        public Object doConcurrentOperation(ProceedingJoinPoint pjp) throws Throwable {
            int numAttempts = 0;
            PessimisticLockingFailureException lockFailureException;
            do {
                numAttempts++;
                try {
                    return pjp.proceed();
                }
                catch(PessimisticLockingFailureException ex) {
                    lockFailureException = ex;
                }
            } while(numAttempts <= this.maxRetries);
            throw lockFailureException;
        }

    }

请注意，该方面实现了`Ordered`接口，以便我们可以将切面的优先级设置为高于事务通知（我们每次重试时都需要一个新的事务）。 `maxRetries`和`order`属性都由Spring配置。主要的操作是在`doConcurrentOperation`的环绕通知中。请注意，请注意，目前，我们将重试逻辑应用于每个 `businessService()`。 尝试执行时，如果失败了，将产生`PessimisticLockingFailureException`异常，但是不用管它，只需再次尝试执行即可，除非已经用尽所有的重试次数。

相应的Spring配置如下：

    <aop:aspectj-autoproxy/>

    <bean id="concurrentOperationExecutor" class="com.xyz.myapp.service.impl.ConcurrentOperationExecutor">
        <property name="maxRetries" value="3"/>
        <property name="order" value="100"/>
    </bean>

为了优化切面以便它只重试幂等操作，我们可以定义以下`Idempotent`注解:

    @Retention(RetentionPolicy.RUNTIME)
    public @interface Idempotent {
        // marker annotation
    }

然后使用它来注解服务操作的实现。对切面的更改只需要重试等幂运算，只需细化切点表达式，以便只匹配`@Idempotent`操作:

    @Around("com.xyz.myapp.SystemArchitecture.businessService() && " +
            "@annotation(com.xyz.myapp.service.Idempotent)")
    public Object doConcurrentOperation(ProceedingJoinPoint pjp) throws Throwable {
        ...
    }

<a id="aop-schema"></a>

### [](#aop-schema)5.5. 基于Schema的AOP支持

如果您更喜欢基于XML的格式，Spring还支持使用新的`aop`命名空间标签定义切面。完全相同的切点表达式和通知类型在使用@AspectJ方式时同样得到支持。 因此，在本节中，我们将重点放在新语法上，并将读者引用到上一节（[@AspectJ支持](#aop-ataspectj)）中的讨论，以了解编写切点表达式和通知参数的绑定。

要使用本节中描述的aop命名空间标签，您需要导入`spring-aop` schema，如[基于XML模式的配置中所述](#xsd-schemas)。 有关如何在`aop`命名空间中导入标记，请参阅[the AOP schema](#xsd-schemas-aop)。

在Spring配置中，所有aspect和advisor元素必须放在 `<aop:config>`元素中（在应用程序上下文配置中可以有多个 `<aop:config>`元素）。 `<aop:config>`元素可以包含切点，通知者和切面元素（请注意，这些元素必须按此顺序声明）。

`<aop:config>`配置样式大量使用了Spring的[自动代理](#aop-autoproxy)机制。如果已经通过使用`BeanNameAutoProxyCreator` 或类似的类使用了显式的自动代理， 则可能会出现问题（如通知还没被编织）。建议的使用模式是仅使用`<aop:config>`样式或仅使用`AutoProxyCreator`样式，并且永远不要混用它们。

<a id="aop-schema-declaring-an-aspect"></a>

#### [](#aop-schema-declaring-an-aspect)5.5.1.声明切面

如果使用schema，那么切面只是在Spring应用程序上下文中定义为bean的常规Java对象。在对象的字段和方法中获取状态和行为，并且在XML中获取切点和通知信息。

您可以使用<aop:aspect>元素声明方面，并使用`ref`属性引用支持bean，如以下示例所示:

    <aop:config>
        <aop:aspect id="myAspect" ref="aBean">
            ...
        </aop:aspect>
    </aop:config>

    <bean id="aBean" class="...">
        ...
    </bean>

支持切面的bean（在这种情况下是`aBean`）当然可以像任何其他Spring bean一样配置和依赖注入。

<a id="aop-schema-pointcuts"></a>

#### [](#aop-schema-pointcuts)5.5.2. 声明切点

您可以在`<aop:config>`元素中声明一个命名切点，让切点定义在多个切面和通知者之间共享。

表示服务层中任何业务服务执行的切点可以定义如下：:

    <aop:config>

        <aop:pointcut id="businessService"
            expression="execution(* com.xyz.myapp.service.*.*(..))"/>

    </aop:config>

切点表达式本身使用的是相同的AspectJ切点表达式语言，如[@AspectJ支持](#aop-ataspectj)所述。如果使用基于schema的声明样式，则可以引用在切点表达式内的类型(@Aspects)中定义的命名切点 。定义上述切入点的另一种方法如下:

    <aop:config>

        <aop:pointcut id="businessService"
            expression="com.xyz.myapp.SystemArchitecture.businessService()"/>

    </aop:config>

假设有一个`SystemArchitecture`的切面（如[共享通用的切点定义](#aop-common-pointcuts)一节所述）。

切面声明切点与声明top-level切点非常相似，如下例所示：:

    <aop:config>

        <aop:aspect id="myAspect" ref="aBean">

            <aop:pointcut id="businessService"
                expression="execution(* com.xyz.myapp.service.*.*(..))"/>

            ...

        </aop:aspect>

    </aop:config>

与@AspectJ方面的方法相同，使用基于schema的定义样式声明的切点可能会收集连接点上下文。例如，以下切点将`this`对象收集为连接点上下文并将其传递给通知：:

    <aop:config>

        <aop:aspect id="myAspect" ref="aBean">

            <aop:pointcut id="businessService"
                expression="execution(* com.xyz.myapp.service.*.*(..)) &amp;&amp; this(service)"/>

            <aop:before pointcut-ref="businessService" method="monitor"/>

            ...

        </aop:aspect>

    </aop:config>

必须通过包含匹配名称的参数来声明接收所收集的连接点上下文的通知，如下所示：:

    public void monitor(Object service) {
        ...
    }

在组合切点表达式中， `&&` 在XML文档中很难处理，因此您可以分别使用 `and`, `or`和`not` 分别用来代替`&&`, `||`, 和 `!` 。例如，以前的切点可以更好地编写如下：:

    <aop:config>

        <aop:aspect id="myAspect" ref="aBean">

            <aop:pointcut id="businessService"
                expression="execution(* com.xyz.myapp.service..(..)) and this(service)"/>

            <aop:before pointcut-ref="businessService" method="monitor"/>

            ...
        </aop:aspect>
    </aop:config>

以这种方式定义的切点由其XML `id`引用，不能用作命名切点以形成复合切点。因此，基于schema定义样式中的命名切点比@AspectJ样式提供的受到更多的限制。

<a id="aop-schema-advice"></a>

#### [](#aop-schema-advice)5.5.3. 声明通知

同样的五种通知类型也支持@AspectJ样式，并且它们具有完全相同的语义。

<a id="aop-schema-advice-before"></a>

##### [](#aop-schema-advice-before)前置通知

前置通知很明显是在匹配方法执行之前被调用， 它通过使用 <aop:before>元素在`<aop:aspect>`中声明，如下例所示:

    <aop:aspect id="beforeExample" ref="aBean">

        <aop:before
            pointcut-ref="dataAccessOperation"
            method="doAccessCheck"/>

        ...

    </aop:aspect>

这里`dataAccessOperation` 是在最外层的(`<aop:config>`)定义的切点id。若要以内联方式定义切点，请将`pointcut-ref`属性替换为切点属性。如下所示:

    <aop:aspect id="beforeExample" ref="aBean">

        <aop:before
            pointcut="execution(* com.xyz.myapp.dao.*.*(..))"
            method="doAccessCheck"/>

        ...

    </aop:aspect>

正如我们在讨论@AspectJ样式时所提到的，使用命名切点可以显着提高代码的可读性。

`method`属性定义的 (`doAccessCheck`)方法用于通知的代码体内。这个方法包含切面元素所引用的bean。在数据访问操作之前通知会被执行(当然连接点匹配中的切点)， 即切面bean的`doAccessCheck`方法会被调用。

<a id="aop-schema-advice-after-returning"></a>

##### [](#aop-schema-advice-after-returning)后置返回通知

在匹配的方法执行正常完成后返回通知运行。 它在 `<aop:aspect>`中以与前置通知相同的方式声明。 以下示例显示了如何声明它:

    <aop:aspect id="afterReturningExample" ref="aBean">

        <aop:after-returning
            pointcut-ref="dataAccessOperation"
            method="doAccessCheck"/>

        ...

    </aop:aspect>

与@AspectJ样式一样，可以在通知代码体内获取返回值。为此，使用returning属性定义参数的名字来传递返回值，如以下示例所示:

    <aop:aspect id="afterReturningExample" ref="aBean">

        <aop:after-returning
            pointcut-ref="dataAccessOperation"
            returning="retVal"
            method="doAccessCheck"/>

        ...

    </aop:aspect>

`doAccessCheck`方法必须声明一个名为`retVal`的参数，此参数的类型约束匹配的方式与`@AfterReturning`所描述的相同。例如，您可以按如下方式声明方法签名:

    public void doAccessCheck(Object retVal) {...

<a id="aop-schema-advice-after-throwing"></a>

##### [](#aop-schema-advice-after-throwing)后置异常通知

就是匹配的方法运行抛出异常后后置异常通知会运行，它在`<aop:aspect>`中使用 after-throwing元素声明。如下例所示:

    <aop:aspect id="afterThrowingExample" ref="aBean">

        <aop:after-throwing
            pointcut-ref="dataAccessOperation"
            method="doRecoveryActions"/>

        ...

    </aop:aspect>

与@AspectJ样式一样，可以在通知代码体内获取抛出的异常，使用throwing属性定义参数的名字来传递异常。如以下示例所示:

    <aop:aspect id="afterThrowingExample" ref="aBean">

        <aop:after-throwing
            pointcut-ref="dataAccessOperation"
            throwing="dataAccessEx"
            method="doRecoveryActions"/>

        ...

    </aop:aspect>

`doRecoveryActions`方法必须声明名为 `dataAccessEx`的参数。此参数的类型约束匹配的方式与`@AfterThrowing`所描述的相同。 例如，方法签名可以声明如下:

    public void doRecoveryActions(DataAccessException dataAccessEx) {...

<a id="aop-schema-advice-after-finally"></a>

##### [](#aop-schema-advice-after-finally)后置通知(总会执行的)

当方法执行完成并退出后，后置通知会被执行(而且是总会被执行)。如以下示例所示:

    <aop:aspect id="afterFinallyExample" ref="aBean">

        <aop:after
            pointcut-ref="dataAccessOperation"
            method="doReleaseLock"/>

        ...

    </aop:aspect>

<a id="aop-schema-advice-around"></a>

##### [](#aop-schema-advice-around)环绕通知

最后一种通知是环绕通知. 环绕通知 “around” 匹配的方法执行运行。它有机会在方法执行之前和之后进行工作，并确定方法何时、 如何以及甚至是否真正执行。环绕通知经常用于需要在方法执行前或后在线程安全的情况下共享状态（例如开始和结束时间）。确认可使用的通知形式， 要符合最小匹配原则。

您可以使用`aop:around`元素声明环绕通知。通知方法的第一个参数必须是`ProceedingJoinPoint`类型。在通知代码体中，调用`ProceedingJoinPoint`实现的`proceed()`会使匹配的方法继续执行。 `proceed`方法也可以通过传递 `Object[]`– 数组的值给原方法作为传入参数。有关调用继续使用 `Object[]`的说明，请参阅[环绕通知](#aop-ataspectj-around-advice)。 以下示例显示如何在XML中声明通知：

    <aop:aspect id="aroundExample" ref="aBean">

        <aop:around
            pointcut-ref="businessService"
            method="doBasicProfiling"/>

        ...

    </aop:aspect>

`doBasicProfiling`通知的运行与@AspectJ示例中的完全相同（当然省略了注解）。如以下示例所示:

    public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
        // start stopwatch
        Object retVal = pjp.proceed();
        // stop stopwatch
        return retVal;
    }

<a id="aop-schema-advice-around"></a>

##### [](#aop-schema-params)通知参数

基于schema的声明样式支持所有类型的通知，其方式与@AspectJ支持的描述相同 - 通过按名称匹配切点参数与通知方法参数相匹配。有关详细信息，请参阅[通知参数](#aop-ataspectj-advice-params)。 如果希望显式指定通知方法的参数名称（不依赖于前面描述的检测策略）则使用通知元素的`arg-names`属性来完成这一操作。其处理方式和通知注解中的`argNames`属性是相同的， 在通知注解中（如[确定参数名称](#aop-ataspectj-advice-params-names)中所述）。 以下示例显示如何在XML中指定参数名称:

    <aop:before
        pointcut="com.xyz.lib.Pointcuts.anyPublicMethod() and @annotation(auditable)"
        method="audit"
        arg-names="auditable"/>

`arg-names` 属性接受以逗号分隔的参数名称列表。

下面是一个基于XSD方式的多调用示例，它说明环绕通知是如何与一些强类型参数共同使用的:

    package x.y.service;

    public interface PersonService {

        Person getPerson(String personName, int age);
    }

    public class DefaultFooService implements FooService {

        public Person getPerson(String name, int age) {
            return new Person(name, age);
        }
    }

接下来定义切面。请注意，`profile(..)`方法接受许多强类型参数，其中第一个是用于方法调用的连接点。这个参数用于声明`profile(..)`作为环绕通知来使用，如以下示例所示：

    package x.y;

    import org.aspectj.lang.ProceedingJoinPoint;
    import org.springframework.util.StopWatch;

    public class SimpleProfiler {

        public Object profile(ProceedingJoinPoint call, String name, int age) throws Throwable {
            StopWatch clock = new StopWatch("Profiling for '" + name + "' and '" + age + "'");
            try {
                clock.start(call.toShortString());
                return call.proceed();
            } finally {
                clock.stop();
                System.out.println(clock.prettyPrint());
            }
        }
    }

最后，下面是为特定连接点执行上述建议所需的XML配置:

    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:aop="http://www.springframework.org/schema/aop"
        xsi:schemaLocation="
            http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

        <!-- this is the object that will be proxied by Spring's AOP infrastructure -->
        <bean id="personService" class="x.y.service.DefaultPersonService"/>

        <!-- this is the actual advice itself -->
        <bean id="profiler" class="x.y.SimpleProfiler"/>

        <aop:config>
            <aop:aspect ref="profiler">

                <aop:pointcut id="theExecutionOfSomePersonServiceMethod"
                    expression="execution(* x.y.service.PersonService.getPerson(String,int))
                    and args(name, age)"/>

                <aop:around pointcut-ref="theExecutionOfSomePersonServiceMethod"
                    method="profile"/>

            </aop:aspect>
        </aop:config>

    </beans>

请考虑以下驱动程序脚本:

    import org.springframework.beans.factory.BeanFactory;
    import org.springframework.context.support.ClassPathXmlApplicationContext;
    import x.y.service.PersonService;

    public final class Boot {

        public static void main(final String[] args) throws Exception {
            BeanFactory ctx = new ClassPathXmlApplicationContext("x/y/plain.xml");
            PersonService person = (PersonService) ctx.getBean("personService");
            person.getPerson("Pengo", 12);
        }
    }

使用这样的Boot类，我们将在标准输出上获得类似于以下内容的输出：:

StopWatch 'Profiling for 'Pengo' and '12'': running time (millis) = 0
\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-
ms     %     Task name
\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-
00000  ?  execution(getFoo)

<a id="aop-ordering"></a>

##### [](#aop-ordering)通知的顺序

当多个通知需要在同一个连接点（执行方法）执行时，排序规则如[通知排序](#aop-ataspectj-advice-ordering)中所述。 方面之间的优先级是通过将`Order`注释添加到支持方面的bean或通过让bean实现 `Ordered`接口来确定的。

<a id="aop-schema-introductions"></a>

#### [](#aop-schema-introductions)5.5.4. 引入

引入（作为AspectJ中内部类型的声明）允许切面定义通知的对象实现给定的接口，并代表这些对象提供该接口的实现。

您可以在`aop:aspect`中使用`aop:declare-parents`元素进行引入。。 您可以使用`aop:declare-parents`元素声明匹配类型具有父级（因此名称）。 例如，给定名为`UsageTracked`的接口和名为`DefaultUsageTracked`的接口的实现，以下方面声明服务接口的所有实现者也实现`UsageTracked` 接口。 （例如，为了通过JMX公开统计信息。）

    <aop:aspect id="usageTrackerAspect" ref="usageTracking">

        <aop:declare-parents
            types-matching="com.xzy.myapp.service.*+"
            implement-interface="com.xyz.myapp.service.tracking.UsageTracked"
            default-impl="com.xyz.myapp.service.tracking.DefaultUsageTracked"/>

        <aop:before
            pointcut="com.xyz.myapp.SystemArchitecture.businessService()
                and this(usageTracked)"
                method="recordUsage"/>

    </aop:aspect>

然后，支持`usageTracking`bean的类将包含以下方法:

    public void recordUsage(UsageTracked usageTracked) {
        usageTracked.incrementUseCount();
    }

要实现的接口由`implement-interface`属性确定。`types-matching`属性的值是AspectJ类型模式。任何匹配类型的bean都将实现`UsageTracked`接口。 请注意，在前面的示例的通知中，服务bean可以直接用作`UsageTracked`接口的实现。要以编程方式访问bean，您可以编写以下代码:

    UsageTracked usageTracked = (UsageTracked) context.getBean("myService");

<a id="aop-schema-instatiation-models"></a>

#### [](#aop-schema-instatiation-models)5.5.5. 切面实例化模型

唯一受支持的schema定义的实例化模型是单例模型，在将来的版本中可能支持其他实例化模型。

<a id="aop-schema-advisors"></a>

#### [](#aop-schema-advisors)5.5.6. 通知者

通知者的概念是在Spring 1.2中提出的，能被AOP支持。而在AspectJ中没有等价的概念。通知者就像迷你的切面，包含单一的通知。通知本身可以通过bean来代表，并且必须实现Spring中的[通知类型](#aop-api-advice-types)中描述的通知接口之一， 通知者可以利用AspectJ的切点表达式

Spring使用`<aop:advisor>`元素支持通知者概念。通常会看到它与事务性通知一起使用，它在Spring中也有自己的命名空间支持。 以下示例显示了一个通知者:

    <aop:config>

        <aop:pointcut id="businessService"
            expression="execution(* com.xyz.myapp.service.*.*(..))"/>

        <aop:advisor
            pointcut-ref="businessService"
            advice-ref="tx-advice"/>

    </aop:config>

    <tx:advice id="tx-advice">
        <tx:attributes>
            <tx:method name="*" propagation="REQUIRED"/>
        </tx:attributes>
    </tx:advice>

除了前面示例中使用的`pointcut-ref`属性之外，您还可以使用切点属性来内联定义切点表达式。

如果想将通知排序，可以定义通知者的优先级。在通知者上可以使用`order`属性来定义`Ordered`值。

<a id="aop-schema-example"></a>

#### [](#aop-schema-example)5.5.7. AOP Schema 例子

本节说明如何使用Schema支持重写[An AOP Example](#aop-ataspectj-example)示例中的并发锁定失败重试示例。

由于并发问题（例如，死锁失败者），业务服务的执行有时会失败。如果重试该操作，则可能在下次尝试时成功。对于适合在这种情况下重试的业务服务（不需要返回给用户来解决冲突的幂等操作）。 希望透明地重试该操作，以避免客户端看到`PessimisticLockingFailureException`异常。这个需求很明显，它跨越了服务层中的多个服务，因此非常适合通过切面来实现。

因为我们想要重试操作，所以我们需要使用环绕通知，以便我们可以多次调用`proceed`。 以下清单显示了基本方面的实现（使用Schema支持的常规Java类）:

    public class ConcurrentOperationExecutor implements Ordered {

        private static final int DEFAULT_MAX_RETRIES = 2;

        private int maxRetries = DEFAULT_MAX_RETRIES;
        private int order = 1;

        public void setMaxRetries(int maxRetries) {
            this.maxRetries = maxRetries;
        }

        public int getOrder() {
            return this.order;
        }

        public void setOrder(int order) {
            this.order = order;
        }

        public Object doConcurrentOperation(ProceedingJoinPoint pjp) throws Throwable {
            int numAttempts = 0;
            PessimisticLockingFailureException lockFailureException;
            do {
                numAttempts++;
                try {
                    return pjp.proceed();
                }
                catch(PessimisticLockingFailureException ex) {
                    lockFailureException = ex;
                }
            } while(numAttempts <= this.maxRetries);
            throw lockFailureException;
        }

    }

请注意，该方面实现了`Ordered`接口，以便我们可以将切面的优先级设置为高于事务通知（我们每次重试时都需要一个新的事务）。 `maxRetries`和`order`属性都由Spring配置。主要的操作是在`doConcurrentOperation`的环绕通知中。请注意，请注意，目前，我们将重试逻辑应用于每个 `businessService()`。 尝试执行时，如果失败了，将产生`PessimisticLockingFailureException`异常，但是不用管它，只需再次尝试执行即可，除非已经用尽所有的重试次数。

此类与@AspectJ示例中使用的类相同，但删除了注释。

相应的Spring配置如下:

    <aop:config>

        <aop:aspect id="concurrentOperationRetry" ref="concurrentOperationExecutor">

            <aop:pointcut id="idempotentOperation"
                expression="execution(* com.xyz.myapp.service.*.*(..))"/>

            <aop:around
                pointcut-ref="idempotentOperation"
                method="doConcurrentOperation"/>

        </aop:aspect>

    </aop:config>

    <bean id="concurrentOperationExecutor"
        class="com.xyz.myapp.service.impl.ConcurrentOperationExecutor">
            <property name="maxRetries" value="3"/>
            <property name="order" value="100"/>
    </bean>

请注意，在当时，我们假设所有业务服务都是幂等的。如果不是这种情况，我们可以通过引入`Idempotent`注解并使用注解来注解服务操作的实现来优化切面，使其重试时是幂等操作，如以下示例所示:

    @Retention(RetentionPolicy.RUNTIME)
    public @interface Idempotent {
        // marker annotation
    }

对切面的更改只需要重试等幂运算，只需细化切点表达式，以便只匹配`@Idempotent`操作，如下所示:

    <aop:pointcut id="idempotentOperation"
            expression="execution(* com.xyz.myapp.service.*.*(..)) and
            @annotation(com.xyz.myapp.service.Idempotent)"/>

<a id="aop-choosing"></a>

### [](#aop-choosing)5.6. 选择要使用的AOP声明样式

一旦确定某个切面是实现给定需求的最佳方法，您如何决定使用Spring AOP或AspectJ以及Aspect语言（代码）样式， @ AspectJ注解样式还是Spring XML样式？ 这些决策受到许多因素的影响，包括应用程序要求，开发工具和团队对AOP的熟悉程度。

<a id="aop-spring-or-aspectj"></a>

#### [](#aop-spring-or-aspectj)5.6.1. 使用Spring AOP还是全面使用AspectJ?

使用最简单的方法。 Spring AOP比使用完整的AspectJ更简单，因为不需要在开发和构建过程中引入AspectJ编译器/ 编织器。如果只是需要在Spring bean上执行通知操作，那么使用Spring AOP是正确的选择。 如果需要的通知不是由Spring容器管理的对象（通常是域对象），那么就需要使用AspectJ。如果想使用通知连接点而不是简单的方法执行，也需要使用AspectJ（例如，字段获取或设置连接点等），则还需要使用AspectJ。

使用AspectJ时，您可以选择AspectJ语言语法（也称为“代码样式”）或@AspectJ注释样式。显然，如果没有使用Java 5+版本那么选择已经确定了...使用代码方式。 如果切面在你的设计中扮演重要角色，并且想使用针对Eclipse的[AspectJ开发工具（AJDT）](https://www.eclipse.org/ajdt/) 插件，那么AspectJ语言语法是首选项：它更清晰和更简单，因为语言是专门用于编写切面的。 如果没有使用Eclipse，或者只有一些切面在应用程序中不起主要作用，那么可能需要考虑使用@AspectJ方式，并在IDE中使用常规Java编译，并加入切面编织阶段构建的脚本。

<a id="aop-ataspectj-or-xml"></a>

#### [](#aop-ataspectj-or-xml)5.6.2. 选择@AspectJ注解还是Spring AOP的XML配置?

如果您选择使用Spring AOP，则可以选择@AspectJ或XML样式。 需要考虑各种权衡。

XML样式可能是现有Spring用户最熟悉的，并且由真正的POJO支持。当使用AOP作为一种工具来配置企业服务时， XML就是一个很好的选择（可以用以下方法测试：是否认为切入点表达式是想要独立改变的配置的一部分）。 使用XML配置的方式，可以从配置中更清楚地了解系统中存在哪些切面。

XML样式有两个缺点。首先，它并没有按实现的要求完全封装到单个地方。DRY原则是说：在任何知识系统中，应该有一个单一的、明确的、权威的职责。使用XML的样式时，如果要求的知识是实现拆分的bean类的声明，并且是配置在文件的XML中。 当使用@AspectJ的风格实现单一的模块时，切面的信息是封装的。其次，XML的样式在能表达的功能方面比@AspectJ风格的有更多的限制，只有“singleton”切面的实例化模式得到支持，这在XML声明的切点中是不可能的。 例如，在@AspectJ样式中，您可以编写如下内容:

    @Pointcut("execution(* get*())")
    public void propertyAccess() {}

    @Pointcut("execution(org.xyz.Account+ *(..))")
    public void operationReturningAnAccount() {}

    @Pointcut("propertyAccess() && operationReturningAnAccount()")
    public void accountPropertyAccess() {}

在XML样式中，您可以声明前两个切入点:

    <aop:pointcut id="propertyAccess"
            expression="execution(* get*())"/>

    <aop:pointcut id="operationReturningAnAccount"
            expression="execution(org.xyz.Account+ *(..))"/>

XML的方法的缺点是，您无法通过组合这些定义来定义`accountPropertyAccess`切点。

@AspectJ的风格支持更多的实例化模式和丰富的切点组合。它的优点是将切面确保为单元模块化，@AspectJ的使用对理解切面也很有优势（也很容易接受）， 无论是通过Spring AOP还是AspectJ的使用 。 所以如果决定需要AspectJ的能力解决额外的要求，然后迁移到一个基于AspectJ的方法，是非常简单的。 Spring团队建议使用@AspectJ的方式。

<a id="aop-mixing-styles"></a>

### [](#aop-mixing-styles)5.7. 混合切面类型

在实际应用中，完全有可能混合使用@AspectJ的切面方式，用于支持自动代理、schema定义`<aop:aspect>`，`<aop:advisor>`声明通知者甚至在同一配置中定义使用Spring 1.2 风格的代理和拦截器。所有这些都是使用相同的底层支持机制实现的，并且可以愉快地共存。

<a id="aop-proxying"></a>

### [](#aop-proxying)5.8. 代理策略

Spring AOP使用JDK动态代理或CGLIB为给定目标对象创建代理。 （只要有选择，JDK动态代理就是首选）。

如果要代理的目标对象实现至少一个接口，则使用JDK动态代理。 目标类型实现的所有接口都是代理的。 如果目标对象未实现任何接口，则会创建CGLIB代理。

如果要强制使用CGLIB代理（例如，代理为目标对象定义的每个方法，而不仅仅是那些由其接口实现的方法），您可以这样做。 但是，您应该考虑以下问题：

*   `final` 声明为final的方法不能使用，因为它们不能被覆盖。

*   从Spring 3.2开始，不再需要将CGLIB添加到项目类路径中，因为CGLIB类在`org.springframework`下重新打包并直接包含在spring-core JAR中。这意味着基于CGLIB的代理支持可以像JDK动态代理那样方便地工作。

*   从Spring 4.0开始，代理对象的构造函数不再被调用两次，因为CGLIB代理实例是通过Objenesis创建的。只有当JVM不允许构造器绕过时，可能会看到来自Spring AOP的代理支持双重调用以及看到相应的调试日志。


要强制使用CGLIB代理，请将`<aop:config>`元素的`proxy-target-class`属性的值设置为`true`，如下所示:

    <aop:config proxy-target-class="true">
        <!-- other beans defined here... -->
    </aop:config>

要在使用@AspectJ自动代理支持时强制CGLIB代理，请将`<aop:aspectj-autoproxy>` 元素的`proxy-target-class`属性设置为`true`，如下所示:

    <aop:aspectj-autoproxy proxy-target-class="true"/>

多个`<aop:config/>`选择被集合到一个统一的自动代理创建器中运行，它使用了一个强代理设置，这些配置是任意 `<aop:config/>` 的子代码段（通常是来自不同的XML bean定义文件） 。这也适用于`<tx:annotation-driven/>`和`<aop:aspectj-autoproxy/>`。

要明确的是，在`<tx:annotation-driven/>`，`<aop:aspectj-autoproxy/>`或`<aop:config/>`元素上使用`proxy-target-class="true"` =“true”会强制使用CGLIB代理 他们。

<a id="aop-understanding-aop-proxies"></a>

#### [](#aop-understanding-aop-proxies)5.8.1. 理解AOP代理

Spring AOP是基于代理的，在编写自定义切面或使用Spring框架提供的任何基于Spring AOP的切面前，掌握上一个语句的实际语义是非常重要的。

首先需要考虑的情况如下，假设有一个普通的、非代理的、没有什么特殊的、直接的引用对象。如下面的代码片段所示:

    public class SimplePojo implements Pojo {

        public void foo() {
            // this next method invocation is a direct call on the 'this' reference
            this.bar();
        }

        public void bar() {
            // some logic...
        }
    }

如果在对象引用上调用方法，则直接在该对象引用上调用该方法，如下图所示：:

![aop proxy plain pojo call](https://github.com/DocsHome/spring-docs/blob/master/pages/images/aop-proxy-plain-pojo-call.png)

    public class Main {

        public static void main(String[] args) {

            Pojo pojo = new SimplePojo();

            // this is a direct method call on the 'pojo' reference
            pojo.foo();
        }
    }

当客户端代码是代理的引用时，事情发生了细微的变化。请考虑以下图表和代码段:

![aop proxy call](https://github.com/DocsHome/spring-docs/blob/master/pages/images/aop-proxy-call.png)

    public class Main {

        public static void main(String[] args) {

            ProxyFactory factory = new ProxyFactory(new SimplePojo());
            factory.addInterface(Pojo.class);
            factory.addAdvice(new RetryAdvice());

            Pojo pojo = (Pojo) factory.getProxy();

            // this is a method call on the proxy!
            pojo.foo();
        }
    }

这里要理解的关键是 `Main`类的 `main(..)`方法中的客户端代码具有对代理的引用。这意味着对该对象引用的方法将在代理上调用，因此代理将能够委托与该特定方法调用相关的所有拦截器（通知）。 然而，一旦调用终于达到了目标对象（在这个例子中是`SimplePojo`引用），任何方法调用都会传递给他，例如`this.bar()`或`this.foo()`， 都会调用这个引用，而不是代理。这具有重要的意义，这意味着自我调用不会导致与方法调用相关联的通知，从而也不会获得执行的机会。

好的，那要做些什么呢？ 最好的方法（这个“最好”的，也是迫不得已的）是重构代码，以便不会发生自我调用。这确实需要您做一些工作，但这是最好的，最少侵入性的方法。 下一个办法绝对是可怕的，我几乎不愿意指出，正是因为它是如此可怕。您可以（对我们来说很痛苦）将类中的逻辑完全绑定到Spring AOP，如下例所示：

    public class SimplePojo implements Pojo {

        public void foo() {
            // this works, but... gah!
            ((Pojo) AopContext.currentProxy()).bar();
        }

        public void bar() {
            // some logic...
        }
    }

这完全将代码与AOP相耦合，这使类本身意识到它正在AOP上下文中使用，犹如在AOP面前耍大刀一般。当创建代理时，它还需要一些额外的配置。如以下示例所示:

    public class Main {

        public static void main(String[] args) {

            ProxyFactory factory = new ProxyFactory(new SimplePojo());
            factory.adddInterface(Pojo.class);
            factory.addAdvice(new RetryAdvice());
            factory.setExposeProxy(true);

            Pojo pojo = (Pojo) factory.getProxy();

            // this is a method call on the proxy!
            pojo.foo();
        }
    }

最后，必须注意的是AspectJ没有这种自我调用问题，因为它不是基于代理的AOP框架。

<a id="aop-aspectj-programmatic"></a>

### [](#aop-aspectj-programmatic)5.9. 编程创建@AspectJ代理

除了在配置中使用`<aop:config>`或 `<aop:aspectj-autoproxy>`来声明切面外，还可以使用编程的方式创建代理的通知目标对象。 有关Spring的AOP API的完整详细信息，请参阅[下一章](#aop-api)。在这里，我们的关注点是希望使用@AspectJ方面自动创建代理的能力。

您可以使用`org.springframework.aop.aspectj.annotation.AspectJProxyFactory` 类为一个或多个@AspectJ切面通知的目标对象创建代理。 此类的基本用法非常简单，如下例所示:

    // create a factory that can generate a proxy for the given target object
    AspectJProxyFactory factory = new AspectJProxyFactory(targetObject);

    // add an aspect, the class must be an @AspectJ aspect
    // you can call this as many times as you need with different aspects
    factory.addAspect(SecurityManager.class);

    // you can also add existing aspect instances, the type of the object supplied must be an @AspectJ aspect
    factory.addAspect(usageTracker);

    // now get the proxy object...
    MyInterfaceType proxy = factory.getProxy();

See the [javadoc](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/aop/aspectj/annotation/AspectJProxyFactory.html) for more information.

<a id="aop-using-aspectj"></a>

### [](#aop-using-aspectj)5.10. 在Spring应用中使用AspectJ

到目前为止，我们在本章中介绍的所有内容都是纯粹的Spring AOP。将介绍如何使用AspectJ编译器/编织器代替AOP，还介绍了超越Spring AOP而单独提供的功能。

Spring有一个小的AspectJ切面库，是一个单独管理的`spring-aspects.jar`包。如果使用到切面那么需要将它添加到类路径中。在[使用Spring中的AspectJ独立注入域对象](#aop-atconfigurable) 和 [在Spring中使用的AspectJ另外的切面](#aop-ajlib-other) 会讨论这个库的内容以及如何使用。[使用Spring的IoC配置AspectJ切面](#aop-aj-configure)讨论如何依赖于使用AspectJ编译器编织的AspectJ切面。最后， 在 [在Spring框架中使用AspectJ的加载时织入](#aop-aj-ltw) 将讨论在Spring的应用中使用AspectJ涉及的编织时机的讨论。

<a id="aop-atconfigurable"></a>

#### [](#aop-atconfigurable)5.10.1. 使用Spring中的AspectJ独立注入域对象

Spring容器实例化和配置会在应用程序上下文中定义bean。也可以让bean工厂配置预先存在的对象，给定一个包含要应用的配置的bean定义名称。`spring-aspects.jar` 包含了注解驱动的切面， 利用这个功能来允许依赖注入到任意对象。该支持旨在用于在创建任何容器控制之外的对象。域对象通常属于这一类，因为它们通常是使用`new`的操作符以编程方式创建的，或由ORM工具为数据库查询的结果创建的。

`@Configurable`注解标记一个类符合Spring驱动配置的条件，在最简单的情况下，您可以纯粹使用它作为标记注解，如下例所示:

    package com.xyz.myapp.domain;

    import org.springframework.beans.factory.annotation.Configurable;

    @Configurable
    public class Account {
        // ...
    }

作为这样一个标识接口, Spring将会为这个注解类型（在例子中是`Account`）利用定义bean的方式（典型的原型作用域）配置一个新实例， 这个实例拥有与完全限定类型相同的名字(`com.xyz.myapp.domain.Account`)。因为一个bean的默认名称是它的类型的完全限定名，这个简便的方式只是省略了它的 `id`属性。如以下示例所示:

    <bean class="com.xyz.myapp.domain.Account" scope="prototype">
        <property name="fundsTransferService" ref="fundsTransferService"/>
    </bean>

如果想要显式指定为原型bean使用的名称，可以直接在注解执行此操作，如以下示例所示:

    package com.xyz.myapp.domain;

    import org.springframework.beans.factory.annotation.Configurable;

    @Configurable("account")
    public class Account {
        // ...
    }

Spring现在查找名为`account` 的bean定义，并将其用作配置新`Account`实例的定义。

也可以使用自动装配以避免指定一个特定的专用bean定义。Spring将利用`@Configurable`注解的自动装配属性来自动装配bean，可以使用`@Configurable(autowire=Autowire.BY_NAME`或者 `@Configurable(autowire=Autowire.BY_TYPE)`分别自动装配基于名称和基于类型的bean。另外，Spring 2.5之后明确地指定了更好的策略， 在类中有`@Configurable`注解的bean上，其域或方法级别上使用`@Autowired`或`@Inject`能够使用注解驱动的依赖注入。 有关更多详细信息，请参阅[基于注解的容器配置](#beans-annotation-config)。

最后，可以使用Spring依赖的名为`dependencyCheck`的特性去检查新建的对象引用以及配置对象（例如， `@Configurable(autowire=Autowire.BY_NAME,dependencyCheck=true)`） 。如果将此特性设置为 `true`，那么Spring将在配置之后确认所有属性（非原始或集合）已被设置。

当然，使用注解本身没有任何作用。这是 `spring-aspects.jar`包中的`AnnotationBeanConfigurerAspect`注解的存在行为。实质上， 该切面表达的是，一个带有`@Configurable`注解类型的新对象在初始化返回之后，按照注解的属性使用Spring配置创建新的对象。在这种情况下，初始化是指新实例化的对象（例如， 用`new` 运算符实例化的对象）以及正在经历反序列化（例如，通过[readResolve()](https://docs.oracle.com/javase/6/docs/api/java/io/Serializable.html)）的可序列化对象。

上一段的一个关键短语是 “实质”.。在大多数情况下，精确的语义从一个新对象初始化后返回是适合的。"初始化后"意味着依赖将会在对象被构建完毕后注入 ， 这意味着依赖在类构造器当中是不能使用的。如果想依赖的注入发生在构造器执行之前，而且能够用在构造器之中，那么需要像下面这样声明 `@Configurable`：

    @Configurable(preConstruction=true)

您可以在 [本附录](https://www.eclipse.org/aspectj/doc/next/progguide/semantics-joinPoints.html) 中 [AspectJ编程指南](https://www.eclipse.org/aspectj/doc/next/progguide/index.html)一书中找到更多有关AspectJ的信息

这个注解类型必须使用AspectJ编织织入才可以工作 ， 开发者可以使用构建组件Ant或Maven来完成这个任务（[AspectJ Development Environment Guide](https://www.eclipse.org/aspectj/doc/released/devguide/antTasks.html)有参考例子），或者在装配时织入（请参考 [在Spring框架中使用AspectJ的加载时织入](#aop-aj-ltw)）。`AnnotationBeanConfigurerAspect`注解本身需要Spring来配置（为了获取一个bean工厂引用，被用于配置新的对象）。如果使用基于Java的配置， 那么只需将`@EnableSpringConfigured` 注解加入到任意的`@Configuration`类中即可，如下所示:

    @Configuration
    @EnableSpringConfigured
    public class AppConfig {

    }

如果基于XML配置，那么只要在Spring [`context`](#xsd-schemas-context)的命名空间声明中添加`context:spring-configured`。您可以按如下方式使用它：

    <context:spring-configured/>

在配置切面之前创建`@Configurable`对象的实例将会向调试日志发消息，并且不会对该对象进行配置。一个例子是在Spring配置中的一个bean，它在Spring初始化时创建域对象。 在这种情况下，可以使用`depends-on`bean属性来手动指定bean依赖的切面配置。以下示例显示了如何使用`depends-on`属性：

    <bean id="myService"
            class="com.xzy.myapp.service.MyService"
            depends-on="org.springframework.beans.factory.aspectj.AnnotationBeanConfigurerAspect">

        <!-- ... -->

    </bean>

不用通过bean的切面配置来激活`@Configurable`处理过程，除非真的想在运行中依赖其语义。特别地，不要在一个已经在容器上注册过的Spring bean上去再去使用`@Configurable`注解。 否则，这个bean将会被初始化两次，容器一次，切面一次。

<a id="aop-configurable-testing"></a>

##### [](#aop-configurable-testing)单元测试`@Configurable`的对象

开启`@Configurable`支持的一个目标就是使单元测试独立于域对象，从而没有碰到诸如硬编码查找一样的困难。如果`@Configurable`注解没有使用AspectJ织入那么它就不会对单元测试造成影响， 这样就可以正常地进行mock或stub测试。如果`@Configurable`是使用AspectJ织入的，那么依然可以在容器之外正常地进行单元测试，但是如果每次都构建一个`@Configurable`对象都会看到警告消息， 它表示此配置并非Spring的配置。

<a id="aop-configurable-container"></a>

##### [](#aop-configurable-container)多个应用上下文一起工作

`AnnotationBeanConfigurerAspect`类在AspectJ中用来实现`@Configurable`支持的单个切面。单个切面的作用域与静态成员的作用域是相同的， 也就是说每一个类加载器都会定义这个切面的实例类型。这意味着，如果使用相同的类加载器层来定义多个应用上下文。那么必须考虑在哪儿定义`@EnableSpringConfigured` bean以及在哪个路径存放 `spring-aspects.jar`包。

考虑一个典型的Spring Web应用程序配置，其中有一个共享的父应用上下文，定义公共业务服务和支持它们所需的所有内容，每个Servlet包含一个子应用上下文， 其中包含特定于Servlet的定义。所有这些上下文共存于相同的类加载器层次，所以`AnnotationBeanConfigurerAspect`能够持有他们之中的一个的引用。在这种情况下， 建议在共享的（父）应用上下文上使用`@EnableSpringConfigured` bean定义，这个定义的服务， 可能想注入到域对象中。结果是，开发者不能在子上下文（特定的Servlet） 中使用@Configurable去定义域对象的引用bean（也许并不想做些什么）。

在同一个容器部署多个Web应用程序时，确保每个Web应用程序加载`spring-aspects.jar`类型是在使用自己的加载器引用（例如，通过 `'WEB-INF/lib'`）。如果`spring-aspects.jar`仅在容器的类路径下（也就是装在父母共享的加载器的引用），所有的Web应用程序将共享相同的切面实例，而这可能不是你想要的。

<a id="aop-ajlib-other"></a>

#### [](#aop-ajlib-other)5.10.2. 在Spring中使用的AspectJ额外的切面

除了`@Configurable`切面，`spring-aspects.jar`还包含AspectJ切面，可以用来驱动Spring的事务管理，用于注解带`@Transactional`注解的类型和方法 。这主要是为那些希望在Spring容器之外使用Spring框架的事务支持的用户而设计的。

解析 `@Transactional` 注解的切面是 `AnnotationTransactionAspect`。当使用这个切面时，必须注解这个实现类（和/或在类的方法上），不是接口（如果有的话） 的实现类。AspectJ遵循Java的规则，注解的接口不能被继承。

`@Transactional`注解的类指定默认的事务语义的各种公共操作的类.

在类的方法上注解`@Transactional`将会覆盖由给定默认事务语义的注解（如果存在），任意可见性的方法都可以被注解，包括私有方法。直接注解非公共方法是获得执行此类方法的事务划分的唯一方法。

从Spring Framework 4.2开始，`spring-aspects`提供了类似的切面，为标准的`javax.transaction.Transactional`注解提供了完全相同的功能。 查看`JtaAnnotationTransactionAspect` 获取更多细节

对于希望使用Spring配置和事务管理支持但不希望（或不能）使用注解的AspectJ程序员， `spring-aspects.jar`还包含可以扩展以提供自定义切点定义的抽象切面。 有关更多信息，请参阅`AbstractBeanConfigurerAspect`和`AbstractTransactionAspect`切面的源码。 作为示例，以下摘录显示了如何使用与完全限定的类名匹配的原型bean定义来编写一个切面 ，用于配置域模型中定义的所有对象实例:

    public aspect DomainObjectConfiguration extends AbstractBeanConfigurerAspect {

        public DomainObjectConfiguration() {
            setBeanWiringInfoResolver(new ClassNameBeanWiringInfoResolver());
        }

        // the creation of a new bean (any object in the domain model)
        protected pointcut beanCreation(Object beanInstance) :
            initialization(new(..)) &&
            SystemArchitecture.inDomainModel() &&
            this(beanInstance);

    }

<a id="aop-aj-configure"></a>

#### [](#aop-aj-configure)5.10.3. 使用Spring IoC配置AspectJ切面

当在Spring应用中使用AspectJ的切面时，很自然的希望能够使用Spring来配置切面。AspectJ运行时本身是负责创建和配置切面的， AspectJ通过Spring创建切面取决于AspectJ实例化模型的方法（`per-xxx`引起的）的切面使用。

多数的AspectJ切面是单例切面。这些切面的配置非常容易，只需正常地创建一个bean定义引用切面的类型，包含bean属性`factory-method="aspectOf"` 。这保证了Spring获得的是AspectJ的实例而不是试图创建实例本身的切面。下示例显示如何使用 `factory-method="aspectOf"`属性：

    <bean id="profiler" class="com.xyz.profiler.Profiler"
            factory-method="aspectOf"> (1)

        <property name="profilingStrategy" ref="jamonProfilingStrategy"/>
    </bean>

**(1)。**请注意`factory-method="aspectOf"` 属性

非单例切面很难配置，但是这样做也是有可能的，通过创建原型bean的定义和从`spring-aspects.jar`中使用`@Configurable`的支持。这些工作需要在AspectJ运行之后在创建之中去配置切面实例才能成功。

如果想要使用AspectJ编写一些@AspectJ切面（例如，针对领域模型类型使用加载时编织）以及希望与Spring AOP一起使用的其他@AspectJ切面，并且这些切面都使用Spring进行配置 。 那么需要告诉Spring AOP @AspectJ自动代理支持在配置中定义的@AspectJ方面的确切子集应该用于自动代理。可以通过在`<aop:aspectj-autoproxy/>`元素中声明使用一个或多个 `<include/>`元素来完成此操作。 每个 `<include/>`元素指定一个名称模式，并且只有名称与至少一个模式相匹配的bean才会用于Spring AOP自动代理配置。以下示例显示了如何使用`<include/>`元素：

    <aop:aspectj-autoproxy>
        <aop:include name="thisBean"/>
        <aop:include name="thatBean"/>
    </aop:aspectj-autoproxy>

不要被 `<aop:aspectj-autoproxy/>` 元素的名称误导。 使用它会导致创建Spring AOP代理。 切面声明的@AspectJ方式只是在这里使用，AspectJ运行时是没有用到的。

<a id="aop-aj-ltw"></a>

#### [](#aop-aj-ltw)5.10.4. 在Spring框架中使用AspectJ的加载时织入

是指AspectJ切面在JVM加载类文件时被织入到程序的类文件的过程。本部分的重点是配置和使用LTW在Spring框架上的具体内容，本节不是LTW的简介。 只有AspectJ能够详细地讲述LTW的特性和配置（与Spring完全没有关系），可以参看[LTW section of the AspectJ Development Environment Guide](https://www.eclipse.org/aspectj/doc/released/devguide/ltw.html)。

Spring框架在AspectJ的LTW织入的过程中提供了更细粒度的控制，'Vanilla' AspectJ LTW是一个高效的使用Java（1.5+)的代理，它会在JVM启动的时候改变一个VM参数。 这是一种JVM范围的设置，在某些情况下可能会很适合，但是太粗粒度了。Spring的LTW能够为LTW提供类加载前的织入，显然这是一个更细粒度的控制，而且它在'single-JVM-multiple-application' 的环境下更具意义（在典型的应用程序服务器环境中就是这样做的）。

此外，在特定的环境中（查看[in certain environments](#aop-aj-ltw-environments)），这种方式可以在对应用程序服务器运行脚本不做任何修改的情形下支持LTW， 但需要添加`-javaagent:path/to/aspectjweaver.jar`（本节稍后将会描述）或`-javaagent:path/to/org.springframework.instrument-{version}.jar`（原名为 `spring-agent.jar`）。 开发人员只需修改构成应用程序上下文的一个或多个文件，以启用加载时编入，而不是依赖通常负责部署配置的管理文件。例如启动脚本。

到此为止，推销宣传部分已经结束了，那么让我们首先介绍使用Spring的AspectJ LTW的快速示例，然后详细介绍示例中介绍的元素。 有关完整示例，请参阅[Petclinic示例应用程序](https://github.com/spring-projects/spring-petclinic)。

<a id="aop-aj-ltw-first-example"></a>

##### [](#aop-aj-ltw-first-example)第一个例子

假设您是一名应用程序开发人员，负责诊断系统中某些性能问题的原因。我们无需打开一个分析工具，而是要打开一个简单的剖析切面，让我们能够很快获得了一些性能指标， 这样我们就可以在随后立即使用更细粒度的分析工具。

这里介绍的例子使用XML格式的配置，也可以使用 [Java配置](#beans-java)和和@AspectJ的方式。特别是`@EnableLoadTimeWeaving`注解可以起 到替代`<context:load-time-weaver/>` （详情[见下文](#aop-aj-ltw-spring)）。

下面是一个用于性能分析的切面，它不需要太花哨，它是一个基于时间的分析器，它使用@ AspectJ样式的方面声明:

    package foo;

    import org.aspectj.lang.ProceedingJoinPoint;
    import org.aspectj.lang.annotation.Aspect;
    import org.aspectj.lang.annotation.Around;
    import org.aspectj.lang.annotation.Pointcut;
    import org.springframework.util.StopWatch;
    import org.springframework.core.annotation.Order;

    @Aspect
    public class ProfilingAspect {

        @Around("methodsToBeProfiled()")
        public Object profile(ProceedingJoinPoint pjp) throws Throwable {
            StopWatch sw = new StopWatch(getClass().getSimpleName());
            try {
                sw.start(pjp.getSignature().getName());
                return pjp.proceed();
            } finally {
                sw.stop();
                System.out.println(sw.prettyPrint());
            }
        }

        @Pointcut("execution(public * foo..*.*(..))")
        public void methodsToBeProfiled(){}
    }

此外还需要创建一个`META-INF/aop.xml` 文件，它将通知AspectJ将`ProfilingAspect`织入到类中。这是文件的惯例， 即在Java类路径中存在名为`META-INF/aop.xml`的文件（或多个文件）是标准AspectJ。 以下示例显示了`aop.xml` 文件:

    <!DOCTYPE aspectj PUBLIC "-//AspectJ//DTD//EN" "http://www.eclipse.org/aspectj/dtd/aspectj.dtd">
    <aspectj>

        <weaver>
            <!-- only weave classes in our application-specific packages -->
            <include within="foo.*"/>
        </weaver>

        <aspects>
            <!-- weave in just this aspect -->
            <aspect name="foo.ProfilingAspect"/>
        </aspects>

    </aspectj>

现在来配置的Spring特定部分。 我们需要配置`LoadTimeWeaver`（稍后解释）。LTW是从一个或多个`META-INF/aop.xml` 文件中织入到应用类的切面配置的主要部分。幸运的是它不需要大量的配置，如下所示（还有一些选项可以指定，但是后面会详细介绍）。如以下示例所示:

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:context="http://www.springframework.org/schema/context"
        xsi:schemaLocation="
            http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/context
            http://www.springframework.org/schema/context/spring-context.xsd">

        <!-- a service object; we will be profiling its methods -->
        <bean id="entitlementCalculationService"
                class="foo.StubEntitlementCalculationService"/>

        <!-- this switches on the load-time weaving -->
        <context:load-time-weaver/>
    </beans>

现在所有必需的材料( aspect, `META-INF/aop.xml`文件, Spring 的配置) 都已到位，我们可以使用`main(..)`方法创建以下驱动程序类，以演示LTW的运行情况：

    package foo;

    import org.springframework.context.support.ClassPathXmlApplicationContext;

    public final class Main {

        public static void main(String[] args) {

            ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml", Main.class);

            EntitlementCalculationService entitlementCalculationService
                = (EntitlementCalculationService) ctx.getBean("entitlementCalculationService");

            // the profiling aspect is 'woven' around this method execution
            entitlementCalculationService.calculateEntitlement();
        }
    }

我们还有最后一件事要做。 本节的介绍确实说可以使用Spring在每个ClassLoader的基础上有选择地打开LTW，这是事实。 但是，对于此示例，我们使用Java代理（随Spring提供）来打开LTW。 我们使用以下命令来运行前面显示的 `Main`类：

java -javaagent:C:/projects/foo/lib/global/spring-instrument.jar foo.Main

`-javaagent`是一个标志，用于指定和启用 [代理程序来检测在JVM上运行的程序](https://docs.oracle.com/javase/6/docs/api/java/lang/instrument/package-summary.html)。Spring Framework附带了一个代理程序`InstrumentationSavingAgent`， 它包装在`spring-instrument.jar`中，它作为前面示例中-javaagent参数的值提供。

主程序的输出将如下所示。（前面已经介绍了`Thread.sleep(..)`声明为 `calculateEntitlement()`实现使分析器实际上捕获了比0毫秒更多的东西（`01234`毫秒不是AOP引入的开销） ）下面的清单显示了输出 我们运行我们的探查器时得到了:

```
Calculating entitlement

StopWatch 'ProfilingAspect': running time (millis) = 1234
\-\-\-\-\-\- \-\-\-\-\- \-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-
ms     %     Task name
\-\-\-\-\-\- \-\-\-\-\- \-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-
01234  100%  calculateEntitlement
```

由于LTW是会对AspectJ产生影响的，而不是仅仅局限在Spring的beans。在 `Main`程序的轻微变化会产生相同的结果:

    package foo;

    import org.springframework.context.support.ClassPathXmlApplicationContext;

    public final class Main {

        public static void main(String[] args) {

            new ClassPathXmlApplicationContext("beans.xml", Main.class);

            EntitlementCalculationService entitlementCalculationService =
                new StubEntitlementCalculationService();

            // the profiling aspect will be 'woven' around this method execution
            entitlementCalculationService.calculateEntitlement();
        }
    }

请注意，在前面的程序中，我们如何引导Spring容器，然后在Spring的上下文之外创建一个新的`StubEntitlementCalculationService`实例。 分析通知依然会被编织。

不可否认，这个例子很简单。但是在Spring中支持LTW的基础都介绍到了，而且为什么使用以及怎样使用配置在后面的章节也将解释。

在这个例子中使用的`ProfilingAspect`可能很基础的，但它非常有用。是一个开发者可以使用在开发过程中使用开发时间切面的例子， 然后很容易地排除来自应用程序被部署到测试或生产中的因素。

<a id="aop-aj-ltw-the-aspects"></a>

##### [](#aop-aj-ltw-the-aspects)切面

在LTW使用的aspects必须是AspectJ的切面。它们可以写在AspectJ语言本身也可以在@AspectJ方式声明。这意味着aspects在AspectJ和Spring AOP的切面都有效。 此外，编译切面的类需要包含在类路径中。

<a id="aop-aj-ltw-aop_dot_xml"></a>

##### [](#aop-aj-ltw-aop_dot_xml)'META-INF/aop.xml'

使用AspectJ LTW的基础设施是一个或多个`META-INF/aop.xml`配置文件，这是在Java类路径中的（直接的或者更通常是一个JAR文件）。

LTW部分 [AspectJ参考文档](https://www.eclipse.org/aspectj/doc/released/devguide/ltw-configuration.html)中详细介绍了此文件的结构和内容。 由于aop.xml文件是100％AspectJ，因此我们不在此进一步描述。

<a id="aop-aj-ltw-libraries"></a>

##### [](#aop-aj-ltw-libraries)需要的类库(JARS)

至少，您需要以下库来使用Spring Framework对AspectJ LTW的支持:

*   `spring-aop.jar` (version 2.5 or later, plus all mandatory dependencies)

*   `aspectjweaver.jar` (version 1.6.8 or later)


如果使用[Spring提供的代理程序启用检测](#aop-aj-ltw-environment-generic)，则还需要：

*   `spring-instrument.jar`


<a id="aop-aj-ltw-spring"></a>

##### [](#aop-aj-ltw-spring)Spring的配置

Spring支持LTW的关键部件是 `LoadTimeWeaver`接口（位于`org.springframework.instrument.classloading`包），而这接口有大部分的实现分布在Spring中。 `LoadTimeWeaver`负责添加一个或多个`java.lang.instrument.ClassFileTransformers` 到运行时的类装载器中。 这为各种有趣的应用程序打开了大门，其中一个恰好是方面的LTW。

如果您不熟悉运行时类文件转换的概念，请在继续之前查看`java.lang.instrument` 包的javadoc API文档。虽然该文档并不全面，但至少可以看到关键接口和类（供您阅读本节时参考）。

配置一个特定的`ApplicationContext` `LoadTimeWeaver`就像加入一行代码一样容易。（请注意，几乎可以肯定会将`ApplicationContext`作为的Spring容器- 通常一个`BeanFactory`是不够的，因为LTW的支持利用到`BeanFactoryPostProcessors`）。

要启用Spring Framework的LTW支持，您需要配置`LoadTimeWeaver`，通常使用`@EnableLoadTimeWeaving`注解来完成，如下所示：

    @Configuration
    @EnableLoadTimeWeaving
    public class AppConfig {

    }

或者，如果您更喜欢基于XML的配置，请使用`<context:load-time-weaver/>`元素。请注意，元素是在 `context`命名空间中定义的。 以下示例显示如何使用`<context:load-time-weaver/>`：

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:context="http://www.springframework.org/schema/context"
        xsi:schemaLocation="
            http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/context
            http://www.springframework.org/schema/context/spring-context.xsd">

        <context:load-time-weaver/>

    </beans>

上面的配置自动为你登记了一些特定的基础beans，例如`LoadTimeWeaver`和`AspectJWeavingEnabler`。 默认的`LoadTimeWeaver`是`DefaultContextLoadTimeWeaver` 类，它试图装饰并自动检测`LoadTimeWeaver`。 “自动检测”的`LoadTimeWeaver`的确切类型取决于您的运行时环境。 下表总结了各种`LoadTimeWeaver`实现:

Table 13. DefaultContextLoadTimeWeaver LoadTimeWeavers

| 运行时环境                                                   | `LoadTimeWeaver` 实现           |
| ------------------------------------------------------------ | ------------------------------- |
| Running in Oracle’s [WebLogic](http://www.oracle.com/technetwork/middleware/weblogic/overview/index-085209.html) | `WebLogicLoadTimeWeaver`        |
| Running in Oracle’s [GlassFish](http://glassfish.dev.java.net/) | `GlassFishLoadTimeWeaver`       |
| Running in [Apache Tomcat](https://tomcat.apache.org/)       | `TomcatLoadTimeWeaver`          |
| Running in Red Hat’s [JBoss AS](http://www.jboss.org/jbossas/) or [WildFly](http://www.wildfly.org/) | `JBossLoadTimeWeaver`           |
| Running in IBM’s [WebSphere](https://www-01.ibm.com/software/webservers/appserv/was/) | `WebSphereLoadTimeWeaver`       |
| JVM started with Spring `InstrumentationSavingAgent` (`java -javaagent:path/to/spring-instrument.jar`) | `InstrumentationLoadTimeWeaver` |
| Fallback, expecting the underlying ClassLoader to follow common conventions (for example applicable to `TomcatInstrumentableClassLoader` and [Resin](http://www.caucho.com/)) | `ReflectiveLoadTimeWeaver`      |

请注意，该表仅列出使用`DefaultContextLoadTimeWeaver`时自动检测的 `LoadTimeWeavers`。 您可以准确指定要使用的`LoadTimeWeaver`实现。

使用Java配置指定特定的 `LoadTimeWeaver`实现`LoadTimeWeavingConfigurer` 接口并覆盖 `getLoadTimeWeaver()` 方法。以下示例指定`ReflectiveLoadTimeWeaver`：

    @Configuration
    @EnableLoadTimeWeaving
    public class AppConfig implements LoadTimeWeavingConfigurer {

        @Override
        public LoadTimeWeaver getLoadTimeWeaver() {
            return new ReflectiveLoadTimeWeaver();
        }
    }

如果使用基于XML的配置，则可以将完全限定的类名指定为`<context:load-time-weaver/>`元素上的 `weaver-class`属性的值。 同样，以下示例指定了 `ReflectiveLoadTimeWeaver`:

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:context="http://www.springframework.org/schema/context"
        xsi:schemaLocation="
            http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd
            http://www.springframework.org/schema/context
            http://www.springframework.org/schema/context/spring-context.xsd">

        <context:load-time-weaver
                weaver-class="org.springframework.instrument.classloading.ReflectiveLoadTimeWeaver"/>

    </beans>

稍后可以使用众所周知的名称 `loadTimeWeaver`从Spring容器中检索由配置定义和注册的`LoadTimeWeaver` 。请记住， `LoadTimeWeaver`只是作为Spring的LTW基础结构的机制来添加一个或多个`ClassFileTransformer`，执行LTW的实际`ClassFileTransformers`是 `ClassPreProcessorAgentAdapter`（来自`org.aspectj.weaver.loadtime`包）。有关详细信息，请参阅`ClassPreProcessorAgentAdapter`类的类级javadoc， 因为编织实际如何实现的细节超出了本文档的范围。

剩下要讨论的配置有一个最终属性：`aspectjWeaving`属性（如果使用XML，则为`aspectj-weaving`）。 此属性控制是否启用LTW。 它接受三个可能值中的一个，如果该属性不存在，则默认值为`autodetect`。 下表总结了三个可能的值:

Table 14. AspectJ织入的属性值

| Annotation Value | XML Value    | Explanation                                                  |
| ---------------- | ------------ | ------------------------------------------------------------ |
| `ENABLED`        | `on`         | AspectJ编织开启，切面在加载时织入。                          |
| `DISABLED`       | `off`        | LTW已关闭。 没有切面加载时织入。                             |
| `AUTODETECT`     | `autodetect` | 如果Spring LTW基础结构可以找到至少一个`META-INF/aop.xml`文件，那么AspectJ编织就会打开。 否则，它关闭。 这是默认值。 |

<a id="aop-aj-ltw-environments"></a>

##### [](#aop-aj-ltw-environments)特定环境的配置

最后一部分包含在应用程序服务器和Web容器等环境中使用Spring LTW支持时所需的任何其他设置和配置。

<a id="aop-aj-ltw-environment-tomcat"></a>

###### [](#aop-aj-ltw-environment-tomcat)Tomcat

从历史上看, [Apache Tomcat](https://tomcat.apache.org/)的默认类加载器不支持类转换，这就是为什么Spring提供了一个增强的实现来满足这一需求。 名字叫`TomcatInstrumentableClassLoader`，加载程序适用于Tomcat 6.0及更高版本。

不要在Tomcat 8.0及更高版本上定义`TomcatInstrumentableClassLoader` 。 相反，让Spring通过`TomcatLoadTimeWeaver` 策略自动使用Tomcat的新的，原生的`InstrumentableClassLoader`工具。

如果仍需要使用`TomcatInstrumentableClassLoader`，则可以为每个Web应用程序单独注册，如下所示:

1.  将`org.springframework.instrument.tomcat.jar`复制到`$CATALINA_HOME/lib`中，其中`$CATALINA_HOME`表示Tomcat安装的根目录

2.  通过编辑Web应用程序上下文文件，指示Tomcat使用自定义类加载器（而不是默认值），如以下示例所示:


    <Context path="/myWebApp" docBase="/my/webApp/location">
        <Loader
            loaderClass="org.springframework.instrument.classloading.tomcat.TomcatInstrumentableClassLoader"/>
    </Context>

Apache Tomcat 6.0+支持多个上下文位置:

*   服务配置文件 : `$CATALINA_HOME/conf/server.xml`

*   默认上下文配置 : `$CATALINA_HOME/conf/context.xml`, which affects all deployed web applications

*   每个应用程序的配置, 可以在服务器端的`$CATALINA_HOME/conf/[enginename]/[hostname]/[webapp]-context.xml`上部署也可以嵌入在web应用程序`META-INF/context.xml`中


为了提高效率，建议使用嵌入式Web应用程序配置风格，因为它只影响使用自定义类装入器的应用程序，不需要对服务器配置进行任何更改。 有关可用上下文位置的更多详细信息，请参阅Tomcat 6.0.x[文档](https://tomcat.apache.org/tomcat-6.0-doc/config/context.html)。

或者，考虑使用Spring提供的通用VM代理，在Tomcat的启动脚本中指定（在本节前面介绍过）。 这将使功能适用于所有部署的Web应用程序，无论它们恰好运行在哪个`ClassLoader`上。

<a id="expressions"></a>

###### [](#aop-aj-ltw-environments-weblogic-oc4j-resin-glassfish-jboss)WebLogic, WebSphere, Resin, GlassFish, and JBoss

最新版本的WebLogic Server（版本10及更高版本），IBM WebSphere Application Server（版本7及更高版本），Resin（版本3.1及更高版本）和JBoss（版本6.x或更高版本） 提供了一个能够进行本地检测的ClassLoader。Spring的原生LTW利用这种ClassLoader实现来实现AspectJ织入。 [如前所述](#aop-using-aspectj)，您可以通过激活加载时织入来启用LTW。 具体来说，您无需修改启动脚本即可添加`-javaagent:path/to/spring-instrument.jar`。

请注意，具有GlassFish功能的`ClassLoader` 仅在其EAR环境中可用。对于GlassFish Web应用程序，请按照[上面概述](#aop-aj-ltw-environment-tomcat)的Tomcat设置说明进行操作。 .

注意在JBoss 6.x中， 应用程序服务器的扫描需要禁用，防止它加载的类的应用之前实际上已经开始。快速的解决方案是增加一个叫`WEB-INF/jboss-scanning.xml`的文档并加入以下内容：

    <scanning xmlns="urn:jboss:scanning:1.0"/>

<a id="aop-aj-ltw-environment-generic"></a>

###### [](#aop-aj-ltw-environment-generic)通用的Java应用

在不支持现有`LoadTimeWeaver`实现或不受现有`LoadTimeWeaver`实现支持的环境中需要类检测时，使用JDK代理可能是唯一的解决方案。对于这种情况， Spring提供了`InstrumentationLoadTimeWeaver`，它需要Spring特有的（但也是非常普通的）VM代理包`org.springframework.instrument-{version}.jar`(以前称为 `spring-agent.jar`).

要使用它，必须通过提供以下JVM选项来启动带有Spring代理的虚拟机。:

-javaagent:/path/to/org.springframework.instrument-{version}.jar

这需要修改VM启动脚本，这可能会阻止在应用服务器环境中使用它（具体取决于操作策略）。此外，JDK代理可以检测整个VM，这可能很昂贵。。

出于性能原因，我们建议您仅在目标环境（例如[Jetty](https://www.eclipse.org/jetty/)）没有（或不支持）专用LTW时才使用此配置。

<a id="aop-resources)"></a>

### [](#aop-resources)5.11. 更多资源

有关AspectJ的更多信息可以在[AspectJ website](https://www.eclipse.org/aspectj)上找到。

_Eclipse AspectJ_的书Eclipse AspectJ (Addison-Wesley, 2005) 提供了详尽的有关AspectJ语言的介绍

_AspectJ in Action_, 一书的第二版由Ramnivas Laddad（Manning，2009)出版，也是强烈推荐的。这本书的重点是AspectJ，但也在一定的深度上探讨了普通的AOP主题。