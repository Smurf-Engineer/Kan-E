## SOFA Boot 框架文档

[![Build Status](https://travis-ci.org/alipay/sofa-boot.svg?branch=master)](https://travis-ci.org/alipay/sofa-boot)
[![Coverage Status](https://coveralls.io/repos/github/alipay/sofa-boot/badge.svg?branch=master)](https://coveralls.io/github/alipay/sofa-boot?branch=master)
[![Gitter](https://img.shields.io/badge/chat-on%20gitter-orange.svg)](https://gitter.im/alipay/sofa-boot)
![license](https://img.shields.io/badge/license-Apache--2.0-green.svg)
![maven](https://img.shields.io/badge/maven-2.3.0-green.svg)

## 一、背景

在我们使用 Java 的日常开发过程中，可能经常遇到一些类冲突问题，经常要解决可能潜在的 Jar 版本冲突问题，同时，不同的应用使用的 Web 容器、开源框架和一些基础的依赖集合都可能不尽相同，如果我们没有一个统一的开发框架来规范和管控大家的开源框架或者基础依赖的版本，那么大家在遇到一些业务场景时比如架构层面某一个能力在全部应用的支持，大家就需要各自去升级指定的依赖版本，这个时候如果没有框架层面做统一约束的话，人力的投入和升级的周期都可能导致最终目标的达成。同时，很多业务应用也期望存在一个统一的开发框架来统一大家的编程接口，并在框架层面提供一些基础能力如健康检查、类隔离和 Spring 上下文隔离能力，除此之外对一些基础依赖的管控也能最大程度的避免引入组件之间的类冲突并统一大家的基础依赖。

基于此，我们在 Spring Boot 上构建了我们自己的开发框架 SOFA Boot，旨在提供一个集成我们金融级中间件并提供统一的编程接口、保证可维护性并构建在开源之上的开发框架，同时在框架层面提供统一的依赖管控、健康检查、类隔离和其他相应框架能力。

## 二、框架功能介绍

SOFA Boot 框架构建在 Spring Boot 之上并完全兼容 Spring Boot 的生态，主要提供了下面的相应能力。

### 2.1 健康检查能力

* 在分布式环境下，我们的应用启动完毕并不意味着我们的应用就是健康的，如果应用此时并非健康，但是一些流量的请求调用到当前应用或者当前应用发起对其他系统的调用均可能导致不可预期的行为。基于此，我们基于 Spring Boot 的健康检查在 SOFA Boot 上扩展了健康检查能力，除了在原有的 Spring Boot 上提供运行时的健康检查（Liveness Check），我们健康检查还提供了启动期的健康检查能力（Readiness Check）。

* 将启动期间的健康检查和运行期间的健康检查区分开来，一个重要的原因就是保证我们健康状态的一致性。如果启动期和运行期检查状态没有加以区分，就会出现类似如下的一个场景：应用启动一开始检查状态提示成功后，然后紧接着就可能发布相应的服务到服务注册中心，与此同时启动过程中会有类似技术栈（或者说启动脚本）去检查应用的健康状态，这个时候可能有组件也在动态注册自己的检查状态，可能注册的就是失败，而技术栈拿到的检查状态就是失败，那么认为应用发布失败，但是问题就发生了，最开始启动期间的检查成功导致服务注册行为的发生，这个时候应用就可能被调用者发现并不断的发生服务调用，但是我们的发布平台却认为发布是失败的。
* 将启动期间和运行期间的健康检查区分开，启动检查成功我们就认为成功，避免应用的检查状态和实际行为发生不一致，从而造成业务损失；而运行期间的健康检查状态我们可以通过类似 Metrics 或者上报的方式来随时关注应用在每个阶段的将康状态。

### 2.2 日志隔离能力

在日常开发中，应用避免不了都会打印日志，我们可能采用的通用日志接口开发框架 SLF4J，而对于具体的日志实现我们可以选择 Logback、Log4j2 或者 Log4j。这里面就会面临一个问题，在同样的一个 class path 下，都是由同一个 ClassLoader 加载的类，如何保证我们开发框架或者中间件的日志实现和业务期望使用的日志实现不冲突呢？我们在框架层面提供了解决方案，即我们的框架或者中间件也只面向日志编程接口 SLF4J 去编程而不去依赖具体的日志实现，具体的日志实现的选择权利交给应用开发者去选择，应用选择哪一个日志实现，我们的框架就选择此日志实现进行打印，而具体的日志实现的选择以及能够自由切换日志实现的能力均是通过框架层面提供的基础功能来达到目标。

### 2.3 中间件的集成管理

基于 Spring Boot 的自动配置能力，提供我们中间件统一易用的编程接口，每一个蚂蚁金服中间件都是独立可插拔的组件，节约开发时间，和后期维护的成本。同时，SOFA Boot 提供蚂蚁金服中间件提供轻量级集成方案，即用户只需要引入对应中间件的 starter ，SOFA Boot 会自动导入所需的依赖并完成必要的启动初始化工作。

### 2.4 提供类隔离的机制

SOFA Ark 是一个在 SOFA Boot 之上基于 Java 实现的轻量级类隔离加载容器。为了解决依赖冲突或者类冲突的问题，构建在 SOFA Boot 之上的类隔离容器 SOFA Ark 为应用程序提供了类隔离能力，通过 SOFA Ark 既可以解决各个中间件插件（或者说 starter）之间的类冲突问题，也可以解决中间件插件（或者说 starter）和业务应用所依赖类的冲突问题。通过 ClassLoader 技术，SOFA Ark 将每一个插件（或者 starter）和业务应用均采用独立的 ClassLoader 实例进行加载来解决中间件和中间件之前以及中间件和业务之间的类冲突问题。应用运行在 SOFA Ark 之上，借助容器插件化隔离达到依赖包的隔离，同时 SOFA Ark 管理插件初始化和应用启动，多应用、多插件相互隔离以及插件的整个生命周期过程。

### 2.5 提供 Spring 上下文隔离能力

日常开发中，一个应用会有多个 JAR 模块，但是一个应用却一般只会有一个 Spring 上下文，这种模块化策略的劣势在于，假设随着业务的发展，应用程序变得越来越庞大，那么应用的 SOA 化势在必行，需要将应用的模块继续拆分成独立的应用，在这种方式下 JAR 模块的拆分比较容易，但是 Spring 配置拆分却十分繁琐，需要从每一个配置文件中区分出来。基于此 SOFA Boot 中提供了 Spring 上下文隔离的能力，应用中的每一个符合规范的模块都会被扫描到并被加载为一个独立的 Spring 上下文，模块之间的调用通过本地服务的方式来完成。这样，当应用需要拆分的时候，可以将整个模块连同它的 Spring 配置文件直接拆出去，所需要修改的只是将本地服务，改成远程服务，十分方便。

> 待开源

### 2.6 多维度应用度量

提供多种度量维度实时监测应用程序的性能，能帮助更好的了解当前应用程序或者服务在线上的各种性能状态。

> 待开源

### 2.7 调用链路监控及治理

SOFA Boot 集成基于 OpenTracing 标准的日志埋点工具 Tracer，提供统一的中间件日志埋点和上下文 ID，通过一个统一的 ID，将上下游系统的调用关系串联起来。通过 Tracer 异步日志组件，将调用链路中的各种网络调用情况以日志的方式记录下来，以达到透视化网络调用的目的。这些日志可用于故障的快速发现，服务治理等用途。

> 待开源

### 2.8 兼容

SOFA Boot 基于 Spring Boot 构建并完全兼容社区的使用方法，同时在其上提供额外的框架扩展能力和中间件能力。SOFA Boot 应用可与 Spring Boot 工程无缝集成，大大减少了用户的迁移成本，同样可以将应用打包成 FAT JAR 的方式进行运行，同时也支持在多种 Servlet 容器（包括 Tomcat，Jetty，Undertow）中运行。

## 三、应用场景介绍

SOFA Boot 可帮助用户快速搭建高效、可靠的分布式应用，同时能与 Spring Boot 工程无缝集成，降低用户的迁移成本。

### 3.1 快速开发分布式应用

SOFA Boot 框架集成了所有蚂蚁金融云中间件，以“依赖即服务”的调用形式实现快速配置，轻松搭建稳定、可靠、安全、可扩展的分布式应用，减少开发、测试、集成成本。

### 3.2 兼容 Spring Boot 工程

对于基于 Spring Boot 框架开发的应用，可迁移至 SOFA Boot 工程，轻松实现对原有框架的支持与优化。

## 四、构建源代码

我们的 SOFA Boot 代码是完全开源的，大家可以直接使用 Git 克隆我们的源代码，为了编译我们的源代码，需要的基础环境为 JDK7 或者 JDK8，[Apache Maven 3.2.5](https://archive.apache.org/dist/maven/maven-3/3.2.5/binaries/) 以及更高版本。

* [如何贡献](./how-to-contribute.md)


## 五、模块介绍 

### 5.1 healthcheck-sofa-boot-starter

此模块提供健康能力的代码集合，构建在 spring-boot-actuator 之上并完全兼容 actuator 的 API 扩展方式，同时提供了 SOFA Boot 维度的健康检查 API 和相应的事件监听机制，并明确区分出启动期健康检查和运行时健康检查。

### 5.2 infra-sofa-boot-starter

SOFA Boot 的基础能力集合，提供相应版本信息汇总展示能力、统一的命名空间管理能力以及一些通用能力的扩展。

### 5.3 runtime-sofa-boot-starter:

SOFA Boot 运行时能力集合，主要提供模块间通信能力的标准绑定模型和扩展方式、模块化开发、组件管理以及相关标准 API 的集合。

### 5.4 sofaboot-dependencies

SOFA Boot 的管控依赖集合，管控 SOFA Boot 的所有开源组件依赖，管控依赖继承自 Spring Boot 即

```java
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.4.2.RELEASE</version>
    </parent>
```

在继承 Spring Boot 的管控依赖之上，管控我们所有的基础依赖。

### 5.5 sofaboot-samples

提供基于我们 SOFA Boot 开发框架下的示例工程

## 六、示例

* 在此工程的 `sofaboot-samples` 下是示例工程，分别为：
	+ [SOFA Boot 示例工程](https://github.com/alipay/sofa-boot/tree/master/sofaboot-samples/sofaboot-sample)
	+ [SOFA Boot 示例工程(包含类隔离能力)](https://github.com/alipay/sofa-boot/tree/master/sofaboot-samples/sofaboot-sample-with-isolation)
 
## 七、SOFA Boot Guides

Reference Document ： 这里面介绍的详细下对使用的文档



