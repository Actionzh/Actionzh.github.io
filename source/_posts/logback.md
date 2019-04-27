---
title: logback实践
date: 2019-04-21 16:01:08
tags:
---
### 一、logback介绍

Logback是由log4j创始人设计的一个开源日志组件。LogBack被分为3个组件，logback-core, logback-classic 和 logback-access。

```
1\. logback-core:提供了LogBack的核心功能，是另外两个组件的基础。
2\. logback-classic:实现了Slf4j的API，所以当想配合Slf4j使用时，需要引入logback-classic。
3\. logback-access:为了集成Servlet环境而准备的，可提供HTTP-access的日志接口。

```

Logback是要与SLF4J结合起来。这两个组件的官方网站如下：

logback官方网站: [logback官方网站](http://logback.qos.ch)

SLF4J官方网站： [SLF4J官方网站](http://www.slf4j.org)

*   Slf4j：简单日志门面(Simple Logging Facade for Java)，不是具体的日志解决方案，它只服务于各种各样的日志系统。
*   在使用SLF4J的时候，不需要在代码中或配置文件中指定你打算使用那个具体的日志系统。

### 二、slf4j + logback是如何绑定的

```
1. private static final Logger LOGGER = LoggerFactory.getLogger(LogbackTest.class);

2. 查看LoggerFactory.getLogger()方法
    public static Logger getLogger(Class<?> clazz) {
        // 获取Logger对象
        Logger logger = getLogger(clazz.getName());
        if (DETECT_LOGGER_NAME_MISMATCH) {
            Class<?> autoComputedCallingClass = Util.getCallingClass();
            if (autoComputedCallingClass != null && nonMatchingClasses(clazz, autoComputedCallingClass)) {
                Util.report(String.format("Detected logger name mismatch. Given name: \"%s\"; computed name: \"%s\".", logger.getName(),
                                autoComputedCallingClass.getName()));
                Util.report("See " + LOGGER_NAME_MISMATCH_URL + " for an explanation");
            }
        }
        return logger;
    }

3. 继续跟进 getLogger()
    /**
     * Return a logger named according to the name parameter using the
     * statically bound {@link ILoggerFactory} instance.
     * 
     * @param name
     *            The name of the logger.
     * @return logger
     */
    public static Logger getLogger(String name) {
        // 获取日志工厂
        ILoggerFactory iLoggerFactory = getILoggerFactory();
        // 返回日志对象 
        return iLoggerFactory.getLogger(name);
    }

4. 获取工厂实例
    /**
     * Return the {@link ILoggerFactory} instance in use.
     * <p/>
     * <p/>
     * ILoggerFactory instance is bound with this class at compile time.  
     * 编译时绑定工厂实例
     * 
     * @return the ILoggerFactory instance in use
     */
    public static ILoggerFactory getILoggerFactory() {
        // 没有初始化情况
        // 双重检测锁
        if (INITIALIZATION_STATE == UNINITIALIZED) {
            synchronized (LoggerFactory.class) {
                if (INITIALIZATION_STATE == UNINITIALIZED) {
                    // 初始化
                    INITIALIZATION_STATE = ONGOING_INITIALIZATION;
                    performInitialization();
                }
            }
        }
        switch (INITIALIZATION_STATE) {
        case SUCCESSFUL_INITIALIZATION:
            return StaticLoggerBinder.getSingleton().getLoggerFactory();
        case NOP_FALLBACK_INITIALIZATION:
            return NOP_FALLBACK_FACTORY;
        case FAILED_INITIALIZATION:
            throw new IllegalStateException(UNSUCCESSFUL_INIT_MSG);
        case ONGOING_INITIALIZATION:
            // support re-entrant behavior.
            // See also http://jira.qos.ch/browse/SLF4J-97
            return SUBST_FACTORY;
        }
        throw new IllegalStateException("Unreachable code");
    }

5. 初始化
    private final static void performInitialization() {
        bind();
        if (INITIALIZATION_STATE == SUCCESSFUL_INITIALIZATION) {
            versionSanityCheck();
        }
    }

6. 绑定
    private final static void bind() {
        try {
            Set<URL> staticLoggerBinderPathSet = null;
            // skip check under android, see also
            // http://jira.qos.ch/browse/SLF4J-328
            if (!isAndroid()) {
                staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();
                reportMultipleBindingAmbiguity(staticLoggerBinderPathSet);
            }
            // the next line does the binding
            // 真正的绑定，将具体的实现绑定到slf4j
            StaticLoggerBinder.getSingleton();
            INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;
            reportActualBinding(staticLoggerBinderPathSet);
            fixSubstituteLoggers();
            replayEvents();
            // release all resources in SUBST_FACTORY
            SUBST_FACTORY.clear();
         }
    }

7. logback-classic: org.slf4j.impl.StaticLoggerBinder
    public static StaticLoggerBinder getSingleton() {
        return SINGLETON;
    }

由此可以看出slf4j在编译时就找了具体的日志实现了，也就是 org.slf4j.impl.StaticLoggerBinder。

```

### 三、logback对配置文件的加载

```
1. getSingleton()方法获取logback实例对象，说明在对象之前已经加载了相关的配置文件，跟进 StaticLoggerBinder
    static {
        // 初始化
        SINGLETON.init();
    }

    private boolean initialized = false;
    private LoggerContext defaultLoggerContext = new LoggerContext();
    private final ContextSelectorStaticBinder contextSelectorBinder = ContextSelectorStaticBinder.getSingleton();

    private StaticLoggerBinder() {
        defaultLoggerContext.setName(CoreConstants.DEFAULT_CONTEXT_NAME);
    }

    public static StaticLoggerBinder getSingleton() {
        return SINGLETON;
    }

2. 查看 init()
    /**
     * Package access for testing purposes.
     */
    void init() {
        try {
            try {
                // 上下文初始化环境 
                new ContextInitializer(defaultLoggerContext).autoConfig();
            } catch (JoranException je) {
                Util.report("Failed to auto configure default logger context", je);
            }
            // logback-292
            if (!StatusUtil.contextHasStatusListener(defaultLoggerContext)) {
                StatusPrinter.printInCaseOfErrorsOrWarnings(defaultLoggerContext);
            }
            contextSelectorBinder.init(defaultLoggerContext, KEY);
            initialized = true;
        } catch (Throwable t) {
            // we should never get here
            Util.report("Failed to instantiate [" + LoggerContext.class.getName() + "]", t);
        }
    }

3. 跟进autoConfig()
    public void autoConfig() throws JoranException {
        StatusListenerConfigHelper.installIfAsked(loggerContext);
        // 寻找默认配置文件
        URL url = findURLOfDefaultConfigurationFile(true);
        if (url != null) {
            configureByResource(url);
        } else {
            Configurator c = EnvUtil.loadFromServiceLoader(Configurator.class);
            if (c != null) {
                try {
                    c.setContext(loggerContext);
                    c.configure(loggerContext);
                } catch (Exception e) {
                    throw new LogbackException(String.format("Failed to initialize Configurator: %s using ServiceLoader", c != null ? c.getClass()
                                    .getCanonicalName() : "null"), e);
                }
            } else {
                // 没有找到配置文件，则使用默认的配置器，那么日志只会打印在控制台
                BasicConfigurator basicConfigurator = new BasicConfigurator();
                basicConfigurator.setContext(loggerContext);
                basicConfigurator.configure(loggerContext);
            }
        }
    }

4. findURLOfDefaultConfigurationFile() logback配置文件加载规则
    public URL findURLOfDefaultConfigurationFile(boolean updateStatus) {
        // 获取当前实例的类加载器，目的是在classpath下寻找配置文件
        ClassLoader myClassLoader = Loader.getClassLoaderOfObject(this);
        // 先找logback.configurationFile文件
        URL url = findConfigFileURLFromSystemProperties(myClassLoader, updateStatus);
        if (url != null) {
            return url;
        }
        // logback.configurationFile文件没找到，再找logback.groovy
        url = getResource(GROOVY_AUTOCONFIG_FILE, myClassLoader, updateStatus);
        if (url != null) {
            return url;
        }
        // logback.groovy没找到，再找logback-test.xml
        url = getResource(TEST_AUTOCONFIG_FILE, myClassLoader, updateStatus);
        if (url != null) {
            return url;
        }
        // logback-test.xml没找到，最后找logback.xml
        return getResource(AUTOCONFIG_FILE, myClassLoader, updateStatus);
    }

```

小结：
编译期间，完成slf4j的绑定已经logback配置文件的加载。slf4j会在classpath中寻找org/slf4j/impl/StaticLoggerBinder.class(会在具体的日志框架如log4j、logback等中存在)，找到并完成绑定；同时，logback也会在classpath中寻找配置文件，先找logback.configurationFile、没有则找logback.groovy，若logback.groovy也没有，则找logback-test.xml，若logback-test.xml还是没有，则找logback.xml，若连logback.xml也没有，那么说明没有配置logback的配置文件，那么logback则会启用默认的配置(日志信息只会打印在控制台)。

### 四、使用步骤

1.引入slf4j、logback相关依赖

```
    <!-- slf4j -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>${slf4j.version}</version>
    </dependency>

    <!-- logback -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-core</artifactId>
        <version>${logback.version}</version>
    </dependency>

    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>${logback.version}</version>
    </dependency>

    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-access</artifactId>
        <version>${logback.version}</version>
    </dependency>

```

2.添加配置文件logback.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 定义日志文件的存储地址 -->
    <!--
        关于catalina.base解释如下：
            catalina.home指向公用信息的位置，就是bin和lib的父目录。
            catalina.base指向每个Tomcat目录私有信息的位置，就是conf、logs、temp、webapps和work的父目录。
    -->
    <property name="LOG_DIR" value="${catalina.base}/logs"/>

    <!--
        %p:输出优先级，即DEBUG,INFO,WARN,ERROR,FATAL
        %r:输出自应用启动到输出该日志讯息所耗费的毫秒数
        %t:输出产生该日志事件的线程名
        %f:输出日志讯息所属的类别的类别名
        %c:输出日志讯息所属的类的全名
        %d:输出日志时间点的日期或时间，指定格式的方式： %d{yyyy-MM-dd HH:mm:ss}
        %l:输出日志事件的发生位置，即输出日志讯息的语句在他所在类别的第几行。
        %m:输出代码中指定的讯息，如log(message)中的message
        %n:输出一个换行符号
    -->
    <!--格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度 %msg：日志消息，%n是换行符-->
    <property name="pattern" value="%d{yyyyMMdd:HH:mm:ss.SSS} [%thread] %-5level  %msg%n"/>

    <!--
        Appender: 设置日志信息的去向,常用的有以下几个
            ch.qos.logback.core.ConsoleAppender (控制台)
            ch.qos.logback.core.rolling.RollingFileAppender (文件大小到达指定尺寸的时候产生一个新文件)
            ch.qos.logback.core.FileAppender (文件)
    -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 字符串System.out（默认）或者System.err -->
        <target>System.out</target>
        <!-- 对记录事件进行格式化 -->
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${pattern}</pattern>
        </encoder>
    </appender>

    <appender name="SQL_INFO" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 被写入的文件名，可以是相对目录，也可以是绝对目录，如果上级目录不存在会自动创建 -->
        <file>${LOG_DIR}/sql_info.log</file>
        <!-- 当发生滚动时，决定RollingFileAppender的行为，涉及文件移动和重命名。属性class定义具体的滚动策略类 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 必要节点，包含文件名及"%d"转换符，"%d"可以包含一个java.text.SimpleDateFormat指定的时间格式，默认格式是 yyyy-MM-dd -->
            <fileNamePattern>${LOG_DIR}/sql_info_%d{yyyy-MM-dd}.log.%i.gz</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>20MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!-- 可选节点，控制保留的归档文件的最大数量，超出数量就删除旧文件。假设设置每个月滚动，如果是6，则只保存最近6个月的文件，删除之前的旧文件 -->
            <maxHistory>10</maxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${pattern}</pattern>
        </encoder>
        <!-- LevelFilter： 级别过滤器，根据日志级别进行过滤 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <!-- 用于配置符合过滤条件的操作 ACCEPT：日志会被立即处理，不再经过剩余过滤器 -->
            <onMatch>ACCEPT</onMatch>
            <!-- 用于配置不符合过滤条件的操作 DENY：日志将立即被抛弃不再经过其他过滤器 -->
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <appender name="SQL_ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_DIR}/sql_error.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_DIR}/sql_error_%d{yyyy-MM-dd}.log.%i.gz</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>20MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <maxHistory>10</maxHistory>
        </rollingPolicy>
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>${pattern}</pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <appender name="APP_INFO" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_DIR}/info.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <FileNamePattern>${LOG_DIR}/info.%d{yyyy-MM-dd}.log
            </FileNamePattern>
        </rollingPolicy>
        <encoder>
            <Pattern>[%date{yyyy-MM-dd HH:mm:ss}] [%-5level] [%thread] [%logger:%line]--%mdc{client} %msg%n</Pattern>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.FileAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="ch.qos.logback.classic.html.HTMLLayout">
                <property name="pattern" value="%d{yyyyMMdd:HH:mm:ss.SSS} [%thread] %-5level  %msg%n"/>
                <pattern>%d{yyyyMMdd:HH:mm:ss.SSS}%thread%-5level%F{32}%M%L%msg</pattern>
            </layout>
        </encoder>
        <file>${LOG_DIR}/test.html</file>
    </appender>

    <!--
        用来设置某一个包或者具体的某一个类的日志打印级别、以及指定<appender>。
        <loger>仅有一个name属性，一个可选的level和一个可选的addtivity属性
        name:
            用来指定受此logger约束的某一个包或者具体的某一个类。
        level:
            用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
            如果未设置此属性，那么当前logger将会继承上级的级别。
        additivity:
            是否向上级loger传递打印信息。默认是true。
        <logger>可以包含零个或多个<appender-ref>元素，标识这个appender将会添加到这个logger
    -->
    <logger name="java.sql" level="info" additivity="false">
        <level value="info" />
        <appender-ref ref="STDOUT"></appender-ref>
        <appender-ref ref="SQL_INFO"></appender-ref>
        <appender-ref ref="SQL_ERROR"></appender-ref>
    </logger>

    <logger name="com.souche.LogbackTest" additivity="false">
        <level value="info" />
        <appender-ref ref="STDOUT" />
        <appender-ref ref="APP_INFO" />
        <appender-ref ref="FILE"/>
    </logger>

    <!--
        也是<logger>元素，但是它是根logger。默认debug
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
        <root>可以包含零个或多个<appender-ref>元素，标识这个appender将会添加到这个logger。
    -->
    <root level="info">
        <level>info</level>
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="SQL_INFO"/>
        <appender-ref ref="SQL_ERROR"/>
        <appender-ref ref="FILE"/>
    </root>

</configuration>

```

### 五、实际应用

就举例最近新做的项目弹个X中的应用吧。因为在与前端联调阶段，api自测感觉没啥问题的，然后联调就会有各种问题，没法避免，技术还是太水了，哈哈哈哈........
调详情页的时候，听到有问题就赶紧看日志去，果然报错了。如下:

![](//upload-images.jianshu.io/upload_images/11806186-f57f61c096181fc4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

错误日志

有错误信息就能当做本地控制台一样，可以直接看到错误信息。

Cause: com.mysql.jdbc.MysqlDataTruncation: Data truncation: Data too long for column 'saler_phone' at row 1
; SQL []; Data truncation: Data too long for column 'saler_phone' at row 1; nested exception is com.mysql.jdbc.MysqlDataTruncation: Data truncation: Data too long for column 'saler_phone' at row 1

然后进一步加了log日志

![](//upload-images.jianshu.io/upload_images/11806186-1008f8cf2b04796e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

log日志

传参错了，查了原因，登录失效，然后salerPhone居然还有值，而且多了一位......

### 六、为什么使用logback

logback具有以下优点：

*   内核重写、测试充分、初始化内存加载更小，这一切让logback性能和log4j相比有诸多倍的提升
*   logback非常自然地直接实现了slf4j，这个严格来说算不上优点，只是这样，再理解slf4j的前提下会很容易理解logback，也同时很容易用其他日志框架替换logbac
*   logback有比较齐全的200多页的文档
*   logback当配置文件修改了，支持自动重新加载配置文件，扫描过程快且安全，它并不需要另外创建一个扫描线程
*   支持自动去除旧的日志文件，可以控制已经产生日志文件的最大数量

总而言之，如果大家的项目里面需要选择一个日志框架，那么我个人非常建议使用logback。

### 七、logback性能大比拼

log4j、logback、log4j2性能测试，直接引用公司的博客吧。[logback log4j log4j2 性能实测](https://blog.souche.com/logback-log4j-log4j2shi-ce/)

### 参考链接

1.  [Java日志框架-logback的介绍及配置使用方法](https://www.cnblogs.com/EasonJim/p/7800880.html)
2.  [从源码来理解slf4j的绑定，以及logback对配置文件的加载](https://www.cnblogs.com/youzhibing/p/6849843.html)
3.  [logback 配置详解和使用](https://www.cnblogs.com/nananana/p/8371805.html)
4.  [logback.xml常用配置详解 : filter](https://blog.csdn.net/haidage/article/details/6794540)
5.  [Java 日志框架：logback 详解](https://mp.weixin.qq.com/s?__biz=MzU1ODUyMDUzNw==&mid=2247483962&idx=1&sn=ee98ec38dbca13730b6d87e793609acb&chksm=fc24038ccb538a9aad19573c06c987dbdb6d72de407811db469f7d06e5afb095bbe81395ca3e&mpshare=1&scene=23&srcid=0614XsPPbErSxUi2MOzweE0X#rd)
6.  [logback log4j log4j2 性能实测](https://blog.souche.com/logback-log4j-log4j2shi-ce/)


链接：https://www.jianshu.com/p/b3dedb8fb61e

