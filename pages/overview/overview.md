<a id="overview"></a>

# Spring Framework 概述

**Spring 使创建 Java 企业应用程序变得更加容易。它提供了在企业环境中接受 Java 语言所需的一切,，并支持 Groovy 和 Kotlin 作为 JVM 上的替代语言，并可根据应用程序的需要灵活地创建多种体系结构。
从 Spring Framework 5.0 开始，Spring 需要 JDK 8(Java SE 8+)，并且已经为 JDK 9 提供了现成的支持。**

Spring支持各种应用场景， 在大型企业中, 应用程序通常需要运行很长时间，而且必须运行在 jdk 和应用服务器上，这种场景开发人员无法控制其升级周期。 其他可能作为一个单独的jar嵌入到服务器去运行，也有可能在云环境中。还有一些可能是不需要服务器的独立应用程序(如批处理或集成的工作任务)。

Spring 是开源的。它拥有一个庞大而且活跃的社区，提供不同范围的，真实用户的持续反馈。这也帮助Spring不断地改进,不断发展。

<a id="overview-spring"></a>

## 1、"Spring"是什么？

Spring在不同的背景下有这不同的含义。它可以指 Spring Framework项目本身，这也是创建他的初衷。随着时间的推移，其他的"Spring" 项目已经构建在Spring框架之上。大多数时候，当人们说 "Spring"，他们指的是整个项目家族。本参考文档主要侧重于Spring的基本：也就是Spring框架本身。

整个Spring框架被分成多个模块，应用程序可以选择需要的部分。core是核心容器的模块，包括模块配置和依赖注入机制 。Spring框架还为不同的应用程序体系结构提供了基础支持，包括消息传递，事务数据和持久化以及Web，还包括基于Servlet的Spring MVC Web框架 ，以及Spring WebFlux响应式Web框架。

关于模块的说明: Spring Framework 的jar包允许部署到JDK 9的模块路径("Jigsaw")。为了在支持Jigsaw的应用程序中使用， Spring Framework  5的jar带有 "Automatic-Module-Name" 清单条目，它定义了稳定的语言级模块名称(例如"spring.code","spring.context"等)独立于jar部件名称(jar遵循相同的命名模式使用"-"号代替"."， 例如"spring-core" 和 "spring-context")。当然，Spring Framework 的jar包在JDK 8和9的类路径上都能保持正常工作。

<a id="overview-history"></a>

## 2、Spring和Spring框架的历史

Spring 的初版发布在2003年，是为了克服早期 J2EE 规范的复杂性。虽然有些人认为 [J2EE](https://en.wikipedia.org/wiki/Java_Platform,_Enterprise_Edition) 和 Spring 是竞争的，但是Spring 实际上是对 Java EE 的补充。Spring编程模型不受 Java EE 的平台制约，相反 它与精心挑选的个别规范的java EE项 目结合：

- Servlet API ([JSR 340](https://jcp.org/en/jsr/detail?id=340))

- WebSocket API ([JSR 356](https://www.jcp.org/en/jsr/detail?id=356))

- Concurrency Utilities ([JSR 236](https://www.jcp.org/en/jsr/detail?id=236))

- JSON Binding API ([JSR 367](https://jcp.org/en/jsr/detail?id=367))

- Bean Validation ([JSR 303](https://jcp.org/en/jsr/detail?id=303))

- JPA ([JSR 338](https://jcp.org/en/jsr/detail?id=338))

- JMS ([JSR 914](https://jcp.org/en/jsr/detail?id=914))

- as well as JTA/JCA setups for transaction coordination, if necessary.

Spring框架还支持依赖注入([JSR 330](https://www.jcp.org/en/jsr/detail?id=330))和通用注解([JSR 250](https://jcp.org/en/jsr/detail?id=250)) 规范， 应用程序开发人员可以选择使用这些规范，而不是Spring Framework提供的Spring特定机制。

在Spring框架5.0版本中，Spring最低要求使用Java EE 7的版本（例如Servlet 3.1+, JPA 2.1+)，同时在运行时能与使用Java EE 8的最新API集成（例如Servlet 4.0, JSON Binding API）。 这使得Spring能完全兼容Tomcat 8/9、WebSphere 9或者JBoss EAP 7等等。

随着时间的不断推移，Java EE在应用程序开发中越发重要，也不断发展、改善。在Java EE和Spring的早期，应用程序被创建为部署到服务器的应用。如今，在有Spring Boot的帮助后，应用可以创建在devops或云端。 而Servlet容器的嵌入和一些琐碎的东西也发生了变化。在Spring框架5中，WebFlux应用程序甚至可以不直接使用Servlet的API，并且可以在非Servlet容器的服务器（如Netty）上运行。

Spring还在继续创新和发展，如今，除了Spring框架以外，还加入了其他项目，如：`Spring Boot`, `Spring Security`, `Spring Data`, `Spring Cloud`, `Spring Batch` 等等。请记住，每一个Spring项目都有自己的源代码库， 问题跟踪以及发布版本。请上[spring.io/projects](https://spring.io/projects)查看所有Spring家族的项目名单。



<a id="overview-philosophy"></a>

## 3、Spring的设计理念

当你学习一个框架时，不仅需要知道他是如何运作的，更需要知道他遵循什么样的原则，以下是 Spring 框架遵循的原则：

- 提供各个层面的选择。Spring 允许您尽可能延迟设计决策。例如，您可以在不更改代码的情况下通过配置切换持久性功能。对于其他基础架构的问题以及与第三方API的集成也是如此。

- 包含多个使用视角。Spring 的灵活性非常高,而不是规定了某部分只能做某一件事.他以不同的视角支持广泛的应用需求

- 保持向后兼容。Spring 的发展经过了精心的设计，在不同版本之间保持与前版本的兼容。Spring 支持一系列精心挑选的 JDK 版本和第三方库，以方便维护依赖于 Spring 的应用程序和库。

- 关心API的设计。Spring团队投入了大量的思考和时间来制作直观的API,并在许多版本和许多年中都保持不变。

- 高标准的代码质量。Spring 框架提供了强有力的、精确的、即时的Javadoc。Spring这种要求干净、简洁的代码结构、包之间没有循环依赖的项目在Java界是少有的。


<a id="overview-feedback"></a>

## 4、反馈和贡献

对于操作方法问题或诊断或调试问题，我们建议使用 Stackoverflow。Spring 在此网站上有一个专用的页面[questions page](https://spring.io/questions) ，列出了建议使用的标记。 如果你相当确定 Spring 框架存在问题，或者想建议添加功能等等，可以使用 [JIRA 的问题追踪](https://jira.spring.io/browse/spr)。

如果你想到了某问题的解决方案或者想建议，你可以在 [Github](https://github.com/spring-projects/spring-framework) 上提交请求。但是， 我们希望在此之前,你可以在问题跟踪页面先提交一波，在那里可以进行讨论并留下记录以作参考。

有关更多详情，请参阅 [CONTRIBUTING](https://github.com/spring-projects/spring-framework/blob/master/CONTRIBUTING.md) 项目页面

<a id="overview-getting-started"></a>

## 5、入门指南

如果你是刚开始使用 Spring 的话，希望你能使用基于 [Spring Boot](https://projects.spring.io/spring-boot/) 来编写应用程序,由此开始使用Spring框架。Spring Boot提供了一种快速（固定设置 ）的方式来创建即时能用的 Spring 应用程序。它基于Spring框架，利用已有的约束配置，目的是让你的程序尽快跑起来。

你可以使用 [start.spring.io](https://start.spring.io/) 里的步骤来生成基本项目。或者参考 ["Getting Started" guides](https://spring.io/guides) ，例如开始创建RESTful风格的Web服务 。除了易于理解，这些指南都是非常注重任务，其中大多数是基于Spring Boot的。当然,里面还囊括了 Spring 其他很多项目，你可能在需要某些功能时用得上。