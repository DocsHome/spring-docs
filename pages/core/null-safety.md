<a id="null-safety"></a>

[](#null-safety)7\. null安全
--------------------------

尽管Java不允许使用类型系统来表示null安全，但Spring框架现在加入了`org.springframework.lang` 包，并提供了以下注解，用来声明API和字段的null特性:

*   [`@NonNull`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/lang/NonNull.html): 注解指示特定参数，返回值或字段不能为`null`（对于`@NonNullApi`和`@NonNullFields` 适用的参数和返回值不需要）

*   [`@Nullable`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/lang/Nullable.html):其中特定参数、返回值或字段可以为`null`.

*   [`@NonNullApi`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/lang/NonNullApi.html): 在包级别将非null声明为参数和返回值的默认行为

*   [`@NonNullFields`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/lang/NonNullFields.html): 在包级别将非null声明为字段的默认行为


Spring框架用到这些注解，但它们也可以用于任意基于Spring的Java项目中，用来声明null安全的API和可选的null安全字段。null特性对于泛型类型参数、varargs参数和数组元素是不受支持的， 但可能会包含在即将发布的版本中，请参阅[SPR-15942](https://jira.spring.io/browse/SPR-15942)以获取最新信息。在Spring框架发行版（包括小版本）之间。可以将 fine-tuned 声明为null。在方法体内判定类型是否为null超出了它的能力范围。

像Reactor或者Spring Data这样的库也用到了null安全的API。

<a id="use-cases"></a>

### [](#use-cases)7.1. Use cases

Spring API除了为null提供了显式的声明外，IDE还可以使用这些注解（如IDEA或Eclipse）为与null安全相关的Java开发人员提供有用的警告，用于避免运行时出现`NullPointerException`。

在Kotlin项目中也使用到Spring API的null安全特性，因为Kotlin本身支持[null安全](https://kotlinlang.org/docs/reference/null-safety.html)。[Kotlin支持文档](languages.html#kotlin-null-safety)提供了更多的详细信息。

<a id="jsr-305-meta-annotations"></a>

### [](#jsr-305-meta-annotations)7.2. JSR 305 元注解

Spring注解是被[JSR 305](https://jcp.org/en/jsr/detail?id=305)注解的元注解（一个潜在的但广泛使用的JSR）。 JSR 305元注解允许工具供应商（如IDEA或Kotlin）以通用方式提供安全支持，而无需为Spring注解提供硬编码支持。

为了使用Spring的null安全API，其实不需要也不建议在项目类路径中添加JSR 305依赖项。而只有在其代码库中使用null安全标识的项目（例如Spring基本库） 即可，应该将`com.google.code.findbugs:jsr305:3.0.2` 添加到仅用于编译的Gradle配置或Maven提供的scope中，就可以避免编译警告了。