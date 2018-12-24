<a id="resources"></a>

[](#resources)2\. 资源
--------------------

本章介绍Spring如何处理资源以及如何在Spring中使用资源。 它包括以下主题:

*   [简介](#resources-introduction)

*   [资源接口](#resources-resource)

*   [内置资源实现](#resources-implementations)

*   [`ResourceLoader`](#resources-resourceloader)

*   [`ResourceLoaderAware` 接口](#resources-resourceloaderaware)

*   [资源依赖](#resources-as-dependencies)

*   [应用上下文和资源路径](#resources-app-ctx)

<a id="resources-introduction"></a>

### [](#resources-introduction)2.1. 简介

遗憾的是，Java的标准`java.net.URL`类和各种 `URL`前缀的标准处理程序不足以完全访问底层资源。例如，没有标准化的 `URL`实现可用于访问需要从类路径或相对于 `ServletContext`获取的资源。 虽然可以为专用 `URL`前缀注册新的处理程序（类似于`http:` :)这样的前缀的现有处理程序，但这通常非常复杂，并且`URL`接口仍然缺少一些理想的功能，例如检查当前资源是否存在的方法。

<a id="resources-resource"></a>

### [](#resources-resource)2.2. 资源接口

Spring的`Resource`接口的目标是成为一个更强大的接口，用于抽象对底层资源的访问。 以下清单显示了`Resource`接口定义：

```java
public interface Resource extends InputStreamSource {

    boolean exists();

    boolean isOpen();

    URL getURL() throws IOException;

    File getFile() throws IOException;

    Resource createRelative(String relativePath) throws IOException;

    String getFilename();

    String getDescription();

}
```

正如`Resource`接口的定义所示，它扩展了`InputStreamSource`接口。 以下清单显示了`InputStreamSource`接口的定义：

```java
public interface InputStreamSource {

    InputStream getInputStream() throws IOException;

}
```

`Resource` 接口中一些最重要的方法是：

*   `getInputStream()`: 用于定位和打开当前资源, 返回当前资源的`InputStream` ，预计每一次调用都会返回一个新的`InputStream`. 因此调用者必须自行关闭当前的输出流.

*   `exists()`: 返回`boolean`值，表示当前资源是否存在。

*   `isOpen()`: 返回`boolean`值，表示当前资源是否有已打开的输入流。如果为 `true`，那么`InputStream`不能被多次读取 ，只能在一次读取后即关闭以防止内存泄漏。除了`InputStreamResource`外，其他常用Resource实现都会返回`false`。

*   `getDescription()`: 返回当前资源的描述，当处理资源出错时，资源的描述会用于输出错误的信息。一般来说，资源的描述是一个完全限定的文件名称，或者是当前资源的真实URL。


其他方法允许您获取表示资源的实际`URL`或`File`对象（如果底层实现兼容并支持该功能）。

在Spring里， `Resource`抽象有着相当广泛的使用，例如，当需要某个资源时， `Resource`可以当作方法签名里的参数类型被使用。在Spring API中，有些方法（例如各种`ApplicationContext`实现的构造函数） 会直接采用普通格式的`String`路径来创建合适的`Resource`，调用者也可以通过在路径里带上指定的前缀来创建特定的`Resource`实现。

不但Spring内部和使用Spring的应用都大量地使用了`Resource`接口,而且开发者在应用代码中将它作为一个通用的工具类也是非常通用的。当你仅需要使用到`Resource`接口实现时， 可以直接忽略Spring的其余部分.虽然这样会与Spring耦合,但是也只是耦合一部分而已。使用这些`Resource`实现代替底层的访问是极其美好的。这与开发者引入其他库的目的也是一样的

`Resource`抽象不会取代功能。 它尽可能地包裹它。 例如，`UrlResource`包装`URL`并使用包装的URL来完成其工作。

<a id="resources-implementations"></a>

### [](#resources-implementations)2.3. 内置的资源实现

Spring包括以下`Resource`实现:

*   [`UrlResource`](#resources-implementations-urlresource)

*   [`ClassPathResource`](#resources-implementations-classpathresource)

*   [`FileSystemResource`](#resources-implementations-filesystemresource)

*   [`ServletContextResource`](#resources-implementations-servletcontextresource)

*   [`InputStreamResource`](#resources-implementations-inputstreamresource)

*   [`ByteArrayResource`](#resources-implementations-bytearrayresource)


<a id="resources-implementations-urlresource"></a>

#### [](#resources-implementations-urlresource)2.3.1. `UrlResource`

`UrlResource` 封装了`java.net.URL`用来访问正常URL的任意对象。例如`file:` ，HTTP目标，FTP目标等。所有的URL都可以用标准化的字符串来表示，例如通过正确的标准化前缀。 可以用来表示当前URL的类型。 这包括`file:`，用于访问文件系统路径，`http:` ：用于通过HTTP协议访问资源，`ftp:`：用于通过FTP访问资源，以及其他。

通过java代码可以显式地使用`UrlResource`构造函数来创建`UrlResource`，但也可以调用API方法来使用代表路径的String参数来隐式创建`UrlResource`。 对于后一种情况，JavaBeans `PropertyEditor`最终决定要创建哪种类型的`Resource`。如果路径字符串包含众所周知的（对于它，那么）前缀（例如 `classpath:`:)，它会为该前缀创建适当的专用`Resource`。 但是，如果它无法识别前缀，则假定该字符串是标准URL字符串并创建`UrlResource`。

<a id="resources-implementations-classpathresource"></a>

#### [](#resources-implementations-classpathresource)2.3.2. `ClassPathResource`

ClassPathResource代表从类路径中获取资源，它使用线程上下文加载器，指定类加载器或给定class类来加载资源。

当类路径上资源存于文件系统中时，`ClassPathResource`支持使用`java.io.File`来访问。但是当类路径上的资源位于未解压(没有被Servlet引擎或其他可解压的环境解压）的jar包中时， `ClassPathResource`就不再支持以`java.io.File`的形式访问。鉴于此，Spring中各种Resource的实现都支持以`java.net.URL`的形式访问资源。

可以显式使用`ClassPathResource`构造函数来创建`ClassPathResource`，但是更多情况下，是调用API方法使用的。即使用一个代表路径的`String`参数来隐式创建`ClassPathResource`。 对于后一种情况，将会由JavaBeans的`PropertyEditor`来识别路径中`classpath:`前缀，并创建`ClassPathResource`。

<a id="resources-implementations-filesystemresource"></a>

#### [](#resources-implementations-filesystemresource)2.3.3. `FileSystemResource`

`FileSystemResource`是用于处理`java.io.File`和`java.nio.file.Path`的实现，显然，它同时能解析作为`File`和作为`URL`的资源。

<a id="resources-implementations-servletcontextresource"></a>

#### [](#resources-implementations-servletcontextresource)2.3.4. `ServletContextResource`

这是`ServletContext`资源的 `Resource`实现，用于解释相关Web应用程序根目录中的相对路径。

ServletContextResource完全支持以流和URL的方式访问资源，但只有当Web项目是解压的（不是以war等压缩包形式存在），而且该ServletContext资源必须位于文件系统中， 它支持以`java.io.File`的方式访问资源。无论它是在文件系统上扩展还是直接从JAR或其他地方（如数据库）（可以想象）访问，实际上都依赖于Servlet容器。

<a id="resources-implementations-inputstreamresource"></a>

#### [](#resources-implementations-inputstreamresource)2.3.5. `InputStreamResource`

`InputStreamResource`是针对`InputStream`提供的`Resource`实现。在一般情况下，如果确实无法找到合适的`Resource`实现时，才去使用它。 同时请优先选择`ByteArrayResource`或其他基于文件的`Resource`实现，迫不得已的才使用它。

与其他`Resource` 实现相比，这是已打开资源的描述符。 因此，它从`isOpen()`返回`true`。

<a id="resources-implementations-bytearrayresource"></a>

#### [](#resources-implementations-bytearrayresource)2.3.6. `ByteArrayResource`

这是给定字节数组的`Resource`实现。 它为给定的字节数组创建一个`ByteArrayInputStream`。

当需要从字节数组加载内容时，ByteArrayResource会是个不错的选择，无需求助于单独使用的`InputStreamResource`。

<a id="resources-resourceloade"></a>

### [](#resources-resourceloader)2.4.`ResourceLoader`

`ResourceLoader`接口用于加载`Resource`对象，换句话说，就是当一个对象需要获取`Resource`实例时，可以选择实现`ResourceLoader`接口，以下清单显示了`ResourceLoader`接口定义：。

```java
public interface ResourceLoader {

    Resource getResource(String location);

}
```

所有应用程序上下文都实现`ResourceLoader`接口。 因此，可以使用所有应用程序上下文来获取 `Resource`实例。

当在特殊的应用上下文中调用`getResource()`方法以及指定的路径没有特殊前缀时，将返回适合该特定应用程序上下文的`Resource`类型。 例如，假设针对`ClassPathXmlApplicationContext` 实例执行了以下代码片段：

    Resource template = ctx.getResource("some/resource/path/myTemplate.txt");

针对`ClassPathXmlApplicationContext`，该代码返回 `ClassPathResource`。如果对`FileSystemXmlApplicationContext` 实例执行相同的方法，它将返回`FileSystemResource`。 对于`WebApplicationContext`，它将返回`ServletContextResource`。 它同样会为每个上下文返回适当的对象。

因此，您可以以适合特定应用程序上下文的方式加载资源。

另一方面，您可以通过指定特殊的`classpath:`前缀来强制使用`ClassPathResource`，而不管应用程序上下文类型如何，如下例所示：

```java
Resource template = ctx.getResource("classpath:some/resource/path/myTemplate.txt");
```

同样，您可以通过指定任何标准`java.net.URL`前缀来强制使用`UrlResource` 。 以下对示例使用`file` 和 `http`前缀：

```java
Resource template = ctx.getResource("file:///some/resource/path/myTemplate.txt");

Resource template = ctx.getResource("http://myhost.com/resource/path/myTemplate.txt");
```

下表总结了将：`String`对象转换为`Resource`对象的策略:

Table 10.资源字符串

| 前缀       | 示例                                                 | 解释                                                         |
| ---------- | ---------------------------------------------------- | ------------------------------------------------------------ |
| classpath: | `classpath:com/myapp/config.xml`                     | 从类路径加载                                                 |
| file:      | [file:///data/config.xml](file:///data/config.xml)   | 从文件系统加载为`URL`。 另请参见[`FileSystemResource` 警告。](#resources-filesystemresource-caveats) |
| http:      | [http://myserver/logo.png](http://myserver/logo.png) | 作为`URL`加载。                                              |
| (none)     | `/data/config.xml`                                   | 取决于底层的`ApplicationContext`。                           |

<a id="resources-resourceloaderaware"></a>

### [](#resources-resourceloaderaware)2.5. `ResourceLoaderAware` 接口

`ResourceLoaderAware`是一个特殊的标识接口，用来提供`ResourceLoader`引用的对象。以下清单显示了`ResourceLoaderAware`接口的定义：

```java
public interface ResourceLoaderAware {

    void setResourceLoader(ResourceLoader resourceLoader);
}
```

当类实现`ResourceLoaderAware`并部署到应用程序上下文（作为Spring管理的bean）时，它被应用程序上下文识别为`ResourceLoaderAware`。 然后，应用程序上下文调用`setResourceLoader(ResourceLoader)`，将其自身作为参数提供（请记住，Spring中的所有应用程序上下文都实现了`ResourceLoader`接口）。

由于`ApplicationContext`实现了`ResourceLoader`，因此bean还可以实现 `ApplicationContextAware` 接口并直接使用提供的应用程序上下文来加载资源。 但是，通常情况下，如果您需要，最好使用专用的`ResourceLoader`接口。 代码只能耦合到资源加载接口（可以被认为是实用程序接口），而不能耦合到整个Spring `ApplicationContext`接口。

从Spring 2.5开始，除了实现`ResourceLoaderAware`接口，还可以采取另外一种替代方案-依赖`ResourceLoader`的自动装配。 “传统”构造函数和byType [自动装配](#beans-factory-autowire)模式都支持对`ResourceLoader`的装配。 前者是以构造参数的形式装配，后者作为setter方法的参数参与装配。如果为了获得更大的灵活性（包括属性注入的能力和多参方法），可以考虑使用基于注解的新型注入方式。 使用注解`@Autowired`标识`ResourceLoader`变量，便可将其注入到成员属性、构造参数或方法参数中。这些参数需要ResourceLoader类型。 有关更多信息，请参阅使用[Using `@Autowired`](#beans-autowired-annotation)。

<a id="resources-as-dependencies"></a>

### [](#resources-as-dependencies)2.6. 资源依赖

如果bean本身要通过某种动态过程来确定和提供资源路径，那么bean使用`ResourceLoader`接口来加载资源就变得更有意义了。假如需要加载某种类型的模板，其中所需的特定资源取决于用户的角色 。如果资源是静态的，那么完全可以不使用`ResourceLoader`接口，只需让bean公开它需要的`Resource`属性，并按照预期注入属性即可。

是什么使得注入这些属性变得如此简单？是因为所有应用程序上下文注册和使用一个特殊的`PropertyEditor` JavaBean，它可以将 `String` paths转换为`Resource`对象。 因此，如果`myBean`有一个类型为`Resource`的模板属性，它可以用一个简单的字符串配置该资源。如下所示：

```xml
<bean id="myBean" class="...">
    <property name="template" value="some/resource/path/myTemplate.txt"/>
</bean>
```

请注意，资源路径没有前缀。 因此，因为应用程序上下文本身将用作`ResourceLoader`， 所以资源本身通过`ClassPathResource`，`FileSystemResource`或`ServletContextResource`加载，具体取决于上下文的确切类型。

如果需要强制使用特定的 `Resource`类型，则可以使用前缀。 以下两个示例显示如何强制`ClassPathResource`和`UrlResource`（后者用于访问文件系统文件）：

```xml
<property name="template" value="classpath:some/resource/path/myTemplate.txt">

<property name="template" value="file:///some/resource/path/myTemplate.txt"/>
```

<a id="resources-app-ctx"></a>

### [](#resources-app-ctx)2.7. 应用上下文和资源路径

本节介绍如何使用资源创建应用程序上下文，包括使用XML的快捷方式，如何使用通配符以及其他详细信息。

<a id="resources-app-ctx-construction"></a>

#### [](#resources-app-ctx-construction)2.7.1. 构造应用上下文

应用程序上下文构造函数（对于特定的应用程序上下文类型）通常将字符串或字符串数组作为资源的位置路径，例如构成上下文定义的XML文件。

当指定的位置路径没有带前缀时，那么从指定位置路径创建`Resource`类型（用于后续加载bean定义），具体取决于所使用应用上下文。 例如，请考虑以下示例，该示例创建`ClassPathXmlApplicationContext`：

```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("conf/appContext.xml");
```

bean定义是从类路径加载的，因为使用了`ClassPathResource`。 但是，请考虑以下示例，该示例创建 `FileSystemXmlApplicationContext`：

```java
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("conf/appContext.xml");
```

现在，bean定义是从文件系统位置加载的（在这种情况下，相对于当前工作目录）。

若位置路径带有classpath前缀或URL前缀，会覆盖默认创建的用于加载bean定义的`Resource`类型。请考虑以下示例：

```java
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("classpath:conf/appContext.xml");
```

使用`FileSystemXmlApplicationContext`从类路径加载bean定义。 但是，它仍然是`FileSystemXmlApplicationContext`。 如果它随后用作`ResourceLoader`，则任何未加前缀的路径仍被视为文件系统路径。

<a id="resources-app-ctx-classpathxml"></a>

##### [](#resources-app-ctx-classpathxml)构造`ClassPathXmlApplicationContext`实例的快捷方式

`ClassPathXmlApplicationContext`提供了多个构造函数，以利于快捷创建`ClassPathXmlApplicationContext`的实例。基础的想法是， 使用只包含多个XML文件名（不带路径信息）的字符串数组和一个Class参数的构造器，所省略路径信息`ClassPathXmlApplicationContext`会从`Class`参数中获取。

请考虑以下目录布局:

com/
  foo/
    services.xml
    daos.xml
    MessengerService.class

以下示例显示如何实例化由名为`services.xml`和`daos.xml`（位于类路径中）的文件中定义的bean组成的`ClassPathXmlApplicationContext`实例：

```java
ApplicationContext ctx = new ClassPathXmlApplicationContext(
    new String[] {"services.xml", "daos.xml"}, MessengerService.class);
```

有关各种构造函数的详细信息，请参阅[`ClassPathXmlApplicationContext`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/jca/context/SpringContextResourceAdapter.html)javadoc。

<a id="resources-app-ctx-wildcards-in-resource-paths"></a>

#### [](#resources-app-ctx-wildcards-in-resource-paths)2.7.2. 使用通配符构造应用上下文

从前文可知，应用上下文构造器的资源路径可以是单一的路径（即一对一地映射到目标资源）。也可以使用高效的通配符。可以包含特殊的"classpath*:"前缀或ant风格的正则表达式（使用Spring的PathMatcher来匹配）。

通配符机制可用于组装应用程序的组件，应用程序里所有组件都可以在一个公用的位置路径发布自定义的上下文片段，那么最终的应用上下文可使用`classpath*:`。 在同一路径前缀（前面的公用路径）下创建，这时所有组件上下文的片段都会被自动装配。

请注意，此通配符特定于在应用程序上下文构造函数中使用资源路径（或直接使用 `PathMatcher`实用程序类层次结构时），并在构造时解析。 它与资源类型本身无关。 您不能使用`classpath*:`前缀来构造实际的`Resource`,，因为资源一次只指向一个资源。

<a id="resources-app-ctx-ant-patterns-in-paths"></a>

##### [](#resources-app-ctx-ant-patterns-in-paths)Ant风格模式

路径位置可以包含Ant样式模式，如以下示例所示:

/WEB-INF/*-context.xml
com/mycompany/**/applicationContext.xml
file:C:/some/path/*-context.xml
classpath:com/mycompany/**/applicationContext.xml

当路径位置包含Ant样式模式时，解析程序遵循更复杂的过程来尝试解析通配符。解释器会先从位置路径里获取最靠前的不带通配符的路径片段， 并使用这个路径片段来创建一个`Resource`，并从中获取一个URL。 如果此URL不是`jar:`URL或特定于容器的变体（例如，在WebLogic中为`zip:`，在WebSphere中为`wsjar`，等等） 则从Resource里获取`java.io.File`对象，并通过其遍历文件系统。进而解决位置路径里通配符。 对于jar URL，解析器要么从中获取`java.net.JarURLConnection`， 要么手动解析jar URL，然后遍历jar文件的内容以解析通配符。

<a id="resources-app-ctx-portability"></a>

###### [](#resources-app-ctx-portability)可移植性所带来的影响

如果指定的路径定为文件URL（不管是显式还是隐式的），首先默认的`ResourceLoader`就是文件系统，其次通配符使用程序可以完美移植。

如果指定的路径是类路径位置，则解析器必须通过 `Classloader.getResource()`方法调用获取最后一个非通配符路径段URL。 因为这只是路径的一个节点（而不是末尾的文件），实际上它是未定义的（在`ClassLoader` javadoc中），在这种情况下并不能确定返回什么样的URL。 实际上，它始终会使用`java.io.File`来解析目录，其中类路径资源会解析到文件系统的位置或某种类型的jar URL，其中类路径资源解析为jar包的位置。 但是，这个操作就碰到了可移植的问题了。

如果获取了最后一个非通配符段的jar包URL，解析器必须能够从中获取`java.net.JarURLConnection`，或者手动解析jar包的URL，以便能够遍历jar的内容。 并解析通配符，这适用于大多数工作环境，但在某些其他特定环境中将会有问题，最后会导致解析失败，所以强烈建议在特定环境中彻底测试来自jar资源的通配符解析，测试成功之后再对其作依赖使用。

<a id="resources-classpath-wildcards"></a>

##### [](#resources-classpath-wildcards)The `classpath*:` 前缀

当构造基于XML文件的应用上下文时，位置路径可以使用`classpath*:`前缀。如以下示例所示：

```java
ApplicationContext ctx =
    new ClassPathXmlApplicationContext("classpath*:conf/appContext.xml");
```

`classpath*:`的使用表示该类路径下所有匹配文件名称的资源都会被获取（本质上就是调用了`ClassLoader.getResources(…​)`方法，接着将获取到的资源装配成最终的应用上下文。

通配符类路径依赖于底层类加载器的`getResources()` 方法。由于现在大多数应用程序服务器都提供自己的类加载器实现，因此行为可能会有所不同，尤其是在处理jar文件时。 要在指定服务器测试`classpath*` 是否有效，简单点可以使用`getClass().getClassLoader().getResources("<someFileInsideTheJar>")`来加载类路径jar包里的文件。 尝试在两个不同的路径加载相同名称的文件，如果返回的结果不一致，就需要查看一下此服务器中与classloader设置相关的文档。

您还可以将`classpath*:` 前缀与位置路径的其余部分中的 `PathMatcher`模式组合在一起（例如，`classpath*:META-INF/*-beans.xml`）。 这种情况的解析策略非常简单，取位置路径最靠前的无通配符片段，然后调用`ClassLoader.getResources()`获取所有匹配到的类层次加载器加载资源，随后将`PathMatcher`的策略应用于每一个得到的资源。

<a id="resources-wildcards-in-path-other-stuff"></a>

##### [](#resources-wildcards-in-path-other-stuff)通配符的补充说明

请注意，除非所有目标资源都存在文件系统中，否则`classpath*:`与Ant样式模式结合，都只能在至少有一个确定了根路径的情况下，才能达到预期的效果。 这意味着`classpath*:*.xml`等模式可能无法从jar文件的根目录中检索文件，而只能从根目录中的扩展目录中检索文件。

问题的根源是JDK的`ClassLoader.getResources()`方法的局限性。当向`ClassLoader.getResources()`传入空串时（表示搜索潜在的根目录）， 只能获取的文件系统的位置路径，即获取不了jar中文件的位置路径。Spring也会评估`URLClassLoader`运行时配置和jar文件中的`java.class.path`清单，但这不能保证导致可移植行为。

扫描类路径包需要在类路径中存在相应的目录条目。 使用Ant构建JAR时，请不要激活JAR任务的文件开关。 此外，在某些环境中，类路径目录可能不会基于安全策略公开 - 例如，JDK 1.7.0_45及更高版本上的独立应用程序（需要在清单中设置'Trusted-Library' 。 请参阅[http://stackoverflow.com/questions/19394570/java-jre-7u45-breaks-classloader-getresources](https://stackoverflow.com/questions/19394570/java-jre-7u45-breaks-classloader-getresources))）。

在JDK 9的模块路径（Jigsaw）上，Spring的类路径扫描通常按预期工作。 此处强烈建议将资源放入专用目录，避免上述搜索jar文件根级别的可移植性问题。

如果有多个类路径上都用搜索到的根包，那么使用`classpath:`和ant风格模式一起指定资源并不保证会找到匹配的资源。请考虑以下资源位置示例：

com/mycompany/package1/service-context.xml

现在考虑一个人可能用来尝试查找该文件的Ant风格路径:

classpath:com/mycompany/**/service-context.xml

这样的资源可能只在一个位置，但是当使用前面例子之类的路径来尝试解析它时，解析器会处理`getResource("com/mycompany");`返回的（第一个）URL。 当在多个类路径存在基础包节点`"com/mycompany"`时(如在多个jar存在这个基础节点），解析器就不一定会找到指定资源。因此，这种情况下建议结合使用`classpath*:` 和ant风格模式，`classpath*:`会让解析器去搜索所有包含基础包节点的类路径。

<a id="resources-filesystemresource-caveats"></a>

#### [](#resources-filesystemresource-caveats)2.7.3. `FileSystemResource` 的警告

当`FileSystemResource`与`FileSystemApplicationContext`之间没有联系（即，当`FileSystemApplicationContext`不是实际的`ResourceLoader`时）时会按预期处理绝对路径和相对路径。 相对路径是相对与当前工作目录而言的，而绝对路径则是相对文件系统的根目录而言的。

但是，出于向后兼容性（历史）的原因，当`FileSystemApplicationContext`是`ResourceLoader`时，这会发生变化。`FileSystemApplicationContext`强制所有有联系的`FileSystemResource`实例将所有位置路径视为相对路径， 无论它们是否以'/'开头。 实际上，这意味着以下示例是等效的：

```java
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("conf/context.xml");

ApplicationContext ctx =
    new FileSystemXmlApplicationContext("/conf/context.xml");
```

以下示例也是等效的（即使它们有所不同，因为一个案例是相对的而另一个案例是绝对的）：

```java
FileSystemXmlApplicationContext ctx = ...;
ctx.getResource("some/resource/path/myTemplate.txt");

FileSystemXmlApplicationContext ctx = ...;
ctx.getResource("/some/resource/path/myTemplate.txt");
```

实际上，如果确实需要使用绝对路径，建议放弃使用`FileSystemResource`和`FileSystemXmlApplicationContext`，而强制使用 `file:`的`UrlResource`。

```java
// actual context type doesn't matter, the Resource will always be UrlResource
ctx.getResource("file:///some/resource/path/myTemplate.txt");

// force this FileSystemXmlApplicationContext to load its definition via a UrlResource
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("file:///conf/context.xml");
```
