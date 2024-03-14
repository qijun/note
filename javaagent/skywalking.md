
SkyWalking是一个观察性分析平台和应用性能管理系统（APM）提供分布式追踪、服务网格遥测分析、度量聚合和可视化一体化解决方案。能非常方便的定位线上性能问题。





SkyWalking整体架构主要分三层，探针、后端和界面。
探针（Probe）
在应用端收集性能度量数据并发送给后端。SkyWalking支持三种探针：
Agent --基于ByteBuddy字节码增强技术实现，通过jvm的agent参数加载，并在程序启动时拦截指定的方法来收集数据。
SDK -- 程序中显式调用SkyWalking提供的SDK来收集数据，对应用有侵入。
Service Mesh -- 通过Service mesh的网络代理来收集数据。
后端（Backend）
接受探针发送过来的数据，进行度量分析，调用链分析和存储。后端主要分为两部分：
OAP（Observability Analysis Platform）-进行度量分析和调用链分析的后端平台，并支持将数据存储到各种数据库中，如：ElasticSearch，MySQL，InfluxDB等。
OAL（Observability Analysis Language）-用来进行度量分析的DSL，类似于SQL，用于查询度量分析结果和警报。
界面（UI）
RocketBot UI -- SkyWalking 7.0.0 的默认web UI
CLI -- 命令行界面

这三个模块的交互流程：





构建SkyWalking
各种编程语言开发的应用（JVM、.Net、Go、Python等）都可以使用SkyWalking探针来收集数据。平时我们用的最多的是基于JVM的技术栈（Java、Scala、Kotlin），对于JVM技术栈，使用agentplugin的方式来收集性能数据有如下优势：
对应用代码无侵入。
对应用进程性能开销小，可以做到线上实时性能分析和调用链追踪。
支持任意粒度性能数据的收集， 比如：API级别、方法级别等。
支持同步、异步、跨线程（Thread）、协程（Coroutine）的性能追踪。
SkyWalking本身已经提供了足够多agentplugins，支持了JVM技术栈常用的开源框架和库（SpringCloud、各种网络框架、Dubbo、各种数据库等），但是JVM的生态系统非常的庞大和活跃，各种开源框架和库层出不穷，成熟框架的版本更新也非常快，SkyWalking本身不一定能够及时的追踪这些新的框架或者新版本的库，所以有时候需要根据具体的项目或者工具来做定制的plugin开发。
SkyWalking提供的完整的plugin开发和自动测试框架，开发一个新的plugin需要下载SkyWalking源码进行构建。构建步骤如下：
git clone  本地址
mvn clean package -DskipTests
IDEA导入SkyWalking工程
mvn compile -DskipTests


开发插件(以fireFly为例)
源码构建成功之后，就可以开始插件开发了。首先我们在源代码的apm-sdk-plugin目录下建立自己的pluginmodule，目录结构如下：
[skywalking]
|-[apm-sniffer]
    |-[apm-sdk-plugin]
        |-[firefly-5.x-plugins]
            |-[firefly-5.x-net-http-client-plugin]
                |-pom.xml
            |-[firefly-5.x-net-http-server-plugin]
                |-pom.xml
        |-pom.xml



定义拦截点
Agentplugin是使用ByteBuddy做字节码增强，类似于AOP，相当于给目标类增加了一个代理。SkyWalking封装了相关的操作，形成了自己的开发框架。





enhanceClass
表示需要拦截的类，这里拦截的是AsyncHttpServerConnectionListener，因为fireflyhttpserver接收到的所有请求都会进入这个listener，只需要对这个类做一个代理就可以拿到所有请求的性能数据了。这里除了可以按类名去拦截，还可以按照Annotation、前缀等方式去拦截目标类，具体可以参考ClassMatch接口的子类：





定义方法拦截点
InstanceMethodsInterceptPoint用来定义，这个类中对哪些方法进行拦截，这里直接按方法名拦截onHttpRequestComplete方法。除了按方法名拦截，SkyWalking还封装了各种方式匹配要拦截方法，这里就不在赘述，可以参考ElementMatchers类相关源码或者文档。
第57行getMethodsInterceptor定义了拦截器的实现类的类名。一会儿我们就要实现这个类来记录性能数据。






上图展示了一个简单的场景，server3通过远程调用server2返回结果给用户。右侧图表中的每一行称为一个跨度（Span），在拦截器中，我们就需要构造对应的Span发送给SkyWalking后端，那么SkyWalking的后端就能够根据Span的相关信息做调用链和度量分析。


跨度（Span）
Span是分布式追踪的主要构造块，表示⼀个独⽴的⼯作单元，它包含如下状态记录：
操作名称（Operation Name）。
开始和结束时间戳。
自定义标签（Tags），k-v结构，一般记录操作的一些信息如：http状态码，http方法、ip、port、db.statement等。
Logs，k-v结构，用来记录错误日志。
SpanContext，用来记录trace_id、parent_id等用作调用链分析。能够把一个API中的跨线程、跨进程调用连接起来。

用yml表示span的结构如下：




其中spanType分三种：
Entry -- 服务端入口，比如Spring的Controller、MQ的Consumer等。
Exit -- 客户端远程调用，如http client、JDBC、redis client，MQproducer等。
Local -- 本地方法调用，用来记录本地方法的执行时间。
Span的Context记录分两种：
ContextCarrier -- 用于跨进程传递上下文数据。
ContextSnapshot -- 用于跨线程传递上下文数据。

现在我们基于Firefly 5.x写一个拦截插件

首先看一下我们准备去拦截的Firefly 5.x这个网络框架目标方法onHttpRequestComplete的代码：





Firefly 5.x是一个基于Coroutine的网络框架，所有的httprequest都会进入这个callback然后找到相关的router来处理request，router的handler会运行在一个coroutine上，为了保持和java的兼容性，异步结果通过CompletableFuture返回。

AsyncHttpServerConnectionListenerInterceptor的实现：





beforeMethod在进入onHttpRequesComplete方法之前执行。
49-57行。创建ContextCarrier，并且判断httpheader中是否有需要恢复的上下文数据，如果有则填充到ContextCarrier中。
62-69行。创建一个Entry类型的Span，并用Tags记录一些请求相关的信息。
71行。捕获上下文快照，并记录到Firefly的RoutingContext中，因为目标方法是异步返回的，我们需要保持当前请求的上下文快照后续可以在另外一个线程中恢复。
72行。设置Span的状态为异步执行。





beforeMethod在进入onHttpRequesComplete方法之前执行。
49-57行。创建ContextCarrier，并且判断httpheader中是否有需要恢复的上下文数据，如果有则填充到ContextCarrier中。
62-69行。创建一个Entry类型的Span，并用Tags记录一些请求相关的信息。
71行。捕获上下文快照，并记录到Firefly的RoutingContext中，因为目标方法是异步返回的，我们需要保持当前请求的上下文快照后续可以在另外一个线程中恢复。
72行。设置Span的状态为异步执行。

afterMethod方法在onHttpRequestComplete方法执行之后执行。这里可以拿到该方法的返回值，CompletableFuture，然后在future执行完成之后调用span.asyncFinish()来结束当前span并把span的相关数据发送给SkyWalking的后端OAP平台。

PS:注意此处的span.asyncFinish()是推送相关链路信息到相应的后端，如果是需要推送到我们自定义的业务后端，则需要将行逻辑进行重新封装




handleMethodException用来处理目标方法抛异常的情况，这里直接记录一下错误日志。
至此Firefly 5.x http server的plugin就开发完成了，httpclient的插件也类似，就不在赘述，区别就在于client的方法拦截器中是先创建ExitSpan，然后吧ContextCarrier中的信息存储到httprequest中发送给server，流程和server是正好相反的。


组件定义配置
开发完所有代码之后，还需要在ComponentsDefine类和component-libraries.yml配置文件中增加新plugin的id、名称等信息的配置。

构建插件
由于我们在下载源码后，已经对SkyWalking做过一次全量构建，开发新的插件之后，就不需要对整个项目进行构建，这里可以只构建agent模块即可。运行命令：mvnclean package -Pagent,dist -DskipTests=true
这里总结一下插件的开发流程如下：
apm-sdk-plugin目录下建立自己的plugin module
定义拦截点（实现ClassInstanceMethodsEnhancePluginDefine）
实现方法拦截器（InstanceMethodsAroundInterceptor）
在ComponentsDefine和component-libraries.yml中配置插件信息
构建agent模块mvn clean package -Pagent,dist -DskipTests=true
运行插件，并在分离的apm

编译分离 搭建 Agent 调试环境。
在 IntelliJ IDEA Terminal 中，在下图所示模块执行 mvn compile -Dmaven.test.skip=true 进行编译。在 /packages/skywalking-agent 目录下，我们可以看到编译出来的 Agent ：
插件代码构建完成之后，我们可以在实际的代码中加载一下新开发的插件看看运行是否正常。





启动SkyWalking
构建完成的SkyWalking目录结构如下：






