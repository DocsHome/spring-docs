<a id="cache"></a>

[](#cache)8\. 缓存抽象
------------------

从3.1版本开始,Spring框架为在现有的Spring应用程序透明地添加缓存提供了支持。 与[事务](data-access.html#transaction)支持类似，缓存抽象允许一致地使用各种缓存解决方案，而对代码的影响最小。

从Spring 4.1开始，通过[JSR-107注解](#cache-jsr-107)和更多自定义选项的支持，缓存抽象得到了显着改进。

<a id="cache-strategies"></a>

### [](#cache-strategies)8.1. 了解缓存抽象

Cache vs Buffer

术语“buffer” 和 “cache,”倾向于可互换使用。 但请注意，它们代表不同的东西。 缓冲区通常用作快速和慢速实体之间的数据的中间临时存储区。由于一方必须等待另一个影响性能的因素， 因此缓冲区会通过允许整个数据块同时移动， 而不是一小块来缓解这一问题。数据只在缓冲区中写入和读取一次。此外， 缓冲区至少对知道它的一方是可见的。

另一方面，理论上的缓存是隐藏的，任何一方都不知道缓存是否发生。它还可以提高性能，而且相同的数据可以快速读取多次。

对两者更多的解释是在[here](https://en.wikipedia.org/wiki/Cache_(computing)#The_difference_between_buffer_and_cache) .

缓存抽象的核心是将缓存应用于Java方法，从而根据缓存中可用的信息减少执行次数。 也就是说，每次调用目标方法时，抽象都会应用一种缓存行为，该行为检查该方法是否已针对给定参数执行。 如果已执行，则返回缓存的结果，而不必执行实际方法。 如果该方法尚未执行，则执行该方法，并将结果缓存并返回给用户，以便下次调用该方法时，返回缓存的结果。 这样，对于给定的一组参数，昂贵的方法（无论是CPU还是IO）只能执行一次，并且重用结果而不必再次实际执行该方法。 缓存逻辑是透明应用的，不会对调用者造成任何干扰。

此方法仅适用于保证为给定输入（或参数）返回相同输出（结果）的方法，无论它执行多少次。

其他与缓存相关的操作由抽象提供，如更新缓存内容的能力或删除所有条目中的一个。如果缓存处理在应用程序过程中可以更改的数据，这些都是很有用的。

就像Spring框架中的其他服务一样，缓存服务是抽象(不是缓存实现)，需要使用实际存储来存储缓存数据，也就是说，抽象使开发人员不必编写缓存逻辑，但不提供实际的存储。此抽象由`org.springframework.cache.Cache`和`org.springframework.cache.CacheManager`接口具体化。

Spring提供了一些抽象实现：基于JDK `java.util.concurrent.ConcurrentMap`的缓存，[Ehcache 2.x](http://ehcache.org/)，Gemfire缓存，[Caffeine](https://github.com/ben-manes/caffeine/wiki)和符合JSR-107的缓存（例如Ehcache 3.x）。 有关插入其他缓存存储和提供程序的详细信息，请参阅[不同的后端缓存](#cache-plug)。

缓存抽象没有针对多线程和多进程环境的特殊处理，因为这些功能由缓存实现处理。

如果您具有多进程环境（即，部署在多个节点上的应用程序），则需要相应地配置缓存提供程序。 根据您的使用情况，几个节点上的相同数据的副本就足够了。 但是，如果在应用程序过程中更改数据，则可能需要启用其他传播机制。

在代码中直接添加缓存的经典流程有 get-if-not-found-then-proceed-and-put-eventually 这里没有用到锁。同时允许多线程同步时加载同一个缓存。同样对于回收缓存也是相似。但如果有多个线程试图更新或者回收数据的话，可能会用到旧数据。某些缓存为程序为该区域的操作提供了高级功能，请参阅您正在使用的缓存提供程序的文档以获取详细信息。

要使用缓存抽象，您需要注意两个方面：

*   缓存声明：确定需要缓存的方法及其策略。
    
*   缓存配置：存储数据并从中读取数据的后端缓存。
    

<a id="cache-annotations"></a>

### [](#cache-annotations)8.2. 基于注解声明缓存

对于缓存声明，Spring的缓存抽象提供了一组Java注解：

*   `@Cacheable`: 触发缓存储存
    
*   `@CacheEvict`: 触发缓存回收
    
*   `@CachePut`: 在不影响方法执行的情况下更新缓存
    
*   `@Caching`: 重新在方法上应用多个缓存操作
    
*   `@CacheConfig`: 在类级别共享与缓存有关的常见设置
    

<a id="cache-annotations-cacheable"></a>

#### [](#cache-annotations-cacheable)8.2.1. `@Cacheable` 注解

顾名思义，`@Cacheable` 用于标定可缓存的方法 ， 即将结果存储到缓存中的方法，以便在后续调用中(使用相同的参数)，缓存的值可以在不执行方法的情况下返回。最简单的，注解声明要求带注解的方法关联缓存的名称。如下:

```java
@Cacheable("books")
public Book findBook(ISBN isbn) {...}
```

在上面的代码段中，方法`findBook`与名为`books`的缓存关联。每次调用该方法时，都会检查缓存以查看是否已经执行了调用，并且不会重复。在大多数情况下，只有一个缓存被声明，注解允许指定多个名称，以便使用一个以上的缓存。在这种情况下，在执行方法之前将检查每个缓存，如果至少命中一个缓存，则将返回关联的值。如下：

即使未实际执行缓存的方法，也会更新不包含该值的所有其他缓存。

以下示例在 `findBook` 方法上使用`@Cacheable`:

```java
@Cacheable({"books", "isbns"})
public Book findBook(ISBN isbn) {...}
```

<a id="cache-annotations-cacheable-default-key"></a>

##### [](#cache-annotations-cacheable-default-key)默认的键生成

由于缓存实质上是键-值存储区，因此需要将每个缓存方法的调用转换为适合缓存访问的键。缓存抽象使用基于以下算法使用简单的键生成器-`KeyGenerator`，做到开箱即用。:

*   如果没有参数被指定，则返回`SimpleKey.EMPTY`.
    
*   如果只有一个参数被指定，则返回实例
    
*   如果多于一个参数被指定，则返回一个`SimpleKey`包含所有的参数
    

这种方法适用于大多数用例。只要参数有自然的键并且实现了有效的`hashCode()` 和 `equals()`方法，如果不是这样的话， 那么战略就需要改变。

要提供不同的默认键生成器，您需要实现`org.springframework.cache.interceptor.KeyGenerator`接口。

默认的键生成策略在Spring 4.0版本有点改变。早期的Spring版本使用的键生成策略，对于多个键参数，只考虑参数的`hashCode()` 而没有考虑到 `equals()`方法，这将导致一定的碰撞（见[SPR-10237](https://jira.spring.io/browse/SPR-10237)）。新的 `SimpleKeyGenerator` 为这种情况使用复合键。

如果您希望继续使用前面的 key strategy, 则可以配置已弃用的`org.springframework.cache.interceptor.DefaultKeyGenerator`类或创建自定义的基于`KeyGenerator`的实现。

<a id="cache-annotations-cacheable-key"></a>

##### [](#cache-annotations-cacheable-key)自定义键生成器

由于缓存是具有普遍性的，因此目标方法很可能具有不同的签名，不能简单地映射到缓存结构的顶部。当目标方法有多个参数时，问题显得更突出，其中只有一部分是适合于缓存的(其余仅由方法逻辑使用)。例如：

```
@Cacheable("books")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

乍一看，虽然两个 `boolean`参数影响了book的发现方式，但它们对缓存没有用处。 如果两个中只有一个重要而另一个不重要怎么办？

对于这种情况，`@Cacheable`注解允许用户指定`key`的生成属性。开发人员可以使用[SpEL](core.html#expressions))来选择感兴趣的参数(或它们的嵌套属性)执行操作，甚至调用任意方法，而不必编写任何代码或实现任何接口。这是在默认生成器。

以下示例是各种SpEL声明（如果您不熟悉SpEL，请阅读 [Spring Expression Language](core.html#expressions)）:

```java
@Cacheable(cacheNames="books", key="#isbn")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

@Cacheable(cacheNames="books", key="#isbn.rawNumber")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

@Cacheable(cacheNames="books", key="T(someType).hash(#isbn)")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

上面的代码段显示了选择某个参数、某个属性甚至是任意(静态)方法是多么容易。

如果负责生成键的算法太特殊，或者需要共享，则可以在操作上定义一个自定义`keyGenerator`。为此，请指定要使用的`KeyGenerator` bean实现的名称。如下:

```java
@Cacheable(cacheNames="books", keyGenerator="myKeyGenerator")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

`key`和 `keyGenerator`参数是互斥的， 指定两者的操作将导致异常。

<a id="cache-annotations-cacheable-default-cache-resolver"></a>

##### [](#cache-annotations-cacheable-default-cache-resolver)默认的缓存解析

缓存抽象使用一个简单的 `CacheResolver`（可即用）它使用`CacheManager`配置检索在操作级别定义的缓存。

为了提供不同默认缓存解析，需要实现`org.springframework.cache.interceptor.CacheResolver` 接口.

<a id="cache-annotations-cacheable-cache-resolver"></a>

##### [](#cache-annotations-cacheable-cache-resolver)自定义缓存解析

默认的缓存解析适合于应用对于使用单个`CacheManager`，并且不需要复杂的解析要求。

对于应用使用不同的缓存管理，可以设置`cacheManager` 使用每个操作。如以下示例所示：

```java
@Cacheable(cacheNames="books", cacheManager="anotherCacheManager") (1)
public Book findBook(ISBN isbn) {...}
```

**1**、Specifying `anotherCacheManager`.

也可以完全以类似于 [key generation](#cache-annotations-cacheable-key)的方式替换`CacheResolver`。为每个缓存操作请求该解决方案， 使实现能够根据运行时参数实际解析要使用的缓存。以下示例显示如何指定 `CacheResolver`：

```java
@Cacheable(cacheResolver="runtimeCacheResolver") (1)
public Book findBook(ISBN isbn) {...}
```

**1**、Specifying the `CacheResolver`.

Spring 4.1版本以后, 缓存注解的 `value`属性不再是强制性的了，因为无论注解的内容如何，`CacheResolver`都可以提供此特定信息。

与 `key`和`keyGenerator`似，`cacheManager`和`cacheResolver`参数是互斥的，同时指定两者的操作将导致异常，因为`CacheResolver`实现忽略了自定义`CacheManager`。这可能不是你所期望的。

<a id="cache-annotations-cacheable-synchronized"></a>

##### [](#cache-annotations-cacheable-synchronized)同步缓存

在多线程环境中，对于同一参数(通常在启动时)，可能会同时调用某些操。默认情况下，缓存抽象不会锁定任何内容，并且可能会多次计算相同的值，从而无法达到缓存的目的。

对于这些特定情况，可以使用`sync`属性指示基础缓存提供程序在计算值时锁定缓存项。因此，只有一个线程将忙于计算该值，而其余的则被阻塞，直到在缓存中更新该项。以下示例显示了如何使用`sync`属性:

```java
@Cacheable(cacheNames="foos", sync=true) (1)
public Foo executeExpensiveOperation(String id) {...}
```

**1**、Using the `sync` attribute.

这是一个可选的功能，您喜欢的缓存库可能不支持它。核心框架提供的所有`CacheManager` 实现都支持它。有关详细信息，请参阅缓存提供程序的文档，

<a id="cache-annotations-cacheable-condition"></a>

##### [](#cache-annotations-cacheable-condition)条件缓存

有时，一个方法做缓存可能不是万能的(例如，它可能依赖于给定的参数)。缓存注解通过`condition`参数支持此类功能，它采用 `SpEL` 表达式，并将其计算为`true`或`false`。如果为`true`，则方法执行缓存。如果不是，则它的行为就像不缓存的方法一样。 每次不管缓存中的值是什么或使用了什么参数，都将执行它。例如，仅当参数名称的长度小于32时，才会缓存以下方法：

```java
@Cacheable(cacheNames="book", condition="#name.length() < 32") (1)
public Book findBook(String name)
```

**1**、Setting a condition on `@Cacheable`.

此外， 除了`condition`参数外， 还可以使用`unless` 参数来决定不想缓存的值。与 `condition`不同，， `unless`在调用方法后计算表达式。扩展上一示例。也许我们只想缓存平装书，如下例所示:

```java
@Cacheable(cacheNames="book", condition="#name.length() < 32", unless="#result.hardback") (1)
public Book findBook(String name)
```

**1**、Using the `unless` attribute to block hardbacks.

缓存抽象支持`java.util.Optional`，仅在其存在时将其内容用作缓存值。 `#result`总是引用业务实体而从不支持包装器，因此前面的示例可以重写如下:

```java
@Cacheable(cacheNames="book", condition="#name.length() < 32", unless="#result?.hardback")
public Optional<Book> findBook(String name)
```

请注意，`result`仍然引用 `Book` 而不是`Optional`。 由于它可能为`null`，我们应该使用安全导航操作符。

<a id="cache-spel-context"></a>

##### [](#cache-spel-context)可用的缓存SpEL表达式内容

每个`SpEL`表达式都可以用于再次指定的上下文值。除了在参数中生成外，框架还提供专用的与缓存相关的元数据(如参数名称)。下表列出了可用于[`context`](core.html#expressions-language-ref)的项，以便可以使用这些项进行键和条件计算。

Table 11. 缓存SpEL可用的元数据  

| 名字     |  位置    |   描述   |  例子    |
| ---- | ---- | ---- | ---- |
| `methodName`     |  Root object    | 正在调用的方法的名称     |  `#root.methodName`    |
|  `method`    | Root object     | 正在调用的方法     | `#root.method.name`     |
|  `target`    | Root object     | 正在调用的目标对象     | `#root.target`     |
|  `targetClass`    |  Root object    | 被调用的目标的类     | `#root.targetClass`     |
|  `args`    | Root object     | 用于调用目标的参数（作为数组）     | `#root.args[0]`     |
|  `caches`    | Root object     | 执行当前方法的高速缓存的集合     |`#root.caches[0].name`      |
|  Argument name    |  Evaluation context    | 任何方法参数的名称。 如果名称不可用（可能由于没有调试信息），参数名称也可以在`#a<#arg>`下获得，其中 `#arg`代表参数索引（从0开始）。     |   `#iban` or `#a0` (you can also use `#p0` or `#p<#arg>` notation as an alias).   |
|  `result`    |  Evaluation context    | 方法调用的结果（要缓存的值）。 仅在`unless`表达式，缓存`cache put`表达式（计算`key`）或缓存`cache evict`表达式（当`beforeInvocation`为 `false`时）中可用。 对于受支持的包装器（例如`Optional`）， `#result`引用实际的对象，而不是包装器。    |   `#result`   |


<a id="cache-annotations-put"></a>

#### [](#cache-annotations-put)8.2.2. `@CachePut` 注解

对于需要更新缓存而不影响方法执行的情况，可以使用`@CachePut`注解。即，将始终执行该方法，并将其结果放入缓存(根据`@CachePut`选项)。它支持与`@Cacheable`相同的选项，，并且应用于缓存填充，而不是方法流优化。以下示例使用`@CachePut`注解:

```java
@CachePut(cacheNames="book", key="#isbn")
public Book updateBook(ISBN isbn, BookDescriptor descriptor)
```

通常强烈建议不要在同一方法上使用`@CachePut`和`@Cacheable` 注解，因为它们具有不同的行为。当后者导致使用缓存跳过方法执行时，前者强制执行以执行缓存更新。这会导致意外的行为，并且除了特定的某些情况(例如，有排除彼此的条件的注解)之外， 应避免此类声明。还要注意，此类条件不应依赖于结果对象(即`#result`变量)，因为它们是预先验证的，以确认排除。

<a id="cache-annotations-evict"></a>

#### [](#cache-annotations-evict)8.2.3. `@CacheEvict` 注解

抽象缓存不仅允许缓存存储区的填充，而且还可以回收。此过程对于从缓存中删除陈旧或未使用的数据非常有用。相比于`@Cacheable`，注解 `@CacheEvict`执行缓存回收的区分方法，即充当从缓存中移除数据的触发器的方法。就像它的同级注解一样， `@CacheEvict`需要指定一个(或多个)受该操作影响的缓存，允许自定义缓存和键解析或指定的条件，但除此之外 ，还具有一个额外的参数`allEntries`，它指示是否需要进行缓存范围的回收，而不是只执行一项(基于键):

```java
@CacheEvict(cacheNames="books", allEntries=true) (1)
public void loadBooks(InputStream batch)
```

**1**、Using the `allEntries` attribute to evict all entries from the cache.

当需要清除整个缓存区域时，此选项会派上用场。然后将每个条目回收(这将需要很长时间，因为它的效率很低)，所有的条目都在一个操作中被移除，如上所示。请注意，框架将忽略此方案中指定的任何键，因为它不适用(整个缓存被回收的不仅仅是一个条目)。

您还可以通过使用`beforeInvocation`属性指示回收是在（默认）之后还是在方法执行之前进行的。前者提供与注解的其余部分相同的语义：一旦方法成功完成，就会执行缓存上的操作（在本例中为回收）。如果方法未执行（因为它可能被缓存）或抛出异常，则不会发生回收。 后者（`beforeInvocation=true`）导致回收始终在调用方法之前发生。 这在回收不需要与方法结果相关联的情况下非常有用。

必须注意的是，`void`方法可以与`@CacheEvict`一起使用-因为这些方法充当触发器,所以返回值被忽略(因为它们不与缓存交互)。`@Cacheable` 将数据添加/更新到缓存中的情况并非如此，因此需要重新请求结果。

<a id="cache-annotations-caching"></a>

#### [](#cache-annotations-caching)8.2.4. `@Caching` 注解

在某些情况下，需要指定相同类型的多个注解(例如`@CacheEvict`或`@CachePut`)。例如，因为条件或键表达式在不同的缓存之间是不同的。 `@Caching`允许在同一方法上使用多个嵌套的`@Cacheable`、`@CachePut`和`@CacheEvict`。以下示例使用两个`@CacheEvict`注解:

```java
@Caching(evict = { @CacheEvict("primary"), @CacheEvict(cacheNames="secondary", key="#p0") })
public Book importBooks(String deposit, Date date)
```

<a id="cache-annotations-config"></a>

#### [](#cache-annotations-config)8.2.5. `@CacheConfig` 注解

到目前为止，我们已经看到缓存操作提供了许多自定义选项，您可以为每个操作设置这些选项。但是，如果这些自定义选项适用于该类的所有操作，则可以对其进行配置。例如，指定要用于该类的每个缓存操作的缓存的名称可以由单个类级别定义替换。这就是`@CacheConfig`发挥作用的地方。 以下示例使用`@CacheConfig`设置缓存的名称:

```java
@CacheConfig("books") (1)
public class BookRepositoryImpl implements BookRepository {

    @Cacheable
    public Book findBook(ISBN isbn) {...}
}
```

**1**、Using `@CacheConfig` to set the name of the cache.

`@CacheConfig`是一个类级注解，它允许共享缓存名称、自定义`KeyGenerator`、自定义`CacheManager`以及最终的自定义`CacheResolver`。将此注解放在类上不会打开任何缓存操作。

操作级自定义项将始终覆盖在`@CacheConfig`上设置的自定义项。因此，每个缓存操作都提供了三个级别的自定义项:

*   全局配置, 适用于`CacheManager`, `KeyGenerator`.
    
*   在类级别, 使用@ `@CacheConfig`.
    
*   在操作级别
    

<a id="cache-annotation-enable"></a>

#### [](#cache-annotation-enable)8.2.6. 启用缓存注解

值得注意的是，即使声明缓存注解不会自动触发它们的操作(如Spring中的许多项)，该功能也必须以声明方式启用(这意味着如果您怀疑缓存是罪魁祸首，您可以通过只删除一个配置行而不是代码中的所有注解)。

要启用缓存注解，请将注解`@EnableCaching`添加到您的`@Configuration`类之一上:

```java
@Configuration
@EnableCaching
public class AppConfig {
}
```

在XML的配置中，可以使用`cache:annotation-driven` 元素:

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:cache="http://www.springframework.org/schema/cache"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/cache http://www.springframework.org/schema/cache/spring-cache.xsd">

        <cache:annotation-driven/>
</beans>
```

`cache:annotation-driven` 元素和`@EnableCaching` 注解允许指定各种选项，从而影响通过AOP将缓存行为添加到应用程序的方式。此配置与[`@Transactional`](data-access.html#tx-annotation-driven-settings)很类似：

处理缓存注释的默认建议模式是`proxy`，它允许仅通过代理拦截调用。 同一类中的本地调用不能以这种方式截获。 对于更高级的拦截模式，请考虑结合编译时或加载时编织切换到`aspectj`模式。

有关实现`CachingConfigurer`所需的高级自定义（使用Java配置）的更多详细信息，请参阅[javadoc](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/cache/annotation/CachingConfigurer.html)。

Table 12. Cache 注解设置 

|      XML属性| 注解属性     |默认值      |描述      |
| ---- | ---- | ---- | ---- |
| `cache-manager`     | N/A (see the [`CachingConfigurer`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/cache/annotation/CachingConfigurer.html) javadoc)     |  要使用的缓存管理器的名称。 使用此缓存管理器（或未设置的`cacheManager`）在后台初始化默认的`CacheResolver`。 要获得更精细的缓存fine-grained 管理，请考虑设置'cache-resolver'属性。    |   `cacheManager`   |
|  `cache-resolver`    |  N/A (see the [`CachingConfigurer`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/cache/annotation/CachingConfigurer.html) javadoc)    |  A `SimpleCacheResolver` using the configured `cacheManager`.    |  用于解析后端缓存的CacheResolver的bean名称。 此属性不是必需的，只需要指定为'cache-manager'属性的替代。    |
| `key-generator`     |  N/A (see the [`CachingConfigurer`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/cache/annotation/CachingConfigurer.html) javadoc)    | `SimpleKeyGenerator`     |  要使用的自定义键生成器的名称。    |
|   `error-handler`   |  N/A (see the [`CachingConfigurer`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/cache/annotation/CachingConfigurer.html) javadoc)    |   `SimpleCacheErrorHandler`   |   要使用的自定义缓存错误处理程序的名称。 默认情况下，在缓存相关操作期间抛出的任何异常都会在客户端返回。   |
|  `mode`    |  `mode`    |  `proxy`    | 默认模式（`proxy`）使用Spring的AOP框架处理要注释的注释bean（遵循代理语义，如前所述，仅适用于通过代理进入的方法调用）。 替代模式（`aspectj`）用Spring的AspectJ缓存方面编织受影响的类，修改目标类字节代码以应用于任何类型的方法调用。 AspectJ编织需要在类路径中使用`spring-aspects.jar`以及启用加载时织入（或编译时织入）。 （有关如何设置加载时织入的详细信息，请参阅[Spring configuration](core.html#aop-aj-ltw-spring)。）     |
|  `proxy-target-class`    | `proxyTargetClass`     | `false`     |  仅适用于代理模式。 控制为使用`@Cacheable`或`@CacheEvict`注解注释的类创建哪种类型的高速缓存代理。 如果`proxy-target-class` 属性设置为`true`，则创建基于类的代理。 如果`proxy-target-class`为`false`或者省略了该属性，则会创建基于标准JDK接口的代理。 （有关不同代理类型的详细检查，请参阅 [代理机制](core.html#aop-proxying)    |
|  `order`    |  `order`    | Ordered.LOWEST\_PRECEDENCE     | 定义应用于使用`@Cacheable`或`@CacheEvict`注解的bean的缓存通知的顺序。 （有关与订购AOP advice相关的规则的更多信息，请参阅[Advice Ordering](core.html#aop-ataspectj-advice-ordering)。）没有指定的排序意味着AOP子系统确定advice的顺序。     |

`<cache:annotation-driven/>`在 bean 中定义的应用程序上下文中只查找`@Cacheable/@CachePut/@CacheEvict/@Caching`。这意味着，如果你在`WebApplicationContext` 中为`DispatcherServlet`放置`<cache:annotation-driven/>`，他只会检查控制器中的bean，而不是你的服务。有关更多信息，请参阅[MVC部分](web.html#mvc-servlet)。

方法可见和缓存注解

使用代理时，应仅将缓存注解应用于具有公共可见性的方法。如果使用这些注解对受保护的、私有的或包可见的方法进行注解，虽然不会引发任何错误，但注解的方法不会显示已配置的缓存设置。 如果需要在更改字节码本身时对非公共方法进行注解，请考虑使用AspectJ(请参阅本节的其余部分)

Spring建议您只用`@Cache*`注解来注解具体类(还有具体类的方法)，而不是注解接口。当然，您可以将`@Cache*`注解放在接口(或接口方法)上，但这只适用于您在使用基于代理时所期望的效果。Java注解不是从接口继承的这一事实意味着， 如果您使用的是基于类的代理(`proxy-target-class="true"`)或weaving-based aspect(`mode="aspectj"`)，则缓存设置无法被代理和编织基础结构，并且对象不会被包装在缓存代理中，这将是绝对糟糕的。

在代理模式(默认情况下), 仅截获通过代理进入的外部方法调用。这意味着，实际上，self-invocation在目标对象中调用目标对象的另一种方法时，在运行时不会导致实际的缓存，即使调用的方法标记为`@Cacheable`，考虑使用`aspectj`模式也是这种情况，此外，代理必须完全初始化以提供预期的行为，因此您不应依赖于初始化代码中的此功能，即`@PostConstruct`。

<a id="cache-annotation-stereotype"></a>

#### [](#cache-annotation-stereotype)8.2.7. 使用自定义的注解

自定义的注解和AspectJ

此功能仅在基于方法的情况下工作，但可以通过使用AspectJ的额外工作来启用。

`spring-aspects` 模块定义了标准注解的切面。如果你定义自己的注解，则还需要为这些注解定义一个切面。请检查`AnnotationCacheAspect` 以查看示例：

缓存抽象允许您使用自己的注解来识别触发缓存储存或回收的方法。这在模板机制中非常方便，因为它消除了重复缓存注解声明的需要(在指定键或条件时尤其有用)，或者在您的代码库中不允许使用外部导入(`org.springframework`)。与[stereotype](core.html#beans-stereotype-annotations)注解的其余部分类似， `@Cacheable`, `@CachePut`、`@CacheEvict`, 和 `@CacheConfig`可以用作[meta-annotations](core.html#beans-meta-annotations)。这是可以注解其他注解的注解（就是元注解）。在下面的示例中，我们使用自己的自定义注释替换常见的`@Cacheable`声明：:

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
@Cacheable(cacheNames="books", key="#isbn")
public @interface SlowService {
}
```

在前面的示例中，我们定义了自己的`SlowService`注释，该注释本身使用`@Cacheable`进行注释。 现在我们可以替换以下代码:

```java
@Cacheable(cacheNames="books", key="#isbn")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

以下示例显示了自定义注释，我们可以使用它来替换前面的代码：

```java
@SlowService
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

尽管`@SlowService`不是Spring注释，但容器会在运行时自动获取其声明并理解其含义。 请注意，[如前所述](#cache-annotation-enable)，需要启用注释驱动的行为。

<a id="cache-jsr-107"></a>

### [](#cache-jsr-107)8.3. JCache (JSR-107) 注解

自Spring框架4.1以来，缓存抽象完全支持JCache标准注释: `@CacheResult`, `@CachePut`, `@CacheRemove`, he `@CacheRemoveAll` 以及 `@CacheDefaults`, `@CacheKey`, 和 `@CacheValue`。这些注解可以正确地使用，而无需将缓存存储迁移到JSR-107。内部实现使用Spring的缓存抽象， 并提供默认的`CacheResolver`和`KeyGenerator`实现，它们符合规范。换言之，如果您已经在使用Spring的缓存抽象，则可以切换到这些标准注解，而无需更改缓存存储(或配置)。

<a id="cache-jsr-107-summary"></a>

#### [](#cache-jsr-107-summary)8.3.1. 特性总结

对于熟悉Spring缓存注解的用户，下表描述了Spring注解和JSR-107对应项之间的主要区别：

Table 13. Spring vs. JSR-107 缓存注解

|  Spring    |  JSR-107    |  备注    |
| ---- | ---- | ---- |
| `@Cacheable`     |`@CacheResult`      |  非常相似。`@CacheResult`可以缓存特定的异常并强制执行该方法，而不管缓存的内容如何。    |
| `@CachePut`     | `@CachePut`     |  当Spring使用方法调用的结果更新缓存时，JCache要求将其作为使用`@CacheValue`注释的参数传递。 由于这种差异，JCache允许在实际方法调用之前或之后更新缓存。    |
| `@CacheEvict`     |`@CacheRemove`      | 非常相似。 当方法调用导致异常时，`@CacheRemove`支持条件回收。     |
|  `@CacheEvict(allEntries=true)`    | `@CacheRemoveAll`     |  See `@CacheRemove`.    |
|  `@CacheConfig`    |   `@CacheDefaults`   | 允许您以类似的方式配置相同的概念。     |

JCache具有与Spring的`CacheResolver`接口相同的`javax.cache.annotation.CacheResolver`，但JCache只支持单个缓存。默认情况下，一个简单的实现是根据注解上声明的名称检索要使用的缓存。 应该注意的是，如果在注解中没有指定缓存名称，则会自动生成默认值，参看`@CacheResult#cacheName()` 。

`CacheResolver`实例由`CacheResolverFactory`检索。 可以为每个缓存操作自定义工厂，如以下示例所示:

```java
@CacheResult(cacheNames="books", cacheResolverFactory=MyCacheResolverFactory.class) (1)
public Book findBook(ISBN isbn)
```

**1**、为此操作自定义工厂。

对于所有引用的类，Spring尝试查找具有给定类型的bean。如果存在多个匹配项，则会创建一个新实例，并且可以使用常规bean生命周期回调(如依赖项注入)。

键由`javax.cache.annotation.CacheKeyGenerator`方法生成，其作用与Spring的`KeyGenerator`一样。默认情况下，所有方法参数都将被考虑，除非至少有一个参数是用`@CacheKey`注解。这类似于Spring的[自定义键生成声明](#cache-annotations-cacheable-key)。例如，同样的操作，一个使用Spring的抽象，另一个用JCache：:

```java
@Cacheable(cacheNames="books", key="#isbn")
public Book findBook(ISBN isbn, boolean checkWarehouse, boolean includeUsed)

@CacheResult(cacheName="books")
public Book findBook(@CacheKey ISBN isbn, boolean checkWarehouse, boolean includeUsed)
```

您还可以在操作上指定`CacheKeyResolver`，类似于指定`CacheResolverFactory`的方式。

JCache可以管理由注解的方法引发的异常。这可以防止缓存的更新，但也可以将异常缓存为失败的指示器，而不是再次调用该方法。让我们假设，如果ISBN的结构错误，则抛出`InvalidIsbnNotFoundException`。这是一个永久性的失败，没有book可以用这样的参数检索。下面抛出缓存异常，以便使用相同的无效ISBN进行进一步调用，直接抛出缓存的异常，而不是再次调用该方法。:

```java
@CacheResult(cacheName="books", exceptionCacheName="failures"
            cachedExceptions = InvalidIsbnNotFoundException.class)
public Book findBook(ISBN isbn)
```

<a id="enabling-jsr-107-support"></a>

#### [](#enabling-jsr-107-support)8.3.2. 启用 JSR-107 支持

除了Spring的声明性注解支持之外，不需要做任何具体的工作来启用JSR-107支持。如果`spring-context-support`模块已经在类加载路径中，那么使用`@EnableCaching`或者`cache:annotation-driven`元素都将自动启用JCache支持。

根据您的使用情况，选择使用与否由你选择。您甚至可以使用JSR-107 API和其他使用Spring自己的注解来混合使用服务。但是，请注意，如果这些服务影响到相同的缓存，则应使用一致的和相同的键生成实现。

<a id="cache-declarative-xml"></a>

### [](#cache-declarative-xml)8.4. 声明式基于XML的缓存

如果注解不是可选的(不能访问源代码或没有外部码)，则可以使用XML进行声明性缓存。因此，您不必对缓存方法进行注解，而是在外部指定目标方法和缓存指令(类似于声明性事务管理[advice](data-access.html#transaction-declarative-first-example))。上一节中的示例可以转换为以下示例:

```xml
<!-- the service we want to make cacheable -->
<bean id="bookService" class="x.y.service.DefaultBookService"/>

<!-- cache definitions -->
<cache:advice id="cacheAdvice" cache-manager="cacheManager">
    <cache:caching cache="books">
        <cache:cacheable method="findBook" key="#isbn"/>
        <cache:cache-evict method="loadBooks" all-entries="true"/>
    </cache:caching>
</cache:advice>

<!-- apply the cacheable behavior to all BookService interfaces -->
<aop:config>
    <aop:advisor advice-ref="cacheAdvice" pointcut="execution(* x.y.BookService.*(..))"/>
</aop:config>

<!-- cache manager definition omitted -->
```

在上面的配置中，`bookService`是可以缓存的。缓存语义被保存在`cache:advice`定义中，指定了方法`findBooks`用于将数据放入缓存，而方法`loadBooks`用于回收数据。这两个定义都可以使用 `books`缓存。

`aop:config`定义使用AspectJ切点表达式将缓存通知应用于程序中的适当的切点(更多信息参看[Spring面向切面编程](core.html#aop))。在前面的示例中，将考虑`BookService`中的所有方法，并将缓存advice应用于它们。

声明性XML缓存支持所有基于注解的模型，因此在两者之间转换应该相当简单。在同一个应用程序中可以进一步使用它们。基于XML的方法不会设计到目标代码，但是编写它非常冗长无聊。在处理具有针对缓存的重载方法的类时，确定正确的方法确实需要额外工作，因为该方法参数不能很好的被辨别。在这些情况下， 在这些情况下，您可以使用AspectJ切入点来挑选目标方法并应用适当的缓存功能。然而，通过XML，更容易应用在包/组/接口范围上的缓存(再次因为AspectJ切点)和创建类似模板的定义(如我们在上面的例子中通过缓存定义目标的`cache:definitions`属性。

<a id="cache-store-configuration"></a>

### [](#cache-store-configuration)8.5. 配置缓存的存储

缓存抽象集成了多个存储功能，可以开箱即用。为了使用他们，您需要声明一个适当的`CacheManager` （一个控制和管理`Cache`实例的实体，可用于检索这些实例以进行存储）。

<a id="cache-store-configuration-jdk"></a>

#### [](#cache-store-configuration-jdk)8.5.1. 基于JDK`ConcurrentMap` 缓存

基于JDK的`Cache`实现位于`org.springframework.cache.concurrent`包下。它允许您使用`ConcurrentHashMap`作为支持`Cache`存储。 以下示例显示如何配置两个缓存:

```xml
<!-- simple cache manager -->
<bean id="cacheManager" class="org.springframework.cache.support.SimpleCacheManager">
    <property name="caches">
        <set>
            <bean class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean" p:name="default"/>
            <bean class="org.springframework.cache.concurrent.ConcurrentMapCacheFactoryBean" p:name="books"/>
        </set>
    </property>
</bean>
```

上面的代码段使用`SimpleCacheManager`为名为`default`和`books`的两个嵌套的`ConcurrentMapCache`实例创建`CacheManager`。请注意，这些名称是为每个缓存直接配置的。

由于缓存是由应用程序创建的，因此它被绑定到其生命周期，使其适合于基本用例、测试或简单应用程序。高速缓存的规模很大，而且速度非常快，但它不提供任何管理或持久性功能，也没有任何回收的程序。

<a id="cache-store-configuration-ehcache"></a>

#### [](#cache-store-configuration-ehcache)8.5.2. 基于Ehcache的缓存

Ehcache 3.x完全与JSR-107兼容， 不需要专门的支持。

Ehcache 2.x实现在`org.springframework.cache.ehcache`包中。同样地，要使用它，需要简单地声明适当的 `CacheManager`。以下示例显示了如何执行此操作：

```xml
<bean id="cacheManager"
        class="org.springframework.cache.ehcache.EhCacheCacheManager" p:cache-manager-ref="ehcache"/>

<!-- EhCache library setup -->
<bean id="ehcache"
        class="org.springframework.cache.ehcache.EhCacheManagerFactoryBean" p:config-location="ehcache.xml"/>
```

此设置引导Spring IoC内部的ehcache库（通过`ehcache` bean），然后将其连接到专用的`CacheManager`实现中。 请注意，从`ehcache.xml`读取整个特定于`ehcache`的配置。

<a id="cache-store-configuration-caffeine"></a>

#### [](#cache-store-configuration-caffeine)8.5.3. Caffeine Cache

Caffeine是Java 8重写了Guava的缓存，他的实现在`org.springframework.cache.caffeine`包中，并且提供了访问Caffeine特性的方法。

以下示例配置一个按需创建缓存的`CacheManager`:

```xml
<bean id="cacheManager"
        class="org.springframework.cache.caffeine.CaffeineCacheManager"/>
```

还可以提供缓存以显式使用，在这种情况下，只有manager才能提供这些内容。以下示例显示了如何执行此操作:

```xml
<bean id="cacheManager" class="org.springframework.cache.caffeine.CaffeineCacheManager">
    <property name="caches">
        <set>
            <value>default</value>
            <value>books</value>
        </set>
    </property>
</bean>
```

Caffeine `CacheManager`也支持自定义的`Caffeine` 和 `CacheLoader`. 查阅 [Caffeine 文档](https://github.com/ben-manes/caffeine/wiki)了解更多信息。

<a id="cache-store-configuration-gemfire"></a>

#### [](#cache-store-configuration-gemfire)8.5.4. 基于GemFire的缓存

GemFire是一个面向内存/磁盘存储的全局的备份数据库，它具有可伸缩的、可持续的、可扩展的、具有内置模式订阅通知功能等等特性。全局复制的数据库。并提供全功能的边缘缓存。 有关如何将GemFire用作`CacheManager`（以及更多）的更多信息，请参阅[Spring Data GemFire参考文档](https://docs.spring.io/spring-gemfire/docs/current/reference/html/)。

<a id="cache-store-configuration-jsr107"></a>

#### [](#cache-store-configuration-jsr107)8.5.5. JSR-107 缓存

JSR-107兼容的缓存也可用于Spring的缓存抽象。JCache实现在`org.springframework.cache.jcache` 包中.

同样，要使用它，需要简单地声明适当的`CacheManager`。简单示例如下:

```xml
<bean id="cacheManager"
        class="org.springframework.cache.jcache.JCacheCacheManager"
        p:cache-manager-ref="jCacheManager"/>

<!-- JSR-107 cache manager setup  -->
<bean id="jCacheManager" .../>
```

<a id="cache-store-configuration-noop"></a>

#### [](#cache-store-configuration-noop)8.5.6.处理没有后端的缓存

有时在切换环境或进行测试时， 可能会只声明缓存， 而没有配置实际的后端缓存。由于这是一个无效的配置， 在运行时将引发异常， 因为缓存基础结构无法找到合适的存储。在这样的情况下， 与其删除缓存声明(这可能会很繁琐)， 你不如声明一个简单的，不执行缓存的虚拟缓存， 即强制每次执行缓存的方法。以下示例显示了如何执行此操作:

```xml
<bean id="cacheManager" class="org.springframework.cache.support.CompositeCacheManager">
    <property name="cacheManagers">
        <list>
            <ref bean="jdkCache"/>
            <ref bean="gemfireCache"/>
        </list>
    </property>
    <property name="fallbackToNoOpCache" value="true"/>
</bean>
```

`CompositeCacheManager`使用了多个`CacheManager`实例。此外，通过`fallbackToNoOpCache` 标志，添加了一个没有op的缓存，为所有的定义没有被配置的缓存管理器处理。 也就是说，在`jdkCache`或`gemfireCache`(上面配置)中找不到的每个缓存定义都将由无op缓存来处理，并且不会存储目标方法的信息而方法每次都会被执行（就是多配置成可执行的无缓存操作）。

<a id="cache-plug"></a>

### [](#cache-plug)8.6. 各种各样的后端缓存插件

显然，有大量的缓存产品可以用作后端存储。要将它们集成，需要提供`CacheManager`和`Cache`实现，因为不幸的是没有可用的标准，我们可以改用它。这听起来可能比使用它更难，因为在实践中，类往往是简单的[adapters](https://en.wikipedia.org/wiki/Adapter_pattern)，它将缓存抽象框架映射到存储API的顶部，就像`ehcache`类可以显示的那样。 大多数`CacheManager`类可以使用`org.springframework.cache.support`包中的类，如`AbstractCacheManager`，它负责处理样板代码，只留下实际的映射即可结束工作。我们希望，提供与Spring集成的库能够及时填补这一小的配置缺口。

<a id="cache-specific-config"></a>

### [](#cache-specific-config)8.7. 我可以如何设置TTL/TTI/Eviction policy/XXX特性?

直接通过缓存提供程序。 缓存抽象是抽象，而不是缓存实现。 您正在使用的解决方案可能支持不同的数据策略和不同的拓扑结构，而其他解决方案不会这样做(例如，JDK `ConcurrentHashMap` - 暴露在缓存抽象中将是无用的，因为没有后端支持)。应该通过后端缓存（配置时）或通过其本机API直接控制此类功能。