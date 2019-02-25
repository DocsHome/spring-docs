
<a id="aop"></a>


[](#aop-api)6\. Spring AOP APIs
-------------------------------

前一章介绍了Spring使用@AspectJ和基于schema的切面定义对AOP的支持。 在本章中，将讨论Spring 1.2应用程序中使用的较底层的Spring AOP API和AOP支持。 对于新的应用程序，推荐使用前一章中介绍的Spring 2.0和更高版本的AOP支持，但是在使用现有应用程序或阅读书籍和文章时，您可能会遇到Spring 1.2方式的示例. Spring 5仍然向后兼容了Spring 1.2。本章中描述的所有内容在Spring 5中都得到了完全支持。


<a id="aop-api-pointcuts"></a>


### [](#aop-api-pointcuts)6.1. Spring中的切点API

本节描述了Spring如何处理切点的关键概念。


<a id="op-api-concepts"></a>


#### [](#aop-api-concepts)6.1.1. 概念

Spring的切点模式能够让切点独立于通知类型。针对不同的通知使用相同的切点是可能的。

`org.springframework.aop.Pointcut` 接口是切点的主要接口，用于特定类和方法的目标通知。完整的接口如下:

    public interface Pointcut {

        ClassFilter getClassFilter();

        MethodMatcher getMethodMatcher();

    }

将`Pointcut`接口分成两部分，允许重用类和方法匹配部分，以及细粒度的组合操作（例如与另一个方法匹配器执行“union”）。

`ClassFilter`接口是用来限制切点的一组给定的目标类。如果`matches()`方法总是返回true，那么表示所有的目标类都将匹配。以下清单显示了`ClassFilter`接口定义:

    public interface ClassFilter {

        boolean matches(Class clazz);
    }

`MethodMatcher` 接口通常更重要。完整的接口如下所示:

    public interface MethodMatcher {

        boolean matches(Method m, Class targetClass);

        boolean isRuntime();

        boolean matches(Method m, Class targetClass, Object[] args);
    }

`matches(Method, Class)`方法用于测试此切点是否曾经匹配到目标类上的给定方法。在创建AOP代理时可以执行此评估，以避免需要对每个方法调用进行测试。 如果对于给定方法，这个双参数 `matches`方法返回`true`，并且MethodMatcher的 `isRuntime()`方法也返回true。 则在每次方法调用时都会调用三参数`matches`方法。这使切点能够在目标通知执行之前，查看传递给方法调用的参数。

大多数`MethodMatcher`实现都是静态的，这意味着它们的 `isRuntime()`方法返回`false`。 在这种情况下，永远不会调用三参数`matches`方法。

如果可以，请尝试将切点设为静态的，从而允许AOP框架在创建AOP代理时缓存对切点评估的结果。


<a id="aop-api-pointcut-ops"></a>


#### [](#aop-api-pointcut-ops)6.1.2. 切点的操作

Spring支持对切点的各种操作，特别是并集和交集

并集意味着这个方法只要有一个切点匹配，交集意味着这个方法需要所有的切点都匹配。 并集使用得更广，您可以使用`org.springframework.aop.support.Pointcuts`类中的静态方法或在同一个包中使用`ComposablePointcut`类来组合切 点。但是，使用AspectJ的切点表达式往往是更简单的方式。


<a id="aop-api-pointcuts-aspectj"></a>


#### [](#aop-api-pointcuts-aspectj)6.1.3. AspectJ切点表达式

自 2.0以来, ，Spring使用的最重要的切点类型是`org.springframework.aop.aspectj.AspectJExpressionPointcut`。这是一个使用AspectJ提供的库来解析AspectJ切点表达式字符串的切点。

有关支持的AspectJ切点语义的讨论， [请参见上一章](#aop)

<a id="aop-api-pointcuts-impls"></a>

#### [](#aop-api-pointcuts-impls)6.1.4.方便的切点实现

Spring提供了几个方便的切点实现，您可以直接使用其中一些。其他的目的是在特定于应用程序的切点中进行子类化。 Others are intended to be subclassed in application-specific pointcuts.

<a id="aop-api-pointcuts-static"></a>

##### [](#aop-api-pointcuts-static)静态切点

静态切点是基于方法和目标类的，而且无法考虑该方法的参数。静态切点在大多数的使用上是充分的、最好的。在第一次调用一个方法时， Spring可能只计算一次静态切点，在这之后，无需在使用每个方法调用时都评估切点。

本节的其余部分描述了Spring中包含的一些静态切点实现。

<a id="aop-api-pointcuts-regex"></a>

###### [](#aop-api-pointcuts-regex)正则表达式切点

指定静态切入点的一个显而易见的实现是正则表达式，几个基于Spring的AOP框架让这成为可能。 `org.springframework.aop.support.JdkRegexpMethodPointcut`是一个通用的正则表达式切点，它使用JDK中的正则表达式支持。

使用`JdkRegexpMethodPointcut`类，可以提供一个匹配的Strings列表。如果其中任意一个都是匹配的，则切点将计算将为true（因此，结果实际上是这些切点的并集）。

以下示例显示如何使用`JdkRegexpMethodPointcut`:

    <bean id="settersAndAbsquatulatePointcut"
            class="org.springframework.aop.support.JdkRegexpMethodPointcut">
        <property name="patterns">
            <list>
                <value>.*set.*</value>
                <value>.*absquatulate</value>
            </list>
        </property>
    </bean>

Spring提供了一个方便使用的类 -`RegexpMethodPointcutAdvisor`。 它允许引用`Advice`（记住`Advice`可能是一个拦截器、前置通知、异常通知等等）。 而在这个类的后面，Spring也是使用`JdkRegexpMethodPointcut`类的。使用`RegexpMethodPointcutAdvisor`来简化织入，用作bean封装的切点和通知。如下例所示:

    <bean id="settersAndAbsquatulateAdvisor"
            class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
        <property name="advice">
            <ref bean="beanNameOfAopAllianceInterceptor"/>
        </property>
        <property name="patterns">
            <list>
                <value>.*set.*</value>
                <value>.*absquatulate</value>
            </list>
        </property>
    </bean>

您可以将`RegexpMethodPointcutAdvisor` 与任何`Advice`类型一起使用。

<a id="aop-api-pointcuts-attribute-driven"></a>

###### [](#aop-api-pointcuts-attribute-driven)基于属性的切点

静态切点的一个重要特征是元数据驱动的切点。它将使用元数据属性的值，通常是使用源等级的元数据。

<a id="aop-api-pointcuts-dynamic"></a>

##### [](#aop-api-pointcuts-dynamic)动态的切点

与静态切点相比，动态切点的评估成本更高。它们考虑了方法参数和静态信息。 这意味着必须使用每个方法调用来评估它们，并且不能缓存结果，因为参数会有所不同。

主要的例子是`control flow` 切点

<a id="aop-api-pointcuts-cflow"></a>

###### [](#aop-api-pointcuts-cflow)控制流切点

Spring控制流切点在概念上类似于AspectJ的`cflow`切点，虽然功能不够它的强大 （目前没有办法指定切点在另一个切点匹配的连接点下面执行）。 控制流切点与当前调用的栈相匹配。例如，如果连接点是由`com.mycompany.web`包中的方法或`SomeCaller`类调用的，则可能会触发它。 使用 `org.springframework.aop.support.ControlFlowPointcut`类指定控制流切点。

在运行时评估控制流切点的成本远远高于其他动态切点。 在Java 1.4中，成本大约是其他动态切入点的五倍。

<a id="aop-api-pointcuts-superclasses"></a>

#### [](#aop-api-pointcuts-superclasses)6.1.5. 切点超类

Spring提供了相当有用的切点超类,帮助开发者实现自定义切点.

因为静态切点最有用,所以可能会继承`StaticMethodMatcherPointcut`.编写子类。 这需要只实现一个抽象方法（尽管您可以覆盖其他方法来自定义行为）。 以下示例显示如何子类化 `StaticMethodMatcherPointcut`:

    class TestStaticPointcut extends StaticMethodMatcherPointcut {

        public boolean matches(Method m, Class targetClass) {
            // return true if custom criteria match
        }
    }

这也是动态切点的超类

您可以在Spring 1.0 RC2及更高版本中使用任何通知类型的自定义切点。

<a id="aop-api-pointcuts-custom"></a>



#### [](#aop-api-pointcuts-custom)6.1.6.自定义切点

由于Spring AOP中的切点是Java类,而不是语言功能(如AspectJ),因此可以声明自定义切点,无论是静态的还是动态的.Spring中的自定义切点可以是任意复杂的。 但是,尽量建议使用AspectJ切点表达式语言。

Later versions of Spring may offer support for “semantic pointcuts” as offered by JAC — for example, “all methods that change instance variables in the target object.”

<a id="aop-api-advice"></a>

### [](#aop-api-advice)6.2. Spring的通知API

接下来介绍Spring AOP是怎么样处理通知的

<a id="aop-api-advice-lifecycle"></a>

#### [](#aop-api-advice-lifecycle)6.2.1. 通知的生命周期

每个通知都是Spring bean.通知实例可以在所有通知对象之间共享，或者对每个通知对象都是唯一的。 这对应于每个类或每个实例的通知。

单类（Per-class) 通知是最常用的。它适用于诸如事务通知者之类的一般性通知。它不依赖于代理对象的状态或添加新状态，它们只是对方法和参数产生作用.

单实例（Per-instance）的通知适合于引入,以支持混合使用.在这种情况下,通知将状态添加到代理对象中。

在同一个AOP代理中，可以使用混合共享的和单实例的通知。

<a id="aop-api-advice-types"></a>

#### [](#aop-api-advice-types)6.2.2. Spring中的通知类型

Spring提供了几种通知类型，并且可以扩展以支持任意通知类型。 本节介绍基本概念和标准通知类型。

<a id="aop-api-advice-around"></a>

##### [](#aop-api-advice-around)拦截环绕通知

在Spring中,最基础的通知类型是拦截环绕通知.

Spring使用方法拦截来满足AOP`Alliance` 接口的要求. `MethodInterceptor`实现环绕通知应该实现以下接口:

    public interface MethodInterceptor extends Interceptor {

        Object invoke(MethodInvocation invocation) throws Throwable;
    }

`invoke()` 方法的参数`MethodInvocation` 公开了将要被触发的方法,目标连接点,AOP代理,以及方法的参数。`invoke()` 方法应该返回调用的结果：连接点的返回值。

以下示例显示了一个简单的`MethodInterceptor`实现:

    public class DebugInterceptor implements MethodInterceptor {

        public Object invoke(MethodInvocation invocation) throws Throwable {
            System.out.println("Before: invocation=[" + invocation + "]");
            Object rval = invocation.proceed();
            System.out.println("Invocation returned");
            return rval;
        }
    }

请注意对`MethodInvocation`的`proceed()`方法的调用。proceed从拦截器链上进入连接点。大多数拦截器调用此方法并返回其返回值。但是， 与任意的环绕通知一样， `MethodInterceptor`可以返回不同的值或引发异常，而不是调用proceed方法。但是，如果没有充分的理由，您不希望这样做。

`MethodInterceptor` 提供与其他AOP Alliance兼容的AOP实现。本节其余部分讨论的其他通知类型实现了常见的AOP概念，但这特定于使用Spring的方式。 尽管使用最具体的通知类型切面总是有优势的，但如果希望在另一个AOP框架中运行该切面面，，则应坚持使用`MethodInterceptor`的通知。请注意，目前切点不会在框架之间进行交互操作， 并且目前的AOP Alliance并没有定义切点接口。

<a id="aop-api-advice-before"></a>

##### [](#aop-api-advice-before)前置通知

前置通知是一种简单的通知，它并不需要`MethodInvocation`对象，因为它只会在执行方法前调用。

前置通知的主要优势就是它没有必要去触发`proceed()`方法，因此当拦截器链失败时对它是没有影响的。

以下清单显示了`MethodBeforeAdvice`接口:

    public interface MethodBeforeAdvice extends BeforeAdvice {

        void before(Method m, Object[] args, Object target) throws Throwable;
    }

(Spring的API设计允许前置通知使用在域上，尽管通常是适用于字段拦截的，而 Spring也不可能实现它）。

注意before方法的返回类型是`void`的。前置通知可以在连接点执行之前插入自定义行为，但不能更改返回值。如果前置通知抛出了异常， 将会中止拦截器链的进一步执行，该异常将会传回给拦截器链。如果它标记了unchecked，或者是在触发方法的签名上，那么它将直接传递给客户端。否则，它由AOP代理包装在未经检查的异常中。

以下示例显示了Spring中的前置通知，该通知计算所有方法调用:

    public class CountingBeforeAdvice implements MethodBeforeAdvice {

        private int count;

        public void before(Method m, Object[] args, Object target) throws Throwable {
            ++count;
        }

        public int getCount() {
            return count;
        }
    }

前置通知可以用在任意的切点上

<a id="aop-api-advice-throws"></a>

##### [](#aop-api-advice-throws)异常通知

异常通知是在连接点返回后触发的，前提是连接点抛出了异常。Spring提供了类型化的抛出通知。请注意，这意味着`org.springframework.aop.ThrowsAdvice`接口不包含任何方法。 它只是标识给定对象实现一个或多个类型化异常通知方法的标识接口,这些应该是以下形式:

    afterThrowing([Method, args, target], subclassOfThrowable)

这个方法只有最后一个参数是必需的。方法签名可以有一个或四个参数，具体取决于通知方法是否对方法和参数有影响。 接下来的两个列表显示了作为异常通知示例的类。.

如果抛出`RemoteException`（包括子类），则调用以下通知:

    public class RemoteThrowsAdvice implements ThrowsAdvice {

        public void afterThrowing(RemoteException ex) throws Throwable {
            // Do something with remote exception
        }
    }

与前面的通知不同，下一个示例声明了四个参数，以便它可以访问被调用的方法，方法参数和目标对象。 如果抛出`ServletException`，则调用以下通知：

    public class ServletThrowsAdviceWithArguments implements ThrowsAdvice {

        public void afterThrowing(Method m, Object[] args, Object target, ServletException ex) {
            // Do something with all arguments
        }
    }

最后的示例演示了如何在单个类中使用这两种方法,它能处理`RemoteException`和`ServletException`异常。任何数量的异常通知方法都可以在单个类中进行组合。以下清单显示了最后一个示例:

    public static class CombinedThrowsAdvice implements ThrowsAdvice {

        public void afterThrowing(RemoteException ex) throws Throwable {
            // Do something with remote exception
        }

        public void afterThrowing(Method m, Object[] args, Object target, ServletException ex) {
            // Do something with all arguments
        }
    }

如果异常通知方法引发了异常，那么它将会重写原始的异常（即更改为向用户抛出异常）。覆盖异常通常是RuntimeException，它与任何方法签名兼容。 但是，如果异常通知方法引发了checked异常，那么它必须与目标方法的已声明的异常相匹配，因此在某种程度上耦合到特定的目标方法签名。不要抛出与目标方法签名不兼容的未声明的checked异常！

异常通知可以被用在任意切点上

<a id="aop-api-advice-after-returning"></a>

##### [](#aop-api-advice-after-returning)后置返回通知

Spring中使用后置返回通知必需实现`org.springframework.aop.AfterReturningAdvice` 接口, 如下所示:

    public interface AfterReturningAdvice extends Advice {

        void afterReturning(Object returnValue, Method m, Object[] args, Object target)
                throws Throwable;
    }

后置返回通知可以访问返回值（不能修改）、调用的方法、方法参数和目标。

下面例子的后置返回通知会统计所有成功的、不引发异常的方法调用次数:

    public class CountingAfterReturningAdvice implements AfterReturningAdvice {

        private int count;

        public void afterReturning(Object returnValue, Method m, Object[] args, Object target)
                throws Throwable {
            ++count;
        }

        public int getCount() {
            return count;
        }
    }

此通知不会更改执行路径，如果抛出异常，将抛出拦截器链而不是返回值。

后置返回通知能被任何切点使用

<a id="aop-api-advice-introduction"></a>

##### [](#aop-api-advice-introduction)引入通知

Spring将引入通知看作是一种特殊的拦截器通知

引入通知需要`IntroductionAdvisor` 和`IntroductionInterceptor`，他们都实现了下面的接口:

    public interface IntroductionInterceptor extends MethodInterceptor {

        boolean implementsInterface(Class intf);
    }

从AOP Alliance `MethodInterceptor`接口继承的`invoke()`方法也都必须实现引入。即如果invoked方法是一个引入接口， 引入拦截器将会负责处理这个方法的调用-它无法触发`proceed()`。

引入通知不能与任何切点一起使用，因为它只适用于类级别，而不是方法级别。开发者只能使用`IntroductionAdvisor`的引入通知，它具有以下方法:

    public interface IntroductionAdvisor extends Advisor, IntroductionInfo {

        ClassFilter getClassFilter();

        void validateInterfaces() throws IllegalArgumentException;
    }

    public interface IntroductionInfo {

        Class[] getInterfaces();
    }

在这里如果没有与`MethodMatcher` 相关的引入通知类。也就不会有`Pointcut` 。此时，只有filtering类是符合逻辑的。

`getInterfaces()`方法返回通知者的引入接口

`validateInterfaces()`方法在内部使用，可以查看引入接口是否可以由配置的 `IntroductionInterceptor`实现。

考虑Spring测试套件中的一个示例，并假设我们要将以下接口引入一个或多个对象:

    public interface Lockable {
        void lock();
        void unlock();
        boolean locked();
    }

这个说明是混合型的。我们希望可以将无论是什么类型的通知对象都转成`Lockable`,这样可以调用它的lock和unlock方法。如果调用的是`lock()`方法，希望所有的setter方法都抛出LockedException异常。 因此，可以添加一个切面，它提供了对象不可变的能力，而不需要对它有任何了解。AOP的一个很好的例子t: a good example of AOP.

首先，我们需要一个可以完成繁重工作的`IntroductionInterceptor`。在这种情况下，我们扩展了`org.springframework.aop.support.DelegatingIntroductionInterceptor`类更方便。 我们可以直接实现`IntroductionInterceptor`，但使用`DelegatingIntroductionInterceptor`最适合大多数情况。

`DelegatingIntroductionInterceptor` 设计是为了将引入委托让给引入接口真正的实现类，从而隐藏了拦截器去做这个事。可以使用构造函数参数将委托设置为任何对象。 默认委托（当使用无参数构造函数时）时是 `this`的。 因此，在下面的示例中， 委托是`DelegatingIntroductionInterceptor` 中的`LockMixin`子类。 给定一个委托 (默认是它本身）， `DelegatingIntroductionInterceptor`实例将查找委托(非`IntroductionInterceptor`）实现的所有接口，并支持对其中任何一个的引入。子类(如 `LockMixin`）可以调用 `suppressInterface(Class intf)`方法来控制不应该公开的接口。 但是，无论 `IntroductionInterceptor`准备支持多少接口，使用`IntroductionAdvisor`都可以控制实际公开的接口。引入接口将隐藏目标对同一接口的任何实现。

因此， `LockMixin` 扩展了`DelegatingIntroductionInterceptor`并实现了`Lockable` 本身。 超类自动选择可以支持`Lockable` 引入，因此我们不需要指定。 我们可以用这种方式引入任意数量的接口。

请注意使用`locked` 实例变量，这有效地将附加状态添加到目标对象中。

以下示例显示了示例`LockMixin`类:

    public class LockMixin extends DelegatingIntroductionInterceptor implements Lockable {

        private boolean locked;

        public void lock() {
            this.locked = true;
        }

        public void unlock() {
            this.locked = false;
        }

        public boolean locked() {
            return this.locked;
        }

        public Object invoke(MethodInvocation invocation) throws Throwable {
            if (locked() && invocation.getMethod().getName().indexOf("set") == 0) {
                throw new LockedException();
            }
            return super.invoke(invocation);
        }

    }

通常，您不需要覆盖`invoke()`方法。 `DelegatingIntroductionInterceptor`实现（如果引入方法则调用`delegate`方法，否则就对连接点进行操作）通常就足够了。 在本例中，我们需要添加一个检查：如果处于锁定模式，则不能调用setter方法。

引入通知者是非常简单的，它需要做的所有事情就是持有一个独特的`LockMixin`实例，并指定引入接口 。 在例子中就是 `Lockable`。 一个更复杂的示例可能会引用引入拦截器 （被定义为原型），在这种情况下，没有与`LockMixin`相关的配置，因此我们使用`new`创建它。 以下示例显示了我们的`LockMixinAdvisor`类:

    public class LockMixinAdvisor extends DefaultIntroductionAdvisor {

        public LockMixinAdvisor() {
            super(new LockMixin(), Lockable.class);
        }
    }

我们可以非常简单地应用这个通知者，因为它不需要配置。（但是，没有`IntroductionAdvisor`就不可能使用`IntroductionInterceptor`。）与通常的引入一样， 通知者必须是个单实例（per-instance），因为它是有状态的。需要为每个通知的对象创建每一个不同的`LockMixinAdvisor`实例和`LockMixin`。通知者也包括通知对象状态的一部分

可以使用 `Advised.addAdvisor()`方法或在在XML配置中（推荐此法）编写通知者，这与其他任何的通知者一样。下面讨论的所有代理创建选项， 包括自动代理创建，都正确处理了引入和有状态的mixin。

<a id="aop-api-advisor"></a>

### [](#aop-api-advisor)6.3.Spring中通知者的API

在Spring中，一个通知者就是一个切面，一个仅包含与单个通知对象关联的切点表达式。

除了引入是一个特殊的例子外，通知者能够用于所有的通知上。`org.springframework.aop.support.DefaultPointcutAdvisor`类是最常使用的通知者类。 它可以与`MethodInterceptor`, `BeforeAdvice`或`ThrowsAdvice`一起使用。

在同一个AOP代理中，可以在Spring中混合使用通知者和通知类型。例如，可以在一个代理配置中同时使用环绕通知、异常通知和前置通知。Spring自动创建必要的拦截链。

<a id="aop-pfb"></a>

### [](#aop-pfb)6.4. 使用`ProxyFactoryBean`来创建AOP代理

如果你为业务对象使用Spring IoC容器（一个 `ApplicationContext` 或 `BeanFactory`）（同时也应该这么做！）， 那么可能希望用到其中一个Spring的AOP `FactoryBean`。 （请记住，工厂bean引入了一个间接层，让它创建一个不同类型的对象。）

Spring AOP支持也使用到了工厂bean

在Spring中创建AOP代理的基本方法是使用`org.springframework.aop.framework.ProxyFactoryBean`. 这将完全控制切点和应用的通知及顺序。 但是，如果不需要这样的控制，可以有更简单的选项。

<a id="aop-pfb-1"></a>

#### [](#aop-pfb-1)6.4.1. 基础设置

`ProxyFactoryBean`与其他Spring `FactoryBean` 的实现一样，引入了一个间接层。如果定义了一个名为`foo`的`ProxyFactoryBean`， 那么引用`foo`的对象不是`ProxyFactoryBean`实例本身，而是由`ProxyFactoryBean` 实现的`getObject()` 方法创建的对象。此方法将创建一个用于包装目标对象的AOP代理

使用`ProxyFactoryBean`或另一个IoC识别类来创建AOP代理的最重要的好处之一是，它意味着建议和切点也可以由IoC容器管理。这是一个强大的功能，能够实现其他AOP框架无法实现的方法。 例如，通知本身可以引用应用程序对象（除了目标，它应该在任何AOP框架中可用），这得益于依赖注入提供的所有可插入功能。

<a id="aop-pfb-2"></a>

#### [](#aop-pfb-2)6.4.2. JavaBean 属性

与Spring提供的大多数`FactoryBean` 实现一样，`ProxyFactoryBean`类本身就是一个JavaBean。 其属性用于:

*   指定需要代理的目标

*   指定是否使用CGLIB（稍后介绍，另请参阅[基于JDK和CGLIB的代理](#aop-pfb-proxy-types)）。


一些关键属性继承自`org.springframework.aop.framework.ProxyConfig`（Spring中所有AOP代理工厂的超类）。 这些关键属性包括以下内容：

*   `proxyTargetClass`: 如果目标类需要代理，而不是目标类的接口时，则为`true`。如果此属性值设置为true，则会创建CGLIB代理（但另请参阅 [基于JDK和CGLIB的代理](#aop-pfb-proxy-types)）。

*   `optimize`:控制是否将进一步优化使用CGLIB创建的代理。除非完全了解相关的AOP代理如何处理优化，否则不应草率地使用此设置。目前这只用于CGLIB代理，它对JDK动态代理不起作用。

*   `frozen`: 如果代理配置被`frozen`,则不再允许对配置进行更改。这既可以作为一种轻微的优化，也适用于当不希望调用方在创建代理后能够操作代理（通过`Advised`接口） 的情况。 此属性的默认值为 `false`，因此如果允许添加其他的通知的话可以更改。

*   `exposeProxy`: 确定当前代理是否应在`ThreadLocal`中公开，以便目标可以访问它。如果目标需要获取代理，并且`exposeProxy`属性设置为`true`。 则目标可以使用`AopContext.currentProxy()`方法。


`ProxyFactoryBean`特有的其他属性包括以下内容:

*   `proxyInterfaces`:字符串接口名称的数组。如果未提供此项，将使用目标类的CGLIB代理（ [基于JDK和CGLIB的代理](#aop-pfb-proxy-types)）。

*   `interceptorNames`:要提供的通知者、拦截器或其他通知名称的字符串数组。在先到先得的服务基础上，Ordering（顺序）是重要的。也就是说， 列表中的第一个拦截器将首先拦截调用。

    这些名称是当前工厂中的bean名称，包括来自上级工厂的bean名称。不能在这里提及bean的引用，因为这样做会导致`ProxyFactoryBean`忽略通知的单例。

    可以追加一个带有星号(`*`)的拦截器名称。这将导致应用程序中的所有被*匹配的通知者bean的名称都会被匹配上。 您可以在[使用全局通知者](#aop-global-advisors)中找到使用此功能的示例。

*   singleton:工厂强制返回单个对象，无论调用`getObject()` 方法多少次。几个`FactoryBean`的实现都提供了这样的方法。默认值是`true`。 如果想使用有状态的通知。例如，对于有状态的mixins - 使用原型建议以及单例值`false`。


<a id="aop-pfb-proxy-types"></a>

#### [](#aop-pfb-proxy-types)6.4.3. 基于JDK和基于CGLIB的代理

本节是关于`ProxyFactoryBean`如何为特定目标对象（即将被代理）选择创建基于JDK或CGLIB的代理的权威性文档。

`ProxyFactoryBean`关于创建基于JDK或CGLIB的代理的行为在Spring的1.2.x和2.0版本之间发生了变化。 现在， `ProxyFactoryBean`在自动检测接口方面表现出与 `TransactionProxyFactoryBean`类相似的语义。

如果要代理的目标对象的类（以下简称为目标类）未实现任何接口，则创建基于CGLIB的代理。这是最简单的方案，因为JDK代理是基于接口的，没有接口意味着甚至不可能进行JDK代理。 一个简单的例子是插入目标bean，并通过`interceptorNames`属性指定拦截器列表。请注意，即使`ProxyFactoryBean`的 `proxyTargetClass`属性被设置为`false`，也会创建CGLIB的代理。 （显然，这个false是没有意义的，最好从bean定义中删除，因为它充其量是冗余的，而且是最容易产生混乱）。

如果目标类实现了一个（或多个）接口，那么所创建代理的类型取决于 `ProxyFactoryBean`的配置。

如果`ProxyFactoryBean`的`proxyTargetClass`属性已设置为`true`，则会创建基于CGLIB的代理。这是有道理的，并且符合最少惊喜的原则。 即使`ProxyFactoryBean`的`proxyInterfaces`属性已设置为一个或多个完全限定的接口名称，`proxyTargetClass`属性设置为`true`这一事实也会导致基于CGLIB的代理生效。

如果`ProxyFactoryBean`的 `proxyInterfaces`属性已设置为一个或多个完全限定的接口名称，则会创建基于JDK的代理。创建的代理实现`proxyInterfaces`属性中指定的所有接口。 如果目标类恰好实现了比`proxyInterfaces`属性中指定的更多的接口，那么这一切都很好，但是这些附加接口将不会由返回的代理实现。

如果`ProxyFactoryBean`的`proxyInterfaces`属性具有没有被设置，而目标类确实实现一个或多个接口，则 `ProxyFactoryBean`将自动检测选择，当目标类实际上至少实现一个接口。 将创建JDK代理。实际上代理的接口将是目标类实现的所有接口。事实上，这与简单地提供了目标类实现到 `proxyInterfaces` 属性的每个接口的列表相同。但是，这明显减轻了负担，还避免配置错误。

<a id="aop-api-proxying-intf"></a>

#### [](#aop-api-proxying-intf)6.4.4. 代理接口

首先看一下`ProxyFactoryBean` 简单的例子，这个例子包含:

*   将被代理的目标bean，下面示例中的 `personTarget` bean定义

*   一个通知者和一个拦截器，用于提供通知.

*   指定目标对象( `personTarget` bean)的AOP代理bean和要代理的接口，以及要应用的通知。


以下清单显示了该示例:

    <bean id="personTarget" class="com.mycompany.PersonImpl">
        <property name="name" value="Tony"/>
        <property name="age" value="51"/>
    </bean>

    <bean id="myAdvisor" class="com.mycompany.MyAdvisor">
        <property name="someProperty" value="Custom string property value"/>
    </bean>

    <bean id="debugInterceptor" class="org.springframework.aop.interceptor.DebugInterceptor">
    </bean>

    <bean id="person"
        class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="proxyInterfaces" value="com.mycompany.Person"/>

        <property name="target" ref="personTarget"/>
        <property name="interceptorNames">
            <list>
                <value>myAdvisor</value>
                <value>debugInterceptor</value>
            </list>
        </property>
    </bean>

注意`interceptorNames`属性是一个`String`列表，放拦截器bean的名字或在当前工厂中的通知者。通知者、拦截器、前置、后置返回和异常通知的对象可以被使用。通知者是按顺序排列。

您可能想知道为什么列表不包含bean引用？理由是如果`ProxyFactoryBean`的单例属性被设置为`false`，它必须能够返回独立的代理实例。如果任意的通知者本身是原型的， 那么就需要返回一个独立的实例，所以有必要从工厂获得原型实例。 只保存一个引用是不够的。

前面显示的`person` bean定义可以用来代替`Person`实现，如下所示:

    Person person = (Person) factory.getBean("person");

与普通Java对象一样，同一IoC上下文中的其他bean可以表达对它的强类型依赖。 以下示例显示了如何执行此操作:

    <bean id="personUser" class="com.mycompany.PersonUser">
        <property name="person"><ref bean="person"/></property>
    </bean>

此示例中的`PersonUser`类将公开类型为 `Person`的属性。就它而言，可以透明地使用AOP代理来代替“real”的person实现。但是，它的类将是动态代理类。 可以将其转换为`Advised`的接口（如下所述）：

通过使用匿名内部bean可以隐藏目标和代理之前的区别，只有`ProxyFactoryBean`的定义是不同的，包含通知只是考虑到完整性。以下示例显示如何使用匿名内部bean：

    <bean id="myAdvisor" class="com.mycompany.MyAdvisor">
        <property name="someProperty" value="Custom string property value"/>
    </bean>

    <bean id="debugInterceptor" class="org.springframework.aop.interceptor.DebugInterceptor"/>

    <bean id="person" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="proxyInterfaces" value="com.mycompany.Person"/>
        <!-- Use inner bean, not local reference to target -->
        <property name="target">
            <bean class="com.mycompany.PersonImpl">
                <property name="name" value="Tony"/>
                <property name="age" value="51"/>
            </bean>
        </property>
        <property name="interceptorNames">
            <list>
                <value>myAdvisor</value>
                <value>debugInterceptor</value>
            </list>
        </property>
    </bean>

这样做的好处是只有一个`Person`类型的对象，如果想阻止应用程序上下文的用户获得对un-advised对象的引用，或者需要避免使用Spring IoC自动装配的任何含糊不清的情况， 那么这个对象就很有用。`ProxyFactoryBean`定义是自包含的，这也是一个好处。但是，有时能够从工厂获得un-advised目标可能是一个优势（例如，在某些测试场景中）。。

<a id="aop-api-proxying-class"></a>

#### [](#aop-api-proxying-class)6.4.5. 代理类

如果需要代理一个类而不是一个或多个接口，又该怎么办?

考虑上面的例子，没有`Person`接口，需要给一个没有实现任何业务接口的`Person`类提供通知。在这种情况下，您可以将Spring配置为使用CGLIB代理而不是动态代理。 简单设置`ProxyFactoryBean`的`proxyTargetClass`属性为`true`。尽管最佳实践是面向接口编程，不是类。但在处理遗留代码时， 通知不实现接口的类的能力可能会非常有用（一般来说，Spring不是规定性的。虽然它可以很容易地应用好的实践，但它避免强制使用特定的方法）。

如果你愿意，即使有接口，也可以强制使用CGLIB代理。

CGLIB代理的原理是在运行时生成目标类的子类。Spring配置这个生成的子类用了委托的方法来调用原始的对象，在通知的编织中，子类被用于实现装饰者模式。

CGLIB代理通常对于用户应当是透明的，然而还有需考虑一些问题：

*   `Final`方法不能被advised，因为它们不能被覆盖。

*   无需添加CGLIB到项目的类路径中，从Spring 3.2开始，CGLIB被重新打包并包含在spring-core JAR中。换句话说，基于CGLIB的AOP“开箱即用”，JDK动态代理也是如此。


CGLIB代理和动态代理之间几乎没有性能差异。 从Spring 1.0开始，动态代理略快一些。 但是，这可能会在未来发生变化。 在这种情况下，性能不应该是决定性的考虑因素。

<a id="aop-global-advisors"></a>

#### [](#aop-global-advisors)6.4.6. 使用全局的通知者

通过将星号追加到拦截器名称上，所有与星号前面部分匹配的bean名称的通知者都将添加到通知者链中。如果需要添加一组标准的全局（ “global”）通知者，这可能会派上用场。以下示例定义了两个全局的通知者程序：

    <bean id="proxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="target" ref="service"/>
        <property name="interceptorNames">
            <list>
                <value>global*</value>
            </list>
        </property>
    </bean>

    <bean id="global_debug" class="org.springframework.aop.interceptor.DebugInterceptor"/>
    <bean id="global_performance" class="org.springframework.aop.interceptor.PerformanceMonitorInterceptor"/>

<a id="aop-concise-proxy"></a>

### [](#aop-concise-proxy)6.5. 简明的代理定义

特别是在定义事务代理时，最终可能会定义了许多类似的代理。使用父级和子级bean定义以及内部bean定义可以使代理定义变得更简洁和更简明。

首先为代理创建一个父级的、模板的bean定义:

    <bean id="txProxyTemplate" abstract="true"
            class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
        <property name="transactionManager" ref="transactionManager"/>
        <property name="transactionAttributes">
            <props>
                <prop key="*">PROPAGATION_REQUIRED</prop>
            </props>
        </property>
    </bean>

它本身是永远不会被实例化的，因此它实际上可能是不完整的。然后，每个需要创建的代理都是只是一个子级的bean定义，它将代理的目标包装为内部bean定义，因为目标永远不会单独使用。以下示例显示了这样的子bean:

    <bean id="myService" parent="txProxyTemplate">
        <property name="target">
            <bean class="org.springframework.samples.MyServiceImpl">
            </bean>
        </property>
    </bean>

您可以覆盖父模板中的属性。 在以下示例中，事务传播设置如下:

    <bean id="mySpecialService" parent="txProxyTemplate">
        <property name="target">
            <bean class="org.springframework.samples.MySpecialServiceImpl">
            </bean>
        </property>
        <property name="transactionAttributes">
            <props>
                <prop key="get*">PROPAGATION_REQUIRED,readOnly</prop>
                <prop key="find*">PROPAGATION_REQUIRED,readOnly</prop>
                <prop key="load*">PROPAGATION_REQUIRED,readOnly</prop>
                <prop key="store*">PROPAGATION_REQUIRED</prop>
            </props>
        </property>
    </bean>

请注意，在上面的例子中，通过使用`abstract`属性显式地将父级的bean定义标记为抽象的（`abstract`），[如前所述](#beans-child-bean-definitions)，这样它就不会被实例化。应用程序上下文（但不是简单的bean工厂）将默认提前实例化所有的单例。 因此，重要的是（至少对于单例bean），如果有一个（父级）bean定义，只打算将它用作模板，而这个定义指定一个类，必须确保将抽象（`abstract`）属性设置为`true`， 否则应用程序上下文将实际尝试提前实例化它。

<a id="aop-prog"></a>

### [](#aop-prog)6.6. 使用`ProxyFactory`编程创建AOP代理

使用Spring以编程的方式创建AOP代理是很容易的。这样允许在不依赖于Spring IoC的情况下使用Spring AOP。

目标对象实现的接口将自动代理。下面的代码显示了使用一个拦截器和一个通知者创建目标对象的代理的过程：

    ProxyFactory factory = new ProxyFactory(myBusinessInterfaceImpl);
    factory.addAdvice(myMethodInterceptor);
    factory.addAdvisor(myAdvisor);
    MyBusinessInterface tb = (MyBusinessInterface) factory.getProxy();

第一步是构建一个类型为`org.springframework.aop.framework.ProxyFactory`的对象。可以使用目标对象创建此对象。 如前面的示例所示，或者在指定的接口中进行代理而不是构造器。

开发者可以添加通知（使用拦截器作为一种专用的通知）和/或通知者，并在`ProxyFactory`的生命周期中进行操作。如果添加`IntroductionInterceptionAroundAdvisor`，则可以使代理实现其他接口。

`ProxyFactory`上还有一些便捷的方法（从`AdvisedSupport`类继承的），允许开发者添加其他通知类型，例如前置和异常通知。`AdvisedSupport`是`ProxyFactory`和`ProxyFactoryBean`的超类

将AOP代理创建与IoC框架集成是多数应用程序的最佳实践，因此强烈建议从Java代码中外部配置使用AOP。

<a id="aop-api-advised"></a>

### [](#aop-api-advised)6.7.处理被通知对象

`org.springframework.aop.framework.Advised`接口对它们进行操作。任何AOP代理都可以转换到这个接口，无论它实现了哪个接口。此接口包括以下方法：

    Advisor[] getAdvisors();

    void addAdvice(Advice advice) throws AopConfigException;

    void addAdvice(int pos, Advice advice) throws AopConfigException;

    void addAdvisor(Advisor advisor) throws AopConfigException;

    void addAdvisor(int pos, Advisor advisor) throws AopConfigException;

    int indexOf(Advisor advisor);

    boolean removeAdvisor(Advisor advisor) throws AopConfigException;

    void removeAdvisor(int index) throws AopConfigException;

    boolean replaceAdvisor(Advisor a, Advisor b) throws AopConfigException;

    boolean isFrozen();

`getAdvisors()` 方法将返回已添加到工厂中的每个`Advisor`、拦截器或其他通知类型的通知者。如果添加了`Advisor`，那么这个索引中的返回的通知者将是添加的对象。 如果添加了拦截器或其他通知类型，那么Spring将在通知者中将一个总是返回`true`的切点封装。因此，如果添加了 `MethodInterceptor`，则返回的通知者将是 `DefaultPointcutAdvisor`返回来的`MethodInterceptor`和与所有类和方法匹配的切点。

`addAdvisor()`方法可用于添加任意的`Advisor`。通常，持有切点和通知的通知者是通用的`DefaultPointcutAdvisor`类，它可以用于任意通知或切点（但不能用于引入）。

默认情况下， 即使已经创建了代理，也可以添加或删除通知者或拦截器。唯一的限制是无法添加或删除引入通知者，因为来自工厂的现有代理将不会展示接口的变化。 (开发者可以从工厂获取新的代理，以避免这种问题）。

将AOP代理转换为通知接口并检查和操作其通知的简单示例 :

    Advised advised = (Advised) myObject;
    Advisor[] advisors = advised.getAdvisors();
    int oldAdvisorCount = advisors.length;
    System.out.println(oldAdvisorCount + " advisors");

    // Add an advice like an interceptor without a pointcut
    // Will match all proxied methods
    // Can use for interceptors, before, after returning or throws advice
    advised.addAdvice(new DebugInterceptor());

    // Add selective advice using a pointcut
    advised.addAdvisor(new DefaultPointcutAdvisor(mySpecialPointcut, myAdvice));

    assertEquals("Added two advisors", oldAdvisorCount + 2, advised.getAdvisors().length);

在生产中修改业务对象的通知是否可取(没有双关语）是值得怀疑的，尽管它是合法的使用案例。但是，它可能在开发中非常有用（例如，在测试中）。有时发现能够以拦截器或其他通知的形式添加测试代码也非常有用， 可以在需要测试的方法调用中获取。（例如，通知可以进入为该方法创建的事务中；例如，在标记要回滚的事务之前运行sql以检查数据库是否已正确更新）。

根据您创建代理的方式，通常可以设置`frozen` 标志。在这种情况下，通知的 `isFrozen()` 方法将返回`true`，任何通过添加或删除修改通知的尝试都将导致`AopConfigException`异常。 在某些情况下冻结通知的对象状态的功能很有用（例如，防止调用代码删除安全拦截器）。如果已知的运行时通知不需要修改的话，它也可以在Spring 1.1中使用以获得最好的优化。

<a id="aop-autoproxy"></a>

### [](#aop-autoproxy)6.8. 使用自动代理功能

到目前为止，上面的章节已经介绍了使用`ProxyFactoryBean`或类似的工厂bean显式地创建AOP代理。

Spring还支持使用 “auto-proxy”（自动代理） 的bean定义, 允许自动代理选择bean定义.这是建立在Spring的’s “bean post processor”基础上的，它允许修改任何bean定义作为容器加载。

在这个模式下，可以在XML bean定义文件中设置一些特殊的bean定义，用来配置基础的自动代理。这允许开发者只需声明符合自动代理的目标即可，开发者无需使用`ProxyFactoryBean`。

有两种方法可以做到这一点：:

*   使用在当前上下文中引用特定bean的自动代理创建器

*   自动代理创建的一个特例值得单独考虑：由源代码级别的元数据属性驱动的自动代理创建。


<a id="aop-autoproxy-choices"></a>

#### [](#aop-autoproxy-choices)6.8.1. 自动代理bean的定义

本节介绍`org.springframework.aop.framework.autoproxy` 包提供的自动代理创建器。

<a id="aop-api-autoproxy"></a>

##### [](#aop-api-autoproxy)`BeanNameAutoProxyCreator`

`BeanNameAutoProxyCreator`类是一个`BeanPostProcessor`的实现，它会自动为具有匹配文本值或通配符的名称的bean创建AOP代理。以下示例显示如何创建`BeanNameAutoProxyCreator` bean：

    <bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
        <property name="beanNames" value="jdk*,onlyJdk"/>
        <property name="interceptorNames">
            <list>
                <value>myInterceptor</value>
            </list>
        </property>
    </bean>

与`ProxyFactoryBean`一样，它拥有`interceptorNames`属性而不是持有拦截器列表，以便为原型通知者提供正确的行为。通知者和任意的通知类型都可命名为“interceptors”。

与普通的自动代理一样，使用`BeanNameAutoProxyCreator`的主要目的是能将相同的配置同时或共享地应用于多个对象，此时配置是最少的。 将声明性事务应用于多个对象是很普遍的例子。

在上例中，名称匹配的Bean定义（例如`jdkMyBean` 和 `onlyJdk`）是带有目标类的、普通的、老式的bean定义。 AOP代理由`BeanNameAutoProxyCreator`自动创建。相同的通知也适用于所有匹配到的bean。注意，如果使用通知着（而不是上述示例中的拦截器），那么切点可能随bean的不同用处而变化。

<a id="aop-api-autoproxy-default"></a>

##### [](#aop-api-autoproxy-default)`DefaultAdvisorAutoProxyCreator`

`DefaultAdvisorAutoProxyCreator`是另一个更通用、功能更强大的自动代理创建器。它会在当前的上下文中自动用于符合条件的通知者，而无需在自动代理通知者的bean定义中包含特定的bean名称。 它具有`BeanNameAutoProxyCreator`相同的配置，以及避免重复定义的有点。.

使用此机制涉及:

*   指定`DefaultAdvisorAutoProxyCreator` bean定义

*   在相同或相关上下文中指定任意数量的通知者。注意，这里必须是通知者，而不是拦截器或其他通知类型。这种约束是必需的，因为必须引入对切点的评估， 以检查每个通知是否符合候选bean定义的要求。


`DefaultAdvisorAutoProxyCreator`将自动评估包含在每个通知者中的切点，以查看它是否适用于每个业务对象（如示例中的`businessObject1` 和 `businessObject2` ）的通知（如果有的话）。

这意味着可以将任意数量的通知者自动用于每个业务对象。如果任意通知者都没有一个切点与业务对象中的任何方法匹配，那么对象将不会被代理。当为新的业务对象添加了bean定义时，如果需要这些对象都将被自动代理。

一般来说，自动代理具有使调用方或依赖项无法获取un-advised对象的优点。在这个`ApplicationContext`调用`getBean("businessObject1")`方法将返回AOP代理， 而不是目标业务对象。（前面显示的 “inner bean” 语义也提供了这种好处）。

以下示例创建一个`DefaultAdvisorAutoProxyCreator` bean以及本节中讨论的其他元素:

    <bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>

    <bean class="org.springframework.transaction.interceptor.TransactionAttributeSourceAdvisor">
        <property name="transactionInterceptor" ref="transactionInterceptor"/>
    </bean>

    <bean id="customAdvisor" class="com.mycompany.MyAdvisor"/>

    <bean id="businessObject1" class="com.mycompany.BusinessObject1">
        <!-- Properties omitted -->
    </bean>

    <bean id="businessObject2" class="com.mycompany.BusinessObject2"/>

如果希望对多个业务对象适用相同的通知，那么`DefaultAdvisorAutoProxyCreator`类会显得非常有用。一旦基础架构已定义，就可以简单地添加新的业务对象， 而不必再设置特定的代理配置。还可以很容易地删除其他切面，例如跟踪或性能监视切面 ， 这样对配置的更改最小。

`DefaultAdvisorAutoProxyCreator`提供对过滤器（filtering）的支持（使用命名约定，以便只评估某些通知者，允许在同一工厂中使用多个不同配置的AdvisorAutoProxyCreators）和排序。 通知者可以实现`org.springframework.core.Ordered`接口，以确保正确的排序，如果需要排序的话。 上面的例子中使用的`TransactionAttributeSourceAdvisor`类具有具有可配置的排序值， 默认的设置是无序的。

<a id="aop-targetsource"></a>

### [](#aop-targetsource)6.9. 使用`TargetSource`实现

Spring提供了`TargetSource`概念，定义在 `org.springframework.aop.TargetSource` 接口中。 这个接口用于返回目标对象实现的连接点。 每次AOP代理处理方法调用时，都会要求目标实例进行 `TargetSource`实现。

使用Spring AOP的开发者通常无需直接使用`TargetSource`，一般都是提供了支持池，热部署和用于其他复杂目标的强大手段。 例如，池化的`TargetSource`可以为每个调用返回一个不同的目标实例，并使用一个池来管理实例。

如果未指定`TargetSource`，则使用默认实现来包装本地对象。 每次调用都会返回相同的目标（正如您所期望的那样）。

将下来介绍Spring提供的标准目标源（target sources），以及如何使用。

当使用自定义的target source,目标通常需要配置成原型而不是单例的bean定义。 这允许Spring按需时创建新的目标实例

<a id="aop-ts-swap"></a>

#### [](#aop-ts-swap)6.9.1. Hot-swappable Target Sources

`org.springframework.aop.target.HotSwappableTargetSource` 的存在是为了允许切换AOP代理的目标。

改变目标源的目标会立即有效，`HotSwappableTargetSource`是线程安全的。

可以通过HotSwappableTargetSource上的`swap()`方法更改目标，如下所示:

    HotSwappableTargetSource swapper = (HotSwappableTargetSource) beanFactory.getBean("swapper");
    Object oldTarget = swapper.swap(newTarget);

以下示例显示了所需的XML定义:

    <bean id="initialTarget" class="mycompany.OldTarget"/>

    <bean id="swapper" class="org.springframework.aop.target.HotSwappableTargetSource">
        <constructor-arg ref="initialTarget"/>
    </bean>

    <bean id="swappable" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="targetSource" ref="swapper"/>
    </bean>

前面的`swap()`方法改变了swappable bean的目标。持有对该bean引用的客户端将不会察觉到目标的更改，但会马上开始处理新目标。

虽然这个例子没有添加任何通知 ， 也没有必要添加通知来使用`TargetSource`，当然任意的 `TargetSource`都可以和任意的通知一起使用。

<a id="aop-ts-pool"></a>

#### [](#aop-ts-pool)6.9.2. 创建目标源池

使用池化的目标源为无状态会话EJB提供了类似的编程模型，它维护了相同实例池，调用方法将会释放池中的对象。

Spring池和SLSB池有一个关键的区别是：Spring池可以应用于任意POJO。和Spring一样，这个服务可以以非侵入的方式应用。

Spring为Commons Pool 2.2，提供了开箱即用的支持，它提供了一个相当高效的池化实现。开发者需要在应用程序的类路径上添加 `commons-pool`的jar包来启用此功能。 也可以对`org.springframework.aop.target.AbstractPoolingTargetSource` to support any other进行子类化来支持任意其它池化的API。

Commons Pool 1.5+ 的版本也是支持的，但是在Spring Framework 4.2已经过时了。

以下清单显示了一个示例配置:

    <bean id="businessObjectTarget" class="com.mycompany.MyBusinessObject"
            scope="prototype">
        ... properties omitted
    </bean>

    <bean id="poolTargetSource" class="org.springframework.aop.target.CommonsPool2TargetSource">
        <property name="targetBeanName" value="businessObjectTarget"/>
        <property name="maxSize" value="25"/>
    </bean>

    <bean id="businessObject" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="targetSource" ref="poolTargetSource"/>
        <property name="interceptorNames" value="myInterceptor"/>
    </bean>

请注意，目标对象 ( 例如示例中的`businessObjectTarget`)必须是原型的。 这允许`PoolingTargetSource`能够实现按需创建目标的新实例，用于扩展池。 请参阅[javadoc of`AbstractPoolingTargetSource`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframeworkaop/target/AbstractPoolingTargetSource.html)以及用于其属性信息的具体子类。 `maxSize` 是最基本的，并且始终保证存在。

在这种情况下, `myInterceptor` 是需要在相同的IoC上下文中定义的拦截器的名称。但是，无需指定拦截器来使用池。如果只希望使用池化功能而不需要通知，那么可以不设置`interceptorNames`属性。

可以对Spring进行配置，以便将任意池对象强制转换到`org.springframework.aop.target.PoolingConfig` 接口,从而引入公开的，有关池的配置和当前大小的信息。 此时需要像下面这样定义通知者:

    <bean id="poolConfigAdvisor" class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
        <property name="targetObject" ref="poolTargetSource"/>
        <property name="targetMethod" value="getPoolingConfigMixin"/>
    </bean>

这个通知者是通过在`AbstractPoolingTargetSource`类上调用一个方便的方法获得的，因此可以调用`MethodInvokingFactoryBean`。通知者的名字（在这里是`poolConfigAdvisor`）必须包含在拦截器名字的列表中，`ProxyFactoryBean`公开了池化的对象。

The cast is defined as follows:

    PoolingConfig conf = (PoolingConfig) beanFactory.getBean("businessObject");
    System.out.println("Max pool size is " + conf.getMaxSize());

池化的无状态服务对象一般是没有必要的。一般这种选择不是默认的，因为大多数无状态的对象本质上是线程安全的，并且如果资源是缓存的话，其实例池化是有问题的。

使用自动代理可以创建更简单的池，可以设置任何自动代理创建者使用的`TargetSource` 。


<a id="aop-ts-prototype"></a>


#### [](#aop-ts-prototype)6.9.3. 原型目标源

设置“prototype” 目标源与合并`TargetSource`类似。在这种情况下，每个方法调用都会创建一个新的目标实例。 尽管在现代JVM中创建新对象的成本并不高， 但是连接新对象（满足其IoC依赖性）的成本可能会更高。因此，如果没有很好的理由，不应该使用这种方法。

为此, 可以修改上面显示的 `poolTargetSource` 定义，如下所示（为清晰起见，我们还更改了名称）：:

    <bean id="prototypeTargetSource" class="org.springframework.aop.target.PrototypeTargetSource">
        <property name="targetBeanName" ref="businessObjectTarget"/>
    </bean>

唯一的属性是目标bean的名称。在`TargetSource`实现中使用继承来确保一致的命名。与池化目标源一样，目标bean必须是原型bean定义。

<a id="aop-ts-threadlocal"></a>

#### [](#aop-ts-threadlocal)6.9.4. `ThreadLocal` 的目标源

如果您需要为每个传入请求创建一个对象（每个线程），`ThreadLocal`目标源很有用。`ThreadLocal`的概念提供了一个JDK范围的工具，用于透明地将资源与线程存储在一起。 设置`ThreadLocalTargetSource`几乎与其他类型的目标源设置一样。如下例所示:

    <bean id="threadlocalTargetSource" class="org.springframework.aop.target.ThreadLocalTargetSource">
        <property name="targetBeanName" value="businessObjectTarget"/>
    </bean>

当在多线程和多类加载器环境中错误地使用它们时，`ThreadLocal`会带来严重的问题（可能导致内存泄漏）。您应该始终考虑将threadlocal包装在其他类中，并且永远不要直接使用 `ThreadLocal`本身（除了在包装类中）。 另外，应该始终记得正确设置和取消设置（后者只需调用`ThreadLocal.set(null)`方法）线程的本地资源。在任何情况下都应该写取消设置，如果不取消将会出问题。 Spring的`ThreadLocal`支持此设置并且应当被考虑支持使用`ThreadLocal`而不是手动操作代码。

<a id="aop-extensibility"></a>

### [](#aop-extensibility)6.10. 定义新的通知类型

Spring AOP被设计为可扩展的。尽管拦截器实施机制目前只在内部使用，但除了围绕通知拥有开箱即用的拦截器之外，还可以支持任意的通知类型，例如前置、异常和后置返回的通知。

The `org.springframework.aop.framework.adapter`包是一个SPI包，允许在不改变核心框架的情况下添加新的自定义通知类型。自定义通知类型的唯一约束是它必须实现 `org.aopalliance.aop.Advice`标识接口。

See the [`org.springframework.aop.framework.adapter`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/aop/framework/adapter/package-frame.html) javadoc for further information.