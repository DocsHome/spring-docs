<a id="expressions"></a>

[](#expressions)4\. Spring 的表达式语言(SpEL)
---------------------------------------

Spring Expression Language（简称“SpEL”）是一种强大的表达式语言，支持在运行时查询和操作对象。语言语法类似于Unified EL，但提供了其他功能，最有名的是方法调用和基本字符串模板功能。

尽管还有其他几种Java表达式语言，例如OGNL, MVEL, and JBoss，但SpEL只是为了向Spring社区提供一种支持良好的表达式语言，开发者可以在所有用到Spring框架的产品中使用SpEL。 其语言特性是由使用Spring框架项目的需求所驱动的，包括基于Eclipse的Spring Tool Suite代码支持的需求。也就是说，SpEL基于一种抽象实现的技术API，允许在需要时集成其他表达式语言来实现功能。

虽然SpEL作为Spring产品组合中的表达式运算操作的基础，但它并不直接与Spring耦合，可以独立使用。要自包含，本章中的许多示例都使用SpEL，就像它是一种独立的表达式语言一样。 这时需要创建一些引导作用的基础实现类，，例如解释器。大多数Spring用户不需要处理这种基础实现类，只需将表达式字符串作为运算操作即可。SpEL典型用途的是集成到创建XML或基于注解的bean定义中， 例如[bean定义的表达式支持](#expressions-beandef)。

本章介绍表达式语言的功能，API及其语言语法。在一些地方，`Inventor`和`Society` 类作为表达式运算操作的目标对象。这些类声明和用于填充它们的数据列在本章末尾。

表达式语言支持以下功能:

*   文字表达

*   布尔和关系运算符

*   正则表达式

*   类表达式

*   访问属性，数组，list和maps

*   方法调用

*   关系运算符

*   声明

*   调用构造器

*   bean的引用

*   数组的构造

*   内嵌的list

*   内嵌的map

*   三元表达式

*   变量

*   用户自定义函数

*   集合映射

*   集合选择

*   模板表达式


<a id="expressions-evaluation"></a>

### [](#expressions-evaluation)4.1. 使用Spring表达式接口的表达式运算

本节介绍SpEL接口及其表达式语言的简单使用。 完整的语言参考可以在[语言参考](#expressions-language-ref)中找到。

以下代码介绍使用SpEL API运算操作文字字符串表达式`Hello World`

    ExpressionParser parser = new SpelExpressionParser();
    Expression exp = parser.parseExpression("'Hello World'"); (1)
    String message = (String) exp.getValue();

**(1)。**变量的值为“Hello World”。 `'Hello World'`.

您最有可能使用的SpEL类和接口位于`org.springframework.expression`包及其子包中，例如`spel.support`。

`ExpressionParser`接口负责解析表达式字符串。在前面的示例中，表达式字符串是由周围的单引号表示的字符串文字。接口`Expression`负责运算操作先前定义的表达式字符串. 当分别调用`parser.parseExpression`和`exp.getValue`时，可能抛出两个异常：`ParseException`和`EvaluationException`。

SpEL支持广泛的功能，例如调用方法，访问属性和调用构造函数。

在下面的方法调用示例中，我们在字符串文字上调用`concat` 方法:

    ExpressionParser parser = new SpelExpressionParser();
    Expression exp = parser.parseExpression("'Hello World'.concat('!')"); (1)
    String message = (String) exp.getValue();

**(1)。**变量现在的值为 'Hello World!'.

以下调用JavaBean属性的示例调用`String`属性`Bytes`property :

    ExpressionParser parser = new SpelExpressionParser();

    // invokes 'getBytes()'
    Expression exp = parser.parseExpression("'Hello World'.bytes"); (1)
    byte[] bytes = (byte[]) exp.getValue();

**(1)。**该行将文字转换为字节数组。

SpEL还支持嵌套属性，使用标准的点符号。即`prop1.prop2.prop3`链式写法和设置属性值。也可以访问公共字段。 以下示例显示如何使用点表示法来获取文字的长度：

    ExpressionParser parser = new SpelExpressionParser();

    // invokes 'getBytes().length'
    Expression exp = parser.parseExpression("'Hello World'.bytes.length"); (1)
    int length = (Integer) exp.getValue();

**(1)。**`'Hello World'.bytes.length` 给出了字符串的长度。

可以调用String的构造函数而不是使用字符串文字，如以下示例所示：

    ExpressionParser parser = new SpelExpressionParser();
    Expression exp = parser.parseExpression("new String('hello world').toUpperCase()"); (1)
    String message = exp.getValue(String.class);

**(1)。**从构造一个新的`String`对象并使其成为大写。

请注意泛型方法的使用: `public <T> T getValue(Class<T> desiredResultType)`。使用此方法不需要将表达式的值转换为所需的结果类型。如果该值不能转换为类型`T`或使用注册的类型转换器转换， 则将抛出`EvaluationException`异常。

SpEL的更常见用法是提供针对特定对象实例（称为根对象）计算的表达式字符串。 以下示例显示如何从`Inventor`类的实例检索`name`属性或创建布尔条件：

    // Create and set a calendar
    GregorianCalendar c = new GregorianCalendar();
    c.set(1856, 7, 9);

    // The constructor arguments are name, birthday, and nationality.
    Inventor tesla = new Inventor("Nikola Tesla", c.getTime(), "Serbian");

    ExpressionParser parser = new SpelExpressionParser();

    Expression exp = parser.parseExpression("name"); (1)
    String name = (String) exp.getValue(tesla);
    // name == "Nikola Tesla"

    exp = parser.parseExpression("name == 'Nikola Tesla'");
    boolean result = exp.getValue(tesla, Boolean.class);
    // result == true

**(1)。**将`name`解析为表达式。

<a id="expressions-evaluation-context"></a>

#### [](#expressions-evaluation-context)4.1.1. 了解 `EvaluationContext`

在评估表达式以解析属性，方法或字段以及帮助执行类型转换时，将使用`EvaluationContext`接口。 Spring提供了两种实现。

*   `SimpleEvaluationContext`: 为不需要SpEL语言语法的完整范围的表达式类别公开必要的SpEL语言特性和配置选项的子集， 并且应该进行有意义的限制。 示例包括但不限于数据绑定表达式和基于属性的过滤器。

*   `StandardEvaluationContext`: 公开全套SpEL语言功能和配置选项。 您可以使用它来指定默认根对象并配置每个可用的与评估相关的策略。


`SimpleEvaluationContext` SimpleEvaluationContext旨在仅支持SpEL语言语法的子集。 它排除了Java类型引用，构造函数和bean引用。 它还要求您明确选择表达式中属性和方法的支持级别。 默认情况下，`create()` 静态工厂方法仅启用对属性的读访问权限。 您还可以获取构建器以配置所需的确切支持级别，定位以下一个或多个组合：

*   仅限自定义 `PropertyAccessor`（无反射）

*   只读访问的数据绑定属性

*   读写的数据绑定属性


<a id="expressions-type-conversion"></a>

##### [](#expressions-type-conversion)类型转换

默认情况下，SpEL使用Spring核心类(`org.springframework.core.convert.ConversionService`)提供的转换服务。此转换服务附带许多转换器，内置很多常用转换，但也支持扩展。 因此可以添加类型之间的自定义转换。此外，它具有泛型感知的关键功能。这意味着在使用表达式中的泛型类型时，SpEL将尝试转换以维护遇到的任何对象的类型正确性。

这在实践中能得到什么好处？假设使用`setValue()`的赋值被用于设置 `List`属性。属性的类型实际上是`List<Boolean>`，SpEL会识别列表的元素需要在被放置在其中之前被转换为布尔值。 以下示例显示了如何执行此操作：

    class Simple {
        public List<Boolean> booleanList = new ArrayList<Boolean>();
    }

    Simple simple = new Simple();
    simple.booleanList.add(true);

    EvaluationContext context = SimpleEvaluationContext().forReadOnlyDataBinding().build();

    // false is passed in here as a string. SpEL and the conversion service
    // correctly recognize that it needs to be a Boolean and convert it
    parser.parseExpression("booleanList[0]").setValue(context, simple, "false");

    // b is false
    Boolean b = simple.booleanList.get(0);

<a id="expressions-parser-configuration"></a>

#### [](#expressions-parser-configuration)4.1.2. 解释器配置

可以使用解释器配置对象(`org.springframework.expression.spel.SpelParserConfiguration`)来配置SpEL表达式解释器。该配置对象控制表达式组件的行为。例如，如果索引到数组或集合，并且指定索引处的元素为null，则可以自动创建该元素。 当使用由一组属性引用组成的表达式时，这是非常有用的。如果创建数组或集合的索引，并指定了超出数组或列表的当前大小的结尾的索引时，它将自动增大数组或列表大小以适应索引。以下示例演示如何自动增长列表：

    class Demo {
        public List<String> list;
    }

    // Turn on:
    // - auto null reference initialization
    // - auto collection growing
    SpelParserConfiguration config = new SpelParserConfiguration(true,true);

    ExpressionParser parser = new SpelExpressionParser(config);

    Expression expression = parser.parseExpression("list[3]");

    Demo demo = new Demo();

    Object o = expression.getValue(demo);

    // demo.list will now be a real collection of 4 entries
    // Each entry is a new empty String

<a id="expressions-spel-compilation"></a>

#### [](#expressions-spel-compilation)4.1.3. SpEL编译

Spring Framework 4.1包含一个基本的表达式编译器。通常，由于表达式在操作过程中提供的大量动态性、灵活性的运算能够被解释，但不能保证提供最佳性能。对于不常使用的表达式使用这是非常好的， 但是当被其他并不真正需要动态灵活性的组件（例如Spring Integration）使用时，性能可能成为瓶颈。

新的SpEL编译器旨在满足这一需求。编译器将在表达行为运算操作期间即时生成一个真正的Java类，并使用它来实现更快的表达式求值。由于缺少对表达式按类型归类，编译器在执行编译时会使用在表达式解释运算期间收集的信息来编译。 例如，它不仅仅需要从表达式中知道属性引用的类型，而且需要在第一个解释运算过程中发现它是什么。当然，如果各种表达式元素的类型随着时间的推移而变化，那么基于此信息的编译可能会发生问题。因此， 编译最适合于重复运算操作而类型信息不会改变的表达式。

请考虑以下基本表达式:

someArray\[0\].someProperty.someOtherProperty < 0.1

这涉及到数组访问，某些属性的取消和数字操作，所以性能增益非常明显。 在50000次迭代的微基准测试示例中，使用解释器评估需要75ms，使用表达式的编译版本只需3ms。

<a id="expressions-compiler-configuration"></a>

##### [](#expressions-compiler-configuration)编译器配置

编译器在默认情况下是关闭的，有两种方法可以打开。您可以使用解析器配置过程（([前面讨论的](#expressions-parser-configuration)）或在将SpEL用法嵌入到另一个组件中时使用系统属性来打开它。 本节讨论这两个选项。

编译器可以以三种模式之一操作，这些模式在`org.springframework.expression.spel.SpelCompilerMode`枚举中获取。 模式如下：

*   `OFF` (default): 编译器已关闭。

*   `IMMEDIATE`: 在即时模式下，表达式将尽快编译。这通常在第一次解释运算之后，如果编译的表达式失败（通常是由于类型更改引起的，参看上一节），则表达式运算操作的调用者将收到异常。

*   `MIXED`:在混合模式下，表达式随着时间的推移在解释模式和编译模式之间静默地切换。经过一些解释运行后，它们将切换到编译模式，如果编译后的表单出现问题（如上所述改变类型）， 表达式将自动重新切换回解释模式。稍后，它可能生成另一个编译表单并切换。基本上，用户进入`IMMEDIATE`模式的异常是内部处理的。


推荐`IMMEDIATE`即时模式，因为`MIXED`模式可能会导致副作用，使得表达式出错。如果编译的表达式在部分成功之后崩掉，此时可能已经影响了系统状态。 如果发生这种情况，调用者可能不希望它在解释模式下静默地重新运行，这样的话表达式的某部分可能会运行两次。

选择模式后，使用`SpelParserConfiguration`配置解析器。 以下示例显示了如何执行此操作：

    SpelParserConfiguration config = new SpelParserConfiguration(SpelCompilerMode.IMMEDIATE,
        this.getClass().getClassLoader());

    SpelExpressionParser parser = new SpelExpressionParser(config);

    Expression expr = parser.parseExpression("payload");

    MyMessage message = new MyMessage();

    Object payload = expr.getValue(message);

指定编译器模式时，还可以指定类加载器（允许传递null）。编译表达式将在任何提供的子类加载器中被定义。重要的是确保是否指定了类加载器，它可以看到表达式运算操作过程中涉及的所有类型。 如果没有指定，那么将使用默认的类加载器（通常是在表达式计算期间运行的线程的上下文类加载器）。

配置编译器的第二种方法是将SpEL嵌入其他组件内部使用，并且可能无法通过配置对象进行配置。在这种情况下，可以使用系统属性配置。属性`spring.expression.compiler.mode` 可以设置为`SpelCompilerMode`枚举值之一（`off`, `immediate`, 或 `mixed`）。

<a id="expressions-compiler-limitations"></a>

##### [](#expressions-compiler-limitations)编译器限制

从Spring Framework 4.1开始，基本编译框架已经存在。 但是，该框架尚不支持编译各种表达式。最初的重点是在可能在性能要求高的关键环境中使用的常见表达式。目前无法编译以下类型的表达式：

*   涉及转让的表达

*   依赖转换服务的表达式

*   使用自定义解释器或访问器的表达式

*   使用选择或投影的表达式


越来越多的类型的表达式将在未来可编译.

<a id="expressions-beandef"></a>

### [](#expressions-beandef)4.2. bean定义的表达式支持

SpEL表达式可以通过XML或基于注解的配置用于定义`BeanDefinition`实例。在这两种情况下，定义表达式的语法是`#{ <expression string> }`

<a id="expressions-beandef-xml-based"></a>

#### [](#expressions-beandef-xml-based)4.2.1. XML 配置

可以使用表达式设置属性或构造函数参数值，如以下示例所示:

    <bean id="numberGuess" class="org.spring.samples.NumberGuess">
        <property name="randomNumber" value="#{ T(java.lang.Math).random() * 100.0 }"/>

        <!-- other properties -->
    </bean>

`systemProperties` 变量是预定义的，因此您可以在表达式中使用它，如以下示例所示：

    <bean id="taxCalculator" class="org.spring.samples.TaxCalculator">
        <property name="defaultLocale" value="#{ systemProperties['user.region'] }"/>

        <!-- other properties -->
    </bean>

请注意，您不必在此上下文中使用＃符号为预定义变量添加前缀。

您还可以按名称引用其他bean属性，如以下示例所示:

    <bean id="numberGuess" class="org.spring.samples.NumberGuess">
        <property name="randomNumber" value="#{ T(java.lang.Math).random() * 100.0 }"/>

        <!-- other properties -->
    </bean>

    <bean id="shapeGuess" class="org.spring.samples.ShapeGuess">
        <property name="initialShapeSeed" value="#{ numberGuess.randomNumber }"/>

        <!-- other properties -->
    </bean>

<a id="expressions-beandef-annotation-based"></a>

#### [](#expressions-beandef-annotation-based)4.2.2. 注解 Configuration

要指定默认值，可以在字段，方法和方法或构造函数参数上放置`@Value`注解。

以下示例设置字段变量的默认值:

    public static class FieldValueTestBean

        @Value("#{ systemProperties['user.region'] }")
        private String defaultLocale;

        public void setDefaultLocale(String defaultLocale) {
            this.defaultLocale = defaultLocale;
        }

        public String getDefaultLocale() {
            return this.defaultLocale;
        }

    }

下面显示了属性setter方法的相同配置:

    public static class PropertyValueTestBean

        private String defaultLocale;

        @Value("#{ systemProperties['user.region'] }")
        public void setDefaultLocale(String defaultLocale) {
            this.defaultLocale = defaultLocale;
        }

        public String getDefaultLocale() {
            return this.defaultLocale;
        }

    }

使用`@Autowired`方法注解的构造方法也可以使用`@Value`注解:

    public class SimpleMovieLister {

        private MovieFinder movieFinder;
        private String defaultLocale;

        @Autowired
        public void configure(MovieFinder movieFinder,
                @Value("#{ systemProperties['user.region'] }") String defaultLocale) {
            this.movieFinder = movieFinder;
            this.defaultLocale = defaultLocale;
        }

        // ...
    }

    public class MovieRecommender {

        private String defaultLocale;

        private CustomerPreferenceDao customerPreferenceDao;

        @Autowired
        public MovieRecommender(CustomerPreferenceDao customerPreferenceDao,
                @Value("#{systemProperties['user.country']}") String defaultLocale) {
            this.customerPreferenceDao = customerPreferenceDao;
            this.defaultLocale = defaultLocale;
        }

        // ...
    }

<a id="expressions-language-ref"></a>

### [](#expressions-language-ref)4.3. 语言引用

本节介绍Spring表达式语言的工作原理。 它涵盖以下主题：

*   [文字表达](#expressions-ref-literal)

*   [属性，数组，list，maps和indexs](#expressions-properties-arrays)

*   [内嵌的list](#expressions-inline-lists)

*   [内嵌的map](#expressions-inline-maps)

*   [数组的构造](#expressions-array-construction)

*   [方法](#expressions-methods)

*   [运算符](#expressions-operators)

*   [类型](#expressions-types)

*   [构造器](#expressions-constructors)

*   [变量](#expressions-ref-variables)

*   [函数](#expressions-ref-functions)

*   [Bean 的引用](#expressions-bean-references)

*   [三元运算符（If-Then-Else）](#expressions-operator-ternary)

*   [Elvis运算符](#expressions-operator-elvis)

*   [安全的引导运算符](#expressions-operator-safe-navigation)


<a id="expressions-ref-literal"></a>

#### [](#expressions-ref-literal)4.3.1. 文字表达

支持的文字表达式的类型是字符串，数值（int，real，hex），boolean和null。 字符串由单引号分隔。 要在字符串中放置单引号，请使用两个单引号字符。

以下清单显示了文字的简单用法。 通常，它们不是像这样单独使用，而是作为更复杂表达式的一部分使用 \- 例如，在逻辑比较运算符的一侧使用文字。

    ExpressionParser parser = new SpelExpressionParser();

    // evals to "Hello World"
    String helloWorld = (String) parser.parseExpression("'Hello World'").getValue();

    double avogadrosNumber = (Double) parser.parseExpression("6.0221415E+23").getValue();

    // evals to 2147483647
    int maxValue = (Integer) parser.parseExpression("0x7FFFFFFF").getValue();

    boolean trueValue = (Boolean) parser.parseExpression("true").getValue();

    Object nullValue = parser.parseExpression("null").getValue();

数字支持使用负号，指数表示法和小数点。 默认情况下，使用`Double.parseDouble()`解析实数。.

<a id="expressions-properties-arrays"></a>

#### [](#expressions-properties-arrays)4.3.2. Properties, Arrays, Lists, Maps, and Indexers

调用属性的引用是很简单的,只要指定内置的属性值即可。`Inventor`类（`pupin` 和 `tesla`）的实例填充了[示例](#expressions-example-classes)部分中使用的类中列出的数据。 下面的表达式用于获得Tesla的出生年和Pupin的出生城市:

    // evals to 1856
    int year = (Integer) parser.parseExpression("Birthdate.Year + 1900").getValue(context);

    String city = (String) parser.parseExpression("placeOfBirth.City").getValue(context);

属性名称的第一个字母允许不区分大小写。 数组和列表的内容是使用方括号表示法获得的，如下例所示：:

    ExpressionParser parser = new SpelExpressionParser();
    EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();

    // Inventions Array

    // evaluates to "Induction motor"
    String invention = parser.parseExpression("inventions[3]").getValue(
            context, tesla, String.class);

    // Members List

    // evaluates to "Nikola Tesla"
    String name = parser.parseExpression("Members[0].Name").getValue(
            context, ieee, String.class);

    // List and Array navigation
    // evaluates to "Wireless communication"
    String invention = parser.parseExpression("Members[0].Inventions[6]").getValue(
            context, ieee, String.class);

maps的内容通过方括号包着文字的键/值定义。在这种情况下， 由于 `Officers`的keys是字符串，则可以定义字符字面值：

    // Officer's Dictionary

    Inventor pupin = parser.parseExpression("Officers['president']").getValue(
            societyContext, Inventor.class);

    // evaluates to "Idvor"
    String city = parser.parseExpression("Officers['president'].PlaceOfBirth.City").getValue(
            societyContext, String.class);

    // setting values
    parser.parseExpression("Officers['advisors'][0].PlaceOfBirth.Country").setValue(
            societyContext, "Croatia");

<a id="expressions-inline-lists"></a>

#### [](#expressions-inline-lists)4.3.3. 内嵌的list

您可以使用`{}`表示法直接在表达式中表达列表。

    // evaluates to a Java list containing the four numbers
    List numbers = (List) parser.parseExpression("{1,2,3,4}").getValue(context);

    List listOfLists = (List) parser.parseExpression("{{'a','b'},{'x','y'}}").getValue(context);

`{}`本身就是一个空列表。 出于性能原因，如果列表本身完全由固定文字组成，则会创建一个常量列表来表示表达式（而不是在每个计算上构建新列表）。

<a id="expressions-inline-maps"></a>

#### [](#expressions-inline-maps)4.3.4. 内嵌的map

您还可以使用`{key:value}`表示法直接在表达式中表达map。 以下示例显示了如何执行此操作：

    // evaluates to a Java map containing the two entries
    Map inventorInfo = (Map) parser.parseExpression("{name:'Nikola',dob:'10-July-1856'}").getValue(context);

    Map mapOfMaps = (Map) parser.parseExpression("{name:{first:'Nikola',last:'Tesla'},dob:{day:10,month:'July',year:1856}}").getValue(context);

`{:}` {：}本身就是一张空map。 出于性能原因，如果map本身由固定文字或其他嵌套常量结构（列表或map）组成， 则会创建一个常量来表示表达式（而不是在每次计算时构建新map）。 map的双引号是可选的。 上面的示例没有使用双引号的key。

<a id="expressions-array-construction"></a>

#### [](#expressions-array-construction)4.3.5. 数组的构造

您可以使用熟悉的Java语法构建数组，可选择提供初始化程序以在构造时填充数组。 以下示例显示了如何执行此操作：:

    int[] numbers1 = (int[]) parser.parseExpression("new int[4]").getValue(context);

    // Array with initializer
    int[] numbers2 = (int[]) parser.parseExpression("new int[]{1,2,3}").getValue(context);

    // Multi dimensional array
    int[][] numbers3 = (int[][]) parser.parseExpression("new int[4][5]").getValue(context);

目前不支持创建多维数组的初始化器。

<a id="expressions-methods"></a>

#### [](#expressions-methods)4.3.6. 方法

方法是使用典型的Java编程语法调用的，还可以对文本调用方法。也支持对参数的调用。

    // string literal, evaluates to "bc"
    String bc = parser.parseExpression("'abc'.substring(1, 3)").getValue(String.class);

    // evaluates to true
    boolean isMember = parser.parseExpression("isMember('Mihajlo Pupin')").getValue(
            societyContext, Boolean.class);

<a id="expressions-operators"></a>

#### [](#expressions-operators)4.3.7. 运算符

Spring Expression Language支持以下类型的运算符：

*   [关系运算符](#expressions-operators-relational)

*   [逻辑运算符](#expressions-operators-logical)

*   [数学运算符](#expressions-operators-mathematical)

*   [赋值运算符](#expressions-assignment)


<a id="expressions-operators-relational"></a>

##### [](#expressions-operators-relational)关系运算符

使用标准运算符表示法支持关系运算符（等于，不等于，小于，小于或等于，大于，等于或等于）。 以下清单显示了一些运算符示例：

    // evaluates to true
    boolean trueValue = parser.parseExpression("2 == 2").getValue(Boolean.class);

    // evaluates to false
    boolean falseValue = parser.parseExpression("2 < -5.0").getValue(Boolean.class);

    // evaluates to true
    boolean trueValue = parser.parseExpression("'black' < 'block'").getValue(Boolean.class);

大于和小于`null`的比较遵循一个简单的规则：`null`被视为空（不是零）。 因此，任何其他值始终大于`null`（`X > null`始终为`true`），并且其他任何值都不会小于任何值（`X < null` 始终为`false`）。

如果您更喜欢数字比较，请避免基于数字的`null`比较，以支持与零进行比较（例如， `X > 0`或 `X < 0`）。

除了标准的关系运算符之外，SpEL支持`instanceof`和基于`matches`的正则表达式运算符，以下列表显示了两者的示例:

    // evaluates to false
    boolean falseValue = parser.parseExpression(
            "'xyz' instanceof T(Integer)").getValue(Boolean.class);

    // evaluates to true
    boolean trueValue = parser.parseExpression(
            "'5.00' matches '^-?\\d+(\\.\\d{2})?$'").getValue(Boolean.class);

    //evaluates to false
    boolean falseValue = parser.parseExpression(
            "'5.0067' matches '^-?\\d+(\\.\\d{2})?$'").getValue(Boolean.class);

使用原始类型的时候留意他们会直接被包装成包装类，因此 `1 instanceof T(int)`是`false`。而`1 instanceof T(Integer)`是`true`。

每一个符号运算符可以使用直接的单词字母（前缀）来定义，这样可以避免在某些特定的表达式会在文件类型中出现问题（例如XML文档）。现在列出文本的替换规则：

*   `lt` (`<`)

*   `gt` (`>`)

*   `le` (`<=`)

*   `ge` (`>=`)

*   `eq` (`==`)

*   `ne` (`!=`)

*   `div` (`/`)

*   `mod` (`%`)

*   `not` (`!`).


所有文本运算符都不区分大小写。.

<a id="expressions-operators-logical"></a>

##### [](#expressions-operators-logical)逻辑运算符

SpEL支持以下逻辑运算符：

*   `and`

*   `or`

*   `not`


以下示例显示如何使用逻辑运算符

    // -- AND --

    // evaluates to false
    boolean falseValue = parser.parseExpression("true and false").getValue(Boolean.class);

    // evaluates to true
    String expression = "isMember('Nikola Tesla') and isMember('Mihajlo Pupin')";
    boolean trueValue = parser.parseExpression(expression).getValue(societyContext, Boolean.class);

    // -- OR --

    // evaluates to true
    boolean trueValue = parser.parseExpression("true or false").getValue(Boolean.class);

    // evaluates to true
    String expression = "isMember('Nikola Tesla') or isMember('Albert Einstein')";
    boolean trueValue = parser.parseExpression(expression).getValue(societyContext, Boolean.class);

    // -- NOT --

    // evaluates to false
    boolean falseValue = parser.parseExpression("!true").getValue(Boolean.class);

    // -- AND and NOT --
    String expression = "isMember('Nikola Tesla') and !isMember('Mihajlo Pupin')";
    boolean falseValue = parser.parseExpression(expression).getValue(societyContext, Boolean.class);

<a id="expressions-operators-mathematical"></a>

##### [](#expressions-operators-mathematical)数学运算符

加法可以用在数值和字符串之间。减法、乘法和除法只能用在数值上。其他算术运算符支持取余和乘方。标准的运算符是支持优先级的。以下示例显示了正在使用的数学运算符：

    // Addition
    int two = parser.parseExpression("1 + 1").getValue(Integer.class);  // 2

    String testString = parser.parseExpression(
            "'test' + ' ' + 'string'").getValue(String.class);  // 'test string'

    // Subtraction
    int four = parser.parseExpression("1 - -3").getValue(Integer.class);  // 4

    double d = parser.parseExpression("1000.00 - 1e4").getValue(Double.class);  // -9000

    // Multiplication
    int six = parser.parseExpression("-2 * -3").getValue(Integer.class);  // 6

    double twentyFour = parser.parseExpression("2.0 * 3e0 * 4").getValue(Double.class);  // 24.0

    // Division
    int minusTwo = parser.parseExpression("6 / -3").getValue(Integer.class);  // -2

    double one = parser.parseExpression("8.0 / 4e0 / 2").getValue(Double.class);  // 1.0

    // Modulus
    int three = parser.parseExpression("7 % 4").getValue(Integer.class);  // 3

    int one = parser.parseExpression("8 / 5 % 2").getValue(Integer.class);  // 1

    // Operator precedence
    int minusTwentyOne = parser.parseExpression("1+2-3*8").getValue(Integer.class);  // -21

<a id="expressions-assignment"></a>

##### [](#expressions-assignment)赋值运算符

要设置属性，请使用赋值运算符(`=`)。 这通常在调用`setValue`时完成，但也可以在调用`getValue`时完成。 以下清单显示了使用赋值运算符的两种方法:

    Inventor inventor = new Inventor();
    EvaluationContext context = SimpleEvaluationContext.forReadWriteDataBinding().build();

    parser.parseExpression("Name").setValue(context, inventor, "Aleksandar Seovic");

    // alternatively
    String aleks = parser.parseExpression(
            "Name = 'Aleksandar Seovic'").getValue(context, inventor, String.class);

<a id="expressions-types"></a>

#### [](#expressions-types)4.3.8. 类型

特殊`T`运算符可用于指定`java.lang.Class`的实例(类型)。也可以使用此运算符调用静态方法。`StandardEvaluationContext`使用`TypeLocator`来查找类型， 而`StandardTypeLocator`(可以替换)是通过对`java.lang`包的解释而生成的。这意味着 `T()`对`java.lang`中的类型的引用不需要完全限定，但所有其他类型引用都是必须的。 以下示例显示如何使用`T`运算符:

    Class dateClass = parser.parseExpression("T(java.util.Date)").getValue(Class.class);

    Class stringClass = parser.parseExpression("T(String)").getValue(Class.class);

    boolean trueValue = parser.parseExpression(
            "T(java.math.RoundingMode).CEILING < T(java.math.RoundingMode).FLOOR")
            .getValue(Boolean.class);

<a id="expressions-constructors"></a>

#### [](#expressions-constructors)4.3.9. 构造器

可以使用`new`运算符调用构造函数。除了基本类型和String外需要使用全限定类名 (`int`, `float`,等等是可以直接使用的)。 以下示例显示如何使用 `new`运算符来调用构造函数：

    Inventor einstein = p.parseExpression(
            "new org.spring.samples.spel.inventor.Inventor('Albert Einstein', 'German')")
            .getValue(Inventor.class);

    //create new inventor instance within add method of List
    p.parseExpression(
            "Members.add(new org.spring.samples.spel.inventor.Inventor(
                'Albert Einstein', 'German'))").getValue(societyContext);

<a id="expressions-ref-variables"></a>

#### [](#expressions-ref-variables)4.3.10. 变量

在表达式中，变量通过`#variableName` 模式来表示。变量的设置用到`EvaluationContext`的`setVariable`方法。`setVariable`

    Inventor tesla = new Inventor("Nikola Tesla", "Serbian");

    EvaluationContext context = SimpleEvaluationContext.forReadWriteDataBinding().build();
    context.setVariable("newName", "Mike Tesla");

    parser.parseExpression("Name = #newName").getValue(context, tesla);
    System.out.println(tesla.getName())  // "Mike Tesla"

<a id="expressions-this-root"></a>

##### [](#expressions-this-root)`#this` 和 `#root` 变量

`#this` 变量始终指向当前的对象（处理没有全限定的引用）。`#root`变量使用指向根上下文对象。尽管`#this`可能根据表达式而不同。但是，`#root`一直指向根引用。以下示例显示了如何使用`#this`和`#root`变量：

    // create an array of integers
    List<Integer> primes = new ArrayList<Integer>();
    primes.addAll(Arrays.asList(2,3,5,7,11,13,17));

    // create parser and set variable 'primes' as the array of integers
    ExpressionParser parser = new SpelExpressionParser();
    EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataAccess();
    context.setVariable("primes", primes);

    // all prime numbers > 10 from the list (using selection ?{...})
    // evaluates to [11, 13, 17]
    List<Integer> primesGreaterThanTen = (List<Integer>) parser.parseExpression(
            "#primes.?[#this>10]").getValue(context);

<a id="expressions-ref-functions"></a>

#### [](#expressions-ref-functions)4.3.11. 函数

可以通过用户自定义函数来扩展SpEL，它可以在表达式字符串中使用，函数使用`EvaluationContext`的方法来注册：

    Method method = ...;

    EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();
    context.setVariable("myFunction", method);

例如，请考虑以下实用程序方法来反转字符串:

    public abstract class StringUtils {

        public static String reverseString(String input) {
            StringBuilder backwards = new StringBuilder(input.length());
            for (int i = 0; i < input.length(); i++)
                backwards.append(input.charAt(input.length() - 1 - i));
            }
            return backwards.toString();
        }
    }

然后，您可以注册并使用上述方法，如以下示例所示：

    ExpressionParser parser = new SpelExpressionParser();

    EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();
    context.setVariable("reverseString",
            StringUtils.class.getDeclaredMethod("reverseString", String.class));

    String helloWorldReversed = parser.parseExpression(
            "#reverseString('hello')").getValue(context, String.class);

<a id="expressions-bean-references"></a>

#### [](#expressions-bean-references)4.3.12. Bean的引用

如果已使用bean解析器配置了评估上下文，则可以使用`@` 符号从表达式中查找bean。 以下示例显示了如何执行此操作：

    ExpressionParser parser = new SpelExpressionParser();
    StandardEvaluationContext context = new StandardEvaluationContext();
    context.setBeanResolver(new MyBeanResolver());

    // This will end up calling resolve(context,"something") on MyBeanResolver during evaluation
    Object bean = parser.parseExpression("@something").getValue(context);

要访问工厂bean本身,bean名称应改为( `&`) 前缀符号. The following example shows how to do so:

    ExpressionParser parser = new SpelExpressionParser();
    StandardEvaluationContext context = new StandardEvaluationContext();
    context.setBeanResolver(new MyBeanResolver());

    // This will end up calling resolve(context,"&foo") on MyBeanResolver during evaluation
    Object bean = parser.parseExpression("&foo").getValue(context);

<a id="expressions-operator-ternary"></a>

#### [](#expressions-operator-ternary)4.3.13. 三元运算符（If-Then-Else）

您可以使用三元运算符在表达式中执行if-then-else条件逻辑。 以下清单显示了一个最小的示例:

    String falseString = parser.parseExpression(
            "false ? 'trueExp' : 'falseExp'").getValue(String.class);

在这种情况下，布尔值`false`会返回字符串值 `'falseExp'`。 一个更复杂的例子如下：

    parser.parseExpression("Name").setValue(societyContext, "IEEE");
    societyContext.setVariable("queryName", "Nikola Tesla");

    expression = "isMember(#queryName)? #queryName + ' is a member of the ' " +
            "+ Name + ' Society' : #queryName + ' is not a member of the ' + Name + ' Society'";

    String queryResultString = parser.parseExpression(expression)
            .getValue(societyContext, String.class);
    // queryResultString = "Nikola Tesla is a member of the IEEE Society"

有关三元运算符的更短语法，请参阅Elvis运算符的下一节。

<a id="expressions-operator-elvis"></a>

#### [](#expressions-operator-elvis)4.3.14. Elvis运算符

Elvis运算符是三元运算符语法的缩写，用于[Groovy](http://www.groovy-lang.org/operators.html#_elvis_operator)语言。 使用三元运算符语法，您通常必须重复两次变量，如以下示例所示：

    String name = "Elvis Presley";
    String displayName = (name != null ? name : "Unknown");

可以使用Elvis运算符来实现，上面例子的也可以使用如下的形式展现：

    ExpressionParser parser = new SpelExpressionParser();

    String name = parser.parseExpression("name?:'Unknown'").getValue(String.class);
    System.out.println(name);  // 'Unknown'

以下列表显示了一个更复杂的示例:

    ExpressionParser parser = new SpelExpressionParser();
    EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();

    Inventor tesla = new Inventor("Nikola Tesla", "Serbian");
    String name = parser.parseExpression("Name?:'Elvis Presley'").getValue(context, tesla, String.class);
    System.out.println(name);  // Nikola Tesla

    tesla.setName(null);
    name = parser.parseExpression("Name?:'Elvis Presley'").getValue(context, tesla, String.class);
    System.out.println(name);  // Elvis Presley

您可以使用Elvis运算符在表达式中应用默认值。 以下示例显示如何在`@Value`表达式中使用Elvis运算符：

    @Value("#{systemProperties['pop3.port'] ?: 25}")

如果已定义，则将注入系统属性`pop3.port`，否则注入25。

<a id="expressions-operator-safe-navigation"></a>

#### [](#expressions-operator-safe-navigation)4.3.15. 安全的引导运算符

安全的引导运算符用于避免`NullPointerException`异常，这种观念来自[Groovy](http://www.groovy-lang.org/operators.html#_safe_navigation_operator)语言。当需要引用一个对象时， 可能需要在访问对象的方法或属性之前验证它是否为null。为避免出现这种情况， 安全引导运算符将简单地返回null，而不是引发异常。

    ExpressionParser parser = new SpelExpressionParser();
    EvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding().build();

    Inventor tesla = new Inventor("Nikola Tesla", "Serbian");
    tesla.setPlaceOfBirth(new PlaceOfBirth("Smiljan"));

    String city = parser.parseExpression("PlaceOfBirth?.City").getValue(context, tesla, String.class);
    System.out.println(city);  // Smiljan

    tesla.setPlaceOfBirth(null);
    city = parser.parseExpression("PlaceOfBirth?.City").getValue(context, tesla, String.class);
    System.out.println(city);  // null - does not throw NullPointerException!!!

<a id="expressions-collection-selection"></a>

#### [](#expressions-collection-selection)4.3.16.集合的选择

Selection是一种功能强大的表达语言功能，通过从条目中进行选择，可以将某些源集合转换为另一种集合。

Selection使用语法是`.?[selectionExpression]`. 它会过滤集合并返回一个新的集合，其包含一个原始数据的子集合。例如，Selection可以简单地获取Serbian inventors的list：

    List<Inventor> list = (List<Inventor>) parser.parseExpression(
            "Members.?[Nationality == 'Serbian']").getValue(societyContext);

Selection可以使用在list和map上。前面的例子中选择独立处理了集合中的元素，而当选择一个map时将会处理每个map的entry（Java类型`Map.Entry`的对象），Map的entry有他的key和value作为属性访问在Selection中使用。

上述表达式将返回一个新的map，包括原有map中所有值小于27的条目：

    Map newMap = parser.parseExpression("map.?[value<27]").getValue();

除了返回所有选定元素外， 还可以只检索第一个或最后一个值。要获得与所选内容匹配的第一个条目语法是 `.^[selectionExpression]`。而获取最后一个匹配的选择语法是`.$[selectionExpression]`。

<a id="expressions-collection-projection"></a>

#### [](#expressions-collection-projection)4.3.17. 集合投影

投影允许集合被一个子表达式处理而且结果是一个新的集合。投影的语法是`.![projectionExpression]`。通过例子可便于理解，假设有一个invertors的list并且希望其生产一个叫cities的list， 有效的做法是对每个在invertor的list调用'placeOfBirth.city'。使用投影：

    // returns ['Smiljan', 'Idvor' ]
    List placesOfBirth = (List)parser.parseExpression("Members.![placeOfBirth.city]");

map可以用于处理投影，在这种情况下投影表达式可以对map中的每个entry进行处理（作为一个Java的 `Map.Entry`）。map投影的结果是一个list，包含对每一个map条目处理的投影表达式。

<a id="expressions-templating"></a>

#### [](#expressions-templating)4.3.18. 表达式模板

表达式模板允许将文字文本与一个或多个评估块混合使用。每个计算块都可以定义的前缀和后缀字符分隔，一般选择使用`#{ }`作为分隔符。如下例所示：

    String randomPhrase = parser.parseExpression(
            "random number is #{T(java.lang.Math).random()}",
            new TemplateParserContext()).getValue(String.class);

    // evaluates to "random number is 0.7038186818312008"

字符串包含文本`'random number is '` 和在`#{ }` 中的表达式的处理结果。这个例子的结果调用了`random()`方法。第二个参数对于`parseExpression()`方法是`ParserContext`的类型。 `ParserContext`接口可以控制表达式的解释，用于支持表达式模板功能。`TemplateParserContext` 的定义如下：

    public class TemplateParserContext implements ParserContext {

        public String getExpressionPrefix() {
            return "#{";
        }

        public String getExpressionSuffix() {
            return "}";
        }

        public boolean isTemplate() {
            return true;
        }
    }

<a id="expressions-example-classes"></a>

### [](#expressions-example-classes)4.4. 例子中用到的类

本节列出了本章示例中使用的类。

Example 1. Inventor.java

    package org.spring.samples.spel.inventor;

    import java.util.Date;
    import java.util.GregorianCalendar;

    public class Inventor {

        private String name;
        private String nationality;
        private String[] inventions;
        private Date birthdate;
        private PlaceOfBirth placeOfBirth;

        public Inventor(String name, String nationality) {
            GregorianCalendar c= new GregorianCalendar();
            this.name = name;
            this.nationality = nationality;
            this.birthdate = c.getTime();
        }

        public Inventor(String name, Date birthdate, String nationality) {
            this.name = name;
            this.nationality = nationality;
            this.birthdate = birthdate;
        }

        public Inventor() {
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public String getNationality() {
            return nationality;
        }

        public void setNationality(String nationality) {
            this.nationality = nationality;
        }

        public Date getBirthdate() {
            return birthdate;
        }

        public void setBirthdate(Date birthdate) {
            this.birthdate = birthdate;
        }

        public PlaceOfBirth getPlaceOfBirth() {
            return placeOfBirth;
        }

        public void setPlaceOfBirth(PlaceOfBirth placeOfBirth) {
            this.placeOfBirth = placeOfBirth;
        }

        public void setInventions(String[] inventions) {
            this.inventions = inventions;
        }

        public String[] getInventions() {
            return inventions;
        }
    }

Example 2. PlaceOfBirth.java

    package org.spring.samples.spel.inventor;

    public class PlaceOfBirth {

        private String city;
        private String country;

        public PlaceOfBirth(String city) {
            this.city=city;
        }

        public PlaceOfBirth(String city, String country) {
            this(city);
            this.country = country;
        }

        public String getCity() {
            return city;
        }

        public void setCity(String s) {
            this.city = s;
        }

        public String getCountry() {
            return country;
        }

        public void setCountry(String country) {
            this.country = country;
        }

    }

Example 3. Society.java

    package org.spring.samples.spel.inventor;

    import java.util.*;

    public class Society {

        private String name;

        public static String Advisors = "advisors";
        public static String President = "president";

        private List<Inventor> members = new ArrayList<Inventor>();
        private Map officers = new HashMap();

        public List getMembers() {
            return members;
        }

        public Map getOfficers() {
            return officers;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public boolean isMember(String name) {
            for (Inventor inventor : members) {
                if (inventor.getName().equals(name)) {
                    return true;
                }
            }
            return false;
        }

    }