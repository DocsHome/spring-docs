

<a id="validation"></a>

[](#validation)3\. 验证, 数据绑定和类型转换
--------------------------------

在业务逻辑中添加数据验证利弊参半，Spring提供的验证(和数据绑定）方案也未必能解决这个问题。具体来说，验证不应该与Web层绑定，并且应该易于本地化，并且应该可以插入任何可用的验证器。 考虑到这些问题，Spring提出了一个基本的，非常有用的`Validator`接口，它在应用程序的每一层都可用。

Spring为开发者提供了一个称作`DataBinder`的对象来处理数据绑定，所谓的数据绑定就是将用户输入自动地绑定到相应的领域模型（或者说用来处理用户所输入的任何对象）。Spring的`Validator`和`DataBinder` 构成了验证包，这个包主要用在Spring MVC框架使用，但绝不限于此。

`BeanWrapper`是Spring框架中的一个基本概念，并且在很多地方使用。但是，在使用过程中可能从未碰过它。鉴于这是一份参考文档，那么对`BeanWrapper`的解释很有必要。 我们将在本章中解释`BeanWrapper`，在开发者尝试将数据绑定到对象的时候一定会使用到它。

Spring的`DataBinder` 和底层的`BeanWrapper`都使用`PropertyEditorSupport`实现来解析和格式化属性值。 `PropertyEditor`和`PropertyEditorSupport`接口是JavaBeans规范的一部分，本章也对其进行了解释。 Spring 3引入了`core.convert`包来提供通用的类型转换工具和高级 “format” 包来格式化UI显示。这两个包提供的工具可以用作`PropertyEditorSupport` 的替代品，这一章也会讨论到它们。

JSR-303/JSR-349 Bean 验证

从Spring4.0开始，Spring框架支持Bean Validation 1.0（JSR-303）和Bean Validation 1.1（JSR-349）验证规范，同时也能兼容Spring的`Validator`验证接口。

应用程序既可以一次性开启控制全局的Bean Validation，如[Spring Validation](#validation-beanvalidation)中所述，并专门用于所有验证需求。

应用程序还可以为每个`DataBinder`实例注册其他Spring `Validator` 实例，如配置[Configuring a `DataBinder`](#validation-binder)中所述。

<a id="validator"></a>

### [](#validator)3.1. 使用Spring的Validator接口来进行数据验证

Spring提供了`Validator`接口用来进行对象的数据验证。`Validator`接口在进行数据验证的时候会要求传入一个`Errors` 对象，当有错误产生时会将错误信息放入该对象。

考虑以下小数据对象的示例：:

    public class Person {

        private String name;
        private int age;

        // the usual getters and setters...
    }

下一个示例通过实现`org.springframework.validation.Validator` 接口的以下两个方法为`Person`类提供验证行为：

*   `supports(Class)`: 判断该`Validator`是否能验证提供的`Class`的实例?

*   `validate(Object, org.springframework.validation.Errors)`: 验证给定的对象，如果有验证失败信息，则将其放入`Errors` 对象。


实现一个`Validator`相当简单，尤其是使用Spring提供的`ValidationUtils`工具类时。以下示例为Person实例实现Validator：

    public class PersonValidator implements Validator {

        /**
         * This Validator validates *only* Person instances
         */
        public boolean supports(Class clazz) {
            return Person.class.equals(clazz);
        }

        public void validate(Object obj, Errors e) {
            ValidationUtils.rejectIfEmpty(e, "name", "name.empty");
            Person p = (Person) obj;
            if (p.getAge() < 0) {
                e.rejectValue("age", "negativevalue");
            } else if (p.getAge() > 110) {
                e.rejectValue("age", "too.darn.old");
            }
        }
    }

ValidationUtils中的静态方法`rejectIfEmpty(..)`方法用于拒绝 `name`属性（如果它为`null` 或空字符串）。更多信息可以参考[`ValidationUtils`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/validation/ValidationUtils.html)的javadocs。

虽然可以实现一个`Validator`类来验证富对象中的每个嵌套对象，但最好将每个嵌套对象类的验证逻辑封装在自己的`Validator`实现中。 例如，有一个名为`Customer`的复杂对象，它有两个`String`类型的属性（first name和second name），另外还有一个`Address`对象。它与`Customer`毫无关系， 它还实现了名为`AddressValidator`的验证器。如果考虑在`Customer`验证器类中重用`Address`验证器的功能（这种重用不是通过简单的代码拷贝）， 那么可以将`Address`验证器的实例通过依赖注入的方式注入到`Customer`验证器中。如以下示例所示：

    public class CustomerValidator implements Validator {

        private final Validator addressValidator;

        public CustomerValidator(Validator addressValidator) {
            if (addressValidator == null) {
                throw new IllegalArgumentException("The supplied [Validator] is " +
                    "required and must not be null.");
            }
            if (!addressValidator.supports(Address.class)) {
                throw new IllegalArgumentException("The supplied [Validator] must " +
                    "support the validation of [Address] instances.");
            }
            this.addressValidator = addressValidator;
        }

        /**
         * This Validator validates Customer instances, and any subclasses of Customer too
         */
        public boolean supports(Class clazz) {
            return Customer.class.isAssignableFrom(clazz);
        }

        public void validate(Object target, Errors errors) {
            ValidationUtils.rejectIfEmptyOrWhitespace(errors, "firstName", "field.required");
            ValidationUtils.rejectIfEmptyOrWhitespace(errors, "surname", "field.required");
            Customer customer = (Customer) target;
            try {
                errors.pushNestedPath("address");
                ValidationUtils.invokeValidator(this.addressValidator, customer.getAddress(), errors);
            } finally {
                errors.popNestedPath();
            }
        }
    }

验证错误信息会上报给作为参数传入的`Errors`对象，如果使用Spring Web MVC。您可以使用`<spring:bind/>`标记来检查错误消息，但您也可以自己检查`Errors`对象。 有关它提供的方法的更多信息可以在[javadoc](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframeworkvalidation/Errors.html)中找到。

<a id="validation-conversion"></a>

### [](#validation-conversion)3.2. 通过错误编码得到错误信息

上一节介绍了数据绑定和数据验证，如何拿到验证错误信息是最后需要讨论的问题。在[上一个](#validator)的例子中，验证器拒绝了`name`和`age` 属性。如果我们想通过使用`MessageSource`输出错误消息， 可以在验证失败时设置错误编码（本例中就是name和age）。当调用（直接或间接地，通过使用 `ValidationUtils`类）`Errors`接口中的`rejectValue`方法或者它的任意一个方法时，它的实现不仅仅注册传入的错误编码参数， 还会注册一些遵循一定规则的错误编码。注册哪些规则的错误编码取决于开发者使用的`MessageCodesResolver`。当使用默认的`DefaultMessageCodesResolver`时， 除了会将错误信息注册到指定的错误编码上，这些错误信息还会注册到包含属性名的错误编码上。假如调用`rejectValue("age", "too.darn.old")`方法，Spring除了会注册`too.darn.old`错误编码外， 还会注册`too.darn.old.age`和`too.darn.old.age.int`这两个错误编码（即一个是包含属性名，另外一个既包含属性名还包含类型的）。在Spring中这种注册称为注册约定，这样所有的开发者都能按照这种约定来定位错误信息。

有关`MessageCodesResolver`和默认策略的更多信息可分别在 [`MessageCodesResolver`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/validation/MessageCodesResolver.html) 和 [`DefaultMessageCodesResolver`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/validation/DefaultMessageCodesResolver.html), 的javadoc中找到.

<a id="beans-beans"></a>

### [](#beans-beans)3.3. 操作bean和`BeanWrapper`

`org.springframework.beans`包遵循Oracle提供的JavaBeans标准，JavaBean只是一个包含默认无参构造器的类，它遵循命名约定（举例来说） 名为 `bingoMadness`属性将拥有设置方法 `setBingoMadness(..)`和获取方法`getBingoMadness()`。有关JavaBeans和规范的更多信息，请参考Oracle的网站([javabeans](https://docs.oracle.com/javase/8/docs/api/java/beans/package-summary.html)）。

beans包里一个非常重要的类是`BeanWrapper`接口和它的相应实现(`BeanWrapperImpl`)。引自java文档：`BeanWrapper`提供了设置和获取属性值(单独或批量）、 获取属性描述符以及查询属性以确定它们是可读还是可写的功能。 `BeanWrapper`还提供对嵌套属性的支持，能够不受嵌套深度的限制启用子属性的属性设置。`BeanWrapper`还提供了无需目标类代码的支持就能够添加标准JavaBeans的 `PropertyChangeListeners`和`VetoableChangeListeners`的能力。 最后但同样重要的是， `BeanWrapper`支持设置索引属性。应用程序代码通常不会直接使用`BeanWrapper`，而是提供给`DataBinder`和`BeanFactory`使用。

`BeanWrapper` 顾名思义，它包装了bean并对其执行操作。例如设置和获取属性。

<a id="beans-beans-conventions"></a>

#### [](#beans-beans-conventions)3.3.1. 设置并获取基本和嵌套的属性

设置和获取属性是通过使用`setPropertyValue`,`setPropertyValues`, `getPropertyValue`, 和 `getPropertyValues`方法完成的，这些方法带有多个重载变体。 Springs javadoc更详细地描述了它们。 JavaBeans规范具有指示对象属性的约定。 下表显示了这些约定的一些示例：

Table 11. Examples of properties

| Expression             | Explanation                                                  |
| ---------------------- | ------------------------------------------------------------ |
| `name`                 | 表示属性 `name`与`getName()`或`isName()`和`setName(..)`方法相对应 |
| `account.name`         | 表示 `account` 属性的嵌套属性`name`与`getAccount().setName()` 或 `getAccount().getName()` 相对应. |
| `account[2]`           | 表示索引属性`account`的第_3_个属性. 索引属性可以是`array`, `list`, 其他自然排序的集合. |
| `account[COMPANYNAME]` | 表示映射属性`account`被键`COMPANYNAME` 索引的映射项的值。    |

（如果您不打算直接使用`BeanWrapper` ，那么下一节对您来说并不重要。如果您只使用`DataBinder`和`BeanFactory`及其默认实现，那么您应该跳到[有关](#beans-beans-conversion)`PropertyEditors`的部分。）

以下两个示例类使用`BeanWrapper`来获取和设置属性：

    public class Company {

        private String name;
        private Employee managingDirector;

        public String getName() {
            return this.name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public Employee getManagingDirector() {
            return this.managingDirector;
        }

        public void setManagingDirector(Employee managingDirector) {
            this.managingDirector = managingDirector;
        }
    }

    public class Employee {

        private String name;

        private float salary;

        public String getName() {
            return this.name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public float getSalary() {
            return salary;
        }

        public void setSalary(float salary) {
            this.salary = salary;
        }
    }

以下代码段显示了如何检索和操作实例化`Companies`和 `Employees`的某些属性的一些示例：

    BeanWrapper company = new BeanWrapperImpl(new Company());
    // setting the company name..
    company.setPropertyValue("name", "Some Company Inc.");
    // ... can also be done like this:
    PropertyValue value = new PropertyValue("name", "Some Company Inc.");
    company.setPropertyValue(value);

    // ok, let's create the director and tie it to the company:
    BeanWrapper jim = new BeanWrapperImpl(new Employee());
    jim.setPropertyValue("name", "Jim Stravinsky");
    company.setPropertyValue("managingDirector", jim.getWrappedInstance());

    // retrieving the salary of the managingDirector through the company
    Float salary = (Float) company.getPropertyValue("managingDirector.salary");

<a id="beans-beans-conversion"></a>

#### [](#beans-beans-conversion)3.3.2. 内置`PropertyEditor`实现

Spring使用 `PropertyEditor`的概念来实现 `Object`和`String`之间的转换，有时使用不同于对象本身的方式来表示属性显得更方便。例如，`Date`可以使用易于阅读的方式(如`String`: `'2007-14-09'`）。 还能将易于阅读的形式转换回原来的`Date`(甚至做得更好：转换任何以易于阅读形式输入的日期，然后返回日期对象）。可以通过注册`java.beans.PropertyEditor`类型的自定义编辑器来实现此行为。 在`BeanWrapper`上注册自定义编辑器，或者在特定的IoC容器中注册自定义编辑器（如前一章所述），使其了解如何将属性转换为所需类型。 有关`PropertyEditor`的更多信息，请参阅[Oracle的`java.beans`包的javadoc](https://docs.oracle.com/javase/8/docs/api/java/beans/package-summary.html)

在Spring中使用属性编辑的几个示例:

*   通过使用`PropertyEditor`实现来设置bean的属性。 当您使用`java.lang.String`作为您在XML文件中声明的某个bean的属性的值时， Spring将(如果相应属性的setter具有类参数）使用`ClassEditor`尝试将参数解析为类对象。

*   在Spring的MVC框架中解析HTTP请求参数是通过使用各种`PropertyEditor`实现来完成的，您可以在`CommandController`的所有子类中手动绑定它们。


Spring内置了许多`PropertyEditor`用于简化处理。它们都位于`org.springframework.beans.propertyeditors`包中。大多数（但不是全部，如下表所示）默认情况下由`BeanWrapperImpl`注册。 当属性编辑器以某种方式进行配置时，开发者仍可以注册自定义的变体用于覆盖默认的变量。下表描述了Spring提供的各种`PropertyEditor`实现：

Table 12. 内置`PropertyEditor` 实现

| 类                        | 解释                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `ByteArrayPropertyEditor` | 字节数组的编辑器。 将字符串转换为其对应的字节表示形式。`BeanWrapperImpl`默认注册。 |
| `ClassEditor`             | 将表示类的字符串解析为实际的类，反之亦然。 找不到类时，抛出`IllegalArgumentException`。 默认情况下，由`BeanWrapperImpl`注册。 |
| `CustomBooleanEditor`     | `Boolean`属性的可自定义属性编辑器。 默认情况下，由`BeanWrapperImpl`注册，但可以通过将其自定义实例注册为自定义编辑器来覆盖。 |
| `CustomCollectionEditor`  | `Collection`的属性编辑器，将任何源`Collection`转换为给定的目标`Collection`类型。 |
| `CustomDateEditor`        | `java.util.Date`的可自定义属性编辑器，支持自定义`DateFormat`。 未默认注册。 必须根据需要使用适当的格式进行用户注册。 |
| `CustomNumberEditor`      | 任何`Number` 子类的可自定义属性编辑器，例如`Integer`, `Long`, `Float`或`Double`。 默认情况下，由`BeanWrapperImpl`注册，但可以通过将其自定义实例注册为自定义编辑器来覆盖。 |
| `FileEditor`              | 将字符串解析为`java.io.File`对象。 默认情况下，由`BeanWrapperImpl`注册。 |
| `InputStreamEditor`       | 单向属性编辑器，可以获取字符串并生成（通过中间`ResourceEditor`和`Resource`）`InputStream`，以便`InputStream`属性可以直接设置为字符串。 请注意，默认用法不会为您关闭 `InputStream`。 默认情况下，由 `BeanWrapperImpl`注册。 |
| `LocaleEditor`            | 可以将字符串解析为`Locale`对象，反之亦然（字符串格式为`_[country]_[variant]`，与`Locale`的 `toString()` 方法相同）。 默认情况下，由`BeanWrapperImpl`注册。 |
| `PatternEditor`           | 可以将字符串解析为`java.util.regex.Pattern`对象，反之亦然。  |
| `PropertiesEditor`        | 可以将字符串（使用 `java.util.Properties`类的javadoc中定义的格式进行格式化）转换为 `Properties` 对象。 默认情况下，由`BeanWrapperImpl`注册。 |
| `StringTrimmerEditor`     | 修剪字符串的属性编辑器。 （可选）允许将空字符串转换为空值。 默认情况下未注册 \- 必须是用户注册的。 |
| `URLEditor`               | 可以将URL的字符串表示形式解析为实际的`URL` 对象。 默认情况下，由`BeanWrapperImpl`注册。 |

Spring使用`java.beans.PropertyEditorManager`设置属性编辑器（可能需要）的搜索路径。搜索路径还包括 `sun.bean.editors`，其中包括`Font`, `Color`和大多数基本类型等类型的`PropertyEditor`实现。 注意，标准的JavaBeans架构可以自动发现 `PropertyEditor`类（无需显式注册），前提是此类与需处理的类位于同一个包，并且与该类具有相同的名称。并以`Editor`单词结尾。 可以使用以下类和包结构，这足以使`SomethingEditor`类被识别并用作`Something`类型属性的`PropertyEditor`。

```
com
  chank
    pop
      Something
      SomethingEditor // the PropertyEditor for the Something class
```

请注意，您也可以在此处使用标准 `BeanInfo` JavaBeans机制（ [这里](https://docs.oracle.com/javase/tutorial/javabeans/advanced/customization.html)描述的是无关紧要的细节）。 以下示例使用`BeanInfo`机制使用关联类的属性显式注册一个或多个`PropertyEditor`实例：

com
  chank
    pop
      Something
      SomethingBeanInfo // the BeanInfo for the Something class

以下引用的`SomethingBeanInfo`类的Java源代码将`CustomNumberEditor`与`Something`类的`age`属性相关联：

    public class SomethingBeanInfo extends SimpleBeanInfo {

        public PropertyDescriptor[] getPropertyDescriptors() {
            try {
                final PropertyEditor numberPE = new CustomNumberEditor(Integer.class, true);
                PropertyDescriptor ageDescriptor = new PropertyDescriptor("age", Something.class) {
                    public PropertyEditor createPropertyEditor(Object bean) {
                        return numberPE;
                    };
                };
                return new PropertyDescriptor[] { ageDescriptor };
            }
            catch (IntrospectionException ex) {
                throw new Error(ex.toString());
            }
        }
    }

<a id="beans-beans-conversion-customeditor-registration"></a>

##### [](#beans-beans-conversion-customeditor-registration)注册额外的自定义`PropertyEditor`

将bean属性设置为字符串值时，Spring IoC容器最终使用标准JavaBeans `PropertyEditor`实现将这些字符串转换为属性的复杂类型。 Spring预先注册了许多自定义`PropertyEditor`实现（例如，将表示为字符串的类名转换为`Class`对象）。 此外，Java的标准JavaBeans `PropertyEditor`查找机制允许对类的`PropertyEditor`进行适当的命名，并将其放置在与其提供支持的类相同的包中，以便可以自动找到它。

如果需要注册其他自定义`PropertyEditors`，可以使用多种机制。通常最麻烦也不推荐的策略是手动、简单的使用`ConfigurableBeanFactory`接口的`registerCustomEditor()`方法， 假设有一个`BeanFactory`引用，另一种（稍微更方便）机制是使用一个名为`CustomEditorConfigurer`的特殊bean工厂后置处理器。尽管您可以将bean工厂后置处理器与`BeanFactory`实现一起使用，但`CustomEditorConfigurer`具有嵌套属性设置， 因此我们强烈建议您将它与`ApplicationContext`一起使用，您可以在其中以类似的方式将其部署到任何其他bean以及它可以在哪里 自动检测并应用。

请注意，所有的bean工厂和应用程序上下文都自动使用了许多内置属性编辑器，在其内部都是使用`BeanWrapper`来进行属性转换的。 `BeanWrapper` 注册的标准属性编辑器列在[上一节](#beans-beans-conversion)中 此外，`ApplicationContexts`还会覆盖或添加其他编辑器，以适合特定应用程序上下文类型的方式处理资源查找。

标准的 `PropertyEditor` JavaBeans实例用于将以字符串表示的属性值转换为属性的实际复杂类型。 `CustomEditorConfigurer`是一个bean后置处理工厂，可用于方便地在`ApplicationContext`中添加额外的 `PropertyEditor`实例。

请考虑以下示例，该示例定义名为`ExoticType`的用户类和另一个名为`DependsOnExoticType`的类，该类需要将`ExoticType`设置为属性：

    package example;

    public class ExoticType {

        private String name;

        public ExoticType(String name) {
            this.name = name;
        }
    }

    public class DependsOnExoticType {

        private ExoticType type;

        public void setType(ExoticType type) {
            this.type = type;
        }
    }

当创建好后，希望将type属性指定为一个字符串，`PropertyEditor`会在幕后将其转换成实际的`ExoticType`实例。以下bean定义显示了如何设置此关系：

    <bean id="sample" class="example.DependsOnExoticType">
        <property name="type" value="aNameForExoticType"/>
    </bean>

`PropertyEditor`实现如下:

    // converts string representation to ExoticType object
    package example;

    public class ExoticTypeEditor extends PropertyEditorSupport {

        public void setAsText(String text) {
            setValue(new ExoticType(text.toUpperCase()));
        }
    }

最后，以下示例显示如何使用`CustomEditorConfigurer`向`ApplicationContext`注册新的`PropertyEditor`，然后可以根据需要使用它：

    <bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
        <property name="customEditors">
            <map>
                <entry key="example.ExoticType" value="example.ExoticTypeEditor"/>
            </map>
        </property>
    </bean>

<a id="beans-beans-conversion-customeditor-registration-per"></a>

###### [](#beans-beans-conversion-customeditor-registration-per)使用 `PropertyEditorRegistrar`

使用Spring容器注册属性编辑器的另一个策略是创建和使用`PropertyEditorRegistrar`。当需要在多种不同的情况下使用相同的属性编辑器集时，这个接口特别有用，编写相应的注册器并在每个案例中重用。 `PropertyEditorRegistrar`与另外一个称为`PropertyEditorRegistry`的接口一起工作。它使用Spring `BeanWrapper`(和`DataBinder`)实现。`PropertyEditorRegistrar`在与`CustomEditorConfigurer`([本节](#beans-beans-conversion-customeditor-registration)介绍的)一起使用时特别方便， 它公开`setPropertyEditorRegistrars(..)`的属性。`PropertyEditorRegistrar`和`CustomEditorConfigurer` 结合使用可以简单的在`DataBinder`和Spring MVC控制之间共享。 它避免了在自定义编辑器上进行同步的需要：`PropertyEditorRegistrar`需要为每个bean创建尝试创建新的`PropertyEditor`实例。

以下示例显示如何创建自己的`PropertyEditorRegistrar`实现:

    package com.foo.editors.spring;

    public final class CustomPropertyEditorRegistrar implements PropertyEditorRegistrar {

        public void registerCustomEditors(PropertyEditorRegistry registry) {

            // it is expected that new PropertyEditor instances are created
            registry.registerCustomEditor(ExoticType.class, new ExoticTypeEditor());

            // you could register as many custom property editors as are required here...
        }
    }

有关`PropertyEditorRegistrar`实现的示例，另请参见`org.springframework.beans.support.ResourceEditorRegistrar`。 请注意，在实现`registerCustomEditors(..)` 方法时，它会创建每个属性编辑器的新实例。

下一个示例显示如何配置`CustomEditorConfigurer`并将 `CustomPropertyEditorRegistrar`的实例注入其中：

    <bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
        <property name="propertyEditorRegistrars">
            <list>
                <ref bean="customPropertyEditorRegistrar"/>
            </list>
        </property>
    </bean>

    <bean id="customPropertyEditorRegistrar"
        class="com.foo.editors.spring.CustomPropertyEditorRegistrar"/>

最后（与本章的重点有所不同，对于那些使用[Spring的MVC Web框架](web.html#mvc)的人来说），使用`PropertyEditorRegistrars`和数据绑定控制器（`SimpleFormController`）可以非常方便。 以下示例在 `initBinder(..)`方法的实现中使用 `PropertyEditorRegistrar`:

    public final class RegisterUserController extends SimpleFormController {

        private final PropertyEditorRegistrar customPropertyEditorRegistrar;

        public RegisterUserController(PropertyEditorRegistrar propertyEditorRegistrar) {
            this.customPropertyEditorRegistrar = propertyEditorRegistrar;
        }

        protected void initBinder(HttpServletRequest request,
                ServletRequestDataBinder binder) throws Exception {
            this.customPropertyEditorRegistrar.registerCustomEditors(binder);
        }

        // other methods to do with registering a User
    }

这种类型的code>PropertyEditor注册方式可以让代码更加简洁（`initBinder(..)`的实现只有一行），并允许将通用`PropertyEditor`注册代码封装在一个类中，然后根据需要在尽可能多的控制器之间共享。

<a id="core-convert"></a>

### [](#core-convert)3.4. Spring 类型转换

Spring 3引入了一个`core.convert`包，它提供了一个通用的类型转换系统。系统定义了一个用于实现类型转换逻辑的SPI和一个用于在运行时执行类型转换的API。 在Spring的容器中，此系统可以用作`PropertyEditor`的替代方法，它将外部bean属性值字符串转换为所需的属性类型。您还可以在需要进行类型转换的应用程序中的任何位置使用公共API。

<a id="core-convert-Converter-API"></a>

#### [](#core-convert-Converter-API)3.4.1. SPI转换器

实现类型转换逻辑的SPI是简易的，而且是强类型的。如以下接口定义所示：

    package org.springframework.core.convert.converter;

    public interface Converter<S, T> {

        T convert(S source);
    }

创建自定义转换器都需要实现`Converter`接口，参数`S`是需要转换的类型，`T`是转换后的类型。这个转换器也可以应用在集合或数组上将`S`参数转换为`T`参数。前提是已经注册了委托数组或集合转换器（`DefaultConversionService`默认情况下也是如此）。 `Converter` interface and parameterize `S`

对于要`convert(S)`的每个调用，源参数需保证不为null。转换失败时，`Converter`可能会引发任意的unchecked异常。具体来说，它应抛出`IllegalArgumentException`以报告无效的源值。 请注意确保您的Converter实现是线程安全的。

为方便起见，`core.convert.support`包中提供了几个转换器实现。 这些包括从字符串到数字和其他常见类型的转换器。 以下清单显示了`StringToInteger` 类，它是典型的`Converter` 实现：

    package org.springframework.core.convert.support;

    final class StringToInteger implements Converter<String, Integer> {

        public Integer convert(String source) {
            return Integer.valueOf(source);
        }
    }

<a id="core-convert-ConverterFactory-SPI)"></a>

#### [](#core-convert-ConverterFactory-SPI)3.4.2. 使用 `ConverterFactory`

当需要集中整个类层次结构的转换逻辑时（例如，从String转换为java.lang.Enum对象时），您可以实现`ConverterFactory`，如以下示例所示：

    package org.springframework.core.convert.converter;

    public interface ConverterFactory<S, R> {

        <T extends R> Converter<S, T> getConverter(Class<T> targetType);
    }

参数化S为您要转换的类型，R是需要转换后的类型的基类。 然后实现getConverter(Class<T>)，其中T是R的子类。

以`StringToEnum` `ConverterFactory`为例：

    package org.springframework.core.convert.support;

    final class StringToEnumConverterFactory implements ConverterFactory<String, Enum> {

        public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {
            return new StringToEnumConverter(targetType);
        }

        private final class StringToEnumConverter<T extends Enum> implements Converter<String, T> {

            private Class<T> enumType;

            public StringToEnumConverter(Class<T> enumType) {
                this.enumType = enumType;
            }

            public T convert(String source) {
                return (T) Enum.valueOf(this.enumType, source.trim());
            }
        }
    }

<a id="core-convert-GenericConverter-SPI"></a>

#### [](#core-convert-GenericConverter-SPI)3.4.3. 使用 `GenericConverter`

当您需要复杂的`Converter`实现时，请考虑使用`GenericConverter`接口。`GenericConverter`具有比`Converter`更灵活但不太强类型的签名，支持在多种源和目标类型之间进行转换。 此外，`GenericConverter`可以在实现转换逻辑时使用可用的源和目标字段上下文。 此上下文类允许通过字段注解或在字段签名上声明的一般信息来驱动类型转换。 以下清单显示了`GenericConverter`的接口定义：

    package org.springframework.core.convert.converter;

    public interface GenericConverter {

        public Set<ConvertiblePair> getConvertibleTypes();

        Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);
    }

要实现`GenericConverter`，请使用`getConvertibleTypes()`返回支持的source→target类型对，然后实现 `convert(Object, TypeDescriptor, TypeDescriptor)`方法并编写转换逻辑。源`TypeDescriptor`提供对保存要转换的值的源字段的访问，目标`TypeDescriptor`提供对要设置转换值的目标字段的访问。

Java数组和集合之间转换的转换器是`GenericConverter`应用的例子。其中 `ArrayToCollectionConverter`内部声明目标集合类型用于解析集合元素类型的字段。 它允许在目标字段上设置集合之前，将源数组中的每个元素转换为集合元素类型。

因为`GenericConverter` 是一个更复杂的SPI接口，所以只有在需要时才应该使用它。 一般使用`Converter`或`ConverterFactory` 足以满足基本的类型转换需求。

<a id="core-convert-ConditionalGenericConverter-SPI"></a>

##### [](#core-convert-ConditionalGenericConverter-SPI)Using `ConditionalGenericConverter`

有时可能只想在特定条件为真时才执行`Converter`，例如，在特定注解的目标上使用`Converter`，或者，在一个特定的目标类方法（例如`static valueOf`方法）中执行`Converter`。 `ConditionalGenericConverter`是`GenericConverter` 和`ConditionalConverter`接口的组合。允许自定义匹配条件

    public interface ConditionalConverter {

        boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType);
    }

    public interface ConditionalGenericConverter extends GenericConverter, ConditionalConverter {
    }

用于持久实体标识符和实体引用之间转换的 `EntityConverter`是`ConditionalGenericConverter` 应用的例子。如果目标实体类型声明静态查找器方法(如`findAccount(Long)`), 那么`EntityConverter`只对匹配的生效。开发者可以实现`matches(TypeDescriptor, TypeDescriptor)`以执行finder方法来检查是否匹配。

<a id="core-convert-ConversionService-API"></a>

#### [](#core-convert-ConversionService-API)3.4.4. The `ConversionService` API

`ConversionService`定义了一个统一的API，用于在运行时执行类型转换逻辑。 转换器通常在以下Facade接口后面执行：

    package org.springframework.core.convert;

    public interface ConversionService {

        boolean canConvert(Class<?> sourceType, Class<?> targetType);

        <T> T convert(Object source, Class<T> targetType);

        boolean canConvert(TypeDescriptor sourceType, TypeDescriptor targetType);

        Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType);

    }

大多数 `ConversionService`实现还实现了`ConverterRegistry`，它提供了一个用于注册转换器的SPI。 在内部，`ConversionService`实现委托其注册的转换器执行类型转换逻辑。

`core.convert.support`包中提供了强大的`ConversionService`实现。 `GenericConversionService`是适用于大多数环境的通用实现。 `ConversionServiceFactory`提供了一个方便的工厂，用于创建常见的`ConversionService`配置。

<a id="core-convert-Spring-config"></a>

#### [](#core-convert-Spring-config)3.4.5. 配置`ConversionService`

`ConversionService`是一个无状态对象，在应用程序启动时就会实例化，可以被多个线程共享。在Spring应用程序中，通常每个Spring容器(或`ApplicationContext`) 配置一个`ConversionService`实例。该`ConversionService`将被Spring获取，然后在框架需要执行类型转换时使用。也可以将`ConversionService`插入任意bean并直接调用它。

如果没有向Spring注册`ConversionService`，则使用基于`PropertyEditor`的原始系统。

要使用Spring注册默认的`ConversionService`，请添加以下bean定义，其 `id` 为`conversionService`：

    <bean id="conversionService"
        class="org.springframework.context.support.ConversionServiceFactoryBean"/>

默认的`ConversionService`可以在字符串，数字，枚举，集合，映射和其他常见类型之间进行转换。 要使用您自己的自定义转换器补充或覆盖默认转换器，请设置`converters`属性。 属性值可以实现任何`Converter`, `ConverterFactory`, 或 `GenericConverter` 接口。

    <bean id="conversionService"
            class="org.springframework.context.support.ConversionServiceFactoryBean">
        <property name="converters">
            <set>
                <bean class="example.MyCustomConverter"/>
            </set>
        </property>
    </bean>

在Spring MVC应用程序中使用`ConversionService`也很常见。 请参阅[Spring MVC章节中的转换和格式化](web.html#mvc-config-conversion)。

在某些情况下，您可能希望在转换期间应用格式。 有关使用`FormattingConversionServiceFactoryBean`的详细信息，请参阅FormatterRegistry SPI。 [`FormatterRegistry` SPI](#format-FormatterRegistry-SPI)

<a id="core-convert-programmatic-usage"></a>

#### [](#core-convert-programmatic-usage)3.4.6. 编程使用`ConversionService`

要以编程方式使用`ConversionService`实例，您可以像对任何其他bean一样注入对它的引用。 以下示例显示了如何执行此操作：

    @Service
    public class MyService {

        @Autowired
        public MyService(ConversionService conversionService) {
            this.conversionService = conversionService;
        }

        public void doIt() {
            this.conversionService.convert(...)
        }
    }

对于大多数用例，您可以使用指定targetType的`convert`方法，但它不适用于更复杂的类型，例如参数化元素的集合。 例如，如果想使用编程的方式将整数列表转换为字符串列表，则需要提供源和目标类型的正规定义。

幸运的是，`TypeDescriptor`提供了各种选项，使得这样做非常简单，如下例所示：

    DefaultConversionService cs = new DefaultConversionService();

    List<Integer> input = ....
    cs.convert(input,
        TypeDescriptor.forObject(input), // List<Integer> type descriptor
        TypeDescriptor.collection(List.class, TypeDescriptor.valueOf(String.class)));

请注意， `DefaultConversionService`会自动注册适合大多数环境的转换器。 这包括集合转换器，基本类型转换器和基本的对象到字符串转换器。 您可以使用`DefaultConversionService`类上的静态`addDefaultConverters`方法向任何`ConverterRegistry` 注册相同的转换器。

值类型的转换器可以重用于数组和集合，因此无需创建特定的转换器即可将`S`的`Collection` 转换为`T`的`Collection`，前提是标准集合处理是合适的。 no need to create a specific converter to convert from a of to a of , assuming that standard collection handling is appropriate.

<a id="format"></a>

### [](#format)3.5. Spring 字段格式化

如前一节所述，[`core.convert`](#core-convert) 是一种通用类型转换系统。 它提供统一的`ConversionService`API以及强类型转换器SPI，用于实现从一种类型到另一种类型的转换逻辑。 Spring容器使用此系统绑定bean属性值。 此外，Spring Expression Language（SpEL）和 `DataBinder`都使用此系统绑定字段值。此外，当SpEL需要将 `Short`类型强转为 `Long`类型， 用于试图完成`expression.setValue(Object bean, Object value)`时，那么`core.convert`系统也可以提供这种功能。

现在考虑典型客户端环境（例如Web或桌面应用程序）的类型转换要求。在这种环境中,在这种环境中,还包括转换成为`String`用于支持视图呈现程序。此外，还通常需要本地化字符串值。 普通的转化器SPI没有提供按照直接进行格式转换的功能。更通用的 `core.convert` `Converter` SPI不能解决此类要求。为了实现这个功能，Spring 3添加了方便的`Formatter` SPI，它提供了简单强健的的 `PropertyEditor`专供客户端环境。

通常， 当需要使用通用类型转换时可以用`Converter`SPI。例如，在`java.util.Date`和 `java.lang.Long`之间进行转换。 在客户端环境（例如Web应用程序）中工作时，可以使用`Formatter` SPI， 并且需要解析和打印本地化的字段值。`ConversionService`为两个SPI提供统一的类型转换API。

<a id="format-Formatter-SPI"></a>

#### [](#format-Formatter-SPI)3.5.1. `Formatter` SPI

`Formatter` SPI实现字段格式化逻辑是简单的，强类型的。 以下清单显示了Formatter接口定义：

    package org.springframework.format;

    public interface Formatter<T> extends Printer<T>, Parser<T> {
    }

`Formatter`继承了 `Printer`和`Parser`内置的接口。以下清单显示了这两个接口的定义：

    public interface Printer<T> {

        String print(T fieldValue, Locale locale);
    }

    import java.text.ParseException;

    public interface Parser<T> {

        T parse(String clientValue, Locale locale) throws ParseException;
    }

如果需要创建自定义的 `Formatter`，需要实现 `Formatter`接口。参数`T`类型是你需要格式化的类型。 例如，`java.util.Date`。实现`print()`操作在客户端本地设置中打印显示的`T`实例。 实现`parse()`操作以从客户端本地设置返回的格式化表示形式分析`T`的实例。如果尝试分析失败，`Formatter`会抛出`ParseException`或 `IllegalArgumentException`异常。注意确保自定义的`Formatter` 是线程安全的。

`format`子包提供了多种`Formatter`实现方便使用。 `number`子包中提供了`NumberStyleFormatter`, `CurrencyStyleFormatter`, 和 `PercentStyleFormatter`用于格式化`java.lang.Number`（使用`java.text.NumberFormat`）。`datetime`子包中提供了`DateFormatter`用于格式化`java.util.Date`（使用`java.text.DateFormat`）。 `datetime.joda`包基于 [Joda-Time 库](http://joda-time.sourceforge.net)提供全面的日期时间格式支持。

以下`DateFormatter`是`Formatter`实现的示例：

    package org.springframework.format.datetime;

    public final class DateFormatter implements Formatter<Date> {

        private String pattern;

        public DateFormatter(String pattern) {
            this.pattern = pattern;
        }

        public String print(Date date, Locale locale) {
            if (date == null) {
                return "";
            }
            return getDateFormat(locale).format(date);
        }

        public Date parse(String formatted, Locale locale) throws ParseException {
            if (formatted.length() == 0) {
                return null;
            }
            return getDateFormat(locale).parse(formatted);
        }

        protected DateFormat getDateFormat(Locale locale) {
            DateFormat dateFormat = new SimpleDateFormat(this.pattern, locale);
            dateFormat.setLenient(false);
            return dateFormat;
        }
    }

更多内容上Spring社区查看`Formatter`的版本信息，请参阅[jira.spring.io](https://jira.spring.io/browse/SPR)进行贡献。

<a id="format-CustomFormatAnnotations"></a>

#### [](#format-CustomFormatAnnotations)3.5.2. 基于注解的格式化

字段格式也可以通过字段类型或注解进行配置。如果要将注解绑定到`Formatter`，请实现`AnnotationFormatterFactory`。以下清单显示了 `AnnotationFormatterFactory` 接口的定义：

    package org.springframework.format;

    public interface AnnotationFormatterFactory<A extends Annotation> {

        Set<Class<?>> getFieldTypes();

        Printer<?> getPrinter(A annotation, Class<?> fieldType);

        Parser<?> getParser(A annotation, Class<?> fieldType);
    }

参数化A是将格式逻辑与(例如`org.springframework.format.annotation.DateTimeFormat`关联到字段`annotationType`。 `getFieldTypes()`返回注解可用的字段类型。 使`getPrinter()`返回`Printer`以打印注解字段值。`getParser()`返回一个 `Parser`以分析注解字段的`clientValue`。

下面的示例 `AnnotationFormatterFactory`实现，将`@NumberFormat`注解绑定到格式化程序。此注解允许指定数字样式或模式

    public final class NumberFormatAnnotationFormatterFactory
            implements AnnotationFormatterFactory<NumberFormat> {

        public Set<Class<?>> getFieldTypes() {
            return new HashSet<Class<?>>(asList(new Class<?>[] {
                Short.class, Integer.class, Long.class, Float.class,
                Double.class, BigDecimal.class, BigInteger.class }));
        }

        public Printer<Number> getPrinter(NumberFormat annotation, Class<?> fieldType) {
            return configureFormatterFrom(annotation, fieldType);
        }

        public Parser<Number> getParser(NumberFormat annotation, Class<?> fieldType) {
            return configureFormatterFrom(annotation, fieldType);
        }

        private Formatter<Number> configureFormatterFrom(NumberFormat annotation, Class<?> fieldType) {
            if (!annotation.pattern().isEmpty()) {
                return new NumberStyleFormatter(annotation.pattern());
            } else {
                Style style = annotation.style();
                if (style == Style.PERCENT) {
                    return new PercentStyleFormatter();
                } else if (style == Style.CURRENCY) {
                    return new CurrencyStyleFormatter();
                } else {
                    return new NumberStyleFormatter();
                }
            }
        }
    }

想要触发格式化，只需在在字段上添加`@NumberFormat`注解即可。

    public class MyModel {

        @NumberFormat(style=Style.CURRENCY)
        private BigDecimal decimal;
    }

<a id="format-annotations-api"></a>

##### [](#format-annotations-api)格式化注解API

`org.springframework.format.annotation`包中存在可移植格式注解API。 您可以使用`@NumberFormat`格式化java.lang.Number字段， 使用`@DateTimeFormat`格式化`java.util.Date`， `java.util.Calendar`，`java.util.Long`或Joda-Time字段。

下面的示例使用`@DateTimeFormat`将`java.util.Date` 化为ISO Date（yyyy-MM-dd）：

    public class MyModel {

        @DateTimeFormat(iso=ISO.DATE)
        private Date date;
    }

<a id="format-FormatterRegistry-SPI"></a>

#### [](#format-FormatterRegistry-SPI)3.5.3. `FormatterRegistry` SPI

`FormatterRegistry`是一个用于注册格式化程序和转换器的SPI。 `FormattingConversionService`适用于大多数环境的`FormatterRegistry`实现。此实现可以通过编程或以声明的方式配置为可用`FormattingConversionServiceFactoryBean`的Spring bean。 由于它也实现了`ConversionService`，因此可以直接配置用于Spring的 `DataBinder`和Spring的表达式语言（SpEL）。

以下清单显示了`FormatterRegistry`:

    package org.springframework.format;

    public interface FormatterRegistry extends ConverterRegistry {

        void addFormatterForFieldType(Class<?> fieldType, Printer<?> printer, Parser<?> parser);

        void addFormatterForFieldType(Class<?> fieldType, Formatter<?> formatter);

        void addFormatterForFieldType(Formatter<?> formatter);

        void addFormatterForAnnotation(AnnotationFormatterFactory<?, ?> factory);
    }

如上所示, Formatters通过fieldType或注解进行注册。

`FormatterRegistry` SPI可以集中配置格式规则，避免跨控制器的重复配置。例如，想要强制所有日期字段都以特定方式格式化，或者具有特定注解的字段以某种方式格式化。 使用共享的`FormatterRegistry`，开发者只需一次定义这些规则，即可到处使用。

<a id="format-FormatterRegistrar-SPI"></a>

#### [](#format-FormatterRegistrar-SPI)3.5.4. `FormatterRegistrar` SPI

`FormatterRegistrar` 是用于注册格式化器和通过FormatterRegistry转换的SPI:

    package org.springframework.format;

    public interface FormatterRegistrar {

        void registerFormatters(FormatterRegistry registry);
    }

`FormatterRegistrar`用于注册多个相关的转换器或格式化器（根据给定的格式化目录注册，例如Date格式化）。在直接注册不能实现时FormatterRegistrar就派上用场了， 例如，当格式化程序需要在不同于其自身`<T>`的特定字段类型下进行索引时，或者在注册`Printer`/`Parser`对时。下一节提供了有关转换器和格式化器注册的更多信息。

<a id="format-configuring-formatting-mvc"></a>

#### [](#format-configuring-formatting-mvc)3.5.5. 在Spring MVC中配置格式化

请参阅Spring MVC章节中的[转换和格式化](web.html#mvc-config-conversion)。

<a id="format-configuring-formatting-globaldatetimeformat"></a>

### [](#format-configuring-formatting-globaldatetimeformat)3.6. 配置全局日期和时间格式

默认情况下，不带 `@DateTimeFormat`注解的日期和时间字段使用`DateFormat.SHORT`（短日期）的格式转换字符串。开发者也可以使用自定义的全局格式覆盖默认格式。

此时需要确保Spring不注册默认格式化器，而应该手动注册所有格式化器，根据是否使用joda时间库，可以选择使用`org.springframework.format.datetime.joda.JodaTimeFormatterRegistrar` or `org.springframework.format.datetime.DateFormatterRegistrar` 类。

例如，以下Java配置注册全局`yyyyMMdd`格式（此示例不依赖于Joda-Time库）：

    @Configuration
    public class AppConfig {

        @Bean
        public FormattingConversionService conversionService() {

            // Use the DefaultFormattingConversionService but do not register defaults
            DefaultFormattingConversionService conversionService = new DefaultFormattingConversionService(false);

            // Ensure @NumberFormat is still supported
            conversionService.addFormatterForFieldAnnotation(new NumberFormatAnnotationFormatterFactory());

            // Register date conversion with a specific global format
            DateFormatterRegistrar registrar = new DateFormatterRegistrar();
            registrar.setFormatter(new DateFormatter("yyyyMMdd"));
            registrar.registerFormatters(conversionService);

            return conversionService;
        }
    }

如果您更喜欢基于XML的配置，则可以使用 `FormattingConversionServiceFactoryBean`。 以下示例显示了如何执行此操作（这次使用Joda Time）:

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="
            http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans.xsd>

        <bean id="conversionService" class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
            <property name="registerDefaultFormatters" value="false" />
            <property name="formatters">
                <set>
                    <bean class="org.springframework.format.number.NumberFormatAnnotationFormatterFactory" />
                </set>
            </property>
            <property name="formatterRegistrars">
                <set>
                    <bean class="org.springframework.format.datetime.joda.JodaTimeFormatterRegistrar">
                        <property name="dateFormatter">
                            <bean class="org.springframework.format.datetime.joda.DateTimeFormatterFactoryBean">
                                <property name="pattern" value="yyyyMMdd"/>
                            </bean>
                        </property>
                    </bean>
                </set>
            </property>
        </bean>
    </beans>

Joda-Time提供单独的不同类型来表示`date`, `time`, 和 `date-time`。`JodaTimeFormatterRegistrar`的`dateFormatter`, `timeFormatter`, 和 `dateTimeFormatter` 属性应用于为每种类型配置不同的格式。`DateTimeFormatterFactoryBean`提供了一种创建格式化器的快捷方法。

如果使用Spring MVC框架，请记住显式配置使用的转换服务。对于基于Java的`@Configuration`，这意味着继承`WebMvcConfigurationSupport`类并覆盖了`mvcConversionService()`。 对于XML，您应该使用 `mvc:annotation-driven` 元素的`conversion-service`属性。 有关详细信息，请参阅[转换和格式。](web.html#mvc-config-conversion)

<a id="validation-beanvalidation"></a>

### [](#validation-beanvalidation)3.7. Spring的验证

Spring 3介绍了对其验证支持的几个改进，首先，完全支持JSR-303 Bean Validation API。 其次，当以编程方式使用时，Spring的`DataBinder`可以验证对象以及绑定它们。 第三，Spring MVC支持声明性地验证`@Controller` 输入。

<a id="validation-beanvalidation-overview"></a>

#### [](#validation-beanvalidation-overview)3.7.1.JSR-303的bean Validation API的总览

JSR-303是Java平台标准化验证约束声明和元数据的标准API，使用此API，可以使用声明性验证约束对域模型属性进行注解，并且在运行时强制执行。开发者可以利用一些内置约束，也可于自定义约束。

请考虑以下示例，该示例显示了具有两个属性的简单`PersonForm` 模型:

    public class PersonForm {
        private String name;
        private int age;
    }

JSR-303允许您为这些属性定义声明性验证约束，如以下示例所示:

    public class PersonForm {

        @NotNull
        @Size(max=64)
        private String name;

        @Min(0)
        private int age;
    }

当JSR-303 Validator验证此类的实例时，将强制执行这些约束。

有关JSR-303和JSR-349的一般信息，请参阅[Bean Validation website](http://beanvalidation.org/)网站。有关默认引用实现的特定功能的信息，请参阅Hibernate Validator [Hibernate Validator](https://www.hibernate.org/412.html) 文档。要了解如何将bean验证提供程序设置为Spring bean，请继续阅读以下内容。 bean, keep reading.

<a id="validation-beanvalidation-spring"></a>

#### [](#validation-beanvalidation-spring)3.7.2. 配置bean Validation提供者

Spring提供了对Bean Validation API的完全支持，包括对以 JSR-303 或 JSR-349 Bean Validation提供作为Spring bean进行引导的快捷支持。允许在应用程序需要验证的任何地方注入`javax.validation.ValidatorFactory` 或 `javax.validation.Validator`。

您可以使用`LocalValidatorFactoryBean` 将默认Validator配置为Spring bean，如以下示例所示：

    <bean id="validator"
        class="org.springframework.validation.beanvalidation.LocalValidatorFactoryBean"/>

上面的基本配置将触发Bean验证以使用其默认的引导机制进行初始化，JSR-303或JSR-349提供程序（例如Hibernate Validator）应该存在于类路径中并自动检测。

<a id="validation-beanvalidation-spring-inject"></a>

##### [](#validation-beanvalidation-spring-inject)注入Validator

`LocalValidatorFactoryBean`实现了`javax.validation.ValidatorFactory`和`javax.validation.Validator`，以及Spring的`org.springframework.validation.Validator`。 您可以将这些接口中的任何一个引用注入到需要调用验证逻辑的bean中。

如果您希望直接使用Bean Validation API，则可以注入对`javax.validation.Validator`的引用，如以下示例所示：

    import javax.validation.Validator;

    @Service
    public class MyService {

        @Autowired
        private Validator validator;

如果您的bean需要Spring Validation API，则可以注入对`org.springframework.validation.Validator`的引用，如以下示例所示：

    import org.springframework.validation.Validator;

    @Service
    public class MyService {

        @Autowired
        private Validator validator;
    }

<a id="validation-beanvalidation-spring-constraints"></a>

##### [](#validation-beanvalidation-spring-constraints)配置自定义约束

每个Bean验证约束都由两部分组成：首先是声明约束及其可配置属性的`@Constraint`注解，然后是实现约束行为的`javax.validation.ConstraintValidator`接口实现。

如果要将声明与实现关联，每个 `@Constraint`注解都会引用相应的`ConstraintValidator`实现类。在运行中，当在域模型中遇到约束注解时，`ConstraintValidatorFactory`会将引用的实现实例化。

默认情况下，`LocalValidatorFactoryBean`会配置`SpringConstraintValidatorFactory`，它会使用Spring去创建`ConstraintValidator`实例。这允许自定义`ConstraintValidators`， 就像任何其他Spring bean一样，从依赖注入中获益。

下面是自定义`@Constraint`声明的例子，使用Spring的依赖注入来管理`ConstraintValidator`的实现:

    @Target({ElementType.METHOD, ElementType.FIELD})
    @Retention(RetentionPolicy.RUNTIME)
    @Constraint(validatedBy=MyConstraintValidator.class)
    public @interface MyConstraint {
    }

    import javax.validation.ConstraintValidator;

    public class MyConstraintValidator implements ConstraintValidator {

        @Autowired;
        private Foo aDependency;

        ...
    }

如前面的示例所示，`ConstraintValidator`实现可以将其依赖项`@Autowired`与任何其他Spring bean一样。

<a id="validation-beanvalidation-spring-method"></a>

##### [](#validation-beanvalidation-spring-method)Spring驱动的方法验证

Bean Validation 1.1支持的方法验证，Hibernate Validator 4.3支持的自定义扩展都可以通过 `MethodValidationPostProcessor` 定义集成到Spring上下文中。如下所示：:

    <bean class="org.springframework.validation.beanvalidation.MethodValidationPostProcessor"/>

为了符合Spring驱动方法验证的条件，所有目标类都需要使用Spring的`@Validated`进行注解，还可以选择声明要使用的验证组。 使用Hibernate Validator和Bean Validation 1.1提供验证的步骤可以查看[`MethodValidationPostProcessor`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/validation/beanvalidation/MethodValidationPostProcessor.html)的javadocs

<a id="validation-beanvalidation-spring-other"></a>

##### [](#validation-beanvalidation-spring-other)额外的配置选项

对于大多数情况，默认的`LocalValidatorFactoryBean`配置就足够了。从消息插入到遍历解析，各种Bean Validation构造有许多配置选项。 有关这些选项的更多信息，请参阅[`LocalValidatorFactoryBean`](https://docs.spring.io/spring-framework/docs/5.1.3.BUILD-SNAPSHOT/javadoc-api/org/springframework/validation/beanvalidation/LocalValidatorFactoryBean.html) javadoc。

<a id="validation-binder"></a>

#### [](#validation-binder)3.7.3. 配置 `DataBinder`

从Spring 3开始，您可以使用`Validator`配置`DataBinder`实例。 配置完成后，您可以通过调用`binder.validate()`来调用 `Validator`。 任何验证 `Errors`都会自动添加到活页夹的`BindingResult`中。

以下示例说明如何在绑定到目标对象后以编程方式使用`DataBinder`来调用验证逻辑：

    Foo target = new Foo();
    DataBinder binder = new DataBinder(target);
    binder.setValidator(new FooValidator());

    // bind to the target object
    binder.bind(propertyValues);

    // validate the target object
    binder.validate();

    // get BindingResult that includes any validation errors
    BindingResult results = binder.getBindingResult();

`DataBinder`还可以通过 `dataBinder.addValidators`和`dataBinder.replaceValidators`来配置多个`Validator`实例。 将全局配置的Bean Validation与本地在`DataBinder`实例上配置的Spring `Validator`相结合，这非常有用。 请参阅[\[validation-mvc-configuring\]](#validation-mvc-configuring)。

<a id="validation-mvc"></a>

#### [](#validation-mvc)3.7.4. Spring MVC 3 验证

See [Validation](web.html#mvc-config-validation) in the Spring MVC chapter.