<a id="mvc"></a>

[](#mvc)1\. Spring Web MVC
--------------------------

Spring Web MVC是构建在Servlet API上的原始Web框架，从一开始就包含在Spring Framework中。 正式名称 “Spring Web MVC,” 来自其源模块([`spring-webmvc`](https://github.com/spring-projects/spring-framework/tree/master/spring-webmvc))的名称，但它通常被称为“Spring MVC”。

与Spring Web MVC并行，Spring Framework 5.0引入了一个反应堆栈Web框架，其名称“Spring WebFlux,”也基于其源模块([`spring-webflux`](https://github.com/spring-projects/spring-framework/tree/master/spring-webflux))。 本节介绍Spring Web MVC。 [下一节](web-reactive.html#spring-web-reactive)将介绍Spring WebFlux。.

有关基本信息以及与Servlet容器和Java EE版本范围的兼容性，请参阅Spring Framework [Wiki](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-Versions)。

<a id="mvc-servlet"></a>

### [](#mvc-servlet)1.1. DispatcherServlet

[与Spring WebFlux相同](web-reactive.html#webflux-dispatcher-handler)

Spring MVC和许多其他Web框架一样，围绕前端控制器模式设计，其中核心 `Servlet``DispatcherServlet`为请求处理提供共享算法，而实际工作由可配置委托组件执行。 该模型非常灵活，支持多种工作流程。

`DispatcherServlet`与任何 `Servlet`一样，需要使用Java配置或 `web.xml`根据Servlet规范进行声明和映射。 反过来，`DispatcherServlet`使用Spring配置来发现请求映射，视图解析，异常处理 [等等](#mvc-servlet-special-bean-types)所需的委托组件。

下面的Java配置示例注册并初始化`DispatcherServlet`，它由Servlet容器自动检测（请参阅[Servlet Config](#mvc-container-config)）：

```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletCxt) {

        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext ac = new AnnotationConfigWebApplicationContext();
        ac.register(AppConfig.class);
        ac.refresh();

        // Create and register the DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(ac);
        ServletRegistration.Dynamic registration = servletCxt.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/app/*");
    }
}
```

除了直接使用ServletContext API之外，您还可以扩展`AbstractAnnotationConfigDispatcherServletInitializer` 并覆盖特定方法（请参阅[Context Hierarchy](#mvc-servlet-context-hierarchy)下的示例）。

以下`web.xml`配置示例注册并初始化`DispatcherServlet`:

```xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/app-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value></param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>

</web-app>
```

Spring Boot遵循不同的初始化顺序。 Spring Boot使用Spring配置来引导自身和嵌入式Servlet容器，而不是挂钩到Servlet容器的生命周期。 在Spring配置中检测`Filter` 和`Servlet`声明，并在Servlet容器中注册。 有关更多详细信息，请参阅[Spring Boot

documentation](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-embedded-container)文档。

<a id="mvc-servlet-context-hierarchy"></a>

#### [](#mvc-servlet-context-hierarchy)1.1.1. 上下文层次结构

`DispatcherServlet`需要一个 `WebApplicationContext`（ApplicationContext的扩展）来配置自己。 `WebApplicationContext`有一个指向`ServletContext`的链接以及与之关联的 `Servlet`。 它还绑定到`ServletContext`，当需要访问它时，应用程序可以使用`RequestContextUtils`上的静态方法来查找`WebApplicationContext`。

对于许多应用程序，拥有一个简单的 `WebApplicationContext`已经足够了。它也有一个上下文层次结构，其中根`WebApplicationContext`在多个 `DispatcherServlet`（或其他 `Servlet`）实例之间共享， 每个实例都有自己的子`WebApplicationContext`配置。 有关上下文层次结构功能的更多信息，请参阅[`ApplicationContext`的其他功能](core.html#context-introduction)。

根WebApplicationContext通常包含bean基础结构，例如需要跨多个Servlet实例共享的数据存储库和业务服务。 这些bean被有效继承，可以在特定于 `Servlet`的子`WebApplicationContext`中重写（即重新声明），它通常包含给定`Servlet`本地的bean。 下图显示了这种关系：

![mvc context hierarchy](https://github.com/DocsHome/spring-docs/blob/master/pages/images/mvc-context-hierarchy.png)

以下示例配置`WebApplicationContext`层次结构:

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { App1Config.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/app1/*" };
    }
}
```

如果不需要应用程序上下文层次结构，则应用程序可以通过`getRootConfigClasses()` 返回所有配置，并从`getServletConfigClasses()`返回`null`。

以下示例显示了`web.xml`配置（和上面效果一样）:

```xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/root-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app1</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/app1-context.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app1</servlet-name>
        <url-pattern>/app1/*</url-pattern>
    </servlet-mapping>

</web-app>
```

如果不需要应用程序上下文层次结构，则应用程序可以仅配置“root”上下文，并将 `contextConfigLocation` Servlet参数保留为空。

<a id="mvc-servlet-special-bean-types"></a>

#### [](#mvc-servlet-special-bean-types)1.1.2. 特殊的Bean类型

[Same as in Spring WebFlux](web-reactive.html#webflux-special-bean-types)

`DispatcherServlet` 委托特殊bean处理请求并渲染视图。 “特殊bean”是指实现WebFlux框架的Spring管理的`Object`实例。 这些通常带有内置联系，但您可以自定义其属性并扩展或替换它们。

下表列出了`DispatcherHandler`检测到的特殊bean:

|  Bean 类型    |  说明    |
| ---- | ---- |
|  `HandlerMapping`    |  将请求映射到处理程序以及用于预处理和后处理的[拦截器](#mvc-handlermapping-interceptor)列表。 其映射规则基于某些标准，其细节因`HandlerMapping`实现而异。两个主要的`HandlerMapping`实现是`RequestMappingHandlerMapping`（它支持`@RequestMapping`带注解的方法） 和`SimpleUrlHandlerMapping` （它维护对处理程序的URI路径模式的显式注册）。|
|   `HandlerAdapter`   |  无论实际调用处理程序如何，都可以帮助`DispatcherServlet` 调用映射到请求的处理程序。 例如，调用带有注解的控制器，需要从注解中解析一些信息。 `HandlerAdapter`的主要目的是保护`DispatcherServlet`不受此类细节的影响。    |
|  [`HandlerExceptionResolver`](#mvc-exceptionhandlers)    |  解决异常的策略，他可以将捕获到的异常映射到处理程序，HTML错误视图或其他目标。 请参阅[Exceptions](#mvc-exceptionhandlers)。    |
| [`ViewResolver`](#mvc-viewresolver)     | 将从处理程序返回的逻辑基于`String`的视图名称解析为用于呈现给响应的实际`View`。 请参阅 [View Resolution](#mvc-viewresolver) and [View Technologies](#mvc-view)。     |
| [`LocaleResolver`](#mvc-localeresolver), [LocaleContextResolver](#mvc-timezone)     | 解析客户端正在使用的 `Locale`以及可能的时区，以便能够提供国际化视图。 请参阅[Locale](#mvc-localeresolver)。|
| [`ThemeResolver`](#mvc-themeresolver)     |解决Web应用程序可以使用的主题 - 例如，提供个性化布局。 见[Themes](#mvc-themeresolver)。      |
|[`MultipartResolver`](#mvc-multipart)      | 解析multi-part的请求（例如：浏览器表单文件上载）。请参阅[Multipart Resolver](#mvc-multipart)。     |
| [`FlashMapManager`](#mvc-flash-attributes)     |存储和检索“input” 和“output”`FlashMap`，可用于将属性从一个请求传递到另一个请求，通常是通过重定向。 请参阅[Flash Attributes](#mvc-flash-attributes)。      |



<a id="mvc-servlet-config"></a>

#### [](#mvc-servlet-config)1.1.3. Web MVC 配置

[Same as in Spring WebFlux](web-reactive.html#webflux-framework-config)

对于每种类型的[特殊bean](#mvc-servlet-special-bean-types)， `DispatcherServlet`首先会检查`WebApplicationContext`。如果没有匹配的bean类型，则会退回检查[`DispatcherServlet.properties`](https://github.com/spring-projects/spring-framework/blob/master/spring-webmvc/src/main/resources/org/springframework/web/servlet/DispatcherServlet.properties)。

在大多数情况下，[MVC Config](#mvc-config)是最佳起点。 它以Java或XML声明所需的bean，并提供更高级别的配置回调API来自定义它。

Spring Boot依赖于MVC Java配置来配置Spring MVC并提供许多额外的便捷选项。

<a id="mvc-container-config"></a>

#### [](#mvc-container-config)1.1.4. Servlet 配置

在Servlet 3.0+环境中，您可以选择以编程方式配置Servlet容器作为替代方法，也可以与`web.xml`文件结合使用。 以下示例注册`DispatcherServlet`:

```java
import org.springframework.web.WebApplicationInitializer;

public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) {
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

        ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
```

`WebApplicationInitializer`是Spring MVC提供的一个接口，实现此接口的任何Servlet 3容器都可被检测到并自动初始化。 `AbstractDispatcherServletInitializer`抽象类实现了`WebApplicationInitializer`接口，通过重写方法来指定servlet映射和`DispatcherServlet` 配置的地址， 从而更方便的注册`DispatcherServlet`。

对于使用基于Java的Spring配置的应用程序，建议使用此方法，如以下示例所示:

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { MyWebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

如果使用基于XML的Spring配置，则应直接从`AbstractDispatcherServletInitializer`扩展，如以下示例所示:

```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    @Override
    protected WebApplicationContext createRootApplicationContext() {
        return null;
    }

    @Override
    protected WebApplicationContext createServletApplicationContext() {
        XmlWebApplicationContext cxt = new XmlWebApplicationContext();
        cxt.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
        return cxt;
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

`AbstractDispatcherServletInitializer`还提供了一种便捷的方法来添加`Filter`实例并将它们自动映射到DispatcherServlet，如以下示例所示：

```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    // ...

    @Override
    protected Filter[] getServletFilters() {
        return new Filter[] {
            new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
    }
}
```

每个过滤器都根据其具体类型添加默认名称，并自动映射到`DispatcherServlet`。

`AbstractDispatcherServletInitializer`的protected方法`isAsyncSupported`提供了一个单独的地址来启用`DispatcherServlet` 上的异步支持以及映射到它的所有过滤器。 默认情况下，此标志设置为`true`。

最后，如果您需要进一步自定义`DispatcherServlet`本身，则可以覆盖`createDispatcherServlet`方法。

<a id="mvc-servlet-sequence"></a>

#### [](#mvc-servlet-sequence)1.1.5. 处理流程

[Same as in Spring WebFlux](web-reactive.html#webflux-dispatcher-handler-sequence)

`DispatcherServlet`按如下方式处理请求：

*   首先，搜索应用的上下文对象`WebApplicationContext`，并把它作为一个属性（attribute)绑定到该请求上。以便让控制器和其他组件能使用它。 属性的键名默认为`DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE`。

*   将locale resolver绑定到请求上，并允许其他组件解析处理请求时使用的语言环境（渲染视图，准备数据等）。 如果您不需要区域解析，则不需要locale resolver。

*   将theme resolver 绑定到请求，以允许视图等组件确定要使用的themes。 如果您不使用themes，则可以忽略它。

*   如果指定multipart 文件处理器，则会检查请求的文件是不是multiparts的， 如果是，请求将包装在`MultipartHttpServletRequest`中， 以便其他组件进一步处理。 有关[Multipart Resolver](#mvc-multipart)的更多信息，请参见[Multipart Resolver](#mvc-multipart)。

*   为该请求查找一个合适的处理器。 如果找到处理程序，则与该处理器关联的整条执行链（前处理器、后处理器、控制器等）都会被执行，以完成相应模型的准备或视图的渲染。 或者，对于带注解的控制器，可以显示响应（在 `HandlerAdapter`中）而不是返回视图。

*   如果处理器返回模型，则渲染视图。 如果没有返回模型（可能是由于前处理器或后处理器拦截请求，可能是出于安全原因），则不会渲染任何视图，因为该请求可能已经完成。


在 `WebApplicationContext` 中声明的`HandlerExceptionResolver`用于解决请求处理过程中引发的异常。这些异常解析程序允许使用自定义的逻辑来解决，有关详细信息，请参阅[Exceptions](#mvc-exceptionhandlers) 。

Spring的 `DispatcherServlet`也允许处理器返回Servlet API规范中定义的最后修改时间戳（`last-modification-date`）值。确定请求最后修改时间的方式是直截了当的： `DispatcherServlet`会先查找合适的处理映射来找到请求对应的处理器，然后检测它是否实现了 `LastModified` 接口。如果是的话，则调用接口的`long getLastModified(request)`方法，并将返回的值传回给客户端。

您可以自定义通过`DispatcherServlet`的配置。可以在`web.xml`文件中，声明元素Servlet的上添加Servlet的初始化参数（`init-param`元素）。 下表列出了支持的参数：



|      参数| 说明     |
| ---- | ---- |
|  `contextClass`    |实现`ConfigurableWebApplicationContext`的类，由此类通过本地配置来初始化 Servlet实例。 默认情况下，使用`XmlWebApplicationContext`。      |
|`contextConfigLocation`      |一个指定了上下文配置文件路径的字符串，并传递给上下文实例（由`contextClass`指定） 。该字符串可能包含多个字符串（使用逗号作为分隔符）以支持多个上下文。 对于具有两次定义的bean的多个上下文位置，最新位置优先（即最后加载的为准）。      |
|`namespace`      |`WebApplicationContext`的命名空间。 默认为`[servlet-name]-servlet`。      |
|`throwExceptionIfNoHandlerFound`      |当没有找到请求的处理程序时是否抛出 `NoHandlerFoundException`。 然后可以使用 `HandlerExceptionResolver`捕获异常（例如，使用`@ExceptionHandler`控制器方法）并像处理其他任何方法一样处理异常。默认情况下，此参数设置为 `false`，在这种情况下，`DispatcherServlet`将响应状态设置为404（NOT_FOUND），而不会引发异常。请注意，如果配置了 [默认servlet处理](#mvc-default-servlet-handler) ，则始终将未解析的请求转发到默认servlet，并且永远不会引发404。      |


<a id="mvc-handlermapping-interceptor"></a>

#### [](#mvc-handlermapping-interceptor)1.1.6. Interception

Spring的处理器映射机制包含了处理器拦截器，，可以实现 `HandlerMapping`，所有 `HandlerMapping`实现都支持处理拦截器，这些拦截器在需要为特定类型的请求应用一些功能时可能很有用非常有用， 例如，检查用户身份等，`org.springframework.web.servlet`包中的 `HandlerInterceptor`实现了三种方法，提供足够的灵活性来执行各种预处理和后处理：

*   `preHandle(..)`: 在执行实际处理程序之前

*   `postHandle(..)`: 在执行实际处理程序之后

*   `afterCompletion(..)`: 完成请求后


`preHandle(..)` 方法返回一个布尔值。 您可以使用此方法来中断或继续执行链的处理。 当此方法返回 `true`时，处理程序执行链继续。 当它返回false时，`DispatcherServlet`假定拦截器本身已处理请求（例如，呈现适当的视图）并且不继续执行执行链中的其他拦截器和实际处理程序。

有关如何配置[Interceptors](#mvc-config-interceptors) 的示例，请参阅MVC配置一节中的拦截器。 您还可以使用各个 `HandlerMapping`实现上的setter方法直接注册它们。

请注意，在HandlerAdapter和postHandle之前，响应被写入并提交。 `postHandle`对于`@ResponseBody`和ResponseEntity方法不太有用， 这意味着对响应进行任何更改都为时已晚，例如添加额外的header。 对于此类方案，您可以实现 `ResponseBodyAdvice`并将其声明为[Controller Advice](#mvc-ann-controller-advice)bean或直接在`RequestMappingHandlerAdapter`上进行配置。

<a id="mvc-exceptionhandlers"></a>

#### [](#mvc-exceptionhandlers)1.1.7. 异常

[Same as in Spring WebFlux](web-reactive.html#webflux-dispatcher-exceptions)

如果在请求映射期间发生异常或从请求处理程序（例如 `@Controller`）抛出异常， 则`DispatcherServlet`委托给 `HandlerExceptionResolver` bean来处理并解决异常，这通常是错误响应。

下表列出了可用的 `HandlerExceptionResolver` 实现：:

Table 2. HandlerExceptionResolver 实现 ：



| `HandlerExceptionResolver`     | Description     |
| ---- | ---- |
|`SimpleMappingExceptionResolver`      |异常类名称和错误视图名称之间的映射。 用于在浏览器应用程序中呈现错误页面。      |
|[`DefaultHandlerExceptionResolver`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/mvc/support/DefaultHandlerExceptionResolver.html)      |解决Spring MVC引发的异常并将它们映射到HTTP状态代码。 另请参阅备用`ResponseEntityExceptionHandler`和[REST API exceptions](#mvc-ann-rest-exceptions)异常。      |
|`ResponseStatusExceptionResolver`      |使用`@ResponseStatus`注解解析异常，并根据注解中的值将它们映射到HTTP状态代码。      |
|`ExceptionHandlerExceptionResolver`      |通过在`@Controller`或 `@ControllerAdvice`类中调用 `@ExceptionHandler`方法来解决异常。 请参阅[@ExceptionHandler methods](#mvc-ann-exceptionhandler)方法。      |

<a id="mvc-excetionhandlers-handling"></a>

##### [](#mvc-excetionhandlers-handling)解析链

您可以通过在Spring配置中声明多个 `HandlerExceptionResolver` bean并根据需要设置其顺序属性来形成异常解析链。 `order`属性越高，异常解析器定位的越晚。

`HandlerExceptionResolver`的约定指定它可以返回:

*   一个指向错误视图的 `ModelAndView`。

*   如果在解析程序中处理异常，则为空的`ModelAndView`。

*   如果异常仍未解析，则为 `null`，以供后续解析器尝试，如果异常保留在最后，则允许冒泡到Servlet容器。.


[MVC Config](#mvc-config)自动声明内置的解析器，用于默认的Spring MVC异常，`@ResponseStatus` 带注解的异常，以及对`@ExceptionHandler`方法的支持。 您可以自定义该列表或替换它。

<a id="mvc-ann-customer-servlet-container-error-page"></a>

##### [](#mvc-ann-customer-servlet-container-error-page)容器错误页面

如果任何`HandlerExceptionResolver`仍未解析异常，并且因此将其传播给servlet容器或者如果响应状态设置为错误状态（即4xx，5xx） ，则Servlet容器可以呈现HTML中的默认错误页面。 要自定义容器的默认错误页面，可以在 `web.xml`.中声明错误页面映射。 以下示例显示了如何执行此操作：

```xml
<error-page>
    <location>/error</location>
</error-page>
```

根据前面的示例，当异常冒泡或响应具有错误状态时，Servlet容器会在容器内对配置的URL进行ERROR调度（例如，`/error`）。 然后由`DispatcherServlet`处理，可能将其映射到 `@Controller`，可以实现该控件以返回带有模型的错误视图名称或呈现JSON响应，如以下示例所示：

```java
@RestController
public class ErrorController {

    @RequestMapping(path = "/error")
    public Map<String, Object> handle(HttpServletRequest request) {
        Map<String, Object> map = new HashMap<String, Object>();
        map.put("status", request.getAttribute("javax.servlet.error.status_code"));
        map.put("reason", request.getAttribute("javax.servlet.error.message"));
        return map;
    }
}
```

Servlet API没有提供在Java中创建错误页面映射的方法。 但是，您可以同时使用`WebApplicationInitializer`和最小的 `web.xml`。

<a id="mvc-viewresolver"></a>

#### [](#mvc-viewresolver)1.1.8. 视图解析

[Same as in Spring WebFlux](web-reactive.html#webflux-viewresolution)

Spring MVC定义了 `ViewResolver`和 `View`接口，使您可以在浏览器中呈现模型，而无需将您与特定的视图技术联系起来。 `ViewResolver`提供视图名称和实际视图之间的映射。 `View`接口负责准备请求，并将请求的渲染交给某种具体的视图技术实现。

下表提供了有关`ViewResolver`层次结构的更多详细信息：:

Table 3. ViewResolver 实现

| ViewResolver     | Description     |
| ---- | ---- |
| `AbstractCachingViewResolver`     | `AbstractCachingViewResolver` 的子类缓存它们解析的视图实例。 缓存可提高某些视图技术的性能。 您可以通过将`cache`属性设置为 `false`.来关闭缓存。 此外，如果必须在运行时刷新某个视图（例如，修改FreeMarker模板时），则可以使用`removeFromCache(String viewName, Locale loc)` 方法。|
| `XmlViewResolver`     |  实现`ViewResolver`，它必须和Spring的XML bean工厂有相同的DTD以。 默认配置文件是`/WEB-INF/views.xml`。    |
| `ResourceBundleViewResolver`     | `ViewResolver`的实现，它使用由bundle根路径指定的`ResourceBundle`中的bean定义作为配置。 对于它应该解析的每个视图，它使用属性`[viewname].(class)`的值作为视图类， 并使用属性 `[viewname].url` 的值作为视图URL。 您可以在 [视图技术](#mvc-view)一章中找到示例。     |
| `UrlBasedViewResolver`     | `ViewResolver`接口的简单实现，它不需要其他任何显式的映射说明，而直接使用URL来解析到逻辑视图名。 如果您的逻辑名称与真正的视图资源的名称匹配，则不需要任何映射。     |
| `InternalResourceViewResolver`     | `UrlBasedViewResolver`的便捷子类，支持 `InternalResourceView`（实际上是Servlet和JSP）和子类，如`JstlView` 和 `TilesView`。 您可以使用`setViewClass(..)`为此解析程序生成的所有视图指定视图类。 有关详细信息，请参阅[`UrlBasedViewResolver`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/reactive/result/view/UrlBasedViewResolver.html)javadoc。     |
|`FreeMarkerViewResolver`      |`UrlBasedViewResolver` 的便捷子类，支持 `FreeMarkerView`及其自定义子类。      |
| `ContentNegotiatingViewResolver`     | 实现 `ViewResolver` 接口，该接口根据请求文件名或Accept头解析视图。 请参阅 [Content Negotiation](#mvc-multiple-representations)。     |


<a id="mvc-viewresolver-handling"></a>

##### [](#mvc-viewresolver-handling)处理

[Same as in Spring WebFlux](web-reactive.html#webflux-viewresolution-handling)

您可以在视图解析器链中声明多个视图解析器，并在必要时通过设置 `order`属性来指定排序。 请记住，order属性越高，视图解析器在链中的位置越晚。 .

`ViewResolver` 可以返回null以指示无法找到该视图。 但是，对于JSP和`InternalResourceViewResolver`, 确定JSP是否存在的唯一方法是通过`RequestDispatcher`执行调度。 因此，您必须始终将`InternalResourceViewResolver`配置为视图解析器的整体顺序中的最后一个。

配置视图解析就像将`ViewResolver` bean添加到Spring配置一样简单。[MVC Config](#mvc-config)为[View Resolvers](#mvc-config-view-resolvers)提供专用配置API，并添加无逻辑视图控制器（[View Controllers](#mvc-config-view-controller) ），这些控制器对于没有控制器逻辑的HTML模板渲染非常有用。

<a id="mvc-redirecting-redirect-prefix"></a>

##### [](#mvc-redirecting-redirect-prefix)重定向

[Same as in Spring WebFlux](web-reactive.html#webflux-redirecting-redirect-prefix)

您可以在视图中使用`redirect:`前缀来执行重定向。`UrlBasedViewResolver`（及其子类）将此识别为需要重定向的指令。 视图名称的其余部分是重定向URL。

控制器本身可以根据逻辑视图名称进行操作。 逻辑视图名称（例如`redirect:/myapp/some/resource`）相对于当前Servlet上下文重定向，而名称如`redirect:http://myhost.com/some/arbitrary/path`重定向到绝对URL。

请注意，如果使用`@ResponseStatus`注解控制器方法，则注解值优先于 `RedirectView`设置的响应状态。

<a id="mvc-redirecting-forward-prefix"></a>

##### [](#mvc-redirecting-forward-prefix)转发

你也可以在视图名称中使用`forward:`前缀，来作为 `UrlBasedViewResolver`和其子类最终解析的视图名称。 这将创建一个 `InternalResourceView`，它执行`RequestDispatcher.forward()`。 因此，此前缀对于 `InternalResourceViewResolver`和`InternalResourceView`（对于JSP）没有用，但如果您使用其他视图技术时仍希望强制Servlet/JSP引擎处理资源的转发，则它可能会有所帮助。 请注意，您也可以链接多个视图解析器。

<a id="mvc-multiple-representations"></a>

##### [](#mvc-multiple-representations)Content Negotiation

[Same as in Spring WebFlux](web-reactive.html#webflux-multiple-representations)

[`ContentNegotiatingViewResolver`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/view/ContentNegotiatingViewResolver.html)本身不解析视图，而是委托给其他视图解析器，并选择类似于客户端请求的表示的视图。 可以从`Accept`头或查询参数（例如， `"/path?format=pdf"`）确定表示。

`ContentNegotiatingViewResolver`通过将请求的媒体类型与其每个ViewResolvers关联的View支持的媒体类型（也称为`Content-Type`）进行比较，选择适当的 `View`来处理请求。列表中具有兼容Content-Type的第一个`View`将表示返回给客户端。 如果 `ViewResolver`链无法提供兼容视图，则会查询通过`DefaultViews` 属性指定的视图列表。 后一个选项适用于单个视图，它可以呈现当前资源的适当表示，而不管逻辑视图名称如何。 `Accept`头可以包含通配符（例如`text/*`），在这种情况下，`Content-Type`为`text/xml`的View是兼容匹配。

有关配置详细信息，请参阅 [MVC Config](#mvc-config) 下的[View Resolvers](#mvc-config-view-resolvers) 。

<a id="mvc-localeresolver"></a>

#### [](#mvc-localeresolver)1.1.9. Locale

正如Spring Web MVC框架所做的那样，Spring架构的大多数部分都支持国际化。 `DispatcherServlet`允许您使用客户端的语言环境自动解析消息。 这是通过`LocaleResolver`对象完成的。

当请求进入时，`DispatcherServlet` 会查找当前语言环境解析器，如果找到，则会尝试使用它来设置语言环境。 您可以通过使用`RequestContext.getLocale()`方法，来获取由区域解析器解析到的结果。

除了自动解析语言环境之外，您还可以在处理程序时添加拦截器（有关拦截器的更多信息，请参阅[Interception](#mvc-handlermapping-interceptor) ），以便于在特定情况下更改语言环境。例如（通过请求中的参数来改变语言环境）

区域解析器和拦截器在 `org.springframework.web.servlet.i18n`包中定义，并以正常方式在应用程序上下文中进行配置。 Spring中包含以下选择的语言环境解析器。

*   [Time Zone](#mvc-timezone)

*   [Header Resolver](#mvc-localeresolver-acceptheader)

*   [Cookie Resolver](#mvc-localeresolver-cookie)

*   [Session Resolver](#mvc-localeresolver-session)

*   [Locale Interceptor](#mvc-localeresolver-interceptor)


<a id="mvc-timezone"></a>

##### [](#mvc-timezone)Time Zone

除了获取客户端的区域设置外，了解其时区通常也很有用。 `LocaleContextResolver` 接口提供了`LocaleResolver`的扩展，它允许解析器提供更丰富的 `LocaleContext`，其中可能包含时区信息。

当此解析器可用时，可以使用`RequestContext.getTimeZone()`方法获取用户的`TimeZone`。 时区信息由Spring的`ConversionService`注册的任何Date/Time `Converter`和`Formatter` 对象自动使用。

<a id="mvc-localeresolver-acceptheader"></a>

##### [](#mvc-localeresolver-acceptheader)Header Resolver

此区域解析器检查客户端（例如，Web浏览器）发送的请求头中的`accept-language`。 通常，此字段包含客户端操作系统的区域设置。 请注意，此解析器不支持时区信息。

<a id="mvc-localeresolver-cookie"></a>

##### [](#mvc-localeresolver-cookie)Cookie Resolver

此区域解析器检查客户端上可能存在的 `Cookie`，以查看是否指定了`Locale`或`TimeZone`。 如果是，则使用指定的详细信息。 通过使用此区域解析器的属性，您可以指定cookie的名称以及失效时间。 以下示例定义`CookieLocaleResolver`：

```xml
<bean id="localeResolver" class="org.springframework.web.servlet.i18n.CookieLocaleResolver">

    <property name="cookieName" value="clientlanguage"/>

    <!-- in seconds. If set to -1, the cookie is not persisted (deleted when browser shuts down) -->
    <property name="cookieMaxAge" value="100000"/>

</bean>
```

下表描述了 `CookieLocaleResolver`的属性：

Table 4. CookieLocaleResolver properties:

| Property     | Default     | Description     |
| ---- | ---- | ---- |
|`cookieName`      |classname + LOCALE      | cookie的名字     |
| `cookieMaxAge`     |Servlet container default      | Cookie在客户端上持续存在的最长时间。 如果指定`-1`，则不会保留cookie。 它仅在客户端关闭浏览器之前可用。     |
| `cookiePath`     | /     | 限制cookie对您网站某个部分的可见性。 当指定了 `cookiePath`时，cookie仅对该路径及其下方的路径可见。 |



<a id="mvc-localeresolver-session"></a>

##### [](#mvc-localeresolver-session)Session Resolver

您可以使用 `SessionLocaleResolver`从与用户请求关联的Session中获取`Locale` 和`TimeZone`。 与`CookieLocaleResolver`相比，此策略将本地选择的区域设置存储在Servlet容器的 `HttpSession`中。 因此，这些设置对于每个会话都是临时的，这些设置在会话结束时会丢失。

请注意，与外部会话管理机制没有直接关系，例如Spring Session项目。 此`SessionLocaleResolver` 根据当前的`HttpServletRequest`评估和修改相应的`HttpSession`属性。

<a id="mvc-localeresolver-interceptor"></a>

##### [](#mvc-localeresolver-interceptor)Locale Interceptor

您可以通过将`LocaleChangeInterceptor`添加到其中一个 `HandlerMapping`定义来启用语言环境的更改。 它会检测请求中的参数并相应地更改语言环境，在程序的应用程序上下文中调用 `LocaleResolver` 上的`setLocale`方法。 下一个示例显示，当调用包含名为 `siteLanguage`的参数的所有`*.view`资源时更改了区域设置。 例如，对URL的请求 `[http://www.sf.net/home.view?siteLanguage=nl](https://www.sf.net/home.view?siteLanguage=nl)`将网站语言更改为荷兰语。 以下示例显示如何拦截区域设置:

```xml
<bean id="localeChangeInterceptor"
        class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor">
    <property name="paramName" value="siteLanguage"/>
</bean>

<bean id="localeResolver"
        class="org.springframework.web.servlet.i18n.CookieLocaleResolver"/>

<bean id="urlMapping"
        class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
    <property name="interceptors">
        <list>
            <ref bean="localeChangeInterceptor"/>
        </list>
    </property>
    <property name="mappings">
        <value>/**/*.view=someController</value>
    </property>
</bean>
```

<a id="mvc-themeresolver"></a>

#### [](#mvc-themeresolver)1.1.10. 主题

您可以使用Spring Web MVC框架自带的主题来设置应用程序的整体外观，从而增强用户体验。 主题是静态资源的集合，通常是样式表和图像，它们会影响应用程序的视觉样式。

<a id="mvc-themeresolver-defining"></a>

##### [](#mvc-themeresolver-defining)定义一个主题

要在Web应用程序中使用主题，必须设置 `org.springframework.ui.context.ThemeSource`接口的实现。 `WebApplicationContext`接口扩展了 `ThemeSource`， 但将其职责委托给专用实现。 默认情况下，委托是 `org.springframework.ui.context.support.ResourceBundleThemeSource`的实现。它从类路径的根目录加载属性文件。 要使用自定义 `ThemeSource`实现或配置 `ResourceBundleThemeSource`的名称前缀，可以在应用程序上下文中使用保留名称`themeSource`注册bean。 Web应用程序上下文自动检测具有该名称的bean并使用它。

使用 `ResourceBundleThemeSource`时，主题在简单属性文件中定义。 属性文件列出构成主题的资源，如以下示例所示：

styleSheet=/themes/cool/style.css
background=/themes/cool/img/coolBg.jpg

属性的键是从视图代码引用主题元素的名称。 对于JSP，通常使用`spring:theme` 自定义标签执行此操作，该标记与`spring:message`标签非常相似。 以下JSP片段使用上一示例中定义的主题来自定义外观：

```jsp
<%@ taglib prefix="spring" uri="http://www.springframework.org/tags"%>
<html>
    <head>
        <link rel="stylesheet" href="<spring:theme code='styleSheet'/>" type="text/css"/>
    </head>
    <body style="background=<spring:theme code='background'/>">
        ...
    </body>
</html>
```

默认情况下，`ResourceBundleThemeSource`使用空的名称前缀。 因此，从类路径的根加载属性文件。 因此，您可以将`cool.properties`主题定义放在类路径根目录的目录中（例如，在 `/WEB-INF/classes`中）。 `ResourceBundleThemeSource`使用标准的Java资源包加载机制，从而使主题也具有国际化。 例如，我们可以有一个`/WEB-INF/classes/cool_nl.properties`，它引用一个带有荷兰文本的特殊背景图像。

<a id="mvc-themeresolver-resolving"></a>

##### [](#mvc-themeresolver-resolving)解析主题

定义主题后，[如上一节所述](#mvc-themeresolver-defining)，您可以决定使用哪个主题。 `DispatcherServlet`查找名为`themeResolver`的bean，以找出要使用的`ThemeResolver`实现。 主题解析器的工作方式与`LocaleResolver`的工作方式大致相同。 它检测用于特定请求的主题，还可以更改请求的主题。 下表描述了Spring提供的主题解析器：

Table 5. ThemeResolver implementations :

|Class      | Description     |
| ---- | ---- |
| `FixedThemeResolver`     |选择使用`defaultThemeName`      |
| `SessionThemeResolver`     | 主题在用户的HTTP会话中维护。 它只需要为每个会话设置一次，但不会在会话之间保留。     |
| `CookieThemeResolver`     |所选主题存储在客户端的cookie中。      |

Spring还提供了一个`ThemeChangeInterceptor`，它允许通过简单的请求参数对每个请求进行主题更改。

<a id="mvc-multipart"></a>

#### [](#mvc-multipart)1.1.11. Multipart 解析器

[Same as in Spring WebFlux](web-reactive.html#webflux-multipart)

`org.springframework.web.multipart` 包中的 `MultipartResolver`是一种用于解析包括文件上传在内的多部分请求的策略。 他包含了一个[Commons FileUpload](https://jakarta.apache.org/commons/fileupload) 的实现，另一个基于Servlet 3.0多部分请求解析。

要启用多部分处理，Spring的配置文件中，在 `DispatcherServlet` 配置名称为`multipartResolver`的`MultipartResolver` bean。 `DispatcherServlet`会自动检测并将其应用于请求中。 当收到内容类型为`multipart/form-data`的POST请求时，解析器会解析内容并将当前的`HttpServletRequest`包装为 `MultipartHttpServletRequest`，以提供对已解析部分的访问，并将其作为请求参数公开。

<a id="mvc-multipart-resolver-commons"></a>

##### [](#mvc-multipart-resolver-commons)Apache Commons `FileUpload`

要使用Apache Commons `FileUpload`，您可以配置名为 `multipartResolver`的`CommonsMultipartResolver`r类型的bean。 您还需要添加`commons-fileupload`依赖。

<a id="mvc-multipart-resolver-standard"></a>

##### [](#mvc-multipart-resolver-standard)Servlet 3.0

需要通过Servlet容器配置启用Servlet 3.0多部分解析:

*   在Java中，在注册Servlet时设置 `MultipartConfigElement`。

*   在`web.xml`中，将“`"<multipart-config>"`”部分添加到servlet声明中。


以下示例显示如何在注册Servlet时设置 `MultipartConfigElement`:

```java
public class AppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    // ...

    @Override
    protected void customizeRegistration(ServletRegistration.Dynamic registration) {

        // Optionally also set maxFileSize, maxRequestSize, fileSizeThreshold
        registration.setMultipartConfig(new MultipartConfigElement("/tmp"));
    }

}
```

一旦您配置好Servlet 3.0，您就可以添加名为`multipartResolver`的`StandardServletMultipartResolver`类型的bean。

<a id="mvc-logging"></a>

#### [](#mvc-logging)1.1.12. 日志

[Same as in Spring WebFlux](web-reactive.html#webflux-logging)

Spring MVC中的DEBUG级别日志记录旨在实现紧凑，简约和人性化。 它侧重于那些一次又一次使用的高价值信息，其他的只有在调试特定问题时才有用。

TRACE级日志记录通常遵循与DEBUG相同的原则（例如，不应该是fire hose），但可以用于调试任何问题。 此外，一些日志消息可能在TRACE与DEBUG中显示不同的详细程度。

良好的日志记录来自使用日志的经验。 如果您发现任何不符合既定目标的事件，请告知我们。

<a id="mvc-logging-sensitive-data"></a>

##### [](#mvc-logging-sensitive-data)敏感数据

[Same as in Spring WebFlux](web-reactive.html#webflux-logging-sensitive-data)

DEBUG和TRACE日志记录可能会记录敏感信息。 这就是默认情况下屏蔽请求参数和请求头的原因，并且必须通过 `DispatcherServlet`上的 `enableLoggingRequestDetails`属性显式启用它们的完整日志记录。

以下示例说明如何使用Java配置执行此操作：

```java
public class MyInitializer
        extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return ... ;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return ... ;
    }

    @Override
    protected String[] getServletMappings() {
        return ... ;
    }

    @Override
    protected void customizeRegistration(Dynamic registration) {
        registration.setInitParameter("enableLoggingRequestDetails", "true");
    }

}
```

<a id="filters"></a>

### [](#filters)1.2. 过滤器

[Same as in Spring WebFlux](web-reactive.html#webflux-filters)

`spring-web`模块提供了一些有用的过滤器:

*   [Form Data（表单数据）](#filters-http-put)

*   [Forwarded Headers（转发请求头）](#filters-forwarded-headers)

*   [Shallow ETag（）](#filters-shallow-etag)

*   [CORS](#filters-cors)


<a id="filters-http-put"></a>

#### [](#filters-http-put)1.2.1. 表单数据

浏览器只能通过HTTP GET或HTTP POST提交表单数据，但非浏览器客户端也可以使用HTTP PUT，PATCH和DELETE提交表单数据。 Servlet API要求`ServletRequest.getParameter*()`方法仅支持HTTP POST的表单字段访问。.

`spring-web`模块提供 `FormContentFilter`过滤器来拦截HTTP PUT，PATCH和DELETE请求，请求类型为 `application/x-www-form-urlencoded`， `FormContentFilter`从请求中读取表单数据， 并包装 `ServletRequest`，然后可以通过 `ServletRequest.getParameter*()`系列方法提供表单数据。

<a id="filters-forwarded-headers"></a>

#### [](#filters-forwarded-headers)1.2.2. Forwarded Headers（转发请求头）

[Same as in Spring WebFlux](web-reactive.html#webflux-forwarded-headers)

当通过代理主机或者端口或者其他方案请求时（例如，负载均衡），这是，从客户端角度看，创建正确的主机，端口或者其他方案成为一项挑战，

[RFC 7239](https://tools.ietf.org/html/rfc7239) RFC 7239定义了代理可以用来提供有关原始请求信息的转发HTTP头。 还有其他非标准头文件，包括`X-Forwarded-Host`, `X-Forwarded-Port`, `X-Forwarded-Proto`, `X-Forwarded-Ssl`, 和 `X-Forwarded-Prefix`。

`ForwardedHeaderFilter`是一个Servlet过滤器，它根据 `Forwarded` 头部信息修改请求的主机，端口和方案，然后删除请求头。

当转发请求头时需要注意的安全事项，因为应用程序无法知道请求头是代理按我们想的那样添加还是由客户端恶意添加，这就是为什么应该将信任边界的代理配置为删除来自外部的不受信任的转发请求头。 您还可以使用`removeOnly=true`配置`ForwardedHeaderFilter`，在这种情况下，它会删除但不使用标头。

<a id="filters-shallow-etag"></a>

#### [](#filters-shallow-etag)1.2.3. Shallow ETag

`ShallowEtagHeaderFilter`过滤器通过缓存写入响应的内容并从中计算MD5哈希来创建 “shallow”ETag。 客户端下次发送时， 它会执行相同操作，但它也会将计算值与`If-None-Match`请求头进行比较，如果两者相等，则返回304（NOT_MODIFIED）。

此策略可以节省网络带宽，但不能节省CPU，因为必须为每个请求计算完整响应。 前面描述的控制器级别的其他策略可以避免计算。 请参阅 [HTTP Caching](#mvc-caching)。

此过滤器具有`writeWeakETag`参数，该参数将过滤器配置为写入弱ETag，类似于以下内容：`W/"02a2d595e6ed9a0b24f027f2b63b134d6"`（如[RFC 7232 Section 2.3](https://tools.ietf.org/html/rfc7232#section-2.3)）。

<a id="filters-cors"></a>

#### [](#filters-cors)1.2.4. CORS

[Same as in Spring WebFlux](web-reactive.html#webflux-filters-cors)

Spring MVC通过控制器上的注解为CORS配置提供细粒度的支持。 但是，当与Spring Security一起使用时，我们建议依赖于必须在Spring Security的过滤器链之前配置的内置`CorsFilter`。

有关更多详细信息，请参阅 [CORS](#mvc-cors)和[CORS Filter](#mvc-cors-filter) 过滤器部分。

<a id="mvc-controller"></a>

### [](#mvc-controller)1.3. 注解控制器

[Same as in Spring WebFlux](web-reactive.html#webflux-controller)

Spring MVC提供了基于注解的编程模型，其中`@Controller` 和 `@RestController` 组件使用注解来表示请求映射、请求输入、异常处理等。被注解的控制器拥有灵活的方法签名，并且无需扩展基类或实现特定的接口。以下示例显示了由注解定义的控制器：

```java
@Controller
public class HelloController {

    @GetMapping("/hello")
    public String handle(Model model) {
        model.addAttribute("message", "Hello World!");
        return "index";
    }
}
```

在前面的示例中，该方法接受`Model`并将视图名称作为String返回，但是存在许多其他选项，本章稍后将对其进行说明。

有关[spring.io](https://spring.io/guides)的指南和教程，请使用本节中介绍的基于注解的编程模型。

<a id="mvc-ann-controller"></a>

#### [](#mvc-ann-controller)1.3.1. 声明

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-controller)

您可以在Servlet的`WebApplicationContext`中使用标准的Spring bean定义来定义控制器bean。`@Controller`模板允许自动检测， 与Spring支持检测类路径中的`@Component`类一样，并会自动注册bean定义。它还充当注解类的模板，表示它充当的是Web组件的角色。

要启用`@Controller` bean的自动检测，您可以将组件扫描添加到Java配置中，如以下示例所示:

    @Configuration
    @ComponentScan("org.example.web")
    public class WebConfig {

        // ...
    }

以下示例显示了与前面示例等效的XML配置:

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:p="http://www.springframework.org/schema/p"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="org.example.web"/>

    <!-- ... -->

</beans>
```

`@RestController`是一个[组合注解](core.html#beans-meta-annotations) ，它本身由`@Controller` 和 `@ResponseBody`元注解组成。 其每个方法都继承类型级别（type-level）的 `@ResponseBody`注解，因此，直接写入响应主体与视图渲染和使用HTML模板。

<a id="mvc-ann-requestmapping-proxying"></a>

##### [](#mvc-ann-requestmapping-proxying)AOP 代理

在某些情况下，您需要在运行时使用AOP代理装饰控制器。 例如，如果您想在控制器上直接使用`@Transactional`注解。 在这种情况下，对于控制器而言，我们建议使用基于类的代理。 这通常也是控制器的默认选择。 但是，如果控制器没有实现Spring Context回调的接口 （例如`InitializingBean`, `*Aware`等）， 则可能需要显式配置基于类的代理。 例如，使用 `<tx:annotation-driven/>`，您可以更改为`<tx:annotation-driven proxy-target-class="true"/>`。

<a id="mvc-ann-requestmapping"></a>

#### [](#mvc-ann-requestmapping)1.3.2. Request Mapping

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestmapping)

`@RequestMapping`注解用于将请求映射到控制器方法。它具有各种属性，可以通过URL、HTTP方法、请求参数、请求头参数（headers）和媒体类型进行匹配。 可以在类级别使用它来表示共享映射，或在方法级别上用于缩小到特定的端点映射范围。

还有`@RequestMapping`的HTTP方法特定的缩写变量:

*   `@GetMapping`

*   `@PostMapping`

*   `@PutMapping`

*   `@DeleteMapping`

*   `@PatchMapping`


这些简洁的注解是[自定义注解](#mvc-ann-requestmapping-composed)，因为，大多数的控制器方法应该映射到HTTP方法而不是使用`@RequestMapping`。默认情况下， `@RequestMapping`和所有HTTP方法匹配。在类上定义的仍然需要`@RequestMapping` 来表示共享映射。

以下示例具有类型和方法级别映射:

```java
@RestController
@RequestMapping("/persons")
class PersonController {

    @GetMapping("/{id}")
    public Person getPerson(@PathVariable Long id) {
        // ...
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public void add(@RequestBody Person person) {
        // ...
    }
}
```

<a id="mvc-ann-requestmapping-uri-templates"></a>

##### [](#mvc-ann-requestmapping-uri-templates)URI 模式匹配

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestmapping-uri-templates)

你可以使用glob模式和通配符来映射请求:

*   `?` 匹配一个字符

*   `*` 匹配路径段一个或多个字符

*   `**` 匹配0个或多个路径段


您还可以使用 `@PathVariable`声明URI变量并访问它们的值，如以下示例所示:

    @GetMapping("/owners/{ownerId}/pets/{petId}")
    public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
        // ...
    }

您可以在类和方法级别声明URI变量，如以下示例所示：

```java
@Controller
@RequestMapping("/owners/{ownerId}")
public class OwnerController {

    @GetMapping("/pets/{petId}")
    public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
        // ...
    }
}
```

URI变量会自动转换为适当的类型，或者引发 `TypeMismatchException`。 默认情况下支持简单类型（`int`, `long`, `Date`等），您也可以注册对任何其他数据类型的支持。 请参见[Type Conversion](#mvc-ann-typeconversion) and [`DataBinder`](#mvc-ann-initbinder)。

你可以显示命名URI 变量(例如, `@PathVariable("customId")`),但是如果名称是相同的，并且代码是使用调试信息编译的，或者在Java 8中使用`-parameters` 编译器标记。 则可以保留该详细信息。

语法`{varName:regex}`声明一个具有正则表达式的URI变量，其语法为`{varName:regex}`。例如，给定URL`"/spring-web-3.0.5 .jar"`，以下方法提取名称，版本和文件扩展名:

    @GetMapping("/{name:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{ext:\\.[a-z]+}")
    public void handle(@PathVariable String version, @PathVariable String ext) {
        // ...
    }

URI路径模式还可以嵌入`${…​}`，在启动时通过`PropertyPlaceHolderConfigurer`解析本地、系统、环境和其他属性源时解析的占位符。例如，这种模式可以使用基于某些外部配置对基URL进行参数化

Spring MVC使用 `PathMatcher`联系和 `AntPathMatcher`实现位于`spring-core` URI路径匹配。

<a id="mvc-ann-requestmapping-pattern-comparison"></a>

##### [](#mvc-ann-requestmapping-pattern-comparison)模式比较

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestmapping-pattern-comparison)

当多个模式与URL匹配时，必须对它们进行比较以找到最佳匹配。 这是通过使用 `AntPathMatcher.getPatternComparator(String path)`来完成的，它会查找更具体的模式。

如果URI变量的数量较少且单个通配符计为1且双通配符计为2，那么模式就不那么具体了。如果模式得到的分数相等，那么会选择较长的模式匹配。如果分数和长度都相同，则会选择拥有比通配符更多的URI变量的模式。

默认映射模式（`/**`）从评分中排除，并始终排在最后。 此外，前缀模式（例如`/public/**`）被认为比没有双通配符的其他模式更不具体。

有关详细信息，请参阅[`AntPathMatcher`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/util/AntPathMatcher.html) 中的 [`AntPatternComparator`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/util/AntPathMatcher.AntPatternComparator.html)。 您可以自定义[`PathMatcher`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/util/PathMatcher.html) 的实现. 请参阅 配置中的[Path Matching](#mvc-config-path-matching)

<a id="mvc-ann-requestmapping-suffix-pattern-match"></a>

##### [](#mvc-ann-requestmapping-suffix-pattern-match)后缀匹配

默认情况下,默认情况下，Spring MVC执行`.*` 后缀模式匹配，以便映射到 `/person`的控制器也隐式映射到 `/person.*`。这里使用文件扩展名来解释用于响应的请求内容类型（即，而不是 `Accept`请求头） \- 例如，`/person.pdf`，`/person.xml`等。

当浏览器用于发送难以持续交互的`Accept`头时，必须以这种方式使用文件扩展名。目前，这不再是必需的，判断 `Accept`头应该是首选。

随着时间的推移，文件扩展名的使用已经证明有多种方式存在问题。 当使用URI变量，路径参数和URI编码进行覆盖时，它可能会导致歧义。 有关基于URL的授权和安全性的推理（有关更多详细信息，请参阅下一节）也变得更加困难。

要完全禁用文件扩展名，必须同时设置以下两项:

*   `useSuffixPatternMatching(false)`, see [PathMatchConfigurer](#mvc-config-path-matching)

*   `favorPathExtension(false)`, see [ContentNegotiationConfigurer](#mvc-config-content-negotiation)


基于URL的内容协商仍然有用（例如，在浏览器中键入URL时）。 为此，我们建议使用基于查询参数的策略来避免文件扩展名带来的大多数问题。 或者，如果必须使用文件扩展名，请考虑通过[ContentNegotiationConfigurer](#mvc-config-content-negotiation)的`mediaTypes`属性将它们限制为显式注册的扩展名列表。

<a id="mvc-ann-requestmapping-rfd"></a>

##### [](#mvc-ann-requestmapping-rfd)后缀匹配和RFD

反射文件下载（Reflected file download）攻击与XSS类似，因为它依赖请求输入，例如查询参数、URI变量，并且在响应中被反射。但是，RFD攻击不是将JavaScript插入HTML，而是依赖浏览器切换来执行下载，进而在之后的双击时将响应作为可执行脚本处理。

在Spring MVC中，`@ResponseBody`和 `ResponseEntity`方法存在风险，因为它们可以呈现不同的内容类型，客户端可以通过URL路径扩展来请求。 禁用后缀模式匹配并使用路径扩展进行内容协商可降低风险，但不足以防止RFD攻击。

为了防止RFD攻击，在呈现响应主体之前，需要在Spring MVC添加 `Content-Disposition:inline;filename=f.txt`头用于提供固定和安全的下载文件。只有在URL路径包含的文件扩展名中既不包含白名单，也没有为内容协商显式注册以时，才需要这样做。 但是，在浏览器直接键入URL时，可能会产生副作用。

默认情况下，有许多常见的路径扩展白名单。具有自定义`HttpMessageConverter`实现的应用程序可以显式注册内容协商的文件扩展名，以避免为这些扩展添加`Content-Disposition` 头。 请参阅[Content Types](#mvc-config-content-negotiation)

有关RFD的其他建议，请参阅[CVE-2015-5211](https://pivotal.io/security/cve-2015-5211)。

<a id="mvc-ann-requestmapping-consumes"></a>

##### [](#mvc-ann-requestmapping-consumes)消费者媒体类型

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestmapping-consumes)

您可以根据请求的`Content-Type`缩小请求映射范围，如以下示例所示:

```java
@PostMapping(path = "/pets", consumes = "application/json") (1)
public void addPet(@RequestBody Pet pet) {
    // ...
}
```

**1**、使用 `consumes`属性来缩小内容类型的映射。

`consumes`属性还支持否定表达式 \- 例如，`!text/plain`表示除 `text/plain`之外的任何内容类型。

您可以在类级别声明共享 `consumes`属性。 但是，与大多数其他请求映射属性不同，在类级别使用时，方法级别会 `consumes` 属性覆盖而不是扩展类级别声明。

`MediaType`为常用媒体类型提供常量，例如`APPLICATION_JSON_VALUE` and `APPLICATION_XML_VALUE`。

<a id="mvc-ann-requestmapping-produces"></a>

##### [](#mvc-ann-requestmapping-produces)生产者媒体类型

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestmapping-produces)

您可以根据`Accept`请求头和控制器方法生成的内容类型列表来缩小请求映射，如以下示例所示：

```java
@GetMapping(path = "/pets/{petId}", produces = "application/json;charset=UTF-8") (1)
@ResponseBody
public Pet getPet(@PathVariable String petId) {
    // ...
}
```

**1**

使用`produces`属性来缩小内容类型的映射。

媒体类型可以指定字符集。 支持否定表达式 \- 例如， `!text/plain`表示"text/plain"以外的任何内容类型。

对于JSON内容类型，即使 [RFC7159](https://tools.ietf.org/html/rfc7159#section-11)明确声明“没有为此注册定义charset参数”，也应指定UTF-8字符集，因为某些浏览器要求它正确解释UTF-8特殊字符。

您可以在类级别声明共享的`produces`属性。 但是，与大多数其他请求映射属性不同，在类级别使用时，方法级别会生成属性覆盖，而不是扩展类级别声明。

`MediaType`为常用媒体类型提供常量，例如`APPLICATION_JSON_UTF8_VALUE` and `APPLICATION_XML_VALUE`。

<a id="mvc-ann-requestmapping-params-and-headers"></a>

##### [](#mvc-ann-requestmapping-params-and-headers)参数, 请求头

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestmapping-params-and-headers)

您可以根据请求参数条件缩小请求映射。 您可以测试是否存在请求参数（`myParam`），缺少一个（`!myParam`）或特定值（`myParam=myValue`）。 以下示例显示如何测试特定值：

```java
@GetMapping(path = "/pets/{petId}", params = "myParam=myValue") (1)
public void findPet(@PathVariable String petId) {
    // ...
}
```

**1**、测试`myParam`是否等于`myValue`。

您还可以将其与请求头条件一起使用，如以下示例所示：:

```java
@GetMapping(path = "/pets", headers = "myHeader=myValue") (1)
public void findPet(@PathVariable String petId) {
    // ...
}
```

**1**、Testing whether `myHeader` equals `myValue` 。.

您可以将 `Content-Type`和 `Accept` 与headers条件匹配，但最好使用[consumes](#mvc-ann-requestmapping-consumes)和[produces](#mvc-ann-requestmapping-produces)替代。

<a id="mvc-ann-requestmapping-head-options"></a>

##### [](#mvc-ann-requestmapping-head-options)HTTP HEAD, OPTIONS

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestmapping-head-options)

`@GetMapping` (和 `@RequestMapping(method=HttpMethod.GET)`)一样，为了请求映射的目的，透明地支持HTTP HEAD以进行请求映射。控制器方法无需更改。 在`javax.servlet.http.HttpServlet`中应用的响应包确保有`Content-Length`标头并且设置为写入的字节数，但实际上不会写入响应。

`@GetMapping` (and `@RequestMapping(method=HttpMethod.GET)`)一样，为了请求映射的目的，被隐式映射到并支持HTTP HEAD，处理HTTP HEAD请求就像它是HTTP GET一样，但不是写入正文，而是计算字节数并设置 `Content-Length`头。

默认情况下，HTTP OPTIONS通过设置`Allow`响应头来为所有具有匹配URL模式的@RequestMapping方法中列出的HTTP方法列表来处理HTTP选项。

对于没有HTTP方法声明的`@RequestMapping`，Allow请求头可以设置为 `GET,HEAD,POST,PUT,PATCH,DELETE,OPTIONS`。 控制器方法应始终声明支持的HTTP方法（例如，通过使用特定于HTTP方法的变体：`@GetMapping`, `@PostMapping`等）。

您可以将 `@RequestMapping` 方法显式映射到HTTP HEAD和HTTP OPTIONS，但在常见情况下这不是必需的。

<a id="mvc-ann-requestmapping-composed"></a>

##### [](#mvc-ann-requestmapping-composed)自定义注解

[Same as in Spring WebFlux](web-reactive.html#mvc-ann-requestmapping-head-options)

Spring MVC支持使用 [组合注解](core.html#beans-meta-annotations)进行请求映射。 这些注解本身是使用 `@RequestMapping`进行元注解的，并且用于重新声明具有更窄，更具体目的的 `@RequestMapping`属性的子集（或全部）。

`@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, 和 `@PatchMapping` 就是组合注解最好的示例， 提供它们是因为，可以说，大多数控制器方法应该映射到特定的HTTP方法，而不是使用`@RequestMapping`，默认情况下，它与所有HTTP方法匹配。 如果您需要组合注解的示例，请查看如何声明这些注解。

Spring MVC还支持使用自定义请求匹配逻辑的自定义请求映射属性。这是一个更高级的选项，需要继承`RequestMappingHandlerMapping` 并覆盖`getCustomMethodCondition` 方法， 您可以在其中检查自定义属性并返回自己的`RequestCondition`。

<a id="mvc-ann-requestmapping-registration"></a>

##### [](#mvc-ann-requestmapping-registration)显式注册

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestmapping-registration)

您可以以编程方式注册处理程序方法，您可以将其用于动态注册或高级情况，例如不同URL下的同一处理程序的不同实例。 以下示例注册处理程序方法：

```java
@Configuration
public class MyConfig {

    @Autowired
    public void setHandlerMapping(RequestMappingHandlerMapping mapping, UserHandler handler) (1)
            throws NoSuchMethodException {

        RequestMappingInfo info = RequestMappingInfo
                .paths("/user/{id}").methods(RequestMethod.GET).build(); (2)

        Method method = UserHandler.class.getMethod("getUser", Long.class); (3)

        mapping.registerMapping(info, handler, method); (4)
    }

}
```

**1**、为控制器注入目标处理程序和处理程序映射

**2**、准备映射元数据的请求。

**3**、获取处理程序方法。

**4**、添加注册。

<a id="mvc-ann-methods"></a>

#### [](#mvc-ann-methods)1.3.3. 程序处理方法

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-methods)

`@RequestMapping` 处理程序方法具有灵活的签名,可以从一系列受支持的控制器方法参数和返回值中进行选择.

<a id="mvc-ann-arguments"></a>

##### [](#mvc-ann-arguments)方法参数

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-arguments)

下表显示了受支持的控制器方法参数，任何参数都不支持响应式(Reactive)类型。

JDK 8’s `java.util.Optional` 作为方法参数来支持的，它与具有必需属性的注解(例如`@RequestParam`, `@RequestHeader`等相结合)。 并且等同于`required=false`。



| 控制器方法参数     |描述      |
| ---- | ---- |
|`WebRequest`, `NativeWebRequest`      |无需直接使用Servlet API即可访问请求参数以及request和session属性。      |
| `javax.servlet.ServletRequest`, `javax.servlet.ServletResponse`     | 选择任何特定的请求或响应类型 \- 例如，`ServletRequest`, `HttpServletRequest`或Spring的`MultipartRequest`, `MultipartHttpServletRequest`。     |
| `javax.servlet.http.HttpSession`     |强制进行会话。 因此，此类参数永远不可能为`null`. 请注意，会话访问不是线程安全的。 如果允许多个请求同时访问会话，请考虑将`RequestMappingHandlerAdapter`实例的`synchronizeOnSession`标志设置为true。      |
|`javax.servlet.http.PushBuilder`      | Spring4.0 push生成器API用于编程HTTP/2资源推送， 请注意，根据Servlet规范，如果客户端不支持该HTTP/2功能，则注入的`PushBuilder`实例可以为null。     |
| `java.security.Principal`     |当前经过身份验证的用户 \- 如果已知，可能是特定的`Principal`实现类。      |
| `HttpMethod`     |请求的HTTP方法。      |
| `java.util.Locale`     |当前请求区域设置，由最可用的`LocaleResolver`（实际上是已配置的 `LocaleResolver`或`LocaleContextResolver`）确定。      |
| `java.util.TimeZone` \+ `java.time.ZoneId`     | 与当前请求关联的时区，由`LocaleContextResolver`确定。     |
|`java.io.InputStream`, `java.io.Reader`|用于访问Servlet API公开的原始请求主体。|
|`java.io.OutputStream`, `java.io.Writer`|用于访问Servlet API公开的原始响应主体。|
|`@PathVariable`|用于访问URI模板变量。 请参阅[URI模式](#mvc-ann-requestmapping-uri-templates)。|
|`@MatrixVariable`|用于访问URI路径段中的名称 - 值对。 请参见[矩阵变量](#mvc-ann-matrix-variables)。|
|`@RequestParam`|用于访问Servlet请求参数，包括多部分文件。 参数值将转换为声明的方法参数类型。 请参阅[`@RequestParam`](#mvc-ann-requestparam)以及[Multipart](#mvc-multipart-forms)。 请注意，对于简单的参数值，使用`@RequestParam`是可选的。 请参阅本表末尾的“任何其他参数”。|
|`@RequestHeader`|用于访问请求头。 头的值将转换为声明的方法参数类型。 请参阅[`@RequestHeader`](#mvc-ann-requestheader)。|
|`@CookieValue`|用于访问cookie。 Cookie值将转换为声明的方法参数类型。 请参阅[`@CookieValue`](#mvc-ann-cookievalue)。|
|`@RequestBody`|用于访问HTTP请求正文。 通过使用`HttpMessageConverter`实现将正文内容转换为声明的方法参数类型。 请参阅 [`@RequestBody`](#mvc-ann-requestbody)。|
|`HttpEntity<B>`|用于访问请求标头和正文。 使用`HttpMessageConverter`转换正文。 见[HttpEntity](#mvc-ann-httpentity)。|
|`@RequestPart`|要访问`multipart/form-data`请求中的部件，请使用`HttpMessageConverter`转换部件的主体。 见[Multipart](#mvc-multipart-forms)。|
|`java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap`|用于访问HTML控制器中使用的模型，并将其作为视图呈现的一部分暴露给模板。|
|`RedirectAttributes`|指定在重定向（即，要附加到查询字符串）时使用的属性，以及临时存储的flash属性，直到重定向后的请求为止。 请参阅 [重定向属性](#mvc-redirecting-passing-data)和[Flash属性](#mvc-flash-attributes)。|
|`@ModelAttribute`|用于访问模型中的现有属性（如果不存在则实例化），并应用数据绑定和验证。 请参阅[`@ModelAttribute`](#mvc-ann-modelattrib-method-args) 以及[Model](#mvc-ann-modelattrib-methods)和[`DataBinder`](#mvc-ann-initbinder)。请注意，使用`@ModelAttribute` 是可选的（例如，设置其属性）。 请参阅本表末尾的“任何其他参数”。|
|`Errors`, `BindingResult`|用于访问来自命令对象的验证和数据绑定的错误（即 `@ModelAttribute`参数）或来自验证 `@RequestBody`或 `@RequestPart` 参数的错误。 您必须在经过验证的方法参数后立即声明`Errors`或`BindingResult`参数。|
|`SessionStatus` \+ class-level `@SessionAttributes`|用于标记表单处理完成，从而触发通过类级别`@SessionAttributes`注解声明的会话属性的清除。 有关更多详细信息，请参阅[`@SessionAttributes`](#mvc-ann-sessionattributes)。|
|`UriComponentsBuilder`|用于准备相对于当前请求的主机，端口，方案，上下文路径和servlet映射的文字部分的URL。 请参阅[URI Links](#mvc-uri-building)。|
|`@SessionAttribute`|用于访问任何会话属性，与由于类级别`@SessionAttributes`声明的结束形成对比。 有关更多详细信息，请参阅 [`@SessionAttribute`](#mvc-ann-sessionattribute)。|
|`@RequestAttribute`|用于访问请求属性。 有关更多详细信息，请参阅[`@RequestAttribute`](#mvc-ann-requestattrib)。|
|Any other argument|如果方法参数与此表中的任何值不匹配，并且它是一个简单类型（由 [BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)确定， 则它被解析为 `@RequestParam`。否则，它将被解析为`@ModelAttribute`。|



<a id="mvc-ann-return-types"></a>

##### [](#mvc-ann-return-types)返回值

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-return-types)

下表描述了支持的控制器方法返回值。 所有返回值都支持响应式类型。

| 控制器方法返回值     | 描述     |
| ---- | ---- |
| `@ResponseBody`     | 返回值通过`HttpMessageConverter`实现转换并写入响应。 请参阅[`@ResponseBody`](#mvc-ann-responsebody)。     |
| `HttpEntity<B>`, `ResponseEntity<B>`     | 指定完整响应（包括HTTP头和主体）的返回值将通过 `HttpMessageConverter`实现转换并写入响应。 请参阅[ResponseEntity](#mvc-ann-responseentity)。|
| `HttpHeaders`     | 用于返回带头部信息且没有正文的响应。     |
| `String`     | 要使用`ViewResolver`实现解析的视图名称，并与隐式模型一起使用 \- 通过命令对象和 `@ModelAttribute`方法确定。 处理程序方法还可以通过声明 `Model`参数以编程方式丰富模型（请参阅 [显式注册](#mvc-ann-requestmapping-registration)）。     |
| `View`     | 用于与隐式模型一起呈现的`View`实例 \- 通过命令对象和`@ModelAttribute`方法确定。 处理程序方法还可以通过声明`Model`参数以编程方式丰富模型（请参阅 [显式注册](#mvc-ann-requestmapping-registration)）。     |
| `java.util.Map`, `org.springframework.ui.Model`     | 要添加到隐式模型的属性，通过`RequestToViewNameTranslator`隐式确定视图名称。     |
| `@ModelAttribute`     |要添加到模型的属性，通过`RequestToViewNameTranslator`隐式确定视图名称。请注意，`@ModelAttribute`是可选的。 请参阅本表末尾的“任何其他返回值”。      |
| `ModelAndView` object     |要使用的视图和模型属性，以及（可选）响应状态。      |
| `void`     |如果具有`void`返回类型（或返回值为 `null` ）的方法，如果它还具有`ServletResponse`，`OutputStream`参数或`@ResponseStatus`注解， 则认为已完全处理该响应。 如果控制器已进行正`ETag`或`lastModified` 时间戳检查，则也是如此（有关详细信息，请参阅[Controllers](#mvc-caching-etag-lastmodified)）。如果以上都不是真的，则`void`返回类型也可以指示REST控制器的“无响应主体”或HTML控制器的默认视图名称选择。      |
| `DeferredResult<V>`     |从任何线程异步生成任何前面的返回值 \- 例如，由于某些事件或回调。 请参阅[Asynchronous Requests](#mvc-ann-async) 和 [`DeferredResult`](#mvc-ann-async-deferredresult).      |
| `Callable<V>`     |在Spring MVC管理的线程中异步生成上述任何返回值。 请参阅 [Asynchronous Requests](#mvc-ann-async) 和 [`Callable`](#mvc-ann-async-callable).      |
|  `ListenableFuture<V>`, `java.util.concurrent.CompletionStage<V>`, `java.util.concurrent.CompletableFuture<V>`    | 作为替代`DeferredResult`的便捷操作（例如，当底层服务返回其中一个时）。     |
|  `ResponseBodyEmitter`, `SseEmitter`    |使用`HttpMessageConverter`实现以异步方式发送对象流以写入响应。 还支持`ResponseEntity`的主体。 请参阅[Asynchronous Requests](#mvc-ann-async) and [HTTP Streaming](#mvc-ann-async-http-streaming).      |
|  `StreamingResponseBody`    | 异步写入响应`OutputStream`。 还支持`ResponseEntity`的主体。 请参阅[Asynchronous Requests](#mvc-ann-async) and [HTTP Streaming](#mvc-ann-async-http-streaming).     |
|Reactive types — Reactor, RxJava, or others through `ReactiveAdapterRegistry`|使用multi-value流（例如，`Flux`, `Observable`）替代`DeferredResult`收集到`List`中。对于流式场景(例如, `text/event-stream`, `application/json+stream`), `SseEmitter` 和 `ResponseBodyEmitter` 使用的是在Spring MVC 管理的线程上执行`ServletOutputStream`阻塞I/O，这是 针对每一个Write的。请参阅 [Asynchronous Requests](#mvc-ann-async) 和 [Reactive Types](#mvc-ann-async-reactive-types).|
|Any other return value|任何与此表中任何早期值不匹配且返回值为`String` 或`void`的返回值都被视为视图名称（通过`RequestToViewNameTranslator`应用默认视图名称选择）， 前提是它不是简单类型，由 [BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)确定，简单类型的值仍未解决。|

<a id="mvc-ann-typeconversion"></a>

##### [](#mvc-ann-typeconversion)类型转换

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-typeconversion)

如果参数声明为String以外的其他参数，则表示某些带注解的控制器方法参数（例如`@RequestParam`, `@RequestHeader`, `@PathVariable`, `@MatrixVariable`, 和 `@CookieValue`）可能需要进行类型转换。

对于此类情况，将根据配置的转换器自动应用类型转换。 默认情况下，支持简单类型（`int`, `long`, `Date`和其他）。 您可以通过`WebDataBinder`（请参阅[`DataBinder`](#mvc-ann-initbinder)）或使用`FormattingConversionService`注册`Formatters`来自定义类型转换。 请参见 [Spring Field Formatting](core.html#format)。

<a id="mvc-ann-matrix-variables"></a>

##### [](#mvc-ann-matrix-variables)矩阵变量

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-matrix-variables)

[RFC 3986](https://tools.ietf.org/html/rfc3986#section-3.3)讨论了路径段中的携带键值对。 在Spring MVC中，我们将那些基于Tim Berners-Lee的 [“old post”](https://www.w3.org/DesignIssues/MatrixURIs.html) 称为“矩阵变量”，但它们也可以称为URI路径参数。

矩阵变量可以在任意路径段落中出现，每对矩阵变量之间使用分号隔开，多个值可以用逗号隔开（例如，`/cars;color=red,green;year=2012`）， 也可以通过重复的变量名称指定多个值（例如，`color=red;color=green;color=blue`）。

如果URL有可能会包含矩阵变量，那么在请求路径的映射配置上就需要使用URI模板来体现。这样才能确保请求可以被正确地映射，而不管矩阵变量在URI中是否出现、出现的次序是怎样的等。以下示例使用矩阵变量：

```java
// GET /pets/42;q=11;r=22

@GetMapping("/pets/{petId}")
public void findPet(@PathVariable String petId, @MatrixVariable int q) {

    // petId == 42
    // q == 11
}
```

由于任意路径段落中都可以含有矩阵变量，在某些场景下，开发者需要用更精确的信息来指定矩阵变量的位置。以下示例说明如何执行此操作：

```java
// GET /owners/42;q=11/pets/21;q=22

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable(name="q", pathVar="ownerId") int q1,
        @MatrixVariable(name="q", pathVar="petId") int q2) {

    // q1 == 11
    // q2 == 22
}
```

矩阵变量可以定义为可选，并指定默认值，如以下示例所示：

```java
// GET /pets/42

@GetMapping("/pets/{petId}")
public void findPet(@MatrixVariable(required=false, defaultValue="1") int q) {

    // q == 1
}
```

要获取所有矩阵变量，可以使用`MultiValueMap`，如以下示例所示:

```java
// GET /owners/42;q=11;r=12/pets/21;q=22;s=23

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable MultiValueMap<String, String> matrixVars,
        @MatrixVariable(pathVar="petId") MultiValueMap<String, String> petMatrixVars) {

    // matrixVars: ["q" : [11,22], "r" : 12, "s" : 23]
    // petMatrixVars: ["q" : 22, "s" : 23]
}
```

请注意，您需要启用矩阵变量的使用。 在MVC Java配置中，您需要通过 [路径匹配](#mvc-config-path-matching)将`removeSemicolonContent=false` 设置为`UrlPathHelper`。 在MVC XML命名空间中，您可以设置`<mvc:annotation-driven enable-matrix-variables="true"/>`。

<a id="mvc-ann-requestparam"></a>

##### [](#mvc-ann-requestparam)`@RequestParam`

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestparam)

您可以使用`@RequestParam`注解将Servlet请求参数（即查询参数或表单数据）绑定到控制器中的方法参数。

以下示例显示了如何执行此操作：

```java
@Controller
@RequestMapping("/pets")
public class EditPetForm {

    // ...

    @GetMapping
    public String setupForm(@RequestParam("petId") int petId, Model model) { (1)
        Pet pet = this.clinic.loadPet(petId);
        model.addAttribute("pet", pet);
        return "petForm";
    }

    // ...

}
```

**1**、使用 `@RequestParam`绑定`petId`。

若参数使用了该注解，则该参数默认是必须提供的.但您可以通过将`@RequestParam`注解的`required`属性设置为false或通过使用 `java.util.Optional`包装器声明参数来指定方法参数是可选的。

如果目标方法参数类型不是`String`，则会自动应用类型转换。 请参阅[类型转换](#mvc-ann-typeconversion)。

将参数类型声明为数组或列表允许为同一参数名称解析多个参数值。

当`@RequestParam`注解声明为`Map<String, String>` 或`MultiValueMap<String, String>`时， 如果注解中未指定参数名称，则会使用每个给定参数名称的请求参数值填充映射。

请注意，使用`@RequestParam`是可选的（例如，设置其属性）。 默认情况下， 任何属于简单值类型的参数（由[BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)确定）并且未被任何其他参数解析器解析，都被视为使用`@RequestParam`进行注解。

<a id="mvc-ann-requestheader"></a>

##### [](#mvc-ann-requestheader)`@RequestHeader`

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestheader)

您可以使用`@RequestHeader`注解将请求标头绑定到控制器中的方法参数。

考虑以下请求，请求头为:

Host                    localhost:8080
Accept                  text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Language         fr,en-gb;q=0.7,en;q=0.3
Accept-Encoding         gzip,deflate
Accept-Charset          ISO-8859-1,utf-8;q=0.7,*;q=0.7
Keep-Alive              300

以下示例获取`Accept-Encoding`和`Keep-Alive` 头的值：

```java
@GetMapping("/demo")
public void handle(
        @RequestHeader("Accept-Encoding") String encoding, (1)
        @RequestHeader("Keep-Alive") long keepAlive) { (2)
    //...
}
```

**1**、获取 `Accept-Encoding` 头部信息.

**2**、获取 `Keep-Alive` 头部信息.

如果目标方法参数类型不是`String`，则会自动应用类型转换。 请参阅[类型转换](#mvc-ann-typeconversion)。

在`Map<String, String>`，`MultiValueMap<String, String>`或`HttpHeaders`参数上使用 `@RequestHeader`注解时，将使用所有请求头值填充映射。

内置支持可用于将逗号分隔的字符串转换为字符串或字符串集或类型转换系统已知的其他类型。 例如，使用 `@RequestHeader("Accept")` 注解的方法参数可以是`String`类型，也可以是`String[]`或`List<String>`。

<a id="mvc-ann-cookievalue"></a>

##### [](#mvc-ann-cookievalue)`@CookieValue`

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-cookievalue)

您可以使用`@CookieValue`注解将HTTP cookie的值绑定到控制器中的方法参数。

考虑使用以下cookie的请求:

JSESSIONID=415A4AC178C59DACE0B2C9CA727CDD84

以下示例显示了如何获取cookie值：

```java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) { (1)
    //...
}
```

**1**、获取`JSESSIONID` cookie的值

如果目标方法参数类型不是`String`，则自动应用类型转换。 请参阅[类型转换](#mvc-ann-typeconversion)。

<a id="mvc-ann-modelattrib-method-args"></a>

##### [](#mvc-ann-modelattrib-method-args)`@ModelAttribute`

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-modelattrib-method-args)

您可以在方法参数上使用`@ModelAttribute`注解来从模型访问属性，或者如果不存在则将其实例化。 model属性还覆盖了名称与字段名称匹配的HTTP Servlet请求参数的值。 这称为数据绑定，它使您不必处理解析和转换单个查询参数和表单字段。 以下示例显示了如何执行此操作：

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) { } (1)
```

**1**、绑定一个`Pet`的实例。

上面的`Pet`实例解析如下：:

*   它可能来自已经添加的[Model](#mvc-ann-modelattrib-methods).

*   它可能因为[`@SessionAttributes`](#mvc-ann-sessionattributes)注解的使用已经存在在model中.

*   它可能是由URI模板变量和`转换`中取得的（下面会详细讲解）.

*   它可能是调用了自身的默认构造器被实例化出来的.

*   他可能从调用具有与Servlet请求参数匹配的参数的“primary constructor”。 参数名称通过JavaBeans`@ConstructorProperties`或字节码中的运行时保留参数名称确定。


虽然通常使用[Model](#mvc-ann-modelattrib-methods)来使用属性填充模型，但另一种替代方法是依赖于`Converter<String, T>`和URI路径变量。 在以下示例中，model属性名称`account`匹配URI路径变量 `account`，并通过将String字符串传递到已注册的`Converter<String, Account>`转换器来加载`Account` ：

```java
@PutMapping("/accounts/{account}")
public String save(@ModelAttribute("account") Account account) {
    // ...
}
```

下一步就是数据的绑定，`WebDataBinder`类能将请求参数，包括字符串的查询参数和表单字段等，通过名称匹配到model的属性上。成功匹配的字段在需要的时候会进行一次类型转换（从String类型到目标字段的类型），然后被填充到model对应的属性中， 有关数据绑定（和验证）的更多信息，请参阅[Validation](core.html#validation)。 有关自定义数据绑定的更多信息，请参阅[`DataBinder`](#mvc-ann-initbinder)。

数据绑定可能导致错误。 默认情况下，会引发`BindException` 。 但是，要在控制器方法中检查此类错误，可以在`@ModelAttribute`旁边添加一个`BindingResult`参数，如以下示例所示：

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute("pet") Pet pet, BindingResult result) { (1)
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```

**1**、在`@ModelAttribute`旁边添加 `BindingResult`。

在某些情况下，您可能希望在没有数据绑定的情况下访问model属性。对于这种情况，您可以将model注入控制器并直接访问它，或者设置`@ModelAttribute(binding=false)`，如下例所示：

```java
@ModelAttribute
public AccountForm setUpForm() {
    return new AccountForm();
}

@ModelAttribute
public Account findAccount(@PathVariable String accountId) {
    return accountRepository.findOne(accountId);
}

@PostMapping("update")
public String update(@Valid AccountUpdateForm form, BindingResult result,
        @ModelAttribute(binding=false) Account account) { (1)
    // ...
}
```

**1**、设置 `@ModelAttribute(binding=false)`.

通过添加`javax.validation.Valid`注解或Spring的`@Validated`注解（[Bean validation](core.html#validation-beanvalidation)和[Spring validation](core.html#validation)），您可以在数据绑定后自动应用验证。 以下示例显示了如何执行此操作：

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@Valid @ModelAttribute("pet") Pet pet, BindingResult result) { (1)
    if (result.hasErrors()) {
        return "petForm";
    }
    // ...
}
```

**1**、验证 `Pet` 实例.

请注意，使用`@ModelAttribute`是可选的（例如，设置其属性）。 默认情况下，任何非简单值类型的参数（由[BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)确定）并且未被任何其他参数解析器解析，都被视为使用`@ModelAttribute`进行注解。

<a id="mvc-ann-sessionattributes"></a>

##### [](#mvc-ann-sessionattributes)`@SessionAttributes`

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-sessionattributes)

`@SessionAttributes`用于在请求之间的HTTP Servlet会话中存储model属性。 它是一个类型级别的注解，用于声明特定控制器使用的会话属性。 这通常列出model属性的名称或model属性的类型，这些属性应该透明地存储在会话中以供后续访问请求使用。

以下示例使用`@SessionAttributes`注解:

```java
@Controller
@SessionAttributes("pet") (1)
public class EditPetForm {
    // ...
}
```

**1**、Using the `@SessionAttributes` annotation.

在第一个请求中，当名称为`pet`的model属性添加到模型中时，他会自动保存到HTTP Servlet会话中，并保持不变，直到另一个控制器方法使用`SessionStatus`方法参数来清除存储，如下例所示：

```java
@Controller
@SessionAttributes("pet") (1)
public class EditPetForm {

    // ...

    @PostMapping("/pets/{id}")
    public String handle(Pet pet, BindingResult errors, SessionStatus status) {
        if (errors.hasErrors) {
            // ...
        }
            status.setComplete(); (2)
            // ...
        }
    }
}
```

**1**、在Servlet会话中存储`Pet`值。

**2**、在Servlet会话中清除`Pet`值。

<a id="mvc-ann-sessionattribute"></a>

##### [](#mvc-ann-sessionattribute)`@SessionAttribute`

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-sessionattribute)

如果需要访问已存在的被全局session属性，例如在控制器之外（如通过过滤器）的（可有可无），请在方法参数上使用`@SessionAttribute`注解：

```java
@RequestMapping("/")
public String handle(@SessionAttribute User user) { (1)
    // ...
}
```

**1**、Using a `@SessionAttribute` annotation.

对于需要添加或删除会话属性的用例，请考虑将`org.springframework.web.context.request.WebRequest`或`javax.servlet.http.HttpSession`注入控制器方法。

作为控制器工作流的一部分，在会话中临时存储模型属性的方法可以使用`@SessionAttributes`，详情请参阅[`@SessionAttributes`](#mvc-ann-sessionattributes)。

<a id="mvc-ann-requestattrib"></a>

##### [](#mvc-ann-requestattrib)`@RequestAttribute`

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestattrib)

与`@SessionAttribute`类似，`@RequestAttribute`注解可用于访问由过滤器（`Filter`）或拦截器（`HandlerInterceptor`）创建的已存在的请求属性:

```java
@GetMapping("/")
public String handle(@RequestAttribute Client client) { (1)
    // ...
}
```

**1**、Using the `@RequestAttribute` annotation.

<a id="mvc-redirecting-passing-data"></a>

##### [](#mvc-redirecting-passing-data)Redirect 属性

默认情况下，所有模型属性都被视为在重定向URL中公开为URI模板变量。 在其余属性中，原始类型或集合或基本类型数组的属性将自动附加为查询参数。

如果专门为重定向准备了模型实例，期望的结果则是将原始类型属性作为查询参数。 但是，在带注解的控制器中，为了渲染目的，模型可以包含其他属性（例如，下拉字段值）。 为了避免在URL中出现此类属性的可能性，`@RequestMapping`方法可以声明`RedirectAttributes`类型的参数， 并使用它来指定可供`RedirectView`使用的确切属性。 如果方法重定向，则使用`RedirectAttributes`的内容。 否则，使用模型的内容。

`RequestMappingHandlerAdapter`提供了一个名为`ignoreDefaultModelOnRedirect`的标志，您可以使用该标志指示如果控制器方法重定向，则永远不应使用默认模型的内容。 相反，控制器方法应声明`RedirectAttributes`类型的属性，如果不这样做，则不应将任何属性传递给`RedirectView`。 MVC命名空间和MVC Java配置都将此标志设置为`false`，以保持向后兼容性。 但是，对于新应用程序，我们建议将其设置为`true`。

请注意，扩展重定向URL时，当前请求中的URI模板变量会自动可用，您需要通过 `Model`或`RedirectAttributes`显式添加它们。 以下示例显示如何定义重定向：

```java
@PostMapping("/files/{path}")
public String upload(...) {
    // ...
    return "redirect:files/{path}";
}
```

将数据传递到重定向目标的另一种方法是使用flash属性。 与其他重定向属性不同，Flash属性保存在HTTP会话中（因此，不会出现在URL中）。 有关更多信息，请参阅 [Flash 属性](#mvc-flash-attributes)。

<a id="mvc-flash-attributes"></a>

##### [](#mvc-flash-attributes)Flash 属性

Flash属性（flash attributes）提供了一个请求为另一个请求存储有用属性的方法。这在重定向的时候最常使用，比如常见的POST/REDIRECT/GET模式。 Flash属性会在重定向前被暂时地保存起来（通常是保存在session中），重定向后会重新被下一个请求取用并立即从原保存地移除。

为支持flash属性，Spring MVC提供了两个抽象。 `FlashMap`被用来存储flash属性，而用`FlashMapManager`来存储、取回、管理`FlashMap`的实例。

对flash属性的支持默认是启用“on” 的，并不需要显式声明，不过没用到它时它绝不会主动地去创建HTTP会话（session）。对于每个请求，框架都会“input” 一个`FlashMap`，里面存储了从上个请求（如果有）保存下来的属性；同时，每个请求也会“output”`FlashMap`，里面保存了要给下个请求使用的属性。 两个`FlashMap`实例在Spring MVC应用中的任何地点都可以通过`RequestContextUtils`工具类的静态方法取得。

控制器通常不需要直接接触`FlashMap`。一般是通过`@RequestMapping`方法去接受`RedirectAttributes`类型的参数，然后直接地往其中添加flash属性。 通过`RedirectAttributes`对象添加进去的flash属性会自动被填充到请求的“output”`FlashMap`对象中去。类似地，重定向后“input”的`FlashMap`属性也会自动被添加到服务重定向URL的控制器参数Model中去

匹配请求所使用的flash属性

flash属性的概念在其他许多的Web框架中也存在，并且实践证明有时可能会导致并发上的问题。这是因为从定义上讲，flash属性保存的时间是到下个请求接收到之前。 问题在于，“next”请求不一定刚好就是需要重定向到的那个请求，它有可能是其他的异步请求（比如polling请求或者资源请求等）。这会导致flash属性在到达真正的目标请求前就被移除了。

为了减少这个问题发生的可能性，重定向视图`RedirectView`会自动为一个`FlashMap`实例记录其目标重定向URL的路径和查询参数。然后，默认的`FlashMapManager`会在为请求查找其该“input”的`FlashMap`时，匹配这些信息。

这并不能完全解决重定向的并发问题，但极大程度地减少了这种可能性，因为它可以从重定向URL已有的信息中来做匹配。因此，一般只有在重定向的场景下，才推荐使用flash属性。

<a id="mvc-multipart-forms"></a>

##### [](#mvc-multipart-forms)Multipart

[Same as in Spring WebFlux](web-reactive.html#webflux-multipart-forms)

启用`MultipartResolver`后，将解析具有`multipart/form-data`的POST请求的内容，并将其作为常规请求参数进行访问。 以下示例访问一个常规表单字段和一个上载文件：

```java
@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(@RequestParam("name") String name,
            @RequestParam("file") MultipartFile file) {

        if (!file.isEmpty()) {
            byte[] bytes = file.getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
```

将参数类型声明为`List<MultipartFile>`允许为同一参数名称解析多个文件。

当`@RequestParam`注解声明为`Map<String, MultipartFile>`或`MultiValueMap<String, MultipartFile>`时，如果注解中未指定参数名称，则会使用每个给定参数名称的多部分文件填充map。

使用Servlet 3.0多部分解析，您也可以将 `javax.servlet.http.Part`而不是Spring的`MultipartFile`声明为方法参数或集合值类型。

您还可以将多部分内容用作绑定到[命令对象](#mvc-ann-modelattrib-method-args)的数据的一部分。 例如，前面示例中的表单字段和文件可以是表单对象上的字段，如以下示例所示：

```java
class MyForm {

    private String name;

    private MultipartFile file;

    // ...
}

@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(MyForm form, BindingResult errors) {
        if (!form.getFile().isEmpty()) {
            byte[] bytes = form.getFile().getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
```

还可以在RESTful服务方案中从非浏览器客户端提交多部分请求。 以下示例显示了带有JSON的文件：

```json
POST /someUrl
Content-Type: multipart/mixed

--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="meta-data"
Content-Type: application/json; charset=UTF-8
Content-Transfer-Encoding: 8bit

{
    "name": "value"
}
--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="file-data"; filename="file.properties"
Content-Type: text/xml
Content-Transfer-Encoding: 8bit
... File Data ...

```

对于名称为"meta-data" 的部分，可以通过控制器方法上的`@RequestParam`String metadata参数来获得。但对于那部分请求体中为JSON格式数据的请求， 可能更想通过接受一个对应的强类型对象，就像`@RequestBody`通过[HttpMessageConverter](integration.html#rest-message-conversion)将一般请求的请求体转换成一个对象一样。使用`@RequestPart`注解访问多部分：

```java
@PostMapping("/")
public String handle(@RequestPart("meta-data") MetaData metadata,
        @RequestPart("file-data") MultipartFile file) {
    // ...
}
```

您可以将`@RequestPart`与`javax.validation.Valid`结合使用，或使用Spring的`@Validated`注解，这两种注解都会导致应用标准Bean验证。 默认情况下，验证错误会导致`MethodArgumentNotValidException`， 并将其转换为400（BAD_REQUEST）响应。 或者，您可以通过`Errors` 或`BindingResult` 参数在控制器内本地处理验证错误，如以下示例所示:

```java
@PostMapping("/")
public String handle(@Valid @RequestPart("meta-data") MetaData metadata,
        BindingResult result) {
    // ...
}
```

<a id="mvc-ann-requestbody"></a>

##### [](#mvc-ann-requestbody)`@RequestBody`

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-requestbody)

您可以使用`@RequestBody`注解通过[`HttpMessageConverter`](integration.html#rest-message-conversion)将请求主体读取并反序列化为`Object`。 以下示例使用`@RequestBody`参数:

```java
@PostMapping("/accounts")
public void handle(@RequestBody Account account) {
    // ...
}
```

您可以使用[MVC Config](#mvc-config) 的[Message Converters](#mvc-config-message-converters)选项来配置或自定义消息转换。

您可以将`@RequestBody`与`javax.validation.Valid`或Spring的`@Validated` 注解结合使用，这两种注解都会导致应用标准Bean验证。 默认情况下，验证错误会导致 `MethodArgumentNotValidException`，并将其转换为400（BAD_REQUEST）响应。 或者，您可以通过`Errors`或`BindingResult`参数在控制器内本地处理验证错误，如以下示例所示：:

```java
@PostMapping("/accounts")
public void handle(@Valid @RequestBody Account account, BindingResult result) {
    // ...
}

```


<a id="mvc-ann-httpentity"></a>

##### [](#mvc-ann-httpentity)HttpEntity

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-httpentity)

`HttpEntity`与使用[`@RequestBody`](#mvc-ann-requestbody)或多或少有些类似，但它基于一个公开请求头和正文的容器对象。 以下清单显示了一个示例:

```java
@PostMapping("/accounts")
public void handle(HttpEntity<Account> entity) {
    // ...
}
```

<a id="mvc-ann-responsebody"></a>

##### [](#mvc-ann-responsebody)`@ResponseBody`

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-responsebody)

您可以在方法上使用`@ResponseBody`注解，以通过[HttpMessageConverter](integration.html#rest-message-conversion)将返回序列化到响应主体。 以下清单显示了一个示例:

```java
@GetMapping("/accounts/{id}")
@ResponseBody
public Account handle() {
    // ...
}
```

类级别也支持`@ResponseBody` ，在这种情况下，它由所有控制器方法继承。 例如`@RestController`的效果，它只不过是一个用`@Controller`和`@ResponseBody`标记的元注解。

您可以将`@ResponseBody`与reactive类型一起使用。 有关更多详细信息，请参阅[异步请求](#mvc-ann-async)和[Reactive 类型](#mvc-ann-async-reactive-types)。

您可以使用[MVC Config](#mvc-config)的 [Message Converters](#mvc-config-message-converters)选项来配置或自定义消息转换。

您可以将`@ResponseBody`方法与JSON序列化视图结合使用。 有关详细信息，请参阅[Jackson JSON](#mvc-ann-jackson)。

<a id="mvc-ann-responseentity"></a>

##### [](#mvc-ann-responseentity)ResponseEntity

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-responseentity)

`ResponseEntity`与 [`@ResponseBody`](#mvc-ann-responsebody) 类似，但具有状态和响应头。 例如:

```java
@GetMapping("/something")
public ResponseEntity<String> handle() {
    String body = ... ;
    String etag = ... ;
    return ResponseEntity.ok().eTag(etag).build(body);
}
```

Spring MVC支持使用单值[reactive type](#mvc-ann-async-reactive-types)异步生成 `ResponseEntity`，and/or 主体的单值和多值reactive类型。

<a id="mvc-ann-jackson"></a>

##### [](#mvc-ann-jackson)Jackson JSON

Spring为Jackson JSON库提供支持。

<a id="mvc-ann-jsonview"></a>

###### [](#mvc-ann-jsonview)Jackson 序列化视图

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-jsonview)

Spring MVC为[Jackson的序列化视图](http://wiki.fasterxml.com/JacksonJsonViews)提供内置支持，允许仅渲染Object中所有字段的子集。 为了与`@ResponseBody`控制器方法或者返回`ResponseEntity`的控制器方法一起使用，可以简单地将`@JsonView`注解放在参数上，指定需要使用的视图类或接口即可。如以下示例所示：

```java
@RestController
public class UserController {

    @GetMapping("/user")
    @JsonView(User.WithoutPasswordView.class)
    public User getUser() {
        return new User("eric", "7!jd#h23");
    }
}

public class User {

    public interface WithoutPasswordView {};
    public interface WithPasswordView extends WithoutPasswordView {};

    private String username;
    private String password;

    public User() {
    }

    public User(String username, String password) {
        this.username = username;
        this.password = password;
    }

    @JsonView(WithoutPasswordView.class)
    public String getUsername() {
        return this.username;
    }

    @JsonView(WithPasswordView.class)
    public String getPassword() {
        return this.password;
    }
}
```

`@JsonView`允许一组视图类，但每个控制器方法只能指定一个。 如果需要激活多个视图，可以使用复合接口。

对于依赖视图的控制器，只需将序列化视图类添加到model中即可。如以下示例所示：

```java
@Controller
public class UserController extends AbstractController {

    @GetMapping("/user")
    public String getUser(Model model) {
        model.addAttribute("user", new User("eric", "7!jd#h23"));
        model.addAttribute(JsonView.class.getName(), User.WithoutPasswordView.class);
        return "userView";
    }
}
```

<a id="mvc-ann-modelattrib-methods"></a>

#### [](#mvc-ann-modelattrib-methods)1.3.4. Model

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-modelattrib-methods)

您可以使用`@ModelAttribute` 注解：

*   在`@RequestMapping`方法中的[方法参数](#mvc-ann-modelattrib-method-args)方法参数，用于从model创建或访问Object并通过`WebDataBinder`将其绑定到请求。

*   作为 `@Controller`或`@ControllerAdvice` 类中的方法级注解，有助于在任何 `@RequestMapping`方法调用之前初始化模型。

*   在`@RequestMapping`方法上标记其返回值是一个模型属性。


本节讨论`@ModelAttribute`注解可被应用在方法或方法参数上 \- 前面列表中的第二项。控制器可以包含任意数量的 `@ModelAttribute`方法。 在同一控制器中的`@RequestMapping`方法之前调用所有这些方法。 `@ModelAttribute`方法也可以通过`@ControllerAdvice`在控制器之间共享。 有关更多详细信息，请参阅[控制器上的通知](#mvc-ann-controller-advice) 部分。

`@ModelAttribute`方法具有灵活的方法签名。 除了与`@ModelAttribute`本身或请求体相关的任何内容外，它们支持许多与`@RequestMapping`方法相同的参数。

以下示例显示了`@ModelAttribute` 方法：

```java
@ModelAttribute
public void populateModel(@RequestParam String number, Model model) {
    model.addAttribute(accountRepository.findAccount(number));
    // add more ...
}
```

以下示例仅添加一个属性:

```java
@ModelAttribute
public Account addAccount(@RequestParam String number) {
    return accountRepository.findAccount(number);
}
```

如果未明确指定名称，框架将根据属性的类型给予一个默认名称，如[`Conventions`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/core/Conventions.html)的javadoc中所述。 你可以通过设置`@ModelAttribute`注解的值来改变默认值。当向Model中直接添加属性时，请使用合适的重载方法`addAttribute`。

`@ModelAttribute` 注解也可以被用在`@RequestMapping`方法上，这种情况下，`@RequestMapping`方法的返回值将会被解释为model的一个属性，而非一个视图名。 此时视图名将以视图命名约定来方式来决议，与返回值为void的方法所采用的处理方法类似。`@ModelAttribute` 还可以自定义模型属性名称，如以下示例所示：

```java
@GetMapping("/accounts/{id}")
@ModelAttribute("myAccount")
public Account handle() {
    // ...
    return account;
}
```

<a id="mvc-ann-initbinder"></a>

#### [](#mvc-ann-initbinder)1.3.5. `数据绑定`

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-initbinder)

`@Controller`或`@ControllerAdvice`类可以使用`@InitBinder` 方法初始化`WebDataBinder`的实例，而这些方法又可以：

*   将请求参数（即表单或查询数据）绑定到模型对象。

*   将基于字符串的请求值（例如请求参数，路径变量，请求头，cookie等）转换为目标类型的控制器方法参数。

*   在呈现HTML表单时将模型对象值格式化为`String`值。


`@InitBinder`方法可以注册特定于控制器的`java.bean.PropertyEditor`或Spring `Converter`和`Formatter`组件。 此外，您可以使用[MVC config](#mvc-config-conversion)在全局共享的`FormattingConversionService`中注册`Converter`和`Formatter`类型。

`@InitBinder` 方法支持许多与`@RequestMapping`方法相同的参数，但`@ModelAttribute`（命令对象）参数除外。 通常，它们使用`WebDataBinder`参数（用于注册）和void返回值进行声明。 以下清单显示了一个示例:

```java
@Controller
public class FormController {

    @InitBinder (1)
    public void initBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
    }

    // ...
}
```

**1**、Defining an `@InitBinder` method.

或者，当使用基于Formatter的设置时，您可以通过共享的`FormattingConversionService`重复使用相同的方法并注册特定于控制器的 `Formatter` 实现，如以下示例所示:

```java
@Controller
public class FormController {

    @InitBinder (1)
    protected void initBinder(WebDataBinder binder) {
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd"));
    }

    // ...
}
```

**1**、Defining an `@InitBinder` method on a custom formatter.

<a id="mvc-ann-exceptionhandler"></a>

#### [](#mvc-ann-exceptionhandler)1.3.6. 异常

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-controller-exceptions)

`@Controller` 和 [@ControllerAdvice](#mvc-ann-controller-advice)可以使用`@ExceptionHandler`方法来处理来自控制器方法的异常，如下例所示：

```java
@Controller
public class SimpleController {

    // ...

    @ExceptionHandler
    public ResponseEntity<String> handle(IOException ex) {
        // ...
    }
}
```

该异常可能与顶级异常（即抛出直接`IOException`）或顶级包装器中的异常（例如，包含在 `IllegalStateException`内的`IOException`）相匹配。

对于匹配的异常类型，最好将目标异常声明为方法参数，如前面的示例所示。当多个异常方法匹配时，根(root)异常匹配通常优先于原因(cause )异常匹配。 更具体地说，`ExceptionDepthComparator` 用于根据抛出的异常类型的深度对异常进行排序。

注解声明可以缩小要匹配的异常类型，如以下示例所示：

```java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(IOException ex) {
    // ...
}
```

您甚至可以使用特定异常类型列表中的非常通用的参数签名，如以下示例所示：:

```java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(Exception ex) {
    // ...
}
```

根(root)和原因(cause )异常匹配之间的区别可能是令人惊讶的。

在前面显示的`IOException`变体中，通常使用实际的`FileSystemException`或`RemoteException`实例作为参数调用该方法，因为它们都是从`IOException`扩展的。 但是，如果任何此类异常在包装器内传播，而该异常本身就是`IOException`，则传入的异常实例就是包装器异常。

在`handle(Exception)`变体中，行为更简单。 这总是在包装场景中使用包装器异常调用，在这种情况下可以通过 `ex.getCause()`找到实际匹配的异常。 传入的异常仅在实际的`FileSystemException`或`RemoteException`实例被抛出为顶级异常时才会发生。

我们通常建议您在参数签名中尽可能具体，减少root和cause异常类型之间不匹配的可能性。 考虑将多匹配方法分解为单独的`@ExceptionHandler`方法，每个方法通过其签名匹配单个特定异常类型。

在具有多个`@ControllerAdvice`组成中，我们建议在`@ControllerAdvice`上声明根异常映射，并使用相应的顺序进行优先级排序。 虽然根异常匹配优先于某个原因，但这是在给定控制器或 `@ControllerAdvice`类的方法中定义的。 这意味着优先级较高的 `@ControllerAdvice` bean上的原因匹配优先于较低优先级的 `@ControllerAdvice` bean上的任何匹配（例如，root）。

最后但同样重要的是， 可以通过`@ExceptionHandler`方法的实现，讲异常以原始的形式重新抛出，并提供给特定的异常实例。 这在您仅对根级别匹配或在特定上下文中无法静态确定的匹配中感兴趣的情况下非常有用。 重新抛出的异常通过后续的解析链传播，就好像给定的`@ExceptionHandler`方法首先不匹配一样。

Spring MVC中对`@ExceptionHandler`方法的支持是基于`DispatcherServlet`级别的[HandlerExceptionResolver](#mvc-exceptionhandlers)机制构建的。

<a id="mvc-ann-exceptionhandler-args"></a>

##### [](#mvc-ann-exceptionhandler-args)方法参数

`@ExceptionHandler` 方法支持以下参数：



| 方法 参数     |描述      |
| ---- | ---- |
| Exception type     | 用于访问引发的异常。     |
| `HandlerMethod`      | 访问控制器方法引发的异常     |
| `WebRequest`, `NativeWebRequest`     | 无需直接使用Servlet API即可访问请求参数以及请求和会话属性。     |
| `javax.servlet.ServletRequest`, `javax.servlet.ServletResponse`     | 选择任何特定的请求或响应类型（例如，`ServletRequest` or `HttpServletRequest` or or Spring’s `MultipartRequest` or `MultipartHttpServletRequest`).     |
|`javax.servlet.http.HttpSession`      | 强制进行会话。 因此，这样的结果永远不会是`null`的。请注意，会话访问不是线程安全的。 如果允许多个请求同时访问会话，请考虑将`RequestMappingHandlerAdapter`实例的`synchronizeOnSession`标志设置为`true`。     |
| `java.security.Principal`     | 当前经过身份验证的用户 \- 如果已知，可能是特定的 `Principal`实现类。     |
| `HttpMethod`     | 请求的HTTP方法。     |
| `java.util.Locale`     |当前请求区域设置，由最可用的 `LocaleResolver`（实际上是已配置的 `LocaleResolver`或`LocaleContextResolver`）确定。      |
| `java.util.TimeZone`, `java.time.ZoneId`     | 与当前请求关联的时区，由`LocaleContextResolver`确定。     |
| `java.io.OutputStream`, `java.io.Writer`     | 用于访问Servlet API公开的原始响应主体。     |
| `java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap`     | 用于访问模型以获取错误响应。 总是为空.     |
|`RedirectAttributes`      |指定在重定向的情况下使用的属性 \- （将附加到查询字符串）和临时存储的flash属性，直到重定向后的请求为止。 请参阅[Redirect 属性](#mvc-redirecting-passing-data)和[Flash 属性](#mvc-flash-attributes)。      |
| `@SessionAttribute`     |用于访问任何会话属性，与由于类级别`@SessionAttributes`声明的结束形成对比。 有关更多详细信息，请参阅[`@SessionAttribute`](#mvc-ann-sessionattribute)。      |
| `@RequestAttribute`     | 用于访问请求属性。 有关更多详细信息，请参阅[`@RequestAttribute`](#mvc-ann-requestattrib)。     |


<a id="mvc-ann-exceptionhandler-return-values"></a>

##### [](#mvc-ann-exceptionhandler-return-values)返回值

`@ExceptionHandler`方法支持以下返回值:



| 返回值     | 描述     |
| ---- | ---- |
| `@ResponseBody`     |返回值通过`HttpMessageConverter`实现转换并写入响应。 请参阅[`@ResponseBody`](#mvc-ann-responsebody)。      |
| `HttpEntity<B>`, `ResponseEntity<B>`     | 指定完整响应（包括HTTP头和主体）的返回值将通过 `HttpMessageConverter`实现转换并写入响应。 请参阅[ResponseEntity](#mvc-ann-responseentity)。     |
| `String`     |要使用`ViewResolver`实现解析的视图名称，并与隐式模型一起使用 \- 通过命令对象和 `@ModelAttribute`方法确定。 处理程序方法还可以通过声明 `Model`参数以编程方式丰富模型（请参阅 [显式注册](#mvc-ann-requestmapping-registration)）。      |
| `View`     | 用于与隐式模型一起呈现的`View`实例 \- 通过命令对象和`@ModelAttribute`方法确定。 处理程序方法还可以通过声明`Model`参数以编程方式丰富模型（请参阅 [显式注册](#mvc-ann-requestmapping-registration)）。     |
|`java.util.Map`, `org.springframework.ui.Model`     |要添加到隐式模型的属性，通过`RequestToViewNameTranslator`隐式确定视图名称。      |
| `@ModelAttribute`     | 要添加到模型的属性，通过隐式确定视图名称。请注意，`@ModelAttribute`是可选的。 请参阅本表末尾的“任何其他返回值”。     |
| `ModelAndView` object     | 要使用的视图和模型属性，以及（可选）响应状态。     |
| `void`     | 如果具有`void`返回类型（或返回值为 `null` ）的方法，如果它还具有`ServletResponse`，`OutputStream`参数或`@ResponseStatus`注解， 则认为已完全处理该响应。 如果控制器已进行正`ETag`或`lastModified` 时间戳检查，则也是如此（有关详细信息，请参阅[Controllers](#mvc-caching-etag-lastmodified)）。 如果以上都不是真的，则`void`返回类型也可以指示REST控制器的“无响应主体”或HTML控制器的默认视图名称选择。    |
|  Any other return value    |任何与此表中任何早期值不匹配且返回值为`String` 或`void`的返回值都被视为视图名称（通过`RequestToViewNameTranslator`应用默认视图名称选择）， 前提是它不是简单类型，由 [BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)确定，简单类型的值仍未解决。      |

<a id="mvc-ann-rest-exceptions"></a>

##### [](#mvc-ann-rest-exceptions)REST API 异常

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-rest-exceptions)

REST服务的一个常见要求是在响应正文中包含错误详细信息。 Spring Framework不会自动执行此操作，因为响应正文中的错误详细信息的表示是特定于应用程序的。 但是，`@RestController`可以使用带有ResponseEntity返回值的`@ExceptionHandler`方法来设置响应的状态和正文。 这些方法也可以在`@ControllerAdvice`类中声明，以全局应用它们。

在响应主体中实现具有错误详细信息的全局异常处理的应用程序应考虑扩展[`ResponseEntityExceptionHandler`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/mvc/method/annotation/ResponseEntityExceptionHandler.html)， 它提供对Spring MVC引发的异常的处理，并提供钩子来定制响应主体。要使用它，请创建`ResponseEntityExceptionHandler`的子类，使用`@ControllerAdvice`注解它，覆盖必要的方法，并将其声明为Spring bean。

<a id="mvc-ann-controller-advice"></a>

#### [](#mvc-ann-controller-advice)1.3.7. 控制器通知

[Same as in Spring WebFlux](web-reactive.html#webflux-ann-controller-advice)

通常，在`@Controller`类上声明`@ExceptionHandler`, `@InitBinder`, 和 `@ModelAttribute`注解。 如果您希望此类方法更全局地应用（跨控制器），则可以在标有`@ControllerAdvice` 或 `@RestControllerAdvice`的类中声明它们。

`@ControllerAdvice``@Component`标记，这意味着可以通过[组件扫描](core.html#beans-java-instantiating-container-scan)将这些类注册为Spring bean。 `@RestControllerAdvice`也是一个用`@ControllerAdvice`和`@ResponseBody`标记的元注解，这实际上意味着 `@ExceptionHandler` 方法通过消息转换（与视图分辨率或模板渲染相对）呈现给响应主体。

在启动时， `@RequestMapping`和`@ExceptionHandler`方法的基础结构类检测 `@ControllerAdvice` 类型的Spring bean，然后在运行时应用它们的方法。 全局`@ExceptionHandler`方法（来自`@ControllerAdvice`）在本地方法之后（来自 `@Controller`）应用。 相比之下，全局`@ModelAttribute`和`@InitBinder`方法在本地方法之前应用。

默认情况下，`@ControllerAdvice`方法适用于每个请求（即所有控制器），但您可以通过使用注解上的属性将其缩小到控制器的子集，如以下示例所示：

```java
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
public class ExampleAdvice3 {}
```

前面示例中的选择器在运行时进行评估，如果广泛使用，可能会对性能产生负面影响。 有关更多详细信息，请参阅[`@ControllerAdvice`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/bind/annotation/ControllerAdvice.html) javadoc 。

<a id="mvc-uri-building"></a>

### [](#mvc-uri-building)1.4. URI 链接

[Same as in Spring WebFlux](web-reactive.html#webflux-uri-building)

本节介绍Spring Framework中可用于处理URI的各种选项。

<a id="web-uricomponents"></a>

#### [](#web-uricomponents)1.4.1. UriComponents

Spring MVC和Spring WebFlux

`UriComponentsBuilder` 有助于从URI模板变量构建URI。 如下例所示:

```java
UriComponents uriComponents = UriComponentsBuilder
        .fromUriString("http://example.com/hotels/{hotel}")  (1)
        .queryParam("q", "{q}")  (2)
        .encode() (3)
        .build(); (4)

URI uri = uriComponents.expand("Westin", "123").toUri();  (5)
```

**1**、带有URI模板的静态工厂方法。

**2**、添加或替换URI组件。

**3**、请求编码URI模板和URI变量。

**4**、构建一个`UriComponents`.

**5**、暴露变量并获取`URI`.

前面的示例可以合并到一个链中，并使用`buildAndExpand`缩短，如下例所示:

```java
URI uri = UriComponentsBuilder
        .fromUriString("http://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .encode()
        .buildAndExpand("Westin", "123")
        .toUri();
```

您可以通过直接转到URI（这意味着编码）来进一步缩短它，如下例所示：

```java
URI uri = UriComponentsBuilder
        .fromUriString("http://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123");
```

您使用完整的URI模板进一步缩短它，如下例所示：

```java
URI uri = UriComponentsBuilder
        .fromUriString("http://example.com/hotels/{hotel}?q={q}")
        .build("Westin", "123");
```

<a id="web-uribuilder"></a>

#### [](#web-uribuilder)1.4.2. UriBuilder

Spring MVC和Spring WebFlux

[`UriComponentsBuilder`](#web-uricomponents) 实现了 `UriBuilder`。 您可以使用`UriBuilderFactory`创建一个 `UriBuilder`。 `UriBuilderFactory`和`UriBuilder`一起提供了一种可插入机制，可以根据共享配置（例如基本URL，编码首选项和其他详细信息）从URI模板构建URI。

您可以使用UriBuilderFactory配置`RestTemplate`和`WebClient`。为自定义URI做准备。`DefaultUriBuilderFactory` 是`UriBuilderFactory` 的默认实现，它在内部使用`UriComponentsBuilder`并公开共享配置选项。

以下示例显示如何配置`RestTemplate`:

```java
// import org.springframework.web.util.DefaultUriBuilderFactory.EncodingMode;

String baseUrl = "http://example.org";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl);
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VARIABLES);

RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(factory);
```

以下示例配置`WebClient`:

```java
// import org.springframework.web.util.DefaultUriBuilderFactory.EncodingMode;

String baseUrl = "http://example.org";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl);
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VARIABLES);

WebClient client = WebClient.builder().uriBuilderFactory(factory).build();
```

此外，您还可以直接使用 `DefaultUriBuilderFactory` 。 它类似于使用`UriComponentsBuilder` ，但它不是静态工厂方法，而是一个保存配置和首选项的实际实例，如下例所示：

```java
String baseUrl = "http://example.com";
DefaultUriBuilderFactory uriBuilderFactory = new DefaultUriBuilderFactory(baseUrl);

URI uri = uriBuilderFactory.uriString("/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123");
```

<a id="web-uri-encoding"></a>

#### [](#web-uri-encoding)1.4.3. URI 编码

Spring MVC 和 Spring WebFlux

`UriComponentsBuilder`在两个级别公开编码选项：

*   [UriComponentsBuilder#encode()](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/util/UriComponentsBuilder.html#encode--): 首先对URI模板进行预编码，然后在扩展时严格编码URI变量。.

*   [UriComponents#encode()](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/util/UriComponents.html#encode--): 扩展URI变量后对URI组件进行编码。


这两个选项都使用转义的八位字节替换非ASCII和非法字符。 但是，第一个选项还会替换出现在URI变量中的保留含义的字符。

考虑";"，这在路径中是合法的但具有保留意义。第一个选项取代";"在URI变量中使用"%3B"但在URI模板中没有。 相比之下，第二个选项永远不会替换“;”，因为它是路径中的合法字符。

对于大多数情况，第一个选项可能会给出预期结果，因为它将URI变量视为完全编码的不透明数据，而选项2仅在URI变量故意包含保留字符时才有用。

以下示例使用第一个选项:

```java
URI uri = UriComponentsBuilder.fromPath("/hotel list/{city}")
            .queryParam("q", "{q}")
            .encode()
            .buildAndExpand("New York", "foo+bar")
            .toUri();

    // Result is "/hotel%20list/New%20York?q=foo%2Bbar"
```

您可以通过直接转到URI（这意味着编码）来缩短前面的示例，如以下示例所示：

```java
URI uri = UriComponentsBuilder.fromPath("/hotel list/{city}")
            .queryParam("q", "{q}")
            .build("New York", "foo+bar")
```

您可以使用完整的URI模板进一步缩短它，如以下示例所示：

```java
URI uri = UriComponentsBuilder.fromPath("/hotel list/{city}?q={q}")
            .build("New York", "foo+bar")
```

`WebClient`和`RestTemplate`通过 `UriBuilderFactory`策略在内部展开和编码URI模板。 两者都可以配置自定义策略。 如下例所示：

```java
String baseUrl = "http://example.com";
DefaultUriBuilderFactory factory = new DefaultUriBuilderFactory(baseUrl)
factory.setEncodingMode(EncodingMode.TEMPLATE_AND_VALUES);

// Customize the RestTemplate..
RestTemplate restTemplate = new RestTemplate();
restTemplate.setUriTemplateHandler(factory);

// Customize the WebClient..
WebClient client = WebClient.builder().uriBuilderFactory(factory).build();
```

`DefaultUriBuilderFactory`实现在内部使用`UriComponentsBuilder`来展开和编码URI模板。 作为工厂，它提供了一个单独的位置来配置编码方法，基于以下编码模式之一：

*   `TEMPLATE_AND_VALUES`: 使用`UriComponentsBuilder#encode()`（对应于前面列表中的第一个选项）对URI模板进行预编码，并在展开时严格编码URI变量。

*   `VALUES_ONLY`: 不对URI模板进行编码，而是在将URI变量展开到模板之前，通过`UriUtils#encodeUriUriVariables`对URI变量严格编码。

*   `URI_COMPONENTS`: 使用`UriComponents#encode()`（对应于前面列表中的第二个选项），在URI变量展开后对URI组件值进行编码。

*   `NONE`: 没有应用编码。


出于历史原因和向后兼容性，`RestTemplate`设置为 `EncodingMode.URI_COMPONENTS`。`WebClient` 依赖于`DefaultUriBuilderFactory`中的默认值， 该值已从5.0.x中的 `EncodingMode.URI_COMPONENTS`更改为5.1中的 `EncodingMode.TEMPLATE_AND_VALUES`。

<a id="mvc-servleturicomponentsbuilder"></a>

#### [](#mvc-servleturicomponentsbuilder)1.4.4. 相对请求

您可以使用`ServletUriComponentsBuilder`创建相对于当前请求的URI，如以下示例所示：

```java
HttpServletRequest request = ...

// Re-uses host, scheme, port, path and query string...

ServletUriComponentsBuilder ucb = ServletUriComponentsBuilder.fromRequest(request)
        .replaceQueryParam("accountId", "{id}").build()
        .expand("123")
        .encode();
```

您可以创建相对于上下文路径的URI，如以下示例所示:

```java
// Re-uses host, port and context path...

ServletUriComponentsBuilder ucb = ServletUriComponentsBuilder.fromContextPath(request)
        .path("/accounts").build()
```

您可以创建相对于Servlet的URI（例如，`/main/*`），如以下示例所示:

```java
// Re-uses host, port, context path, and Servlet prefix...

ServletUriComponentsBuilder ucb = ServletUriComponentsBuilder.fromServletMapping(request)
        .path("/accounts").build()
```

从5.1开始，`ServletUriComponentsBuilder`忽略来自`Forwarded`和`X-Forwarded-*`头部的信息，这些头部信息指定了客户端发起的地址。 考虑使用[`ForwardedHeaderFilter`](#filters-forwarded-headers)来提取和使用或丢弃此类请求头。

<a id="mvc-links-to-controllers"></a>

#### [](#mvc-links-to-controllers)1.4.5. 控制器链接

Spring MVC也提供了构造指定控制器方法链接的机制。 例如，以下MVC控制器允许创建链接：

```java
@Controller
@RequestMapping("/hotels/{hotel}")
public class BookingController {

    @GetMapping("/bookings/{booking}")
    public ModelAndView getBooking(@PathVariable Long booking) {
        // ...
    }
}
```

您可以通过引用方法名字的办法来准备一个链接，如以下示例所示：

```java
UriComponents uriComponents = MvcUriComponentsBuilder
    .fromMethodName(BookingController.class, "getBooking", 21).buildAndExpand(42);

URI uri = uriComponents.encode().toUri();
```

在前面的示例中，为方法参数准备了填充值（在本例中，long值： `21`），以用于填充路径变量并插入到URL中。此外，我们提供了值42，来填充任何剩余的URI变量，比如从类层级的请求映射中继承来的hotel变量。 如果方法还有更多的参数，你可以为那些不需要参与URL构造的变量赋予null值。 通常，只有`@PathVariable`和`@RequestParam`参数与构造URL相关。

还有其他方法可以使用`MvcUriComponentsBuilder`。 例如，例如可以通过类似mock掉测试对象的方法，用代理来避免直接通过名字引用一个控制，如以下示例所示（该示例假设静态导入`MvcUriComponentsBuilder.on`）：

```java
UriComponents uriComponents = MvcUriComponentsBuilder
    .fromMethodCall(on(BookingController.class).getBooking(21)).buildAndExpand(42);

URI uri = uriComponents.encode().toUri();
```

Controller method signatures are limited in their design when they are supposed to be usable for link creation with `fromMethodCall`. Aside from needing a proper parameter signature, there is a technical limitation on the return type (namely, generating a runtime proxy for link builder invocations), so the return type must not be `final`. In particular, the common `String` return type for view names does not work here. You should use `ModelAndView` or even plain `Object` (with a `String` return value) instead.

上面的代码例子中使用了`MvcUriComponentsBuilder`类的静态方法，内部实现中，它依赖于`ServletUriComponentsBuilder`来从当前请求中抽取schema、主机名、端口号、context路径和Servlet路径， 并准备一个基本URL。大多数情况下它能良好工作，但有时还不行，比如，在准备链接时你可能在当前请求的上下文（context）之外（比如，执行一个准备链接links的批处理），或你可能需要为路径插入一个前缀（比如一个地区性前缀，它从请求中被移除，然后又重新被插入到链接中去）。

对于上面所提的场景，开发者可以使用重载过的静态方法`fromXxx`，它接收一个`UriComponentsBuilder`参数，然后从中获取基本URL以便使用，或你也可以使用一个基本URL创建一个`MvcUriComponentsBuilder`对象， 然后使用实例对象的`withXxx`方法。以下列表使用 `withMethodCall`:

```java
UriComponentsBuilder base = ServletUriComponentsBuilder.fromCurrentContextPath().path("/en");
MvcUriComponentsBuilder builder = MvcUriComponentsBuilder.relativeTo(base);
builder.withMethodCall(on(BookingController.class).getBooking(21)).buildAndExpand(42);

URI uri = uriComponents.encode().toUri();
```

从5.1开始，`MvcUriComponentsBuilder`忽略来自`Forwarded` 和`X-Forwarded-*`头的信息，这些头指定了客户端发起的地址。 考虑使用ForwardedHeaderFilter来提取和使用或丢弃此类标头。

<a id="mvc-links-to-controllers-from-views"></a>

#### [](#mvc-links-to-controllers-from-views)1.4.6. 视图链接

在Thymeleaf，FreeMarker或JSP等视图中，您可以通过引用每个请求映射的隐式或显式指定名称来构建指向带注解控制器的链接。

请考虑以下示例：

```java
@RequestMapping("/people/{id}/addresses")
public class PersonAddressController {

    @RequestMapping("/{country}")
    public HttpEntity getAddress(@PathVariable String country) { ... }
}
```

给定前面的控制器，可以按照以下方式准备来自JSP的链接，如下所示：

```jsp
<%@ taglib uri="http://www.springframework.org/tags" prefix="s" %>
...
<a href="${s:mvcUrl('PAC#getAddress').arg(0,'US').buildAndExpand('123')}">Get Address</a>
```

前面的示例依赖于Spring标签库中声明的`mvcUrl`函数（即 META-INF/spring.tld），但可以很容易地定义自定义函数或使用自定义标记文件。

这是如何工作的，在启动时，每个`@RequestMapping` 都通过`HandlerMethodMappingNamingStrategy`分配一个默认名称，其默认实现使用类的大写字母和方法名称（例如， `ThingController`中的`getThing`方法变为"TC#getThing"）。如果存在名称冲突，则可以使用`@RequestMapping(name="..")`分配显式名称或实现自己的`HandlerMethodMappingNamingStrategy`。

<a id="mvc-ann-async"></a>

### [](#mvc-ann-async)1.5. 异步请求

[Compared to WebFlux](#mvc-ann-async-vs-webflux)

Spring MVC与Servlet 3.0异步请求[处理](#mvc-ann-async-processing)有广泛的集成：

*   在控制器方法中返回[`DeferredResult`](#mvc-ann-async-deferredresult)和[`Callable`](#mvc-ann-async-callable)，并为单个异步返回值提供基本支持。

*   控制器可以[stream](#mvc-ann-async-http-streaming)多个值，包括[SSE](#mvc-ann-async-sse) 和[raw data](#mvc-ann-async-output-stream)。

*   控制器可以使用reactive clients 并返回[reactive types](#mvc-ann-async-reactive-types)以进行响应处理。


<a id="mvc-ann-async-deferredresult"></a>

#### [](#mvc-ann-async-deferredresult)1.5.1. `DeferredResult`

[Compared to WebFlux](#mvc-ann-async-vs-webflux)

一旦在Servlet容器中[启用](#mvc-ann-async-configuration) 了异步请求处理功能，控制器方法就可以使用 `DeferredResult`包装任何支持的控制器方法返回值，如以下示例所示：

```java
@GetMapping("/quotes")
@ResponseBody
public DeferredResult<String> quotes() {
    DeferredResult<String> deferredResult = new DeferredResult<String>();
    // Save the deferredResult somewhere..
    return deferredResult;
}

// From some other thread...
deferredResult.setResult(data);
```

控制器可以从不同的线程异步生成返回值 \- 例如，响应外部事件（JMS消息），计划任务或其他事件。

<a id="mvc-ann-async-callable"></a>

#### [](#mvc-ann-async-callable)1.5.2. `Callable`

[Compared to WebFlux](#mvc-ann-async-vs-webflux)

控制器可以使用 `java.util.concurrent.Callable`包装任何支持的返回值，如以下示例所示：

```java
@PostMapping
public Callable<String> processUpload(final MultipartFile file) {

    return new Callable<String>() {
        public String call() throws Exception {
            // ...
            return "someView";
        }
    };

}
```

然后可以通过配置的[configured](#mvc-ann-async-configuration-spring-mvc) `TaskExecutor`运行给定任务来获取返回值。

<a id="mvc-ann-async-processing"></a>

#### [](#mvc-ann-async-processing)1.5.3. 处理过程

[Compared to WebFlux](#mvc-ann-async-vs-webflux)

以下是Servlet异步请求处理的简要概述:

*   Servlet请求`ServletRequest`可以通过调用 `request.startAsync()`方法而进入异步模式。这样做的主要结果就是该Servlet以及所有的过滤器都可以结束，但其响应（response）会留待异步处理结束后再返回调用。

*   `request.startAsync()` 方法会返回一个`AsyncContext`对象 ，可用它对异步处理进行进一步的控制和操作。比如说它也提供了一个与转向（forward）很相似的`dispatch`方法，只不过它允许应用恢复Servlet容器的请求处理进程。

*   `ServletRequest` 提供了获取当前`DispatcherType`的方式，后者可以用来区别当前处理的是原始请求、异步分发请求、转向、或是其他类型的请求分发类型。

`DeferredResult` 处理的工作方式如下：

*   控制器先返回一个`DeferredResult`对象，并把它存取在内存（队列或列表等）中以便存取。

*   Spring MVC调用`request.startAsync()`方法，开始进行异步处理。

*   `DispatcherServlet`和所有过滤器都退出Servlet容器线程，但此时方法的响应对象仍未返回。

*   由处理该请求的线程对`DeferredResult`进行设值，然后SpringM VC会重新把请求分派回Servlet容器，恢复处理。

*   `DispatcherServlet`再次被调用, 恢复对该异步返回结果的处理。


`Callable`处理的工作方式如下：

*   控制器先返回一个`Callable`对象.

*   Spring MVC调用 `request.startAsync()`方法，开始进行异步处理，并把该`Callable`对象提交给另一个独立线程的执行器 `TaskExecutor`处理。

*   `DispatcherServlet`和所有过滤器都退出Servlet容器线程，但此时方法的响应对象仍未返回。

*   `Callable`对象最终产生一个返回结果，此时Spring MVC会重新把请求分派回Servlet容器，恢复处理。

*   `DispatcherServlet`再次被调用,恢复对`Callable`异步处理所返回结果的处理。


关于引入异步请求处理的背景和原因，以及什么时候使用它，为什么使用异步请求处理等问题。可以阅读这个系列的[博客文章](https://spring.io/blog/2012/05/07/spring-mvc-3-2-preview-introducing-servlet-3-async-support) 。

<a id="mvc-ann-async-exceptions"></a>

##### [](#mvc-ann-async-exceptions)异常处理

若方法返回的是一个`DeferredResult`对象，你可以选择调Exception实例的`setResult`方法还是`setErrorResult`方法。在这两种情况下，Spring MVC都会将请求发送回Servlet容器以完成处理。 然后将其视为控制器方法返回给定值或者就好像它产生了给定的异常一样。 然后异常通过常规异常处理机制（例如，调用`@ExceptionHandler`方法）。更具体地说呢，当Callable抛出异常时，Spring MVC会把一个Exception对象分派给Servlet容器进行处理，而不是正常返回方法的返回值，然后容器恢复对此异步请求异常的处理。

当您使用`Callable`时，会出现类似的处理逻辑，主要区别在于从`Callable`返回结果，或者由它引发异常。

<a id="mvc-ann-async-interception"></a>

##### [](#mvc-ann-async-interception)拦截异步请求

处理器拦截器`HandlerInterceptor`可以实现`AsyncHandlerInterceptor`接口拦截异步请求，因为在异步请求开始时，被调用的回调方法是该接口的`afterConcurrentHandlingStarted`方法，而非一般的`postHandle`和`afterCompletion`方法。

如果需要与异步请求处理的生命流程有更深入的集成，比如需要处理timeout的事件等。则`HandlerInterceptor`需要注册`CallableProcessingInterceptor`或`DeferredResultProcessingInterceptor`拦截器， 具体的细节可以参考 [`AsyncHandlerInterceptor`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/AsyncHandlerInterceptor.html)类的Java文档

`DeferredResult`类还提供了`onTimeout(Runnable)` 和 `onCompletion(Runnable)` 等回调， 具体的细节可以参考[`DeferredResult`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/context/request/async/DeferredResult.html)类的Java文档 `Callable`可以替代`WebAsyncTask`，它公开了超时和完成回调的其他方法。

<a id="mvc-ann-async-vs-webflux"></a>

##### [](#mvc-ann-async-vs-webflux)Compared to WebFlux

The Servlet API was originally built for making a single pass through the Filter-Servlet chain. Asynchronous request processing, added in Servlet 3.0, lets applications exit the Filter-Servlet chain but leave the response open for further processing. The Spring MVC asynchronous support is built around that mechanism. When a controller returns a `DeferredResult`, the Filter-Servlet chain is exited, and the Servlet container thread is released. Later, when the `DeferredResult` is set, an `ASYNC` dispatch (to the same URL) is made, during which the controller is mapped again but, rather than invoking it, the `DeferredResult` value is used (as if the controller returned it) to resume processing.

By contrast, Spring WebFlux is neither built on the Servlet API, nor does it need such an asynchronous request processing feature, because it is asynchronous by design. Asynchronous handling is built into all framework contracts and is intrinsically supported through all stages of request processing.

From a programming model perspective, both Spring MVC and Spring WebFlux support asynchronous and [Reactive Types](#mvc-ann-async-reactive-types) as return values in controller methods. Spring MVC even supports streaming, including reactive back pressure. However, individual writes to the response remain blocking (and are performed on a separate thread), unlike WebFlux, which relies on non-blocking I/O and does not need an extra thread for each write.

Another fundamental difference is that Spring MVC does not support asynchronous or reactive types in controller method arguments (for example, `@RequestBody`, `@RequestPart`, and others), nor does it have any explicit support for asynchronous and reactive types as model attributes. Spring WebFlux does support all that.

<a id="mvc-ann-async-http-streaming"></a>

#### [](#mvc-ann-async-http-streaming)1.5.4. HTTP 流

[Same as in Spring WebFlux](web-reactive.html#webflux-codecs-streaming)

您可以将`DeferredResult`和`Callable`用于单个异步返回值。 如果要生成多个异步值并将其写入响应，该怎么办？ 本节介绍如何执行此操作。

<a id="mvc-ann-async-objects"></a>

##### [](#mvc-ann-async-objects)Objects

您可以使用`ResponseBodyEmitter`返回值来生成对象流，其中每个对象都使用 [`HttpMessageConverter`](integration.html#rest-message-conversion)进行序列化并写入响应，如以下示例所示：

```java
@GetMapping("/events")
public ResponseBodyEmitter handle() {
    ResponseBodyEmitter emitter = new ResponseBodyEmitter();
    // Save the emitter somewhere..
    return emitter;
}

// In some other thread
emitter.send("Hello once");

// and again later on
emitter.send("Hello again");

// and done at some point
emitter.complete();
```

`ResponseBodyEmitter`也可以被放到`ResponseEntity`体里面使用，这可以对响应状态和响应头做一些定制。

当`emitter` 抛出`IOException`时（例如，如果远程客户端消失），应用程序不负责清理连接，不应调用`emitter.complete`或`emitter.completeWithError`。 相反，servlet容器会自动启动`AsyncListener` 错误通知，其中Spring MVC进行 `completeWithError`调用。 反过来，此调用会对应用程序执行一次最终`ASYNC` 调度，在此期间，Spring MVC将调用已配置的异常解析程序并完成请求。

<a id="mvc-ann-async-sse"></a>

##### [](#mvc-ann-async-sse)SSE

`SseEmitter` （`ResponseBodyEmitter`的子类）为[Server-Sent Events](https://www.w3.org/TR/eventsource/),提供支持，其中从服务器发送的事件根据W3C SSE规范进行格式化。 要从控制器生成SSE流，请返回`SseEmitter`，如以下示例所示：

```java
@GetMapping(path="/events", produces=MediaType.TEXT_EVENT_STREAM_VALUE)
public SseEmitter handle() {
    SseEmitter emitter = new SseEmitter();
    // Save the emitter somewhere..
    return emitter;
}

// In some other thread
emitter.send("Hello once");

// and again later on
emitter.send("Hello again");

// and done at some point
emitter.complete();
```

虽然SSE是流式传输到浏览器的主要选项，但请注意Internet Explorer不支持Server-Sent Events。 考虑将Spring的[WebSocket messaging](#websocket)传递与针对各种浏览器的 [SockJS fallback](#websocket-fallback) 传输（包括SSE）一起使用。

有关异常处理的说明，另请参见[上一节](#mvc-ann-async-objects) 。

<a id="mvc-ann-async-output-stream"></a>

##### [](#mvc-ann-async-output-stream)Raw Data

有时，跳过消息转换的阶段，直接把数据写回响应的输出流`OutputStream`可能更有效，比如文件下载这样的场景，这可以通过返回一个`StreamingResponseBody`类型的对象来实现，如以下示例所示：

```java
@GetMapping("/download")
public StreamingResponseBody handle() {
    return new StreamingResponseBody() {
        @Override
        public void writeTo(OutputStream outputStream) throws IOException {
            // write...
        }
    };
}
```

`StreamingResponseBody`也可以被放到`ResponseEntity` 体里面使用，这可以对响应状态和响应头做一些定制。

<a id="mvc-ann-async-reactive-types"></a>

#### [](#mvc-ann-async-reactive-types)1.5.5. Reactive Types（响应式类型）

[Same as in Spring WebFlux](web-reactive.html#webflux-codecs-streaming)

如果使用`spring-webflux`中的响应式`WebClient`，或其他客户端（也可以阅读WebFlux部分中的[Reactive Libraries](web-reactive.html#webflux-reactive-libraries)），又或者是带响应式支持的数据存储，开发者可以直接从Spring MVC控制器方法返回响应式类型。 p>

Reactive 返回值的处理方式如下:

*   如果返回类型有single-value流的语义，如Reactor`Mono`或RxJava `Single`，那么它是适配并等效于 `DeferredResult`。

*   如果返回类型有multi-value流的语义，如Reactor `Flux`或RxJava `Observable`，并且如果媒体类型也表示为流，（例如，`application/stream+json`或`text/event-stream`）。 则它是适配并等效于使用`ResponseBodyEmitter`或 `SseEmitter`。还可以返回`Flux<ServerSentEvent>`或 `Observable<ServerSentEvent>`。

*   如果返回类型multi-value流的语义，但媒体类型并不表示为流。例如`application/json`，则它是适配并等效于使用`DeferredResult<List<?>>`。


Spring MVC对使用中的响应式库进行了适配 – 例如，预计有多少值，这是在`spring-core`包的[`ReactiveAdapterRegistry`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/core/ReactiveAdapterRegistry.html) 的帮助下完成的。 它为响应式和异步类型提供可插拔的支持。注册表内置了对RxJava的支持，但其他可以注册。

对于流式传输到响应，支持响应式响应，但是对响应的写入仍然是阻塞的，并且通过[配置](#mvc-ann-async-configuration-spring-mvc) 的`TaskExecutor`在单独的线程上执行， 以避免阻塞上游源（例如从`WebClient`返回的`Flux`）。 默认情况下，`SimpleAsyncTaskExecutor`用于阻塞写入，但在加载时不适用。 如果计划使用响应类型进行流式处理，则应使用MVC配置来配置任务执行程序。

<a id="mvc-ann-async-disconnects"></a>

#### [](#mvc-ann-async-disconnects)1.5.6. 断开

[Same as in Spring WebFlux](web-reactive.html#webflux-codecs-streaming)

当远程客户端消失时，Servlet API不提供任何通知。 因此，在通过stream传输到响应时，无论是通过[SseEmitter](#mvc-ann-async-sse)还是<<mvc-ann-async-reactive-types,reactive types>，定期发送数据都很重要， 因为如果客户端断开连接，写入将失败。 发送可以采用空（仅限注解）SSE事件或另一方必须解释为心跳并忽略的任何其他数据的形式。

或者，考虑使用具有内置心跳机制的Web消息传递解决方案（例如基于WebSocket的[STOMP](#websocket-stomp)或具有[SockJS](#websocket-fallback)的WebSocket）。

<a id="mvc-ann-async-configuration"></a>

#### [](#mvc-ann-async-configuration)1.5.7. 配置

[Compared to WebFlux](#mvc-ann-async-vs-webflux)

必须在Servlet容器级别启用异步请求处理功能。 MVC配置还公开了异步请求的几个选项。

<a id="mvc-ann-async-configuration-servlet3"></a>

##### [](#mvc-ann-async-configuration-servlet3)Servlet 容器

Filter和Servlet声明具有`asyncSupported`标志，需要将其设置为`true`以启用异步请求处理。 此外，应声明Filter映射以处理`ASYNC` `javax.servlet.DispatchType`。

在Java配置中，当您使用`AbstractAnnotationConfigDispatcherServletInitializer`初始化Servlet容器时，这将自动完成。

在`web.xml` 配置中，您可以将`<async-supported>true</async-supported>` 添加到`DispatcherServlet`和`Filter`声明，并添加`<dispatcher>ASYNC</dispatcher>`以过滤映射。

<a id="mvc-ann-async-configuration-spring-mvc"></a>

##### [](#mvc-ann-async-configuration-spring-mvc)Spring MVC

MVC配置公开以下与异步请求处理相关的选项：

*   Java configuration:在`WebMvcConfigurer`上使用`configureAsyncSupport`回调。

*   XML namespace:使用 `<mvc:annotation-driven>`下的`<async-support>`元素。


您可以配置以下内容：

*   异步请求的默认超时值（如果未设置）取决于底层Servlet容器（例如，Tomcat上的10秒）。

*   `AsyncTaskExecutor`用于在使用[Reactive Types](#mvc-ann-async-reactive-types)进行流式处理时阻止写入，以及用于执行从控制器方法返回的`Callable`实例。 如果您使用reactive types进行流式传输或者具有返回`Callable`的控制器方法， 我们强烈建议您配置此属性，因为默认情况下，它是`SimpleAsyncTaskExecutor`。

*   `DeferredResultProcessingInterceptor`实现和`CallableProcessingInterceptor`实现。


请注意，您还可以在`DeferredResult`， `ResponseBodyEmitter`和`SseEmitter`上设置默认超时值。 对于`Callable`，您可以使用`WebAsyncTask` 来提供超时值。

<a id="mvc-cors"></a>

### [](#mvc-cors)1.6. CORS

[Same as in Spring WebFlux](web-reactive.html#webflux-cors)

Spring MVC允许您处理CORS（跨源资源共享）。 本节介绍如何执行此操作。

<a id="mvc-cors-intro"></a>

#### [](#mvc-cors-intro)1.6.1. 简介

[Same as in Spring WebFlux](web-reactive.html#webflux-cors-intro)

出于安全原因，浏览器禁止对当前源外的资源进行AJAX调用。 例如，您可以将您的银行帐户放在一个标签页中，将evil.com放在另一个标签页中。 来自evil.com的脚本不应该使用您的凭据向您的银行API发出AJAX请求 - 例如从您的帐户中提取资金！

Cross-Origin Resource Sharing (CORS) 是[大多数浏览器](https://caniuse.com/#feat=cors) 实现的[W3C规范](https://www.w3.org/TR/cors/)，它允许以灵活的方式指定哪些类型的跨域请求被授权， 而不是使用一些安全程度较低、功能较差的实现(如IFRAME或JSONP)。

<a id="mvc-cors-processing"></a>

#### [](#mvc-cors-processing)1.6.2. 处理

[Same as in Spring WebFlux](web-reactive.html#webflux-cors-processing)

CORS规范区分了预检查，简单和实际请求。 要了解CORS的工作原理，您可以[>阅读本文](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)以及其他许多内容，或者查看规范以获取更多详细信息。

Spring MVC `HandlerMapping`为实现CORS提供内置支持。成功将请求映射到处理程序后，`HandlerMapping` 实现检查给定请求和处理程序的CORS配置并采取进一步操作。 直接处理预检查请求，同时拦截，验证简单和实际的CORS请求，并设置所需的CORS响应头。

为了启用跨源请求（即，存在`Origin`头并且与请求的主机不同），您需要具有一些显式声明的CORS配置。 如果未找到匹配的CORS配置，则拒绝预检请求。 没有CORS头添加到简单和实际CORS请求的响应中，因此浏览器拒绝它们。

可以使用基于URL模式的`CorsConfiguration`映射单独 [配置](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/handler/AbstractHandlerMapping.html#setCorsConfigurations-java.util.Map-)每个`HandlerMapping`。 在大多数情况下，应用程序使用MVC Java配置或XML命名空间来声明此类映射，这会导致将单个全局映射传递给所有`HandlerMappping` 实例。

您可以将`HandlerMapping`级别的全局CORS配置与更细粒度的处理程序级CORS配置相结合。 例如，带注解的控制器可以使用类或方法级别的`@CrossOrigin`注解（其他处理程序可以实现`CorsConfigurationSource`）。

组合全局和本地配置的规则通常是附加的 \- 例如，所有全局和所有本地源。 对于只能接受单个值的属性（例如 `allowCredentials` 和 `maxAge`）， 本地会覆盖全局值。 有关详细信息，请参阅 [`CorsConfiguration#combine(CorsConfiguration)`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/cors/CorsConfiguration.html#combine-org.springframework.web.cors.CorsConfiguration-)。

要从source中了解更多信息或进行高级自定义，请查看后面的代码:

*   `CorsConfiguration`

*   `CorsProcessor`, `DefaultCorsProcessor`

*   `AbstractHandlerMapping`


<a id="mvc-cors-controller"></a>

#### [](#mvc-cors-controller)1.6.3. `@CrossOrigin`

[Same as in Spring WebFlux](web-reactive.html#webflux-cors-controller)

在带注解的控制器方法上使用[`@CrossOrigin`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/bind/annotation/CrossOrigin.html)注解启用跨源请求，如以下示例所示：

```java
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin
    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
```

默认情况下，`@CrossOrigin`允许：

*   All origins.

*   All headers.

*   All HTTP methods（可以映射到控制器的方法）


默认情况下不启用`allowedCredentials`，因为它建立了一个信任级别，该信任级别公开敏感的用户特定信息（例如cookie和CSRF令牌），并且只应在适当的地方使用。

`maxAge` 设置为30 分钟.

`@CrossOrigin`在类级别也受支持，并且由所有方法继承，如以下示例所示：

```java
@CrossOrigin(origins = "http://domain2.com", maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
```

您可以在类级别和方法级别使用`@CrossOrigin` ，如以下示例所示:

```java
@CrossOrigin(maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin("http://domain2.com")
    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
```

<a id="mvc-cors-global"></a>

#### [](#mvc-cors-global)1.6.4. 全局配置

[Same as in Spring WebFlux](web-reactive.html#webflux-cors-global)

除了细粒度，基于注解的配置以外，您可能还希望定义一些全局CORS配置。您可以在任何`HandlerMapping`上单独设置基于URL的 `CorsConfiguration`映射。 但是，大多数应用程序使用MVC Java配置或MVC XNM命名空间来执行此操作。

默认情况下，全局配置启用以下内容：

*   All origins.

*   All headers.

*   `GET`, `HEAD`, and `POST` methods.


默认情况下不启用`allowedCredentials`，因为它建立了一个信任级别，该信任级别公开敏感的用户特定信息（例如cookie和CSRF令牌），并且只应在适当的地方使用。

`maxAge` 设置为30分钟.

<a id="mvc-cors-global-java"></a>

##### [](#mvc-cors-global-java)Java 配置

[Same as in Spring WebFlux](web-reactive.html#webflux-cors-global)

要在MVC Java配置中启用CORS，可以使用`CorsRegistry`回调，如以下示例所示:

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {

        registry.addMapping("/api/**")
            .allowedOrigins("http://domain2.com")
            .allowedMethods("PUT", "DELETE")
            .allowedHeaders("header1", "header2", "header3")
            .exposedHeaders("header1", "header2")
            .allowCredentials(true).maxAge(3600);

        // Add more mappings...
    }
}
```

<a id="mvc-cors-global-xml"></a>

##### [](#mvc-cors-global-xml)XML 配置

要在XML命名空间中启用CORS，可以使用`<mvc:cors>`元素，如以下示例所示:

```xml
<mvc:cors>

    <mvc:mapping path="/api/**"
        allowed-origins="http://domain1.com, http://domain2.com"
        allowed-methods="GET, PUT"
        allowed-headers="header1, header2, header3"
        exposed-headers="header1, header2" allow-credentials="true"
        max-age="123" />

    <mvc:mapping path="/resources/**"
        allowed-origins="http://domain1.com" />

</mvc:cors>
```

<a id="mvc-cors-filter"></a>

#### [](#mvc-cors-filter)1.6.5. CORS 过滤器

[Same as in Spring WebFlux](web-reactive.html#webflux-cors-webfilter)

您可以通过内置的 [`CorsFilter`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/filter/CorsFilter.html)应用CORS支持。

如果您尝试将`CorsFilter` 与Spring Security一起使用，请记住Spring Security[内置了对CORS的支持](https://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#cors)。

要配置过滤器，请将`CorsConfigurationSource`传递给其构造函数，如以下示例所示:

```java
CorsConfiguration config = new CorsConfiguration();

// Possibly...
// config.applyPermitDefaultValues()

config.setAllowCredentials(true);
config.addAllowedOrigin("http://domain1.com");
config.addAllowedHeader("*");
config.addAllowedMethod("*");

UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
source.registerCorsConfiguration("/**", config);

CorsFilter filter = new CorsFilter(source);
```

<a id="mvc-web-security"></a>

### [](#mvc-web-security)1.7. Web 安全

[Same as in Spring WebFlux](web-reactive.html#webflux-web-security)

Spring Security项目为保护Web应用程序免受恶意攻击提供支持。 请参阅[Spring Security](https://projects.spring.io/spring-security/)参考文档，包括：

*   [Spring MVC Security](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#mvc)

*   [Spring MVC Test Support](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#test-mockmvc)

*   [CSRF protection](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#csrf)

*   [Security Response Headers](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#headers)


[HDIV](http://hdiv.org/) 是另一个与Spring MVC集成的Web安全框架。

<a id="mvc-caching"></a>

### [](#mvc-caching)1.8. HTTP 缓存

[Same as in Spring WebFlux](web-reactive.html#webflux-caching)

HTTP缓存可以显着提高Web应用程序的性能。HTTP缓存围绕 `Cache-Control` 响应头，随后是条件请求头（例如`Last-Modified`和`ETag`）。 HTTP的响应头`Cache-Control`主要帮助私有缓存（比如浏览器端缓存）和公共缓存（比如代理端缓存）了解它们应该如果缓存HTTP响应。如果内容未更改，则`ETag`头用于生成条件请求， 该条件请求可能导致304（NOT_MODIFIED）没有正文。可以认为它是`Last-Modified`头的一个更精细的后续版本。

本节介绍Spring Web MVC中可用的与HTTP缓存相关的选项。

<a id="mvc-caching-cachecontrol"></a>

#### [](#mvc-caching-cachecontrol)1.8.1. `CacheControl`

[Same as in Spring WebFlux](web-reactive.html#webflux-caching-cachecontrol)

[`CacheControl`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/http/CacheControl.html)支持配置与`Cache-Control`标头相关的设置，并在许多地方被接受为参数：

*   [`WebContentInterceptor`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/mvc/WebContentInterceptor.html)

*   [`WebContentGenerator`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/support/WebContentGenerator.html)

*   [Controllers](#mvc-caching-etag-lastmodified)

*   [Static Resources](#mvc-caching-static-resources)


虽然[RFC 7234](https://tools.ietf.org/html/rfc7234#section-5.2.2)描述了`Cache-Control`响应头的所有可能的指令，但`CacheControl`类型采用面向用例的方法，该方法侧重于常见场景：

```java
// Cache for an hour - "Cache-Control: max-age=3600"
CacheControl ccCacheOneHour = CacheControl.maxAge(1, TimeUnit.HOURS);

// Prevent caching - "Cache-Control: no-store"
CacheControl ccNoStore = CacheControl.noStore();

// Cache for ten days in public and private caches,
// public caches should not transform the response
// "Cache-Control: max-age=864000, public, no-transform"
CacheControl ccCustom = CacheControl.maxAge(10, TimeUnit.DAYS).noTransform().cachePublic();
```

`WebContentGenerator`还接受一个更简单的`cachePeriod`属性（以秒为单位定义），其工作方式如下：

*   A `-1` 值不会生成`Cache-Control` 的响应头。

*   A `0`值将防止缓存使用`'Cache-Control: no-store'` 指令.

*   An `n > 0` 一个大于0的值将缓存给定的响应在`n`秒使用`'Cache-Control: max-age=n'`


<a id="mvc-caching-etag-lastmodified"></a>

#### [](#mvc-caching-etag-lastmodified)1.8.2. Controllers

[Same as in Spring WebFlux](web-reactive.html#webflux-caching-etag-lastmodified)

控制器可以添加对HTTP缓存的显式支持。 我们建议这样做，因为资源的`lastModified`或`ETag`值需要先计算才能与条件请求头进行比较。 控制器可以向`ResponseEntity`添加`ETag`头和`Cache-Control`设置，如以下示例所示：

```java
@GetMapping("/book/{id}")
public ResponseEntity<Book> showBook(@PathVariable Long id) {

    Book book = findBook(id);
    String version = book.getVersion();

    return ResponseEntity
            .ok()
            .cacheControl(CacheControl.maxAge(30, TimeUnit.DAYS))
            .eTag(version) // lastModified is also available
            .body(book);
}
```

如果与条件请求头的比较表明内容未更改，则前面的示例发送带有空响应体的304（NOT_MODIFIED）响应。 否则，`ETag` 和`Cache-Control`标头将添加到响应中。

您还可以对控制器中的条件请求头进行检查，如以下示例所示：

```java
@RequestMapping
public String myHandleMethod(WebRequest webRequest, Model model) {

    long eTag = ... (1)

    if (request.checkNotModified(eTag)) {
        return null; (2)
    }

    model.addAttribute(...); (3)
    return "myViewName";
}
```

**1**、特定于应用的计算。

**2**、响应已设置为304（NOT_MODIFIED） - 无需进一步处理。

**3**、继续请求处理。

有三种变体可用于检查针对`eTag`值，`lastModified`值或两者的条件请求。 对于条件`GET`和`HEAD`请求， 您可以将响应设置为304（NOT_MODIFIED）。对于`POST`, `PUT`, and `DELETE`，您可以将响应设置为409（PRECONDITION_FAILED），以防止并发修改。

<a id="mvc-caching-static-resources"></a>

#### [](#mvc-caching-static-resources)1.8.3. 静态资源

[Same as in Spring WebFlux](web-reactive.html#webflux-caching-static-resources)

您应该使用`Cache-Control` 和条件响应头来提供静态资源，以获得最佳性能。 请参阅有关配置[静态资源](#mvc-config-static-resources)的部分。

<a id="mvc-httpcaching-shallowetag"></a>

#### [](#mvc-httpcaching-shallowetag)1.8.4. `ETag` 过滤器

您可以使用`ShallowEtagHeaderFilter`添加从响应内容计算的“shallow” eTag值，从而节省带宽但不节省CPU时间。 见 [Shallow ETag](#filters-shallow-etag)。

<a id="mvc-view"></a>

### [](#mvc-view)1.9. 视图技术

[Same as in Spring WebFlux](web-reactive.html#webflux-view)

无论您决定使用Thymeleaf，Groovy标记模板，JSP还是其他技术，Spring MVC中视图技术的使用都是可插拔的， 主要是配置更改的问题。 本章介绍了与Spring MVC集成的视图技术。 我们假设您已经熟悉[解析视图](#mvc-viewresolver)。

<a id="mvc-view-thymeleaf"></a>

#### [](#mvc-view-thymeleaf)1.9.1. Thymeleaf

[Same as in Spring WebFlux](web-reactive.html#webflux-view-thymeleaf)

Thymeleaf是一个现代服务器端Java模板引擎，它强调可以通过双击在浏览器中预览的自然HTML模板，这对于UI模板的独立工作（例如，由设计人员）非常有用，而无需运行服务器。 如果您想要替换JSP，Thymeleaf提供了一组最广泛的功能，使这种转换更容易。 Thymeleaf积极开发和维护。 有关更完整的介绍，请参阅[Thymeleaf](http://www.thymeleaf.org/)项目主页。

Thymeleaf与Spring MVC的集成由Thymeleaf项目管理。 配置涉及一些bean声明， 例如`ServletContextTemplateResolver`, `SpringTemplateEngine`, 和 `ThymeleafViewResolver`.。 有关详细信息，请参阅[Thymeleaf+Spring](http://www.thymeleaf.org/documentation.html)。

<a id="mvc-view-freemarker"></a>

#### [](#mvc-view-freemarker)1.9.2. FreeMarker

[Same as in Spring WebFlux](web-reactive.html#webflux-view-freemarker)

[Apache FreeMarker](http://www.freemarker.org) 是一个模板引擎，用于生成从HTML到电子邮件和其他的任何类型的文本输出。 Spring Framework有一个内置的集成，可以将Spring MVC与FreeMarker模板结合使用。

<a id="mvc-view-freemarker-contextconfig"></a>

##### [](#mvc-view-freemarker-contextconfig)视图配置

[Same as in Spring WebFlux](web-reactive.html#webflux-view-freemarker-contextconfig)

以下示例显示如何将FreeMarker配置为视图技术：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.freemarker();
    }

    // Configure FreeMarker...

    @Bean
    public FreeMarkerConfigurer freeMarkerConfigurer() {
        FreeMarkerConfigurer configurer = new FreeMarkerConfigurer();
        configurer.setTemplateLoaderPath("/WEB-INF/freemarker");
        return configurer;
    }
}
```

以下示例显示如何在XML中配置相同的内容：

```xml
<mvc:annotation-driven/>

<mvc:view-resolvers>
    <mvc:freemarker/>
</mvc:view-resolvers>

<!-- Configure FreeMarker... -->
<mvc:freemarker-configurer>
    <mvc:template-loader-path location="/WEB-INF/freemarker"/>
</mvc:freemarker-configurer>
```

或者，您也可以声明`FreeMarkerConfigurer` bean以完全控制所有属性，如以下示例所示：

```xml
<bean id="freemarkerConfig" class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
    <property name="templateLoaderPath" value="/WEB-INF/freemarker/"/>
</bean>
```

模板需要存储在上面所示的FreeMarkerConfigurer指定的目录中，根据前面的配置，如果您的控制器返回`welcome`视图名称，解析器将查找`/WEB-INF/freemarker/welcome.ftl`模板。

<a id="mvc-views-freemarker"></a>

##### [](#mvc-views-freemarker)FreeMarker 配置

[Same as in Spring WebFlux](web-reactive.html#webflux-views-freemarker)

通过设置`FreeMarkerConfigurer` bean可以将FreeMarker的'Settings' 和 'SharedVariables' 值直接传递Spring管理的FreeMarker对象。 `freemarkerSettings`属性需要`java.util.Properties`对象。 而`freemarkerVariables`属性需要`java.util.Map`。以下示例显示了如何执行此操作：

```xml
<bean id="freemarkerConfig" class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
    <property name="templateLoaderPath" value="/WEB-INF/freemarker/"/>
    <property name="freemarkerVariables">
        <map>
            <entry key="xml_escape" value-ref="fmXmlEscape"/>
        </map>
    </property>
</bean>

<bean id="fmXmlEscape" class="freemarker.template.utility.XmlEscape"/>
```

有关更多的Configuration内容的`设置`和变量可以查看FreeMarker文档

<a id="mvc-view-freemarker-forms"></a>

##### [](#mvc-view-freemarker-forms)表单处理

Spring本身提供了用于JSP的标签库，其中包含（当然还有很多）`<spring:bind/>`标签，这个标签用来展示从Web上的`验证器`或业务层抛出的失败验证表单。 Spring还支持FreeMarker中的相同功能，并提供了方便的宏来生成表单输入元素。

<a id="mvc-view-bind-macros"></a>

###### [](#mvc-view-bind-macros)绑定宏命令

`spring-webmvc.jar` 包文件包含Velocity和FreeMarker的一组标准宏，因此两者都适用。

Spring库中定义的某些宏被认为是内部的(私有的），但在宏定义中不存在这样的范围，其实所有宏都可以在调用代码和用户模板时看到。以下各节仅集中于需要从模板中直接调用的宏， 如果希望直接查看宏代码， 那么可以看文件`spring.ftl`，定义在`org.springframework.web.servlet.view.freemarker`包中。

<a id="mvc-view-simple-binding"></a>

###### [](#mvc-view-simple-binding)简单的绑定

HTML表单(vm或ftl模板),充当了Spring MVC控制器的表单视图,可以使用类似下面的代码绑定字段值,也可以类似JSP那样在每个输入字段后面添加错误信息. 以下示例显示了之前配置的personForm视图：

```html
<!-- freemarker macros have to be imported into a namespace. We strongly
recommend sticking to 'spring' -->
<#import "/spring.ftl" as spring/>
<html>
    ...
    <form action="" method="POST">
        Name:
        <@spring.bind "myModelObject.name"/>
        <input type="text"
            name="${spring.status.expression}"
            value="${spring.status.value?html}"/><br>
        <#list spring.status.errorMessages as error> <b>${error}</b> <br> </#list>
        <br>
        ...
        <input type="submit" value="submit"/>
    </form>
    ...
</html>
```

`<@spring.bind>`需要一个包含命令对象的 'path' 参数（默认是'command',除非在`FormController`属性中被改变了），后面跟着写需要绑定到命令对象上的字段名. 可以使用嵌套字段,例如 `command.address.street`,绑定宏可以在`web.xml`中设置ServletContext的参数`defaultHtmlEscape`，用于定义HTML的转义行为。

`<@spring.bindEscaped>` 宏命令是可选的，它接收第二个参数并显式地指定是否应在状态错误消息或值中使用HTML转义。按需设置为`true`或`false`，还有很多其它的宏，它们将在下一节中介绍。

<a id="mvc-views-form-macros"></a>

###### [](#mvc-views-form-macros)输入宏命令

Velocity和FreeMarker都使用宏简化了绑定和表单的生成（包括验证错误的显示），没有必要使用这些宏来生成表单输入字段，实际上他们都可以直接绑定在简单的HTML中，并且可混合使用。

下表中的可用宏显示了FTL定义和每个参数列表：

Table 6. 宏命令定义表

|  宏命令    |  FTL 定义表    |
| ---- | ---- |
| `message`（根据代码参数从资源包中输出字符串）     | <@spring.message code/>     |
| `messageText` （根据代码参数从资源包中输出一个字符串，失败则使用默认参数的值）     |<@spring.messageText code, text/>      |
| `url`（使用应用程序的上下文根作为相对URL的前缀）     | <@spring.url relativeUrl/>     |
| `formInput` (标准输入域用户收集用户信息)     | <@spring.formInput path, attributes, fieldType/>     |
|`formHiddenInput` (用于提交肥输入域的隐藏字段)      | <@spring.formHiddenInput path, attributes/>     |
| `formPasswordInput` (用户收集密码的标准输入字段，请注意，此类型的字段中不会填充任何值)     | <@spring.formPasswordInput path, attributes/>     |
|`formTextarea` (大文本域，用于收集大而自由的文本输入)      |<@spring.formTextarea path, attributes/>      |
|`formSingleSelect` (下拉选项框，可以选择一个必需的值)      | <@spring.formSingleSelect path, options, attributes/>     |
|`formMultiSelect` (一个选项列表框，允许用户选择0或更多值)      |<@spring.formMultiSelect path, options, attributes/>      |
| `formRadioButtons` (单选按钮，可以从可用选项中进行单个选择)     |  <@spring.formRadioButtons path, options separator, attributes/>    |
|  `formCheckboxes` (一组允许选择0或更多值的复选框)    | <@spring.formCheckboxes path, options, separator, attributes/>     |
|`formCheckbox` (单个复选框)|<@spring.formCheckbox path, attributes/>|
|`showErrors` (简化绑定字段的验证错误显示)|<@spring.showErrors separator, classOrStyle/>|


*   在FTL（FreeMarker）中, `formHiddenInput` 和 `formPasswordInput` 这两个宏实际上并不需要，因为可以使用普通的 `formInput`宏。将`hidden` 或 `password`指定为`fieldType` 参数的值


上述任何宏的参数都具有一致的含义

*   `路径`: 要绑定到的字段的名称（例如 "command.name"）

*   `选项`: 可从输入字段中选择的所有可用值的映射，map的键表示从表单POST后得到的对象的值（已绑定的），Map对象保存这些键用于返回值后能在表单上显示出来。 通常这样map由控制器提供数据，任何map都可以实现按需使用，可以使用`SortedMap`，例如 `TreeMap`和适当的`Comparator`为所有的值排序，使用来自`commons-collections`包中的 `LinkedHashMap` 或 `LinkedMap`也是相同的原理。

*   `分隔符`:多个选项可以作为元素（单选按钮或复选框）可以使用标签对字符序列进行分隔（例如`<br>`）。

*   `属性`: HTML标签本身中可以包含任意标签或文本的附加字符串。字符串与上面的宏分别对应，例如，在一个文本字段提供属性'rows="5" cols="60"'字段， 也可以添加css，例如'style="border:1px solid silver"'。

*   `classOrStyle`: 对于`showErrors` 宏, 可以使用`span`标签包装每个错误的CSS类的名称。如果未提供任何信息 (或该值为空），则错误将包含在`<b></b>` 标签中


以下部分概述了宏的示例（一些在FTL中，一些在VTL中）。 如果两种语言之间存在使用差异，则会在说明中对其进行说明。

<a id="mvc-views-form-macros-input"></a>

[](#mvc-views-form-macros-input)输入域

`formInput`宏采用 `path` 参数（`command.name`）和附加属性参数（在下一个示例中为空）。宏与所有其他表单生成宏一起在path参数上执行隐式Spring绑定。在出现新绑定之前， 前一个绑定仍然有效，因此`showErrors` 宏不需要再次传递path参数，它只对上次为其创建绑定的任何字段进行操作。

`showErrors`宏采用分隔符参数(将用于分隔给定字段上的多个错误的字符，同时还接受第二个参数：类名或样式属性。请注意，FreeMarker能够为属性参数指定默认值，这与Velocity不同， 以下示例显示如何使用 `formInput`和 `showWErrors`宏：：

    <@spring.formInput "command.name"/>
    <@spring.showErrors "<br>"/>

下一个示例显示表单片段的输出，生成名称字段并在提交表单后在字段中没有值时显示验证错误。 验证通过Spring的验证框架进行。

生成的HTML类似于以下示例：

```html
Name:
<input type="text" name="name" value="">
<br>
    <b>required</b>
<br>
<br>
```

`formTextarea` 宏类似于`formInput`宏，连接收的参数都是相同的。通常，第二个参数（属性）将被使用用于传递格式信息或`rows`和`cols` 的属性。

<a id="mvc-views-form-macros-select"></a>

[](#mvc-views-form-macros-select)选择字段

有四个字段宏可以用于生产HTML表单中的公共UI值作为选择的输入：

*   `formSingleSelect`

*   `formMultiSelect`

*   `formRadioButtons`

*   `formCheckboxes`


这四个宏都可以从表单字段中接收`Map`，其实需要的就是标签的值。当然值和标签是可以取相同的名。

下一个例子是FTL中的单选按钮。表单使用'London'作为这个字段的默认值，因此不需用进行验证。当渲染表单时，要选择的整个城市列表都在'cityMap'中，cityMap是数据模型。以下清单显示了该示例：

    ...
    Town:
    <@spring.formRadioButtons "command.address.town", cityMap, ""/><br><br>

前面的列表呈现一行单选按钮，一个用于`cityMap`中的每个值，并使用分隔符`""`。没有提供其他属性（缺少宏的最后一个参数）。cityMap对Map中的每个键值对使用相同的`String`。 映射的键是表单实际提交为POSTed请求参数的键。 map值是用户看到的标签。 在前面的示例中，给定一个包含三个众所周知的城市的列表以及表单支持对象中的默认值，HTML类似于以下内容：

```html
Town:
<input type="radio" name="address.town" value="London">London</input>
<input type="radio" name="address.town" value="Paris" checked="checked">Paris</input>
<input type="radio" name="address.town" value="New York">New York</input>
```

如果您的应用程序希望通过内部代码来处理城市，可以写一个name为cityMap的Map传递给模板，如下面的例子：

```java
protected Map<String, String> referenceData(HttpServletRequest request) throws Exception {
    Map<String, String> cityMap = new LinkedHashMap<>();
    cityMap.put("LDN", "London");
    cityMap.put("PRS", "Paris");
    cityMap.put("NYC", "New York");

    Map<String, String> model = new HashMap<>();
    model.put("cityMap", cityMap);
    return model;
}
```

代码将按你的设置输出，可以看到更多的城市名字。

```html
Town:
<input type="radio" name="address.town" value="LDN">London</input>
<input type="radio" name="address.town" value="PRS" checked="checked">Paris</input>
<input type="radio" name="address.town" value="NYC">New York</input>
```

<a id="mvc-views-form-macros-html-escaping"></a>

###### [](#mvc-views-form-macros-html-escaping)HTML 转义

由于HTML的版本问题，上面的表单宏在HTML的4.01版本中需要使用到转义，转义可以在`web.xml`中通过Spring的绑定来定义。为了使标签遵守XHTML的规定以及覆盖默认的HTML转义值， 可以在模板中定义两个变量（或者使你的模型设置为模板可见形式）。在模板中指定的优点是：它们可以在模板处理后更改为不同的值，以便为表单中的不同字段提供不同的行为。

要切换为标记的XHTML合规性，请为名为`xhtmlCompliant`的模型或上下文变量指定值`true` ，如以下示例所示：

```html
<#-- for FreeMarker -->
<#assign xhtmlCompliant = true>
```

处理完该指令后，Spring宏生成的任何元素现在都符合XHTML标准。

以类似的方式，您可以指定每个字段的HTML转义，如以下示例所示：

```html
<#-- until this point, default HTML escaping is used -->

<#assign htmlEscape = true>
<#-- next field will use HTML escaping -->
<@spring.formInput "command.name"/>

<#assign htmlEscape = false in spring>
<#-- all future fields will be bound with HTML escaping off -->
```

<a id="mvc-view-groovymarkup"></a>

#### [](#mvc-view-groovymarkup)1.9.3. Groovy Markup

[Groovy标记模板引擎](http://groovy-lang.org/templating.html#_the_markuptemplateengine)主要用于生成类似XML的标记（XML，XHTML，HTML5等），但您可以使用它来生成任何基于文本的内容。 Spring Framework有一个内置的集成，可以将Spring MVC与Groovy Markup结合使用。

目前要求使用Groovy 2.3.1+的版本.

<a id="mvc-view-groovymarkup-configuration"></a>

##### [](#mvc-view-groovymarkup-configuration)配置

以下示例显示如何配置Groovy标记模板引擎：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.groovy();
    }

    // Configure the Groovy Markup Template Engine...

    @Bean
    public GroovyMarkupConfigurer groovyMarkupConfigurer() {
        GroovyMarkupConfigurer configurer = new GroovyMarkupConfigurer();
        configurer.setResourceLoaderPath("/WEB-INF/");
        return configurer;
    }
}
```

以下示例显示如何在XML中配置相同的内容：

```xml
<mvc:annotation-driven/>

<mvc:view-resolvers>
    <mvc:groovy/>
</mvc:view-resolvers>

<!-- Configure the Groovy Markup Template Engine... -->
<mvc:groovy-configurer resource-loader-path="/WEB-INF/"/>
```

<a id="mvc-view-groovymarkup-example"></a>

##### [](#mvc-view-groovymarkup-example)例子

与传统的模板引擎不同，Groovy是依赖于使用生成器语法的DSL。 以下示例显示了HTML页面的示例模板：

```html
yieldUnescaped '<!DOCTYPE html>'
html(lang:'en') {
    head {
        meta('http-equiv':'"Content-Type" content="text/html; charset=utf-8"')
        title('My page')
    }
    body {
        p('This is an example of HTML contents')
    }
}
```

<a id="mvc-view-script"></a>

#### [](#mvc-view-script)1.9.4. 脚本视图

[Same as in Spring WebFlux](web-reactive.html#webflux-view-script)

Spring Framework有一个内置的集成，可以将Spring MVC与任何可以在[JSR-223](https://www.jcp.org/en/jsr/detail?id=223) Java脚本引擎之上运行的模板库一起使用。 我们在不同的脚本引擎上测试了以下模板库：:



Scripting Library

Scripting Engine

[Handlebars](http://handlebarsjs.com/)

[Nashorn](http://openjdk.java.net/projects/nashorn/)

[Mustache](https://mustache.github.io/)

[Nashorn](http://openjdk.java.net/projects/nashorn/)

[React](https://facebook.github.io/react/)

[Nashorn](http://openjdk.java.net/projects/nashorn/)

[EJS](http://www.embeddedjs.com/)

[Nashorn](http://openjdk.java.net/projects/nashorn/)

[ERB](http://www.stuartellis.eu/articles/erb/)

[JRuby](http://jruby.org)

[String templates](https://docs.python.org/2/library/string.html#template-strings)

[Jython](http://www.jython.org/)

[Kotlin Script templating](https://github.com/sdeleuze/kotlin-script-templating)

[Kotlin](https://kotlinlang.org/)

集成任何其他脚本引擎的基本规则是它必须实现`ScriptEngine` 和`Invocable` 接口。

<a id="mvc-view-script-dependencies"></a>

##### [](#mvc-view-script-dependencies)要求

[Same as in Spring WebFlux](web-reactive.html#webflux-view-script-dependencies)

您需要在类路径上安装脚本引擎，其详细信息因脚本引擎而异:

*   The [Nashorn](http://openjdk.java.net/projects/nashorn/) Javascript引擎提供了内置的Java 8+。强烈建议使用最新的可用更新版本。

*   为了获得[JRuby](http://jruby.org) 支持，应添加JRuby依赖性

*   为了获得[Jython](http://www.jython.org)支持，应添加Jython依赖性。

*   `org.jetbrains.kotlin:kotlin-script-util`依赖项和包含在 `META-INF/services/javax.script.ScriptEngineFactory`文件里的`org.jetbrains.kotlin.script.jsr223.KotlinJsr223JvmLocalScriptEngineFactory`行应添加到Kotlin脚本支持中。 有关详细信息，请参阅[此示例](https://github.com/sdeleuze/kotlin-script-templating)。


还需要为基于脚本的模板引擎添加依赖项。例如，对于javascript，可以使用[WebJars](http://www.webjars.org/)。

<a id="mvc-view-script-integrate"></a>

##### [](#mvc-view-script-integrate)脚本模板

[Same as in Spring WebFlux](web-reactive.html#webflux-script-integrate)

您可以声明`ScriptTemplateConfigurer`bean以指定要使用的脚本引擎，要加载的脚本文件，要调用以呈现模板的函数，等等。 以下示例使用Mustache模板和Nashorn JavaScript引擎：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.scriptTemplate();
    }

    @Bean
    public ScriptTemplateConfigurer configurer() {
        ScriptTemplateConfigurer configurer = new ScriptTemplateConfigurer();
        configurer.setEngineName("nashorn");
        configurer.setScripts("mustache.js");
        configurer.setRenderObject("Mustache");
        configurer.setRenderFunction("render");
        return configurer;
    }
}
```

以下示例显示了XML中的相同排列：:

```xml
<mvc:annotation-driven/>

<mvc:view-resolvers>
    <mvc:script-template/>
</mvc:view-resolvers>

<mvc:script-template-configurer engine-name="nashorn" render-object="Mustache" render-function="render">
    <mvc:script location="mustache.js"/>
</mvc:script-template-configurer>
```

对于Java和XML配置，控制器看起来没有什么不同，如以下示例所示：

```java
@Controller
public class SampleController {

    @GetMapping("/sample")
    public String test(Model model) {
        model.addObject("title", "Sample title");
        model.addObject("body", "Sample body");
        return "template";
    }
}
```

以下示例显示了Mustache模板：

```html
<html>
    <head>
        <title>{{title}}</title>
    </head>
    <body>
        <p>{{body}}</p>
    </body>
</html>
```

使用以下参数调用render函数：

*   `String template`: 模板内容

*   `Map model`: 视图模型

*   `RenderingContext renderingContext`: [`RenderingContext`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/view/script/RenderingContext.html)，提供对应用程序上下文，区域设置，模板加载器和URL的访问（自5.0起）。


`Mustache.render()` 方法会与本地兼容，因此可以直接调用。

如果模板化技术需要自定义，则可以提供实现自定义渲染函数的脚本。例如，[Handlerbars](http://handlebarsjs.com)需要在使用模板之前进行编译，并且需要使用 [polyfill](https://en.wikipedia.org/wiki/Polyfill)以模拟服务器端脚本引擎中不可用的某些浏览器功能。

以下示例显示了如何执行此操作：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.scriptTemplate();
    }

    @Bean
    public ScriptTemplateConfigurer configurer() {
        ScriptTemplateConfigurer configurer = new ScriptTemplateConfigurer();
        configurer.setEngineName("nashorn");
        configurer.setScripts("polyfill.js", "handlebars.js", "render.js");
        configurer.setRenderFunction("render");
        configurer.setSharedEngine(false);
        return configurer;
    }
}
```

当要求非线程安全地使用脚本引擎时，需要将`sharedEngine`的属性设置为 `false` ，因为模板库不是为了并发而设计的，具体可以看运行在Nashorn上的Handlerbars或react。据此，需要Java 8u60+的版本来修复这个 [bug](https://bugs.openjdk.java.net/browse/JDK-8076099)。

`polyfill.js` 只需定义一个window对象，就可以被Handlerbars运行，如下所示：

    var window = {};

脚本`render.js`会在使用该模板之前被编译，一个好的产品应当保存和重用模板（使用缓存的方法），这样高效些。这可以在脚本中完成，并且可以自定义它(例如管理模板引擎配置。以下示例显示了如何执行此操作：

```js
function render(template, model) {
    var compiledTemplate = Handlebars.compile(template);
    return compiledTemplate(model);
}
```

有关更多配置示例，请查看Spring Framework单元测试，[Java](https://github.com/spring-projects/spring-framework/tree/master/spring-webmvc/src/test/java/org/springframework/web/servlet/view/script)和[resources](https://github.com/spring-projects/spring-framework/tree/master/spring-webmvc/src/test/resources/org/springframework/web/servlet/view/script)。

<a id="mvc-view-jsp"></a>

#### [](#mvc-view-jsp)1.9.5. JSP 和 JSTL

Spring为JSP和JSTL视图提供了一些现成的解决方案

<a id="mvc-view-jsp-resolver"></a>

##### [](#mvc-view-jsp-resolver)视图解析

使用JSP进行开发时，可以声明`InternalResourceViewResolver`或`ResourceBundleViewResolver` bean。

`ResourceBundleViewResolver`依赖于属性文件来定义映射到类和URL的视图名称。使用`ResourceBundleViewResolver`，您可以通过仅使用一个解析器来混合不同类型的视图，如以下示例所示：

```xml
<!-- the ResourceBundleViewResolver -->
<bean id="viewResolver" class="org.springframework.web.servlet.view.ResourceBundleViewResolver">
    <property name="basename" value="views"/>
</bean>

# And a sample properties file is uses (views.properties in WEB-INF/classes):
welcome.(class)=org.springframework.web.servlet.view.JstlView
welcome.url=/WEB-INF/jsp/welcome.jsp

productList.(class)=org.springframework.web.servlet.view.JstlView
productList.url=/WEB-INF/jsp/productlist.jsp
```

`InternalResourceBundleViewResolver`也可用于JSP。 作为最佳实践，我们强烈建议将JSP文件放在`'WEB-INF'`目录下的目录中，以便客户端无法直接访问。

```xml
<bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="viewClass" value="org.springframework.web.servlet.view.JstlView"/>
    <property name="prefix" value="/WEB-INF/jsp/"/>
    <property name="suffix" value=".jsp"/>
</bean>
```

<a id="mvc-view-jsp-jstl"></a>

##### [](#mvc-view-jsp-jstl)JSPs 和 JSTL

当使用Java标准标记库时，必须使用特殊的视图类`JstlView`，因为JSTL需要一些准备工作，例如I18N功能。

<a id="mvc-view-jsp-tags"></a>

##### [](#mvc-view-jsp-tags)Spring的JSP标签库

Spring提供了请求参数与命令对象的数据绑定，如前面章节所述。为了方便开发JSP页面，结合这些数据绑定功能，Spring提供了一些使事情变得更容易的标记。所有的Spring标记都haveHTML转义功能以启用或禁用字符转义。

标签库描述符(TLD) 在`spring-webmvc.jar`包中。更多的信息，请浏览[API参考](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/tags/package-summary.html#package.description)或查看标签库说明。

<a id="mvc-view-jsp-formtaglib"></a>

##### [](#mvc-view-jsp-formtaglib)Spring的表单标签库

从2.0版本开始, Spring在使用JSP和Spring Web MVC时为处理表单元素提供了一套完整的数据绑定识别标签。每个标签都支持其相应的HTML标签对应的属性集，使标签熟悉和直观地使用，标签生成的HTML 4.01/XHTML 1.0兼容。

不同于其他的表单或输入标签库，Spring的表单标签库是集成在Spring Web MVC中，标签可以使用控制器处理的命令对象和引用数据。因此在下面的例子中将会看到，表单标签使得JSP更加方便开发、阅读和维护。

让我们浏览一下表单标签，看看如何使用每个标签的例子。其中已经包括了生成的HTML片段，而某些标签需要进一步的讨论。

<a id="mvc-view-jsp-formtaglib-configuration"></a>

###### [](#mvc-view-jsp-formtaglib-configuration)配置

表单标签库捆绑在`spring-webmvc.jar`中. 库描述符名字为`spring-form.tld`.

如果需要使用到这些标签，在JSP页面的头部必须添加对应的标签库

```jsp
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
```

其中 `form`是后面引用标签的前缀。

<a id="mvc-view-jsp-formtaglib-formtag"></a>

###### [](#mvc-view-jsp-formtaglib-formtag)表单标签

标签'form'绑定了引用库的内部标签，可以被HTML解析。它将命令对象放在`PageContext`中，以便可以通过内部标记访问命令对象。此库中的所有其他标记都是`form`标记的嵌套标记。

假设我们有一个名为 `User`的域对象。 它是一个JavaBean，具有`firstName`和`lastName`等属性。我们将使用它作为表单控制器的形式支持对象，输出给 `form.jsp`。以下示例显示了form.jsp的显示：

```html
<form:form>
    <table>
        <tr>
            <td>First Name:</td>
            <td><form:input path="firstName"/></td>
        </tr>
        <tr>
            <td>Last Name:</td>
            <td><form:input path="lastName"/></td>
        </tr>
        <tr>
            <td colspan="2">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form:form>
```

`firstName`和`lastName` 值会从页面控制器放置在`PageContext`的命令对象中查找。更多复杂的例子都是这样延伸的，重点就是内部标签是如何与表单标签一起使用的。

以下清单显示了生成的HTML，它看起来像标准格式：

```html
<form method="POST">
    <table>
        <tr>
            <td>First Name:</td>
            <td><input name="firstName" type="text" value="Harry"/></td>
        </tr>
        <tr>
            <td>Last Name:</td>
            <td><input name="lastName" type="text" value="Potter"/></td>
        </tr>
        <tr>
            <td colspan="2">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form>
```

之前的JSP假设表单的变量名是`command`。如果对象已经封装到另一个名称中了，表单也支持从自定义名称中绑定变量（这是最佳实践）。如以下示例所示：

```html
<form:form modelAttribute="user">
    <table>
        <tr>
            <td>First Name:</td>
            <td><form:input path="firstName"/></td>
        </tr>
        <tr>
            <td>Last Name:</td>
            <td><form:input path="lastName"/></td>
        </tr>
        <tr>
            <td colspan="2">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form:form>
```

<a id="mvc-view-jsp-formtaglib-inputtag"></a>

###### [](#mvc-view-jsp-formtaglib-inputtag)输入标签

这个标签其实就是HTML的`input`标签（当然是解析后的），此标签或默认绑定值和 `type='text'`属性。有关此的示例，请参阅[表单标签](#mvc-view-jsp-formtaglib-formtag)。 您还可以使用特定于HTML5的类型，例如`email`, `tel`, `date`等。

<a id="mvc-view-jsp-formtaglib-checkboxtag"></a>

###### [](#mvc-view-jsp-formtaglib-checkboxtag)复选框标签

复选框也会解析成HTML的输入标签。

假设`User`对象拥有新闻订阅和爱好列表属性，显示了`Preferences`类：

```java
public class Preferences {

    private boolean receiveNewsletter;
    private String[] interests;
    private String favouriteWord;

    public boolean isReceiveNewsletter() {
        return receiveNewsletter;
    }

    public void setReceiveNewsletter(boolean receiveNewsletter) {
        this.receiveNewsletter = receiveNewsletter;
    }

    public String[] getInterests() {
        return interests;
    }

    public void setInterests(String[] interests) {
        this.interests = interests;
    }

    public String getFavouriteWord() {
        return favouriteWord;
    }

    public void setFavouriteWord(String favouriteWord) {
        this.favouriteWord = favouriteWord;
    }
}
```

相应的`form.jsp` 可能类似于以下内容：

```html
<form:form>
    <table>
        <tr>
            <td>Subscribe to newsletter?:</td>
            <%-- Approach 1: Property is of type java.lang.Boolean --%>
            <td><form:checkbox path="preferences.receiveNewsletter"/></td>
        </tr>

        <tr>
            <td>Interests:</td>
            <%-- Approach 2: Property is of an array or of type java.util.Collection --%>
            <td>
                Quidditch: <form:checkbox path="preferences.interests" value="Quidditch"/>
                Herbology: <form:checkbox path="preferences.interests" value="Herbology"/>
                Defence Against the Dark Arts: <form:checkbox path="preferences.interests" value="Defence Against the Dark Arts"/>
            </td>
        </tr>

        <tr>
            <td>Favourite Word:</td>
            <%-- Approach 3: Property is of type java.lang.Object --%>
            <td>
                Magic: <form:checkbox path="preferences.favouriteWord" value="Magic"/>
            </td>
        </tr>
    </table>
</form:form>
```

checkbox标签有三种方法，可满足您的所有复选框需求。

*   方法一: 当绑定值为`java.lang.Boolean`,如果绑定值为 `true`。则`input(checkbox)`被标记为`checked` 。value属性对应于`setValue(Object)`的值（当然是解析后的）。

*   方法二: 当绑定值是`array` 或`java.util.Collection`,如果绑定集合中存在已配置的 `setValue(Object)` 则输入（复选框）将标记为已选中。。

*   方法三: 对于任何其他绑定值类型, 如果配置的`setValue(Object)`等于绑定值，则`input(checkbox)`被标记为已选中。


请注意，无论采用何种方法，都会生成相同的HTML结构。 以下HTML代码段定义了一些复选框：

```html
<tr>
    <td>Interests:</td>
    <td>
        Quidditch: <input name="preferences.interests" type="checkbox" value="Quidditch"/>
        <input type="hidden" value="1" name="_preferences.interests"/>
        Herbology: <input name="preferences.interests" type="checkbox" value="Herbology"/>
        <input type="hidden" value="1" name="_preferences.interests"/>
        Defence Against the Dark Arts: <input name="preferences.interests" type="checkbox" value="Defence Against the Dark Arts"/>
        <input type="hidden" value="1" name="_preferences.interests"/>
    </td>
</tr>
```

可能不希望看到的是每个复选框后都附加隐藏域，如果html页中的复选框一个都没有选中，则在提交表单后，它的值将不会作为HTTP请求参数的一部分发送到服务器，因此为了使Spring表单数据绑定工作。 需要在html中使用此奇怪的变通方法。复选框标记遵循现有的Spring约定，其中包括每个复选框都以下划线`_`为前缀的隐藏参数。通过这样做，可以有效地告诉Spring"该复选框在表单中是可见的,并且希望将表单数据绑定到其上的对象能够反映复选框的状态".

<a id="mvc-view-jsp-formtaglib-checkboxestag"></a>

###### [](#mvc-view-jsp-formtaglib-checkboxestag)复选框标签

checkbox标签相当于多个HTML的input标签

上一个例子展示了复选框标签的生成。有时候，不希望在JSP页面中列出User的所有爱好。你更希望在运行提供可选的列表，并传递给复选框标签。这是复选框标记的用途。 可以传入一个数组、 一个列表或一个包含`items`属性中的可用选项的Map。绑定属性通常是一个集合，因此它可以保存用户选择的多个值。下面是使用此标签的JSP示例

```html
<form:form>
    <table>
        <tr>
            <td>Interests:</td>
            <td>
                <%-- Property is of an array or of type java.util.Collection --%>
                <form:checkboxes path="preferences.interests" items="${interestList}"/>
            </td>
        </tr>
    </table>
</form:form>
```

本实例假定`interestList`是一个模型的属性`List`，包含需要的字符串值。在使用MAP的情况下，Map的key将用作值，map的value将用作要显示的标签。还可以使用自定义对象，可以使用`itemValue`和使用`itemLabel`的标签作为该值提供属性名称。

<a id="mvc-view-jsp-formtaglib-radiobuttontag"></a>

###### [](#mvc-view-jsp-formtaglib-radiobuttontag)单选框标签

还有一个可以解析成HTMLinput标签的是radio标签

radio很简单，提供多个值，但是一次只能选其中一个。如以下示例所示：

```html
<tr>
    <td>Sex:</td>
    <td>
        Male: <form:radiobutton path="sex" value="M"/> <br/>
        Female: <form:radiobutton path="sex" value="F"/>
    </td>
</tr>
```

<a id="mvc-view-jsp-formtaglib-radiobuttonstag"></a>

###### [](#mvc-view-jsp-formtaglib-radiobuttonstag)单选标签 `radiobuttons`

这个形式的`radio`也可以解析成HTML的`input`标签，只是它是多个单选。

就像上面的[`checkboxes` 标签](#mvc-view-jsp-formtaglib-checkboxestag)一样，可能希望将可用选项作为运行时变量传入。对于此用法，可以使用单选标签。可以传入一个数组、一个列表或一个包含 `items`属性的Map。如果使用map，map的key将使用作为值并且map的值将使用作为标签来显示。还可以使用自定义对象，可以使用`itemValue`和使用`itemLabel`的标签作为该值提供属性名称。

    <tr>
        <td>Sex:</td>
        <td><form:radiobuttons path="sex" items="${sexOptions}"/></td>
    </tr>

<a id="mvc-view-jsp-formtaglib-passwordtag"></a>

###### [](#mvc-view-jsp-formtaglib-passwordtag)密码框标签

`password` 标签页会解析成HTML的`input`标签 只是它有自己的特性。

```html
<tr>
    <td>Password:</td>
    <td>
        <form:password path="password"/>
    </td>
</tr>
```

请注意，密码值是不可见的。如果希望密码值可见，需要设置`showPassword`属性为`true`，如下所示：

```html
<tr>
    <td>Password:</td>
    <td>
        <form:password path="password" value="^76525bvHGq" showPassword="true"/>
    </td>
</tr>
```

<a id="mvc-view-jsp-formtaglib-selecttag"></a>

###### [](#mvc-view-jsp-formtaglib-selecttag)选择标签

T这个标签就是HTML的select元素。支持单层选项或嵌套选项的选择，数据利用项来绑定。

让我们假设`User`，他有一个技能列表如下:

```html
<tr>
    <td>Skills:</td>
    <td><form:select path="skills" items="${skills}"/></td>
</tr>
```

如果User选中的技能是Herbology，那么这个Skills的HTML源代码是这样的：

```html
<tr>
    <td>Skills:</td>
    <td>
        <select name="skills" multiple="true">
            <option value="Potions">Potions</option>
            <option value="Herbology" selected="selected">Herbology</option>
            <option value="Quidditch">Quidditch</option>
        </select>
    </td>
</tr>
```

<a id="mvc-view-jsp-formtaglib-optiontag"></a>

###### [](#mvc-view-jsp-formtaglib-optiontag)选项标签

这个标签就是HTML的option(配合select中）元素。它会对被绑定的值设置属性为selected，以下HTML显示了它的典型输出：

```html
<tr>
    <td>House:</td>
    <td>
        <form:select path="house">
            <form:option value="Gryffindor"/>
            <form:option value="Hufflepuff"/>
            <form:option value="Ravenclaw"/>
            <form:option value="Slytherin"/>
        </form:select>
    </td>
</tr>
```

如果User的家是在Gryffindor，那么House的HTML源代码长这样：

```html
<tr>
    <td>House:</td>
    <td>
        <select name="house">
            <option value="Gryffindor" selected="selected">Gryffindor</option> (1)
            <option value="Hufflepuff">Hufflepuff</option>
            <option value="Ravenclaw">Ravenclaw</option>
            <option value="Slytherin">Slytherin</option>
        </select>
    </td>
</tr>
```

**1**、请注意添加所选属性。

<a id="mvc-view-jsp-formtaglib-optionstag"></a>

###### [](#mvc-view-jsp-formtaglib-optionstag)选项标签

这个标签就是HTML的option(配合select中)元素,但是它处理的是一个列表，它会对被绑定的值设置属性为selected，如下所示：

```html
<tr>
    <td>Country:</td>
    <td>
        <form:select path="country">
            <form:option value="-" label="--Please Select"/>
            <form:options items="${countryList}" itemValue="code" itemLabel="name"/>
        </form:select>
    </td>
</tr>
```

如果User住在UK，那么Country的HTML源代码长这这样:

```html
<tr>
    <td>Country:</td>
    <td>
        <select name="country">
            <option value="-">--Please Select</option>
            <option value="AT">Austria</option>
            <option value="UK" selected="selected">United Kingdom</option> (1)
            <option value="US">United States</option>
        </select>
    </td>
</tr>
```

**1**、Note the addition of a `selected` attribute.

看上面的两个例子， `option`和`options`标签都生成了相同的标准的HTML，但允许你在JSP中显式地按需显示属性值，例如默认的字符串在例子中是"-- Please Select"（就是默认的，选择为空的那个，这个很有用）。

`items`属性通常使用项对象的集合或数组填充， `itemValue`和`itemLabel`就是对应指定bean对象的属性，如果没有指定，对象将被转成字符串。或者， 可以定义一个Map的items，`Map`的key对应选项值，value对应选项标签。如果如果`itemValue`和`itemLabel`都被指定了，那么item值属性对应key，item标签属性对应value。

<a id="mvc-view-jsp-formtaglib-textareatag"></a>

###### [](#mvc-view-jsp-formtaglib-textareatag)文本框标签



这个标签解析成HTML中的`textarea`标签：

```html
<tr>
    <td>Notes:</td>
    <td><form:textarea path="notes" rows="3" cols="20"/></td>
    <td><form:errors path="notes"/></td>
</tr>
```

<a id="mvc-view-jsp-formtaglib-hiddeninputtag"></a>

###### [](#mvc-view-jsp-formtaglib-hiddeninputtag)隐藏标签

`hidden`标签解析为HTML的`hidden`，用在`input` 标签中用于暗中绑定值，目的很明显就是隐藏，如下

```html
<form:hidden path="house"/>
```

如果我们选择`house`值作为隐藏域提交, HTML长这样:

```html
<input name="house" type="hidden" value="Gryffindor"/>
```

<a id="mvc-view-jsp-formtaglib-errorstag"></a>

###### [](#mvc-view-jsp-formtaglib-errorstag)错误标签

这个标签会在HTML的 `span`标签中展示错误，它提供对在控制器中创建的错误的访问，或对与控制器关联的任何验证程序创建的出错信息进行显示。

假设我们希望在提交表单后显示 `firstName` 和 `lastName`字段的所有错误信息，我们有一个验证器的实例的 `User` 类称为`UserValidator`。如下例所示：

```java
public class UserValidator implements Validator {

    public boolean supports(Class candidate) {
        return User.class.isAssignableFrom(candidate);
    }

    public void validate(Object obj, Errors errors) {
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "firstName", "required", "Field is required.");
        ValidationUtils.rejectIfEmptyOrWhitespace(errors, "lastName", "required", "Field is required.");
    }
}
```

这个 `form.jsp`看起来是这样的:

```html
<form:form>
    <table>
        <tr>
            <td>First Name:</td>
            <td><form:input path="firstName"/></td>
            <%-- Show errors for firstName field --%>
            <td><form:errors path="firstName"/></td>
        </tr>

        <tr>
            <td>Last Name:</td>
            <td><form:input path="lastName"/></td>
            <%-- Show errors for lastName field --%>
            <td><form:errors path="lastName"/></td>
        </tr>
        <tr>
            <td colspan="3">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form:form>
```

如果我们将`firstName` 和 `lastName`的域设置空值并提交，则html看起来是这样的:

```html
<form method="POST">
    <table>
        <tr>
            <td>First Name:</td>
            <td><input name="firstName" type="text" value=""/></td>
            <%-- Associated errors to firstName field displayed --%>
            <td><span name="firstName.errors">Field is required.</span></td>
        </tr>

        <tr>
            <td>Last Name:</td>
            <td><input name="lastName" type="text" value=""/></td>
            <%-- Associated errors to lastName field displayed --%>
            <td><span name="lastName.errors">Field is required.</span></td>
        </tr>
        <tr>
            <td colspan="3">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form>
```

如果我们要显示给定页面的整个错误列表，该怎么办？下面的示例显示了错误标记还支持一些基本的通用功能

*   `path="*"`: 展示所有的错误.

*   `path="lastName"`: 展示`lastName`域的所有错误

*   如果 `path` 被省略，只会显示当前对象的错误。


下面的示例将显示页面顶部的错误列表，后跟字段旁边的特定于字段的错误：

```html
<form:form>
    <form:errors path="*" cssClass="errorBox"/>
    <table>
        <tr>
            <td>First Name:</td>
            <td><form:input path="firstName"/></td>
            <td><form:errors path="firstName"/></td>
        </tr>
        <tr>
            <td>Last Name:</td>
            <td><form:input path="lastName"/></td>
            <td><form:errors path="lastName"/></td>
        </tr>
        <tr>
            <td colspan="3">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form:form>
```

html看起来是这样的：

```html
<form method="POST">
    <span name="*.errors" class="errorBox">Field is required.<br/>Field is required.</span>
    <table>
        <tr>
            <td>First Name:</td>
            <td><input name="firstName" type="text" value=""/></td>
            <td><span name="firstName.errors">Field is required.</span></td>
        </tr>

        <tr>
            <td>Last Name:</td>
            <td><input name="lastName" type="text" value=""/></td>
            <td><span name="lastName.errors">Field is required.</span></td>
        </tr>
        <tr>
            <td colspan="3">
                <input type="submit" value="Save Changes"/>
            </td>
        </tr>
    </table>
</form>
```

The `spring-form.tld` tag library descriptor (TLD) is included in the `spring-webmvc.jar`. For a comprehensive reference on individual tags, browse the [API reference](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/tags/form/package-summary.html#package.description) or see the tag library description.

<a id="mvc-rest-method-conversion"></a>

###### [](#mvc-rest-method-conversion)HTTP方法转换

REST的一个关键原则是使用统一的接口。这意味着所有资源(URL)都可以使用相同的四种HTTP方法进行操作GET, PUT, POST,和 DELETE。对于每个方法，HTTP规范都定义了精确的语义。例如， GET应该始终是一个安全的操作，这意味着它对服务器的数据没有任何影响。而PUT或DELETE应该是幂等的，这意味着可以反复重复这些操作，其最终结果应该是相同的。虽然HTTP定义了这四种方法，但是HTML只支持两个：GET和POST， 幸运的是，有两种可能的解决方法：1，可以使用JavaScript来执行PUT或DELETE。或者2，简单地用“real”的方式作为附加参数(作为HTML表单中的隐藏输入字段)进行POST。后者是使用Spring的`HiddenHttpMethodFilter`做的。 这个过滤器是一个简单的Servlet过滤器，因此它可以与任何Web框架(不仅仅是Spring MVC)结合使用，只需将此筛选器添加到 web.xml,并将具有隐藏域`_method`参数转换为相应的HTTP方法请求。

为了支持HTTP方法转换，Spring MVC表单标签被重新设计来支持设置HTTP方法。例如，更新后的Petclinic示例有以下判断：

```html
<form:form method="delete">
    <p class="submit"><input type="submit" value="Delete Pet"/></p>
</form:form>
```

实际上它就是一个HTTP POST，DELETE方法只是隐藏在请求参数中的假正经方法而已，这个DELETE将被定义在web.xml的 `HiddenHttpMethodFilter`来处理,如以下示例所示：

```xml
<filter>
    <filter-name>httpMethodFilter</filter-name>
    <filter-class>org.springframework.web.filter.HiddenHttpMethodFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>httpMethodFilter</filter-name>
    <servlet-name>petclinic</servlet-name>
</filter-mapping>
```

以下示例显示了相应的`@Controller`方法：

```java
@RequestMapping(method = RequestMethod.DELETE)
public String deletePet(@PathVariable int ownerId, @PathVariable int petId) {
    this.clinic.deletePet(petId);
    return "redirect:/owners/" + ownerId;
}
```

<a id="mvc-view-jsp-formtaglib-html5"></a>

###### [](#mvc-view-jsp-formtaglib-html5)HTML5标签

表单标签库允许输入动态属性，这意味着您可以输入任何HTML5的特定属性。

表单`input`标签支持输入文本以外的类型属性。 他允许HTML5定义输入类型，例如`email`, `date`,`range`等。 请注意，因为text是默认类型，因此不需要输入`type='text'`

<a id="mvc-view-tiles"></a>

#### [](#mvc-view-tiles)1.9.6. Tiles

Spring Web应用还可以集成Tiles，就像其它视图技术一样。下面将描述怎样集成。

本节重点介绍Spring在`org.springframework.web.servlet.view.tiles3`包中对Tiles版本3的支持。

<a id="mvc-view-tiles-dependencies"></a>

##### [](#mvc-view-tiles-dependencies)依赖

为了能够使用Tiles，您必须在Tiles 3.0.1或更高版本上添加依赖项及其对项目的[传递依赖性](https://tiles.apache.org/framework/dependency-management.html)。

<a id="mvc-view-tiles-integrate"></a>

##### [](#mvc-view-tiles-integrate)配置

为了能够使用Tiles，您必须使用包含定义的文件对其进行配置（有关定义和其他Tiles概念的基本信息，请参阅[http://tiles.apache.org](https://tiles.apache.org)）。 在Spring中，这是通过使用 `TilesConfigurer`完成的。 以下示例`ApplicationContext`配置显示了如何执行此操作：

```xml
<bean id="tilesConfigurer" class="org.springframework.web.servlet.view.tiles3.TilesConfigurer">
    <property name="definitions">
        <list>
            <value>/WEB-INF/defs/general.xml</value>
            <value>/WEB-INF/defs/widgets.xml</value>
            <value>/WEB-INF/defs/administrator.xml</value>
            <value>/WEB-INF/defs/customer.xml</value>
            <value>/WEB-INF/defs/templates.xml</value>
        </list>
    </property>
</bean>
```

这里的Tiles定义了五个文件，都位于`WEB-INF/defs` 文件夹中。在初始化`WebApplicationContext`时 ，文件将被加载，定义工厂将被初始化。完成此操作之后，在Spring Web应用程序中，定义文件中包含的Tiles可以用作视图。 之后Spring使用Tiles与使用其他视图是一样的：通过`ViewResolver`解析，`ViewResolver`可以选择`UrlBasedViewResolver`或`ResourceBundleViewResolver`。

您可以通过添加下划线然后添加区域设置来指定特定于区域设置的Tiles定义，如以下示例所示：

```xml
<bean id="tilesConfigurer" class="org.springframework.web.servlet.view.tiles3.TilesConfigurer">
    <property name="definitions">
        <list>
            <value>/WEB-INF/defs/tiles.xml</value>
            <value>/WEB-INF/defs/tiles_fr_FR.xml</value>
        </list>
    </property>
</bean>
```

使用上述配置，`tiles_fr_FR.xml`用于具有 `fr_FR`语言环境的请求，默认情况下使用`tiles.xml`。

由于下划线用于表示区域设置，因此我们建议不要在Tiles定义的文件名中使用它们。

<a id="mvc-view-tiles-url"></a>

###### [](#mvc-view-tiles-url)`UrlBasedViewResolver`

`UrlBasedViewResolver`对给定的`viewClass`进行实例化，即会解析所有的视图。 以下bean定义了`UrlBasedViewResolver`：

```xml
<bean id="viewResolver" class="org.springframework.web.servlet.view.UrlBasedViewResolver">
    <property name="viewClass" value="org.springframework.web.servlet.view.tiles3.TilesView"/>
</bean>
```

<a id="mvc-view-tiles-resource"></a>

###### [](#mvc-view-tiles-resource)`ResourceBundleViewResolver`

`ResourceBundleViewResolver`必须提供一个包含viewnames和viewclasses的属性文件。以下示例显示了`ResourceBundleViewResolver`的bean定义以及相应的视图名称和视图类（取自Pet Clinic示例）：

```xml
<bean id="viewResolver" class="org.springframework.web.servlet.view.ResourceBundleViewResolver">
    <property name="basename" value="views"/>
</bean>

...
welcomeView.(class)=org.springframework.web.servlet.view.tiles3.TilesView
welcomeView.url=welcome (this is the name of a Tiles definition)

vetsView.(class)=org.springframework.web.servlet.view.tiles3.TilesView
vetsView.url=vetsView (again, this is the name of a Tiles definition)

findOwnersForm.(class)=org.springframework.web.servlet.view.JstlView
findOwnersForm.url=/WEB-INF/jsp/findOwners.jsp
...
```

使用`ResourceBundleViewResolver`时，可以轻松混合使用不同的视图技术。

请注意， `TilesView` 类支持JSTL（JSP标准标记库）。

<a id="mvc-view-tiles-preparer"></a>

###### [](#mvc-view-tiles-preparer)`SimpleSpringPreparerFactory` 和 `SpringBeanPreparerFactory`

作为一个高级功能，Spring还支持两个特殊的Tiles `PreparerFactory`实现，有关如何在Tiles定义文件中使用`ViewPreparer`引用的详细信息，请参阅Tiles文档。

您可以指定SimpleSpringPreparerFactory以基于指定的preparer类自动装配ViewPreparer实例，应用Spring的容器回调以及应用已配置的Spring BeanPostProcessors。 如果已激活Spring的上下文范围注释配置，则会自动检测并应用ViewPreparer类中的注释。 请注意，这需要Tiles定义文件中的preparer类，作为默认的PreparerFactory。

您可以指定`SpringBeanPreparerFactory`来操作指定的preparer名称（而不是类），从DispatcherServlet的应用程序上下文中获取相应的Spring bean。在这种情况下，完整的bean创建过程控制着Spring应用程序上下文，允许使用显式依赖项注入配置，作用域bean等。 请注意，您需要为每个preparer名称定义一个Spring bean定义（在Tiles定义中使用）。 以下示例显示如何在`TilesConfigurer`上定义一个 `SpringBeanPreparerFactory`属性集：

```xml
<bean id="tilesConfigurer" class="org.springframework.web.servlet.view.tiles3.TilesConfigurer">
    <property name="definitions">
        <list>
            <value>/WEB-INF/defs/general.xml</value>
            <value>/WEB-INF/defs/widgets.xml</value>
            <value>/WEB-INF/defs/administrator.xml</value>
            <value>/WEB-INF/defs/customer.xml</value>
            <value>/WEB-INF/defs/templates.xml</value>
        </list>
    </property>

    <!-- resolving preparer names as Spring bean definition names -->
    <property name="preparerFactoryClass"
            value="org.springframework.web.servlet.view.tiles3.SpringBeanPreparerFactory"/>

</bean>
```

<a id="mvc-view-feeds"></a>

#### [](#mvc-view-feeds)1.9.7. RSS 和 Atom

`AbstractAtomFeedView`和`AbstractRssFeedView`都继承自`AbstractFeedView`基类，分别用于提供Atom和RSS Feed视图。 它们基于java.net的[ROME](https://rome.dev.java.net)项目，位于`org.springframework.web.servlet.view.feed`包中。

`AbstractAtomFeedView` 要求实现`buildFeedEntries()` 方法，并可选择重写 `buildFeedMetadata()` 方法(默认实现为空).以下示例显示了如何执行此操作：

```java
public class SampleContentAtomView extends AbstractAtomFeedView {

    @Override
    protected void buildFeedMetadata(Map<String, Object> model,
            Feed feed, HttpServletRequest request) {
        // implementation omitted
    }

    @Override
    protected List<Entry> buildFeedEntries(Map<String, Object> model,
            HttpServletRequest request, HttpServletResponse response) throws Exception {
        // implementation omitted
    }

}
```

类似的要求适用于实现`AbstractRssFeedView`，如以下示例所示：

```java
public class SampleContentAtomView extends AbstractRssFeedView {

    @Override
    protected void buildFeedMetadata(Map<String, Object> model,
            Channel feed, HttpServletRequest request) {
        // implementation omitted
    }

    @Override
    protected List<Item> buildFeedItems(Map<String, Object> model,
            HttpServletRequest request, HttpServletResponse response) throws Exception {
        // implementation omitted
    }
}
```

`buildFeedItems()` 和 `buildFeedEntries()`方法在HTTP请求中传递，以防需要访问区域设置。仅为cookie或其他http头的设置传递http响应。该feed将在方法返回后自动写入响应对象。

有关创建Atom视图的示例，请参阅Alef Arendsen的Spring Team Blog [条目](https://spring.io/blog/2009/03/16/adding-an-atom-view-to-an-application-using-spring-s-rest-support).

<a id="mvc-view-document"></a>

#### [](#mvc-view-document)1.9.8. PDF 和 Excel

Spring提供了返回HTML以外的输出的方法，包括PDF和Excel电子表格。 本节介绍如何使用这些功能。

<a id="mvc-view-document-intro"></a>

##### [](#mvc-view-document-intro)文档视图简介

返回HTML页并不总是用户查看模型输出的最佳方式，Spring让开发者可以从模型数据动态生成PDF文档或Excel电子表格。该文档是视图，将从具有正确内容类型的服务器流式传输到HTML，使客户端PC能够运行其电子表格或PDF查看器应用程序以进行响应。

要使用Excel视图，需要将Apache POI库添加到类路径中。对于PDF生成，您需要添加（最好）OpenPDF库。

如果可能，您应该使用最新版本的基础文档生成库。 特别是，我们强烈建议使用OpenPDF（例如，OpenPDF 1.0.5）而不是过时的原始iText 2.1.7，因为OpenPDF是主动维护的，并修复了不受信任的PDF内容的重要漏洞。

<a id="mvc-view-document-pdf"></a>

##### [](#mvc-view-document-pdf)PDF 视图

单词列表的简单PDF视图可以扩展`org.springframework.web.servlet.view.document.AbstractPdfView`并实现`buildPdfDocument()` 方法，如以下示例所示：

```java
public class PdfWordList extends AbstractPdfView {

    protected void buildPdfDocument(Map<String, Object> model, Document doc, PdfWriter writer,
            HttpServletRequest request, HttpServletResponse response) throws Exception {

        List<String> words = (List<String>) model.get("wordList");
        for (String word : words) {
            doc.add(new Paragraph(word));
        }
    }
}
```

控制器可以从外部视图定义（通过名称引用它）返回这样的视图，也可以从处理程序方法返回`View`实例。

<a id="mvc-view-document-excel"></a>

##### [](#mvc-view-document-excel)Excel 视图

从Spring Framework 4.2开始，`org.springframework.web.servlet.view.document.AbstractXlsView` 作为Excel视图的基类提供。 它基于Apache POI，具有专门的子类（`AbstractXlsxStreamingView`和`AbstractExcelView`），取代了过时的`AbstractXlsxView`类。

编程模型类似于 `AbstractPdfView`，`buildExcelDocument()`作为核心模板方法，控制器能够从外部定义（通过名称）返回这样的视图，或者从处理程序方法返回`View`实例。

<a id="mvc-view-jackson"></a>

#### [](#mvc-view-jackson)1.9.9. Jackson

[Same as in Spring WebFlux](web-reactive.html#webflux-view-httpmessagewriter)

Spring为Jackson JSON库提供支持。

<a id="mvc-view-json-mapping"></a>

##### [](#mvc-view-json-mapping)基于Jackson 的JSON 视图

[Same as in Spring WebFlux](web-reactive.html#webflux-view-httpmessagewriter)

`MappingJackson2JsonView`使用Jackson库的`ObjectMapper`将响应内容呈现为JSON。 默认情况下，模型映射的全部内容（特定于框架的类除外）都编码为JSON。 对于需要过滤Map内容的情况，您可以使用`modelKeys`属性指定要编码的特定模型属性集。 您还可以使用 `extractValueFromSingleKeyModel`属性将single-key模型中的值直接提取和序列化，而不是作为模型属性的映射。

您可以使用Jackson提供的注解根据需要自定义JSON映射。 当您需要进一步控制时，可以通过 `ObjectMapper`属性注入自定义`ObjectMapper`，以用于需要为特定类型提供自定义JSON序列化程序和反序列化程序的情况。

<a id="mvc-view-xml-mapping"></a>

##### [](#mvc-view-xml-mapping)基于Jackson的XML视图

[Same as in Spring WebFlux](web-reactive.html#webflux-view-httpmessagewriter)

`MappingJackson2XmlView`使用[Jackson XML扩展的](https://github.com/FasterXML/jackson-dataformat-xml) `XmlMapper`将响应内容呈现为XML。 如果模型包含多个条目，则应使用`modelKey`bean属性显式设置要序列化的对象。 如果模型包含单个条目，则会自动序列化。

您可以使用JAXB或Jackson提供的注解根据需要自定义XML映射。 当您需要进一步控制时，可以通过`ObjectMapper`属性注入自定义`XmlMapper`，以便自定义XML需要为特定类型提供序列化程序和反序列化程序。

<a id="mvc-view-xml-marshalling"></a>

#### [](#mvc-view-xml-marshalling)1.9.10. XML编组

`MarshallingView`使用XML `Marshaller`（在`org.springframework.oxm`包中定义）将响应内容呈现为XML。 您可以使用 `MarshallingView` 实例的 `modelKey` bean属性显式设置要编组的对象。 或者，视图会迭代所有模型属性，并封送`Marshaller`支持的第一种类型。 有关`org.springframework.oxm`包中功能的更多信息，请参阅使用 [Marshalling XML using O/X Mappers](data-access.html#oxm)。

<a id="mvc-view-xslt"></a>

#### [](#mvc-view-xslt)1.9.11. XSLT视图

XSLT是一个用于转换XML的语言,能够在web的视图技术中使用.如果应用需要处理XML（或者将模型转换为XML），那么XSLT是一个很适合的视图技术。以下部分显示如何将XML文档生成为模型数据，并在Spring Web MVC应用程序中使用XSLT进行转换。

这个例子是一个简单的Spring应用程序，它在Controller中创建一个单词列表并将它们添加到模型映射中。该映射与使用的XSLT视图名称一起返回。有关Spring Web MVC控制器接口的详细信息， 请参阅[带注解的控制器](#mvc-controller)。 XSLT控制器将单词列表转换为准备转换的简单XML文档。

<a id="mvc-view-xslt-beandefs"></a>

##### [](#mvc-view-xslt-beandefs)Beans

Configuration配置是Spring应用程序的标配，MVC配置必须定义`XsltViewResolver` bean和常规MVC注解配置，以下示例显示了如何执行此操作：

```java
@EnableWebMvc
@ComponentScan
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public XsltViewResolver xsltViewResolver() {
        XsltViewResolver viewResolver = new XsltViewResolver();
        viewResolver.setPrefix("/WEB-INF/xsl/");
        viewResolver.setSuffix(".xslt");
        return viewResolver;
    }
}
```

<a id="mvc-view-xslt-controllercode"></a>

##### [](#mvc-view-xslt-controllercode)控制器

并且我们需要一个控制器，用来处理单词的生成逻辑。

控制器逻辑封装在`@Controller` 类中，处理程序方法定义如下：

```java
@Controller
public class XsltController {

    @RequestMapping("/")
    public String home(Model model) throws Exception {
        Document document = DocumentBuilderFactory.newInstance().newDocumentBuilder().newDocument();
        Element root = document.createElement("wordList");

        List<String> words = Arrays.asList("Hello", "Spring", "Framework");
        for (String word : words) {
            Element wordNode = document.createElement("word");
            Text textNode = document.createTextNode(word);
            wordNode.appendChild(textNode);
            root.appendChild(wordNode);
        }

        model.addAttribute("wordList", root);
        return "home";
    }
}
```

到目前为止，我们只创建了一个DOM文档并将其添加到模型映射中。 请注意，您还可以将XML文件作为`Resource` 加载，并使用它而不是自定义DOM文档。

当然，有软件包可以自动 'domify'对象图，在Spring中，您可以完全灵活地以您选择的任何方式从模型中创建DOM。这可以防止XML在模型数据的结构中扮演太大的角色，这在使用工具管理DOM化过程时是一种危险。。

<a id="mvc-view-xslt-transforming"></a>

##### [](#mvc-view-xslt-transforming)转换

最后, `XsltViewResolver` 将解析“home” XSLT 模板文件，并将DOM文档合并到其中以生成所需视图。例如`XsltViewResolver`配置所示，XSLT模板在`WEB-INF/xsl`目录中的`war`文件中， 并以`xslt`文件扩展名结束。

以下示例显示了XSLT转换：

```xml
<?xml version="1.0" encoding="utf-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">

    <xsl:output method="html" omit-xml-declaration="yes"/>

    <xsl:template match="/">
        <html>
            <head><title>Hello!</title></head>
            <body>
                <h1>My First Words</h1>
                <ul>
                    <xsl:apply-templates/>
                </ul>
            </body>
        </html>
    </xsl:template>

    <xsl:template match="word">
        <li><xsl:value-of select="."/></li>
    </xsl:template>

</xsl:stylesheet>
```

上述转换呈现为以下HTML：

```xml
<html>
    <head>
        <META http-equiv="Content-Type" content="text/html; charset=UTF-8">
        <title>Hello!</title>
    </head>
    <body>
        <h1>My First Words</h1>
        <ul>
            <li>Hello</li>
            <li>Spring</li>
            <li>Framework</li>
        </ul>
    </body>
</html>
```

<a id="mvc-config"></a>

### [](#mvc-config)1.10. MVC 配置

[Same as in Spring WebFlux](web-reactive.html#webflux-config)

MVC Java配置和MVC命名空间提供了适用于大多数应用程序的默认配置以及配置API来对其进行自定义。

有关配置API中没有的高级自定义设置请参阅[高级 Java 配置](#mvc-config-advanced-java) 和 [高级 XML 配置](#mvc-config-advanced-xml).

您无需了解MVC Java配置和MVC命名空间创建的基础bean。 如果您想了解更多信息，请参阅特殊Bean类型和Web MVC配置。[特殊Bean类型](#mvc-servlet-special-bean-types) 和 [Web MVC配置](#mvc-servlet-config).

<a id="mvc-config-enable"></a>

#### [](#mvc-config-enable)1.10.1. 启用 MVC 配置

[Same as in Spring WebFlux](web-reactive.html#webflux-config-enable)

在Java配置中，您可以使用`@EnableWebMvc` 注解启用MVC配置，如以下示例所示:

    @Configuration
    @EnableWebMvc
    public class WebConfig {
    }

在XML配置中，您可以使用`<mvc:annotation-driven>` 元素来启用MVC配置，如以下示例所示:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven/>

</beans>
```

前面的示例注册了许多Spring MVC基础结构bean，并适应类路径上可用的依赖项（例如，JSON，XML等的有效负载转换器）。

<a id="mvc-config-customize"></a>

#### [](#mvc-config-customize)1.10.2. MVC 配置 API

[Same as in Spring WebFlux](web-reactive.html#webflux-config-customize)

在Java配置中，您可以实现`WebMvcConfigurer`接口，如以下示例所示:

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // Implement configuration methods...
}
```

在XML中，您可以检查`<mvc:annotation-driven/>`的属性和子元素。 您可以查看[Spring MVC XML schema](https://schema.spring.io/mvc/spring-mvc.xsd) 或使用IDE的代码完成功能来发现可用的属性和子元素。

<a id="mvc-config-conversion"></a>

#### [](#mvc-config-conversion)1.10.3. 类型转换

[Same as in Spring WebFlux](web-reactive.html#webflux-config-conversion)

数字的 `Number`类型和日期`Date` 类型的格式化是默认安装了的，包括`@NumberFormat`注解和`@DateTimeFormat`注解，如果类路径中存在Joda-Time，则还会安装对Joda-Time格式库的完全支持。

在Java配置中，您可以注册自定义格式化程序和转换器，如以下示例所示：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        // ...
    }
}
```

以下示例显示如何在XML中实现相同的配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven conversion-service="conversionService"/>

    <bean id="conversionService"
            class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
        <property name="converters">
            <set>
                <bean class="org.example.MyConverter"/>
            </set>
        </property>
        <property name="formatters">
            <set>
                <bean class="org.example.MyFormatter"/>
                <bean class="org.example.MyAnnotationFormatterFactory"/>
            </set>
        </property>
        <property name="formatterRegistrars">
            <set>
                <bean class="org.example.MyFormatterRegistrar"/>
            </set>
        </property>
    </bean>

</beans>
```

有关何时使用FormatterRegistrar实现的更多信息，请参阅 [`FormatterRegistrar` SPI](core.html#format-FormatterRegistrar-SPI)和 `FormattingConversionServiceFactoryBean`。

<a id="mvc-config-validation"></a>

#### [](#mvc-config-validation)1.10.4. 验证

[Same as in Spring WebFlux](web-reactive.html#webflux-config-validation)

默认情况下，如果类路径上存在[Bean Validation](core.html#validation-beanvalidation-overview)(例如Hibernate Validator），则 `LocalValidatorFactoryBean`将注册为全局[Validator](core.html#validator) 。 以便与 `@Valid` 和 `Validated` 一起使用并在控制器方法参数上进行验证。

在Java配置中，您可以自定义全局`Validator`实例，如以下示例所示：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public Validator getValidator(); {
        // ...
    }
}
```

以下示例显示如何在XML中实现相同的配置:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven validator="globalValidator"/>

</beans>
```

请注意，您还可以在本地注册`Validator`实现，如以下示例所示：

```java
@Controller
public class MyController {

    @InitBinder
    protected void initBinder(WebDataBinder binder) {
        binder.addValidators(new FooValidator());
    }

}
```

如果需要在某处注入`LocalValidatorFactoryBean`，请创建一个bean并使用`@Primary`标记它，以避免与MVC配置中声明的那个冲突。

<a id="mvc-config-interceptors"></a>

#### [](#mvc-config-interceptors)1.10.5. 拦截器

在Java配置中，注册拦截器应用于传入的请求。如以下示例所示：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LocaleChangeInterceptor());
        registry.addInterceptor(new ThemeChangeInterceptor()).addPathPatterns("/**").excludePathPatterns("/admin/**");
        registry.addInterceptor(new SecurityInterceptor()).addPathPatterns("/secure/*");
    }
}
```

以下示例显示如何在XML中实现相同的配置：

```xml
<mvc:interceptors>
    <bean class="org.springframework.web.servlet.i18n.LocaleChangeInterceptor"/>
    <mvc:interceptor>
        <mvc:mapping path="/**"/>
        <mvc:exclude-mapping path="/admin/**"/>
        <bean class="org.springframework.web.servlet.theme.ThemeChangeInterceptor"/>
    </mvc:interceptor>
    <mvc:interceptor>
        <mvc:mapping path="/secure/*"/>
        <bean class="org.example.SecurityInterceptor"/>
    </mvc:interceptor>
</mvc:interceptors>
```

<a id="mvc-config-content-negotiation"></a>

#### [](#mvc-config-content-negotiation)1.10.6. 内容类型

[Same as in Spring WebFlux](web-reactive.html#webflux-config-content-negotiation)

您可以配置Spring MVC如何根据请求确定所请求的媒体类型（例如，`Accept`头，URL路径扩展，查询参数等）。

默认情况下，首先检查URL路径扩展 - 将 `json`, `xml`, `rss`, 和 `atom`注册为已知扩展（取决于类路径依赖性）。 第二个检查 `Accept`头。

将这些默认值更改为只接受`Accept`头，并且如果必须使用基于内容类型解析，请考虑路径扩展上的查询参数策略。 有关更多详细信息，请参阅 [后缀匹配](#mvc-ann-requestmapping-suffix-pattern-match) and [后缀匹配以及RFD](#mvc-ann-requestmapping-rfd)。

在Java配置中，您可以自定义请求的内容类型解析，如以下示例所示：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer.mediaType("json", MediaType.APPLICATION_JSON);
        configurer.mediaType("xml", MediaType.APPLICATION_XML);
    }
}
```

以下示例显示如何在XML中实现相同的配置：

```xml
<mvc:annotation-driven content-negotiation-manager="contentNegotiationManager"/>

<bean id="contentNegotiationManager" class="org.springframework.web.accept.ContentNegotiationManagerFactoryBean">
    <property name="mediaTypes">
        <value>
            json=application/json
            xml=application/xml
        </value>
    </property>
</bean>
```

<a id="mvc-config-message-converters"></a>

#### [](#mvc-config-message-converters)1.10.7. 消息转换

[Same as in Spring WebFlux](web-reactive.html#webflux-config-message-codecs)

使用MVC Java编程配置方式时，如果想替换Spring MVC提供的默认转换器，完全定制自己的`HttpMessageConverter` ，这可以通过覆写[`configureMessageConverters()`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurer.html#configureMessageConverters-java.util.List-)方法来实现。 如果只是想自定义，或者想在默认转换器之外再添加其他的转换器，那么可以通过覆写[`extendMessageConverters()`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/config/annotation/WebMvcConfigurer.html#extendMessageConverters-java.util.List-)方法来实现。

以下示例使用自定义的`ObjectMapper`而不是默认的`ObjectMapper`添加XML和Jackson JSON转换器：

```java
@Configuration
@EnableWebMvc
public class WebConfiguration implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder()
                .indentOutput(true)
                .dateFormat(new SimpleDateFormat("yyyy-MM-dd"))
                .modulesToInstall(new ParameterNamesModule());
        converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
        converters.add(new MappingJackson2XmlHttpMessageConverter(builder.createXmlMapper(true).build()));
    }
}
```

在上面的例子中，[`Jackson2ObjectMapperBuilder`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/http/converter/json/Jackson2ObjectMapperBuilder.html) 用于为 `MappingJackson2HttpMessageConverter` 和`MappingJackson2XmlHttpMessageConverter` 转换器创建公共的配置，比如启用tab缩进、定制的日期格式，并注册了模块 [`jackson-module-parameter-names`](https://github.com/FasterXML/jackson-module-parameter-names)用于获取参数名（Java 8新增的特性）。

该builder会使用以下的默认属性对Jackson进行配置

*   [`DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES`](https://fasterxml.github.io/jackson-databind/javadoc/2.6/com/fasterxml/jackson/databind/DeserializationFeature.html#FAIL_ON_UNKNOWN_PROPERTIES) is disabled.

*   [`MapperFeature.DEFAULT_VIEW_INCLUSION`](https://fasterxml.github.io/jackson-databind/javadoc/2.6/com/fasterxml/jackson/databind/MapperFeature.html#DEFAULT_VIEW_INCLUSION) is disabled.


同时，如果检测到在classpath路径下存在这些模块，该builder也会自动地注册它们。

*   [jackson-datatype-jdk7](https://github.com/FasterXML/jackson-datatype-jdk7): Support for Java 7 types, such as `java.nio.file.Path`.

*   [jackson-datatype-joda](https://github.com/FasterXML/jackson-datatype-joda): Support for Joda-Time types.

*   [jackson-datatype-jsr310](https://github.com/FasterXML/jackson-datatype-jsr310): Support for Java 8 Date and Time API types.

*   [jackson-datatype-jdk8](https://github.com/FasterXML/jackson-datatype-jdk8): Support for other Java 8 types, such as `Optional`.


除了[`jackson-dataformat-xml`](https://search.maven.org/#search%7Cga%7C1%7Ca%3A%22jackson-dataformat-xml%22) 之外，要启用Jackson XML的tab缩进还需要[`woodstox-core-asl`](https://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22org.codehaus.woodstox%22%20AND%20a%3A%22woodstox-core-asl%22)依赖。

还有其他有用的Jackson模块可以使用

*   [jackson-datatype-money](https://github.com/zalando/jackson-datatype-money): 提供了对`javax.money` 类型的支持（非官方模块）

*   [jackson-datatype-hibernate](https://github.com/FasterXML/jackson-datatype-hibernate):提供了Hibernate相关的类型和属性支持（包含懒加载aspects）


以下示例显示如何在XML中实现相同的配置：

```xml
<mvc:annotation-driven>
    <mvc:message-converters>
        <bean class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
            <property name="objectMapper" ref="objectMapper"/>
        </bean>
        <bean class="org.springframework.http.converter.xml.MappingJackson2XmlHttpMessageConverter">
            <property name="objectMapper" ref="xmlMapper"/>
        </bean>
    </mvc:message-converters>
</mvc:annotation-driven>

<bean id="objectMapper" class="org.springframework.http.converter.json.Jackson2ObjectMapperFactoryBean"
      p:indentOutput="true"
      p:simpleDateFormat="yyyy-MM-dd"
      p:modulesToInstall="com.fasterxml.jackson.module.paramnames.ParameterNamesModule"/>

<bean id="xmlMapper" parent="objectMapper" p:createXmlMapper="true"/>
```

<a id="mvc-config-view-controller"></a>

#### [](#mvc-config-view-controller)1.10.8. 视图控制器

以下的一段代码相当于定义`ParameterizableViewController` 视图控制器的快捷方式，该控制器会立即将请求转发（forwards）给视图。请确保仅在以下情景下才使用这个类：当控制器除了将视图渲染到响应中外不需要执行任何逻辑时。

以下Java配置示例将对 `/`的请求转发给名为`home`的视图:

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/").setViewName("home");
    }
}
```

以下示例与前面的示例实现相同的功能，但使用XML，通过使用`<mvc:view-controller>`元素:

    <mvc:view-controller path="/" view-name="home"/>

<a id="mvc-config-view-resolvers"></a>

#### [](#mvc-config-view-resolvers)1.10.9. 视图解析器

[Same as in Spring WebFlux](web-reactive.html#webflux-config-view-resolvers)

MVC提供的配置简化了视图解析器的注册工作

以下Java配置示例使用JSP和Jackson作为JSON呈现的默认视图来配置内容协商视图解析：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.enableContentNegotiation(new MappingJackson2JsonView());
        registry.jsp();
    }
}
```

以下示例显示如何在XML中实现相同的配置：

```xml
<mvc:view-resolvers>
    <mvc:content-negotiation>
        <mvc:default-views>
            <bean class="org.springframework.web.servlet.view.json.MappingJackson2JsonView"/>
        </mvc:default-views>
    </mvc:content-negotiation>
    <mvc:jsp/>
</mvc:view-resolvers>
```

但请注意，FreeMarker，Tiles，Groovy Markup和脚本模板也需要配置底层视图技术。

MVC名称空间提供专用元素。 以下示例适用于FreeMarker：

```xml
<mvc:view-resolvers>
    <mvc:content-negotiation>
        <mvc:default-views>
            <bean class="org.springframework.web.servlet.view.json.MappingJackson2JsonView"/>
        </mvc:default-views>
    </mvc:content-negotiation>
    <mvc:freemarker cache="false"/>
</mvc:view-resolvers>

<mvc:freemarker-configurer>
    <mvc:template-loader-path location="/freemarker"/>
</mvc:freemarker-configurer>
```

在Java配置中，您可以添加相应的`Configurer` bean，如以下示例所示：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.enableContentNegotiation(new MappingJackson2JsonView());
        registry.freeMarker().cache(false);
    }

    @Bean
    public FreeMarkerConfigurer freeMarkerConfigurer() {
        FreeMarkerConfigurer configurer = new FreeMarkerConfigurer();
        configurer.setTemplateLoaderPath("/freemarker");
        return configurer;
    }
}
```

<a id="mvc-config-static-resources"></a>

#### [](#mvc-config-static-resources)1.10.10. 静态资源

[Same as in Spring WebFlux](web-reactive.html#webflux-config-static-resources)

此选项提供了一种从 [`资源`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/core/io/Resource.html)库位置列表中使用静态资源的便捷方法

在下面的示例中，给定以 `/resources`开头的请求，相对路径用于在Web应用程序根目录下或在或在`/static`下的类路径上查找和提供相对于`/public`的静态资源。 资源的有效期为1年，以确保最大程度地使用浏览器缓存，并减少浏览器发出的HTTP请求。如果返回 `304`状态代码，`Last-Modified` 头也会计算到。

以下清单显示了如何使用Java配置执行此操作:

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
            .addResourceLocations("/public", "classpath:/static/")
            .setCachePeriod(31556926);
    }
}
```

以下示例显示如何在XML中实现相同的配置：

    <mvc:resources mapping="/resources/**"
        location="/public, classpath:/static/"
        cache-period="31556926" />

查看 [静态资源的HTTP缓存支持](#mvc-caching-static-resources).

资源处理还支持一系列 [`ResourceResolver`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/resource/ResourceResolver.html) 实现 和 [`ResourceTransformer`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/resource/ResourceTransformer.html) 实现, 可用于创建用于使用优化资源的工具

`VersionResourceResolver`可用于基于内容、固定应用程序版本或其他的MD5哈希计算的版本化资源url。`ContentVersionStrategy`(MD5 hash)方法是一个很好的选择， 有一些值得注意的例外，例如与模块加载器一起使用的JavaScript资源。

以下示例显示如何在Java配置中使用 `VersionResourceResolver`：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
                .addResourceLocations("/public/")
                .resourceChain(true)
                .addResolver(new VersionResourceResolver().addContentVersionStrategy("/**"));
    }
}
```

以下示例显示如何在XML中实现相同的配置：

```xml
<mvc:resources mapping="/resources/**" location="/public/">
    <mvc:resource-chain>
        <mvc:resource-cache/>
        <mvc:resolvers>
            <mvc:version-resolver>
                <mvc:content-version-strategy patterns="/**"/>
            </mvc:version-resolver>
        </mvc:resolvers>
    </mvc:resource-chain>
</mvc:resources>
```

您可以使用`ResourceUrlProvider`来重写URL并应用完整的解析器和转换器链，例如插入版本。MVC配置提供了 `ResourceUrlProvider` bean，因此可以将其注入到其他用户。 您还可以使用`ResourceUrlEncodingFilter` 的Thymeleaf、jsp、FreeMarker和其他依赖于`HttpServletResponse#encodeURL`的URL标记来做重写转换。

请注意，当同时使用`EncodedResourceResolver`（例如，用于提供gzipped或brotli编码的资源）和`VersionedResourceResolver`时，必须按此顺序注册它们。 这可确保始终基于未编码的文件可靠地计算基于内容的版本。

[WebJars](http://www.webjars.org/documentation)也支持使用`WebJarsResourceResolver`和自动注册，当 `org.webjars:webjars-locator`存在于类路径中时。解析器可以重写URL来包含jar的版本，也可以与传入的URL匹配，而不需要版本 。 例如， `/jquery/jquery.min.js` 到 `/jquery/1.2.0/jquery.min.js`。

<a id="mvc-default-servlet-handler"></a>

#### [](#mvc-default-servlet-handler)1.10.11. 默认 Servlet

这些配置允许将`DispatcherServlet`映射到`/`路径（也即覆盖了容器默认Servlet的映射），但依然保留容器默认的Servlet以处理静态资源的请求。这可以通过配置一个URL映射到 `/**` 的处理器`DefaultServletHttpRequestHandler`来实现，并且该处理器在其他所有URL映射关系中优先级应该是最低的。

该处理器会将所有请求转发（forward）到默认的Servlet，因此需要保证它在所有URL处理器映射`HandlerMappings`的最后。如果你是通过`<mvc:annotation-driven>`的方式进行配置， 或自定义 `HandlerMapping` 实例，那么需要确保该处理器`order`属性的值比`DefaultServletHttpRequestHandler`的次序值`Integer.MAX_VALUE`小。

以下示例显示如何使用默认设置启用该功能：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
```

以下示例显示如何在XML中实现相同的配置：

    <mvc:default-servlet-handler/>

不过需要注意，覆写了`/`的Servlet映射后，默认Servlet的`RequestDispatcher`就必须通过名字而非路径来取得了。 `DefaultServletHttpRequestHandler`会尝试在容器初始化的时候自动检测默认Servlet， 这里它使用的是一份主流Servlet容器（包括Tomcat, Jetty, GlassFish, JBoss, Resin, WebLogic, and WebSphere）已知的名称列表。如果默认Servlet被配置了一个其他的名字，或者使用了一个列表里未提供默认Servlet名称的容器，那么默认Servlet的名称必须被显式指定，正如下面代码所示：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable("myCustomDefaultServlet");
    }

}
```

以下示例显示如何在XML中实现相同的配置：

```xml
<mvc:default-servlet-handler default-servlet-name="myCustomDefaultServlet"/>
```

<a id="mvc-config-path-matching"></a>

#### [](#mvc-config-path-matching)1.10.12. 路径匹配

[Same as in Spring WebFlux](web-reactive.html#webflux-config-path-matching)

您可以自定义与路径匹配和URL处理相关的选项。 有关各个选项的详细信息，请参阅 [`PathMatchConfigurer`](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/web/servlet/config/annotation/PathMatchConfigurer.html) javadoc.

以下示例显示如何在Java配置中自定义路径匹配：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer
            .setUseSuffixPatternMatch(true)
            .setUseTrailingSlashMatch(false)
            .setUseRegisteredSuffixPatternMatch(true)
            .setPathMatcher(antPathMatcher())
            .setUrlPathHelper(urlPathHelper())
            .addPathPrefix("/api",
                    HandlerTypePredicate.forAnnotation(RestController.class));
    }

    @Bean
    public UrlPathHelper urlPathHelper() {
        //...
    }

    @Bean
    public PathMatcher antPathMatcher() {
        //...
    }

}
```

以下示例显示如何在XML中实现相同的配置：

```xml
<mvc:annotation-driven>
    <mvc:path-matching
        suffix-pattern="true"
        trailing-slash="false"
        registered-suffixes-only="true"
        path-helper="pathHelper"
        path-matcher="pathMatcher"/>
</mvc:annotation-driven>

<bean id="pathHelper" class="org.example.app.MyPathHelper"/>
<bean id="pathMatcher" class="org.example.app.MyPathMatcher"/>
```

<a id="mvc-config-advanced-java"></a>

#### [](#mvc-config-advanced-java)1.10.13. 高级 Java 配置

[Same as in Spring WebFlux](web-reactive.html#webflux-config-advanced-java)

`@EnableWebMvc` 导入 `DelegatingWebMvcConfiguration`, 其中:

*   为Spring MVC应用程序提供了默认的Spring配置

*   检测到并委派到`WebMvcConfigurer`的自定义该配置


对于高级模式，请删除`@EnableWebMvc`并直接从 `DelegatingWebMvcConfiguration`继承 ，而不是实现`WebMvcConfigurer`，如以下示例所示：

```java
@Configuration
public class WebConfig extends DelegatingWebMvcConfiguration {

    // ...

}
```

可以在`WebConfig`中保留现有的方法，但现在也可以重写基类中的bean声明，并且在类路径上仍然可以有任意数量的其他`WebMvcConfigurer` 。

<a id="mvc-config-advanced-xml"></a>

#### [](#mvc-config-advanced-xml)1.10.14. 高级 XML 配置

MVC命名空间没有高级模式，如果需要自定义无法更改的bean上的属性，可以使用 `ApplicationContext`的`BeanPostProcessor`生命周期挂钩，如以下示例所示：

```java
@Component
public class MyPostProcessor implements BeanPostProcessor {

    public Object postProcessBeforeInitialization(Object bean, String name) throws BeansException {
        // ...
    }
}
```

请注意，`MyPostProcessor`需要用XML显式声明为bean，或通过 `<component-scan/>`声明检测。

<a id="mvc-http2"></a>

### [](#mvc-http2)1.11. HTTP/2

[Same as in Spring WebFlux](web-reactive.html#webflux-http2)

Servlet 4容器需要支持HTTP/2，Spring Framework 5与Servlet API 4兼容。从编程模型的角度来看，应用程序不需要特定的任何操作。 但是，存在与服务器配置相关的注意事项。 有关更多详细信息，请参阅 [HTTP/2 wiki 页面。](https://github.com/spring-projects/spring-framework/wiki/HTTP-2-support)

Servlet API确实公开了一个与HTTP/2相关的构造。 您可以使用`javax.servlet.http.PushBuilder` 主动将资源推送到客户端，并且它被支持作为 `@RequestMapping`方法的[方法参数](#mvc-ann-arguments)。