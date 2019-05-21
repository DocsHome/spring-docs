<a id="mail"></a>

[](#mail)6\. 电子邮件
-----------------

本节介绍如何使用Spring Framework发送电子邮件。

依赖库

以下JAR需要位于应用程序的类路径中才能使用Spring Framework的电子邮件库:

*   The [JavaMail](https://javaee.github.io/javamail/) library
    

该库可在Web上免费获取 - 例如，在Maven Central中以`com.sun.mail:javax.mail`的形式提供。 .

Spring提供了一个发送电子邮件的高级抽象层，它向用户屏蔽了底层邮件系统的一些细节，同时代表客户端负责底层的资源处理。

`org.springframework.mail`包是Spring Framework电子邮件支持的主要包，它包括了发送电子邮件的主要接口 `MailSender`，和值对象`SimpleMailMessage`，它封装了简单邮件的属性如`from` 和 `to`（以及许多其他邮件）。 此程序包还包含已检查异常的层次结构，这些异常提供较低级别邮件系统异常的更高抽象级别，根异常为`MailException`。 有关富邮件异常层次结构的详细信息，请参阅[javadoc](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/mail/MailException.html)。

为了使用JavaMail中的一些特色，比如MIME类型的信件，Spring提供了`MailSender`的一个子接口（内嵌了的)，即`org.springframework.mail.javamail.JavaMailSender`。`JavaMailSender`还提供了一个回调接口`org.springframework.mail.javamail.MimeMessagePreparator`，用于准备`MimeMessage`。

<a id="mail-usage"></a>

### [](#mail-usage)6.1. Usage

假设我们有一个名为`OrderManager`的业务接口，如下例所示：

```java
public interface OrderManager {

    void placeOrder(Order order);

}
```

进一步假设我们要求说明需要生成带有订单号的电子邮件消息并将其发送给下达相关订单的客户。

<a id="mail-usage-simple"></a>

#### [](#mail-usage-simple)6.1.1. `MailSender`与 `SimpleMailMessage`的基本用法

以下示例显示了当有人下订单时如何使用`MailSender`和`SimpleMailMessage`发送电子邮件:

```java
import org.springframework.mail.MailException;
import org.springframework.mail.MailSender;
import org.springframework.mail.SimpleMailMessage;

public class SimpleOrderManager implements OrderManager {

    private MailSender mailSender;
    private SimpleMailMessage templateMessage;

    public void setMailSender(MailSender mailSender) {
        this.mailSender = mailSender;
    }

    public void setTemplateMessage(SimpleMailMessage templateMessage) {
        this.templateMessage = templateMessage;
    }

    public void placeOrder(Order order) {

        // Do the business calculations...

        // Call the collaborators to persist the order...

        // Create a thread safe "copy" of the template message and customize it
        SimpleMailMessage msg = new SimpleMailMessage(this.templateMessage);
        msg.setTo(order.getCustomer().getEmailAddress());
        msg.setText(
            "Dear " + order.getCustomer().getFirstName()
                + order.getCustomer().getLastName()
                + ", thank you for placing order. Your order number is "
                + order.getOrderNumber());
        try{
            this.mailSender.send(msg);
        }
        catch (MailException ex) {
            // simply log it and go on...
            System.err.println(ex.getMessage());
        }
    }

}
```

以下示例显示了上述代码的bean定义：

```xml
<bean id="mailSender" class="org.springframework.mail.javamail.JavaMailSenderImpl">
    <property name="host" value="mail.mycompany.com"/>
</bean>

<!-- this is a template message that we can pre-load with default state -->
<bean id="templateMessage" class="org.springframework.mail.SimpleMailMessage">
    <property name="from" value="[email protected]"/>
    <property name="subject" value="Your order"/>
</bean>

<bean id="orderManager" class="com.mycompany.businessapp.support.SimpleOrderManager">
    <property name="mailSender" ref="mailSender"/>
    <property name="templateMessage" ref="templateMessage"/>
</bean>
```

<a id="mail-usage-mime"></a>

#### [](#mail-usage-mime)6.1.2. 使用 `JavaMailSender` 和 `MimeMessagePreparator`

本节描述了使用`MimeMessagePreparator`回调接口的`OrderManager`的另一个实现。 在以下示例中，`mailSender`属性的类型为`JavaMailSender`，以便我们能够使用JavaMail `MimeMessage`类：

```java
import javax.mail.Message;
import javax.mail.MessagingException;
import javax.mail.internet.InternetAddress;
import javax.mail.internet.MimeMessage;

import javax.mail.internet.MimeMessage;
import org.springframework.mail.MailException;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessagePreparator;

public class SimpleOrderManager implements OrderManager {

    private JavaMailSender mailSender;

    public void setMailSender(JavaMailSender mailSender) {
        this.mailSender = mailSender;
    }

    public void placeOrder(final Order order) {
        // Do the business calculations...
        // Call the collaborators to persist the order...

        MimeMessagePreparator preparator = new MimeMessagePreparator() {
            public void prepare(MimeMessage mimeMessage) throws Exception {
                mimeMessage.setRecipient(Message.RecipientType.TO,
                        new InternetAddress(order.getCustomer().getEmailAddress()));
                mimeMessage.setFrom(new InternetAddress("[email protected]"));
                mimeMessage.setText("Dear " + order.getCustomer().getFirstName() + " " +
                        order.getCustomer().getLastName() + ", thanks for your order. " +
                        "Your order number is " + order.getOrderNumber() + ".");
            }
        };

        try {
            this.mailSender.send(preparator);
        }
        catch (MailException ex) {
            // simply log it and go on...
            System.err.println(ex.getMessage());
        }
    }

}
```

以上的邮件代码是一个横切关注点，能被完美地重构为[自定义Spring AOP切面](core.html#aop)的候选者，这样它就可以在目标对象`OrderManager`的一些合适的连接点（joinpoint)中被执行了。

Spring Framework的邮件支持附带标准的JavaMail实现。 有关更多信息，请参阅相关的javadoc。

<a id="mail-javamail-mime"></a>

### [](#mail-javamail-mime)6.2. 使用JavaMail `MimeMessageHelper`

`org.springframework.mail.javamail.MimeMessageHelper`是处理JavaMail邮件时比较顺手组件之一。它可以让你摆脱繁复的JavaMail API。通过使用`MimeMessageHelper`，创建一个`MimeMessage`实例将非常容易。如下例所示：

```java
// of course you would use DI in any real-world cases
JavaMailSenderImpl sender = new JavaMailSenderImpl();
sender.setHost("mail.host.com");

MimeMessage message = sender.createMimeMessage();
MimeMessageHelper helper = new MimeMessageHelper(message);
helper.setTo("[email protected]");
helper.setText("Thank you for ordering!");

sender.send(message);
```

<a id="mail-javamail-mime-attachments-attachment"></a>

#### [](#mail-javamail-mime-attachments)6.2.1. 发送附件和内部资源

Multipart email 允许添加附件和内嵌资源(inline resources)。内嵌资源可能是你在信件中希望使用的图像或样式表，但是又不想把它们作为附件。

<a id="mail-javamail-mime-attachments-attachment"></a>

##### [](#mail-javamail-mime-attachments-attachment)附件

以下示例显示如何使用`MimeMessageHelper`发送包含单个JPEG图像附件的电子邮件:

```java
JavaMailSenderImpl sender = new JavaMailSenderImpl();
sender.setHost("mail.host.com");

MimeMessage message = sender.createMimeMessage();

// use the true flag to indicate you need a multipart message
MimeMessageHelper helper = new MimeMessageHelper(message, true);
helper.setTo("[email protected]");

helper.setText("Check out this image!");

// let's attach the infamous windows Sample file (this time copied to c:/)
FileSystemResource file = new FileSystemResource(new File("c:/Sample.jpg"));
helper.addAttachment("CoolImage.jpg", file);

sender.send(message);
```

<a id="mail-javamail-mime-attachments-inline"></a>

##### [](#mail-javamail-mime-attachments-inline)内嵌资源

以下示例显示如何使用`MimeMessageHelper`发送带有内嵌图像的电子邮件：

```java
JavaMailSenderImpl sender = new JavaMailSenderImpl();
sender.setHost("mail.host.com");

MimeMessage message = sender.createMimeMessage();

// use the true flag to indicate you need a multipart message
MimeMessageHelper helper = new MimeMessageHelper(message, true);
helper.setTo("[email protected]");

// use the true flag to indicate the text included is HTML
helper.setText("<html><body><img src='cid:identifier1234'></body></html>", true);

// let's include the infamous windows Sample file (this time copied to c:/)
FileSystemResource res = new FileSystemResource(new File("c:/Sample.jpg"));
helper.addInline("identifier1234", res);

sender.send(message);
```

通过使用指定的 `Content-ID`（上例中的`identifier1234`）将内联资源添加到`MimeMessage`。 添加文本和资源的顺序非常重要。 请务必先添加文本，然后再添加资源。 如果你反过来这样做，它就行不通。

<a id="mail-templates"></a>

#### [](#mail-templates)6.2.2. 使用模板库创建电子邮件内容

在之前的代码示例中，所有邮件的内容都是显式定义的，并通过调用`message.setText(..)`来设置邮件内容。 这种做法针对简单的情况或在上述的例子中没什么问题。因为在这里只是为了向你展示基础API。

但是，在典型的企业应用程序中，开发人员通常不会使用之前显示的方法创建电子邮件内容，原因如下:

*   使用Java代码创建基于HTML的电子邮件内容非常繁琐且容易出错。
    
*   显示逻辑和业务逻辑之间没有明确的区别。
    
*   更改电子邮件内容的显示结构需要编写Java代码，重新编译，重新部署等。
    

通常，解决这些问题的方法是使用模板库（例如FreeMarker）来定义电子邮件内容的显示结构。这使您的代码仅负责创建要在电子邮件模板中呈现的数据并发送电子邮件。 当您的电子邮件内容变得相当复杂时，这绝对是一种最佳实践，而且，使用Spring Framework的FreeMarker支持类，它变得非常容易。