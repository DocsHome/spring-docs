<a id="cci"></a>

[](#cci)5\. JCA CCI
-------------------

JJava EE提供了一个规范， 用于对企业信息系统(EIS)标准化的访问： JCA(Java EE Connector Architecture：Java EE连接框架)。该规范分为两个不同的部分：

*   连接器提供程序必须实现的SPI(服务提供程序接口)。这些接口构成了可部署在Java EE应用程序服务器上的资源适配器。在这种情况下， 服务器管理连接池、事务和安全 (托管模式)。应用程序服务器还负责管理在客户端应用程序之外进行的配置。连接器也可以在没有应用服务器的情况下使用。在这种情况下， 应用程序必须直接配置它(非托管模式)。
    
*   CCI (通用客户端接口) 供应用来使用来连接和通讯和EIS。还提供了用于本地事务划分的API。
    

Spring CCI支持的目的是使用Spring Framework的一般资源和事务管理工具提供以典型Spring方式访问CCI连接器的类。

连接器的客户端并不是总要使用CCI。 某些连接器公开自己的API， 只提供JCA资源适配器来使用Java EE Container的系统协定(连接池，全局事务和安全性）。 Spring不为这种特定于连接器的API提供特殊支持。

<a id="cci-config"></a>

### [](#cci-config)5.1. 配置 CCI

本节介绍如何配置通用客户端接口（CCI）。 它包括以下主题：

*   [连接器配置](#cci-config-connector)
    
*   [Spring中的`ConnectionFactory`配置](#cci-config-connectionfactory)
    
*   [配置 CCI 连接](#cci-config-cci-connections)
    
*   [使用单独的CCI连接](#cci-config-single-connection)
    

<a id="cci-config-connector"></a>

#### [](#cci-config-connector)5.1.1. 连接器配置

使用JCA CCI的基本资源是`ConnectionFactory` 接口。 您使用的连接器必须提供此接口的实现。

若要使用连接器，可以将其部署到应用程序服务器上，并从服务器的JNDI环境(托管模式)中获取`ConnectionFactory`。连接器必须打包为RAR文件(资源适配器存档)， 并包含 `ra.xml`文件来描述其部署特性。资源的实际名称在部署时指定。要在Spring中访问它，只需使用Spring的`JndiObjectFactoryBean`/`<jee:jndi-lookup>`以其JNDI名称获取工厂。

使用连接器的另一种方法是将其嵌入到应用程序(非托管模式)中，而不是使用应用程序服务器部署和配置它。Spring提供了通过提供的`FactoryBean`(`LocalConnectionFactoryBean`)将连接器配置为bean的可能性。以这种方式，您只需要类路径中的连接器库(无RAR文件，且不需要`ra.xml`描述符)。如果需要，必须从连接器的RAR文件中提取该库。

一旦你获得了你的`ConnectionFactory`实例， 你就可以将它注入你的组件中。这些组件可以是针对普通的 API， 或者利用Spring的支持类来访问CCI(例如`CciTemplate`)。

在非托管模式下使用连接器时， 不能使用全局事务， 因为在当前线程的当前全局事务中从未登记或摘出资源。该资源根本不知道可能正在运行的任何全局Java EE事务。

<a id="cci-config-connectionfactory"></a>

#### [](#cci-config-connectionfactory)5.1.2. 在Spring中配置`ConnectionFactory`

为了与EIS建立连接，如果您处于托管模式，则需要从应用服务器获取`ConnectionFactory`。如果你处于非托管模式，则可以直接从Spring中获取。

在托管模式中，你通过JNDI来访问`ConnectionFactory`，它的属性将在应用服务器中配置。以下示例说明了如何执行此操作：

```
<jee:jndi-lookup id="eciConnectionFactory" jndi-name="eis/cicseci"/>
```

在非托管模式下，您必须将要在Spring配置中使用的`ConnectionFactory`配置为JavaBean。`LocalConnectionFactoryBean`类提供这种设置样式，传递连接器的`ManagedConnectionFactory`实现，公开应用程序级的CCI `ConnectionFactory`。以下示例说明了如何执行此操作：

```xml
<bean id="eciManagedConnectionFactory" class="com.ibm.connector2.cics.ECIManagedConnectionFactory">
    <property name="serverName" value="TXSERIES"/>
    <property name="connectionURL" value="tcp://localhost/"/>
    <property name="portNumber" value="2006"/>
</bean>

<bean id="eciConnectionFactory" class="org.springframework.jca.support.LocalConnectionFactoryBean">
    <property name="managedConnectionFactory" ref="eciManagedConnectionFactory"/>
</bean>
```

你不能直接实例化特定的`ConnectionFactory`，您需要为连接器的`ManagedConnectionFactory` 接口执行相应的实现。此接口是JCA SPI规范的一部分。

<a id="cci-config-cci-connections"></a>

#### [](#cci-config-cci-connections)5.1.3. 配置CCI连接

JCA CCI 允许开发者使用连接器的`ConnectionSpec`实现来配置到EIS的连接。为了配置其属性，您需要用专用的适配器(`ConnectionSpecConnectionFactoryAdapter`)包装目标连接工厂。因此，，专用的`ConnectionSpec`可以配置为`connectionSpec`属性(作为一个内部bean)。

此属性不是必需的，因为CCI `ConnectionFactory`接口定义了两种不同的方法来获取一个与之关联的CCI连接。某些`ConnectionSpec`属性通常可以在应用程序服务器(在托管模式下)或相应的本地`ManagedConnectionFactory`实现中进行配置。以下清单显示了`ConnectionFactory`接口定义的相关部分：

```java
public interface ConnectionFactory implements Serializable, Referenceable {
    ...
    Connection getConnection() throws ResourceException;
    Connection getConnection(ConnectionSpec connectionSpec) throws ResourceException;
    ...
}
```

Spring提供了一个`ConnectionSpecConnectionFactoryAdapter`，它允许指定一个`ConnectionSpec`实例来用于给定工厂的所有操作。如果指定了适配器的`connectionSpec`属性， 那么适配器将使用带有`ConnectionSpec`参数的`getConnection`变量，否则该变量不带参数。以下示例显示如何配置`ConnectionSpecConnectionFactoryAdapter`:

```xml
<bean id="managedConnectionFactory"
        class="com.sun.connector.cciblackbox.CciLocalTxManagedConnectionFactory">
    <property name="connectionURL" value="jdbc:hsqldb:hsql://localhost:9001"/>
    <property name="driverName" value="org.hsqldb.jdbcDriver"/>
</bean>

<bean id="targetConnectionFactory"
        class="org.springframework.jca.support.LocalConnectionFactoryBean">
    <property name="managedConnectionFactory" ref="managedConnectionFactory"/>
</bean>

<bean id="connectionFactory"
        class="org.springframework.jca.cci.connection.ConnectionSpecConnectionFactoryAdapter">
    <property name="targetConnectionFactory" ref="targetConnectionFactory"/>
    <property name="connectionSpec">
        <bean class="com.sun.connector.cciblackbox.CciConnectionSpec">
            <property name="user" value="sa"/>
            <property name="password" value=""/>
        </bean>
    </property>
</bean>
```

<a id="cci-config-single-connection"></a>

#### [](#cci-config-single-connection)5.1.4. 使用单独的CCI连接

如果您想使用单独的CCI连接，Spring提供了一个进一步的`ConnectionFactory`适配器来管理这个。`SingleConnectionFactory`适配器类将会延迟打开一个连接，并在应用程序关闭时关闭此bean。此类将公开相应行为的特殊`Connection`代理，所有共享相同的底层物理连接。以下示例显示如何使用`SingleConnectionFactory`适配器类：

```xml
<bean id="eciManagedConnectionFactory"
        class="com.ibm.connector2.cics.ECIManagedConnectionFactory">
    <property name="serverName" value="TEST"/>
    <property name="connectionURL" value="tcp://localhost/"/>
    <property name="portNumber" value="2006"/>
</bean>

<bean id="targetEciConnectionFactory"
        class="org.springframework.jca.support.LocalConnectionFactoryBean">
    <property name="managedConnectionFactory" ref="eciManagedConnectionFactory"/>
</bean>

<bean id="eciConnectionFactory"
        class="org.springframework.jca.cci.connection.SingleConnectionFactory">
    <property name="targetConnectionFactory" ref="targetEciConnectionFactory"/>
</bean>
```

无法直接使用`ConnectionSpec`配置此`ConnectionFactory`适配器。如果您需要一个特定`ConnectionSpec`的单独连接，请使用`SingleConnectionFactory`对话的中介`ConnectionSpecConnectionFactoryAdapter`。

<a id="cci-using"></a>

### [](#cci-using)5.2. 使用Spring的CCI访问支持

本节介绍如何使用Spring对CCI的支持来实现各种目的。 它包括以下主题：

*   [记录转换](#cci-record-creator)
    
*   [使用 `CciTemplate`](#cci-using-template)
    
*   [DAO支持](#cci-using-dao)
    
*   [自动输出记录生成](#automatic-output-generation)
    
*   [`CciTemplate` `Interaction` 总结](#template-summary)
    
*   [直接使用CCI连接和交互](#cci-straight)
    
*   [`CciTemplate`用法示例](#cci-template-example)
    

<a id="cci-record-creator"></a>

#### [](#cci-record-creator)5.2.1. 记录转换

JCA CCI支持的目标之一是提供方便的设置来操作CCI记录。开发人员可以指定创建记录和从记录中提取数据的策略，在使用Spring的`CciTemplate`时。如果您不想在应用程序中直接处理记录，以下接口将配置策略以使用输入和输出记录。

为了创建输入记录， 开发人员可以使用 `RecordCreator`接口的专用实现。以下清单显示了`RecordCreator` 接口定义:

```java
public interface RecordCreator {

    Record createRecord(RecordFactory recordFactory) throws ResourceException, DataAccessException;

}
```

`createRecord(..)`方法接收一个`RecordFactory`实例作为参数，它对应于`ConnectionFactory`所使用的`RecordFactory` 。此引用可用于创建 `IndexedRecord`或`MappedRecord`实例。下面的示例演示如何使用`RecordCreator`接口和索引/映射记录：

```java
public class MyRecordCreator implements RecordCreator {

    public Record createRecord(RecordFactory recordFactory) throws ResourceException {
        IndexedRecord input = recordFactory.createIndexedRecord("input");
        input.add(new Integer(id));
        return input;
    }

}
```

输出记录可用于从EIS接收数据。因此，可以将`RecordExtractor`接口的特定实现传递到Spring的`CciTemplate`， 以便从输出记录中提取数据。以下清单显示了`RecordExtractor`接口定义：

```java
public interface RecordExtractor {

    Object extractData(Record record) throws ResourceException, SQLException, DataAccessException;

}
```

以下示例显示如何使用：`RecordExtractor`接口:

```java
public class MyRecordExtractor implements RecordExtractor {

    public Object extractData(Record record) throws ResourceException {
        CommAreaRecord commAreaRecord = (CommAreaRecord) record;
        String str = new String(commAreaRecord.toByteArray());
        String field1 = string.substring(0,6);
        String field2 = string.substring(6,1);
        return new OutputObject(Long.parseLong(field1), field2);
    }

}
```

<a id="cci-using-template"></a>

#### [](#cci-using-template)5.2.2. 使用 `CciTemplate`

`CciTemplate`是核心的CCI支持包(`org.springframework.jca.cci.core`)的中心类。它简化了对管理的使用，因为它处理资源的创建和释放。这有助于避免常见的错误，如忘记总是需要的关闭连接。它关心连接和交互对象的生命周期，让应用程序代码专注于从应用程序数据生成输入记录，并从输出记录中提取应用程序数据。

JCA的CCI规范定义了两种不同的方法来调用EIS上的操作。CCI的 `Interaction`接口提供了两个execute方法：

```java
public interface javax.resource.cci.Interaction {

    ...

    boolean execute(InteractionSpec spec, Record input, Record output) throws ResourceException;

    Record execute(InteractionSpec spec, Record input) throws ResourceException;

    ...

}
```

根据调用的模板方法，`CciTemplate`知道在交互时调用哪个执行方法。 无论如何，正确初始化的 `InteractionSpec`实例是必需的。

您可以通过两种方式使用`CciTemplate.execute(..)` :

*   直接记录参数。在这种情况下，您只需要在CCI输入中传递记录， 返回的对象就是相应的CCI输出记录。
    
*   与应用程序对象一起使用记录映射。在这种情况下， 您需要提供相应的`RecordCreator` 和 `RecordExtractor`实例。
    

采用第一种方法，将使以下模板方法，这些方法直接对应于`Interaction`接口上的那些:

```java
public class CciTemplate implements CciOperations {

    public Record execute(InteractionSpec spec, Record inputRecord)
            throws DataAccessException { ... }

    public void execute(InteractionSpec spec, Record inputRecord, Record outputRecord)
            throws DataAccessException { ... }

}
```

对于第二种方式，你需要指定记录创建和记录获取策略作为参数。使用的接口是在[上一节中描述的记录转换](#cci-record-creator)。相应的`CciTemplate`方法如下:

```java
public class CciTemplate implements CciOperations {

    public Record execute(InteractionSpec spec,
            RecordCreator inputCreator) throws DataAccessException {
        // ...
    }

    public Object execute(InteractionSpec spec, Record inputRecord,
            RecordExtractor outputExtractor) throws DataAccessException {
        // ...
    }

    public Object execute(InteractionSpec spec, RecordCreator creator,
            RecordExtractor extractor) throws DataAccessException {
        // ...
    }

}
```

除非在模板上设置了`outputRecordCreator`属性(请参见下一节）， 负责每一个方法都将调用带有两个参数的CCI `Interaction`方法`InteractionSpec` 和input `Record`，接收输出记录作为返回值。

`CciTemplate`还提供了在`RecordCreator`实现之外创建`IndexRecord`和`MappedRecord`的方法，是通过其`createIndexRecord(..)`和`createMappedRecord(..)`方法。这可以在DAO实现中使用，以创建要传递到相应`CciTemplate.execute(..)`方法。以下清单显示了`CciTemplate`接口定义:

```java
public class CciTemplate implements CciOperations {

    public IndexedRecord createIndexedRecord(String name) throws DataAccessException { ... }

    public MappedRecord createMappedRecord(String name) throws DataAccessException { ... }

}
```

<a id="cci-using-dao"></a>

#### [](#cci-using-dao)5.2.3. DAO支持

Spring的CCI支持为DAO提供了一个抽象类，支持支持`ConnectionFactory`或`CciTemplate`实例的注入。该类的名称为`CciDaoSupport`：它提供了简单的`setConnectionFactory`和`setCciTemplate`方法。在内部， 此类将为传入的`ConnectionFactory`创建一个`CciTemplate`实例，并将其公开给子类中的具体数据访问实现。以下示例显示如何使用`CciDaoSupport`：

```java
public abstract class CciDaoSupport {

    public void setConnectionFactory(ConnectionFactory connectionFactory) {
        // ...
    }

    public ConnectionFactory getConnectionFactory() {
        // ...
    }

    public void setCciTemplate(CciTemplate cciTemplate) {
        // ...
    }

    public CciTemplate getCciTemplate() {
        // ...
    }

}
```

<a id="automatic-output-generation"></a>

#### [](#automatic-output-generation)5.2.4. 自动输出记录生成

如果连接器只支持使用输入和输出记录的`Interaction.execute(..)`方法作为参数（即它需要传递所需的输出记录，而不是返回适当的输出记录)，您可以设置`CciTemplate` 的`outputRecordCreator`属性，以便在收到响应时自动生成由JCA连接器填充的输出记录。此记录将被返回到模板的调用方。

此属性仅包含用于此目的的[`RecordCreator`](#cci-record-creator)接口的实现， 您必须直接在`CciTemplate`上指定`outputRecordCreator`属性。 以下示例显示了如何执行此操作：

```
cciTemplate.setOutputRecordCreator(new EciOutputRecordCreator());
```

或者（我们建议使用此方法），在Spring配置中，如果将`CciTemplate` 配置为专用bean实例，则可以按以下方式定义bean:

```xml
<bean id="eciOutputRecordCreator" class="eci.EciOutputRecordCreator"/>

<bean id="cciTemplate" class="org.springframework.jca.cci.core.CciTemplate">
    <property name="connectionFactory" ref="eciConnectionFactory"/>
    <property name="outputRecordCreator" ref="eciOutputRecordCreator"/>
</bean>
```

由于`CciTemplate`类是线程安全的，因此通常将其配置为共享实例。

<a id="template-summary"></a>

#### [](#template-summary)5.2.5. `CciTemplate` `Interaction` 总结

下表总结了`CciTemplate` 类的机制以及在CCI `Interaction`接口上调用的相应方法:

Table 9. 操作execute方法的使用  

|  `CciTemplate` 调用方法签名    |  `CciTemplate` `outputRecordCreator` 属性    |   `execute` 方法调用对CCI的操作   |
| ---- | ---- | ---- |
|  `Record execute(InteractionSpec, Record)`    |  Not set    | `Record execute(InteractionSpec, Record)`     |
|  `Record execute(InteractionSpec, Record)`    |  Set    | `boolean execute(InteractionSpec, Record, Record)`     |
|  void execute(InteractionSpec, Record, Record)    | Not set     | void execute(InteractionSpec, Record, Record)     |
| `void execute(InteractionSpec, Record, Record)`     | Set     |  `void execute(InteractionSpec, Record, Record)`    |
| `Record execute(InteractionSpec, RecordCreator)`     | Not set     |  `Record execute(InteractionSpec, Record)`    |
|  `Record execute(InteractionSpec, RecordCreator)`    |   Set   |  `void execute(InteractionSpec, Record, Record)`    |
|  `Record execute(InteractionSpec, Record, RecordExtractor)`    |  Not set    | `Record execute(InteractionSpec, Record)`     |
|   `Record execute(InteractionSpec, Record, RecordExtractor)`   |  Set    | `void execute(InteractionSpec, Record, Record)`     |
|   `Record execute(InteractionSpec, RecordCreator, RecordExtractor)`   |  Not set    |   `Record execute(InteractionSpec, Record)`   |
|  `Record execute(InteractionSpec, RecordCreator, RecordExtractor)`    |  Set    |   `void execute(InteractionSpec, Record, Record)`   |


<a id="cci-straight"></a>

#### [](#cci-straight)5.2.6. 直接使用CCI连接和交互

`CciTemplate` 也提供了与`JdbcTemplate` 和 `JmsTemplate`相同的方式直接使用CCI连接和交互。 例如，当您想要在CCI连接或交互上执行多个操作时，这非常有用。

`ConnectionCallback`接口提供了CCI `Connection`作为参数，用于执行自定义的操作，加到CCI `ConnectionFactory`的`Connection`创建。。 后者可用于获取关联的`RecordFactory`实例并创建索引/映射记录，以下清单显示了`ConnectionCallback`接口定义:

```java
public interface ConnectionCallback {

    Object doInConnection(Connection connection, ConnectionFactory connectionFactory)
            throws ResourceException, SQLException, DataAccessException;

}
```

接口`InteractionCallback`提供了CCI `Interaction`，是为了对其执行自定义操作，添加到相应的`ConnectionFactory`。以下清单显示了`InteractionCallback`接口定义：:

```java
public interface InteractionCallback {

    Object doInInteraction(Interaction interaction, ConnectionFactory connectionFactory)
        throws ResourceException, SQLException, DataAccessException;

}
```

`InteractionSpec`对象可以在多个模板调用之间共享，或者在每个回调方法中新创建。这完全由DAO实现来完成。

<a id="cci-template-example"></a>

#### [](#cci-template-example)5.2.7. `CciTemplate` 用法示例

在本节中，`CciTemplate`的使用将被显示为访问到具有ECI模式的CICS，使用IBM CICS ECI连接器。

首先, 必须对CCI `InteractionSpec`进行一些初始化，以指定要访问的CICS程序以及如何与之交互:

```
ECIInteractionSpec interactionSpec = new ECIInteractionSpec();
interactionSpec.setFunctionName("MYPROG");
interactionSpec.setInteractionVerb(ECIInteractionSpec.SYNC_SEND_RECEIVE);
```

然后，程序可以通过Spring模板使用CCI， 指定自定义对象和记录之间的映射。如下:

```java
public class MyDaoImpl extends CciDaoSupport implements MyDao {

    public OutputObject getData(InputObject input) {
        ECIInteractionSpec interactionSpec = ...;

    OutputObject output = (ObjectOutput) getCciTemplate().execute(interactionSpec,
        new RecordCreator() {
            public Record createRecord(RecordFactory recordFactory) throws ResourceException {
                return new CommAreaRecord(input.toString().getBytes());
            }
        },
        new RecordExtractor() {
            public Object extractData(Record record) throws ResourceException {
                CommAreaRecord commAreaRecord = (CommAreaRecord)record;
                String str = new String(commAreaRecord.toByteArray());
                String field1 = string.substring(0,6);
                String field2 = string.substring(6,1);
                return new OutputObject(Long.parseLong(field1), field2);
            }
        });

        return output;
    }
}
```

如前所述，您可以使用回调直接处理CCI连接或交互。 以下示例显示了如何执行此操作：

```java
public class MyDaoImpl extends CciDaoSupport implements MyDao {

    public OutputObject getData(InputObject input) {
        ObjectOutput output = (ObjectOutput) getCciTemplate().execute(
            new ConnectionCallback() {
                public Object doInConnection(Connection connection,
                        ConnectionFactory factory) throws ResourceException {

                    // do something...

                }
            });
        }
        return output;
    }

}
```

使用`ConnectionCallback`时，所使用的`Connection`由`CciTemplate`管理和关闭，但回调实现必须管理在连接上创建的任何交互。

对于更具体的回调，您可以实现`InteractionCallback`。 如果这样做，传入的`Interaction` 将由`CciTemplate`管理和关闭。 以下示例显示了如何执行此操作:

```java
public class MyDaoImpl extends CciDaoSupport implements MyDao {

    public String getData(String input) {
        ECIInteractionSpec interactionSpec = ...;
        String output = (String) getCciTemplate().execute(interactionSpec,
            new InteractionCallback() {
                public Object doInInteraction(Interaction interaction,
                        ConnectionFactory factory) throws ResourceException {
                    Record input = new CommAreaRecord(inputString.getBytes());
                    Record output = new CommAreaRecord();
                    interaction.execute(holder.getInteractionSpec(), input, output);
                    return new String(output.toByteArray());
                }
            });
        return output;
    }

}
```

对于前面的示例，所涉及的Spring bean的相应配置可能类似于非托管模式中的以下示例：:

```xml
<bean id="managedConnectionFactory" class="com.ibm.connector2.cics.ECIManagedConnectionFactory">
    <property name="serverName" value="TXSERIES"/>
    <property name="connectionURL" value="local:"/>
    <property name="userName" value="CICSUSER"/>
    <property name="password" value="CICS"/>
</bean>

<bean id="connectionFactory" class="org.springframework.jca.support.LocalConnectionFactoryBean">
    <property name="managedConnectionFactory" ref="managedConnectionFactory"/>
</bean>

<bean id="component" class="mypackage.MyDaoImpl">
    <property name="connectionFactory" ref="connectionFactory"/>
</bean>
```

在托管模式下（即在Java EE环境中），配置可能类似于以下示例：

```xml
<jee:jndi-lookup id="connectionFactory" jndi-name="eis/cicseci"/>

<bean id="component" class="MyDaoImpl">
    <property name="connectionFactory" ref="connectionFactory"/>
</bean>
```

<a id="cci-object"></a>

### [](#cci-object)5.3. 将CCI访问建模为操作对象

`org.springframework.jca.cci.object`包中包含的支持类允许你以另一种风格访问EIS：通过可重用的操作对象，类似于Spring的JDBC操作对象（参见[数据访问章节中的JDBC](data-access.html#jdbc))。它通常都封装了CCI的API，将应用级的输入对象传入到操作对象，从而它能创建输入record然后转换接收到的record数据到一个应用级输出对象并返回它。

这种方法内在地基于`CciTemplate`类和`RecordCreator`/`RecordExtractor`接口，重用了Spring核心CCI支持的机制。

<a id="cci-object-mapping-record"></a>

#### [](#cci-object-mapping-record)5.3.1. 使用 `MappingRecordOperation`

`MappingRecordOperation`本质上与`CciTemplate`做的事情是一样的，但是它表达了一个明确的、预配置（pre-configured）的操作作为对象。它提供了两个模板方法来指明如何将一个输入对象转换为输入记录，以及如何将一个输出记录转换为输出对象（记录映射）。:

*   `createInputRecord(..)`: 指定了如何将一个输入对象转换为输入`Record`。
    
*   `extractOutputData(..)`:指定了如何从输出`Record`中提取输出对象。
    

以下清单显示了这些方法的签名：

```java
public abstract class MappingRecordOperation extends EisOperation {

    ...

    protected abstract Record createInputRecord(RecordFactory recordFactory,
            Object inputObject) throws ResourceException, DataAccessException {
        // ...
    }

    protected abstract Object extractOutputData(Record outputRecord)
            throws ResourceException, SQLException, DataAccessException {
        // ...
    }

    ...

}
```

此后，为了执行一个EIS操作，你需要使用一个单独的`execute`方法，传递一个应用级（application-level）输入对象，并接收一个应用级输出对象作为结果。如下所示：:

```java
public abstract class MappingRecordOperation extends EisOperation {

    ...

    public Object execute(Object inputObject) throws DataAccessException {
    }

    ...
}
```

与`CciTemplate` 类相反，此`execute(..)`方法没有`InteractionSpec`作为参数。 相反，`InteractionSpec` 对于操作是全局的。 必须使用以下构造函数来实例化具有特定`InteractionSpec`的操作对象。 以下示例显示了如何执行此操作:

```java
InteractionSpec spec = ...;
MyMappingRecordOperation eisOperation = new MyMappingRecordOperation(getConnectionFactory(), spec);
...
```

<a id="cci-object-mapping-comm-area"></a>

#### [](#cci-object-mapping-comm-area)5.3.2. 使用 `MappingCommAreaOperation`

一些连接器使用了基于COMMAREA的记录，该记录包含了发送给EIS的参数和返回数据的字节数组。Spring提供了一个专门的操作类用于直接操作COMMAREA而不是操作记录。 `MappingCommAreaOperation`类扩展了`MappingRecordOperation`类以提供这种专门的COMMAREA支持。它隐含地使用了`CommAreaRecord`类作为输入和输出record类型，并提供了两个新的方法来转换输入对象到输入COMMAREA，以及转换输出COMMAREA到输出对象。以下清单显示了相关的方法签名：:

```java
public abstract class MappingCommAreaOperation extends MappingRecordOperation {

    ...

    protected abstract byte[] objectToBytes(Object inObject)
            throws IOException, DataAccessException;

    protected abstract Object bytesToObject(byte[] bytes)
        throws IOException, DataAccessException;

    ...

}
```

<a id="cci-automatic-record-gen"></a>

#### [](#cci-automatic-record-gen)5.3.3. 自动的输出记录生成

由于每个`MappingRecordOperation`子类的内部都是基于`CciTemplate`的，所以用CciTemplate以相同的方式自动生成输出record都是有效的。每个操作对象提供一个相应的`setOutputRecordCreator(..)`方法，有关详细信息，请参阅[自动输出记录生成](#automatic-output-generation)。

<a id="cci-object-summary"></a>

#### [](#cci-object-summary)5.3.4. 总结

操作对象方法使用了跟`CciTemplate`相同的方式来使用记录。

Table 10. Interaction的execute方法的使用  

|  `MappingRecordOperation` 方法签名    |  `MappingRecordOperation` `outputRecordCreator` 属性    | 在CCI Interaction上调用的 `execute` 方法  |
| ---- | ---- | ---- |
|  `Object execute(Object)`    |  Not set    |  `Record execute(InteractionSpec, Record)`    |
|  `Object execute(Object)`    | Set     |  `boolean execute(InteractionSpec, Record, Record)`    |


<a id="cci-objects-mappring-record-example"></a>

#### [](#cci-objects-mappring-record-example)5.3.5. `MappingRecordOperation` 使用例子

在本节中，我们将展示如何使用`MappingRecordOperation` 访问具有Blackbox CCI连接器的数据库。

此连接器的原始版本由Java EE SDK（版本1.3）提供，可从Oracle获得。

首先，必须对CCI `InteractionSpec`进行一些初始化，在下面的示例中，我们直接定义将请求的参数转换为CCI记录的方式以及将CCI结果记录转换为`Person`类的实例的方法：:

```java
public class PersonMappingOperation extends MappingRecordOperation {

    public PersonMappingOperation(ConnectionFactory connectionFactory) {
        setConnectionFactory(connectionFactory);
        CciInteractionSpec interactionSpec = new CciConnectionSpec();
        interactionSpec.setSql("select * from person where person_id=?");
        setInteractionSpec(interactionSpec);
    }

    protected Record createInputRecord(RecordFactory recordFactory,
            Object inputObject) throws ResourceException {
        Integer id = (Integer) inputObject;
        IndexedRecord input = recordFactory.createIndexedRecord("input");
        input.add(new Integer(id));
        return input;
    }

    protected Object extractOutputData(Record outputRecord)
            throws ResourceException, SQLException {
        ResultSet rs = (ResultSet) outputRecord;
        Person person = null;
        if (rs.next()) {
            Person person = new Person();
            person.setId(rs.getInt("person_id"));
            person.setLastName(rs.getString("person_last_name"));
            person.setFirstName(rs.getString("person_first_name"));
        }
        return person;
    }
}
```

然后应用程序会以person标识符作为参数来得到操作对象。注意，操作对象可以被设为共享实例，因为它是线程安全的。 以下以person标识符作为参数执行操作对象:

```java
public class MyDaoImpl extends CciDaoSupport implements MyDao {

    public Person getPerson(int id) {
        PersonMappingOperation query = new PersonMappingOperation(getConnectionFactory());
        Person person = (Person) query.execute(new Integer(id));
        return person;
    }
}
```

在非托管模式下，Spring bean的相应配置如下：

```xml
<bean id="managedConnectionFactory"
        class="com.sun.connector.cciblackbox.CciLocalTxManagedConnectionFactory">
    <property name="connectionURL" value="jdbc:hsqldb:hsql://localhost:9001"/>
    <property name="driverName" value="org.hsqldb.jdbcDriver"/>
</bean>

<bean id="targetConnectionFactory"
        class="org.springframework.jca.support.LocalConnectionFactoryBean">
    <property name="managedConnectionFactory" ref="managedConnectionFactory"/>
</bean>

<bean id="connectionFactory"
        class="org.springframework.jca.cci.connection.ConnectionSpecConnectionFactoryAdapter">
    <property name="targetConnectionFactory" ref="targetConnectionFactory"/>
    <property name="connectionSpec">
        <bean class="com.sun.connector.cciblackbox.CciConnectionSpec">
            <property name="user" value="sa"/>
            <property name="password" value=""/>
        </bean>
    </property>
</bean>

<bean id="component" class="MyDaoImpl">
    <property name="connectionFactory" ref="connectionFactory"/>
</bean>
```

在托管模式下（即在Java EE环境中），配置可能如下所示：

```xml
<jee:jndi-lookup id="targetConnectionFactory" jndi-name="eis/blackbox"/>

<bean id="connectionFactory"
        class="org.springframework.jca.cci.connection.ConnectionSpecConnectionFactoryAdapter">
    <property name="targetConnectionFactory" ref="targetConnectionFactory"/>
    <property name="connectionSpec">
        <bean class="com.sun.connector.cciblackbox.CciConnectionSpec">
            <property name="user" value="sa"/>
            <property name="password" value=""/>
        </bean>
    </property>
</bean>

<bean id="component" class="MyDaoImpl">
    <property name="connectionFactory" ref="connectionFactory"/>
</bean>
```

<a id="cci-objects-mapping-comm-area-example"></a>

#### [](#cci-objects-mapping-comm-area-example)5.3.6. `MappingCommAreaOperation` 的使用例子

在本节中，我们将展示如何使用`MappingCommAreaOperation`来使用IBM CICS ECI连接器访问具有ECI模式的CICS。

首先，我们需要初始化CCI `InteractionSpec`以指定要访问的CICS程序以及如何与之交互，如以下示例所示：:

```java
public abstract class EciMappingOperation extends MappingCommAreaOperation {

    public EciMappingOperation(ConnectionFactory connectionFactory, String programName) {
        setConnectionFactory(connectionFactory);
        ECIInteractionSpec interactionSpec = new ECIInteractionSpec(),
        interactionSpec.setFunctionName(programName);
        interactionSpec.setInteractionVerb(ECIInteractionSpec.SYNC_SEND_RECEIVE);
        interactionSpec.setCommareaLength(30);
        setInteractionSpec(interactionSpec);
        setOutputRecordCreator(new EciOutputRecordCreator());
    }

    private static class EciOutputRecordCreator implements RecordCreator {
        public Record createRecord(RecordFactory recordFactory) throws ResourceException {
            return new CommAreaRecord();
        }
    }

}
```

然后我们可以将抽象的 `EciMappingOperation`类子类化为指定自定义对象和Records之间的映射，如下例所示:

```java
public class MyDaoImpl extends CciDaoSupport implements MyDao {

    public OutputObject getData(Integer id) {
        EciMappingOperation query = new EciMappingOperation(getConnectionFactory(), "MYPROG") {

            protected abstract byte[] objectToBytes(Object inObject) throws IOException {
                Integer id = (Integer) inObject;
                return String.valueOf(id);
            }

            protected abstract Object bytesToObject(byte[] bytes) throws IOException;
                String str = new String(bytes);
                String field1 = str.substring(0,6);
                String field2 = str.substring(6,1);
                String field3 = str.substring(7,1);
                return new OutputObject(field1, field2, field3);
            }
        });

        return (OutputObject) query.execute(new Integer(id));
    }

}
```

在非托管模式下，Spring bean的相应配置如下：

```xml
<bean id="managedConnectionFactory" class="com.ibm.connector2.cics.ECIManagedConnectionFactory">
    <property name="serverName" value="TXSERIES"/>
    <property name="connectionURL" value="local:"/>
    <property name="userName" value="CICSUSER"/>
    <property name="password" value="CICS"/>
</bean>

<bean id="connectionFactory" class="org.springframework.jca.support.LocalConnectionFactoryBean">
    <property name="managedConnectionFactory" ref="managedConnectionFactory"/>
</bean>

<bean id="component" class="MyDaoImpl">
    <property name="connectionFactory" ref="connectionFactory"/>
</bean>
```

在托管模式下（即在Java EE环境中），配置可能如下所示：

```xml
<jee:jndi-lookup id="connectionFactory" jndi-name="eis/cicseci"/>

<bean id="component" class="MyDaoImpl">
    <property name="connectionFactory" ref="connectionFactory"/>
</bean>
```

<a id="cci-tx"></a>

### [](#cci-tx)5.4. 事务

JCA为资源适配器（resource adapters)指定了几个级别的事务支持。你可以在`ra.xml`文件中指定你的资源适配器支持的事务类型。它本质上有三个选项：none（例如，使用CICS EPI连接器），本地事务（例如，使用CICS ECI连接器）和全局事务（例如，使用IMS连接器）。 以下示例配置全局选项：:

```xml
<connector>
    <resourceadapter>
        <!-- <transaction-support>NoTransaction</transaction-support> -->
        <!-- <transaction-support>LocalTransaction</transaction-support> -->
        <transaction-support>XATransaction</transaction-support>
    <resourceadapter>
<connector>
```

对于全局事务，您可以使用Spring的通用事务基础结构来划分事务，使用`JtaTransactionManager`作为后端（委托给下面的Java EE服务器的分布式事务协调器）。

对于单个CCI `ConnectionFactory`上的本地事务，Spring为CCI提供了特定的事务管理策略，类似于JDBC的`DataSourceTransactionManager`。CCI API定义了本地事务对象和相应的本地事务划分方法。 Spring的`CciLocalTransactionManager`执行这样的本地CCI事务，完全依照Spring中常见的`PlatformTransactionManager`抽象。以下示例配置 `CciLocalTransactionManager`：

```xml
<jee:jndi-lookup id="eciConnectionFactory" jndi-name="eis/cicseci"/>

<bean id="eciTransactionManager"
        class="org.springframework.jca.cci.connection.CciLocalTransactionManager">
    <property name="connectionFactory" ref="eciConnectionFactory"/>
</bean>
```

您可以将这两种事务策略与Spring的任何事务划分工具一起使用，无论是声明性的还是编程式的。这是Spring通用的`PlatformTransactionManager`抽象的结果，它解耦了实际运行策略中的事务划分。你可以保持现在的事务划分，只需要在`JtaTransactionManager`和`CciLocalTransactionManager`之间转换即可。

有关Spring的事务处理机制的更多信息，请参阅[事务管理](data-access.html#transaction)。