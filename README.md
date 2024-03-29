# P6spy
쿼리 튜닝을 쾌적하게 ! P6spy 개조하기

![image](https://user-images.githubusercontent.com/71188307/154077509-12d8cd08-9b9a-485a-94d1-d1030808e1e5.png)

대략 이러한 결과를 얻을 수 있습니다.

<br />

# 🚫 주의!

---

- 굉장히 비싼 자원을 사용하므로 운영 환경에선 절대 사용하지 말 것을 권장드립니다.

<br />

# ✅ 개발 환경

---

소스코드는 [GitHub](https://github.com/shirohoo/P6spy)에 공개되어 있습니다.

- Java 11
- Gradle 6.8.3
- Spring-Boot 2.5.0
- Spring-Data-JPA
- H2 Database
- p6spy 1.7.1

<br />

# ✅ 필수 설정

---

```groovy
// file: 'build.gradle'
implementation 'com.github.gavlyukovskiy:p6spy-spring-boot-starter:1.7.1'
```

<br />

```yaml
# file: 'application.yaml'
decorator:
  datasource:
    p6spy:
      enable-logging: true
```

<br />

명시하지 않아도 기본값은 `true`로 설정돼있지만, `profile`을 이용하여 개발/운영에서 명확하게 사용하기 위해 선언해주었다.

운영환경은 운영환경 `profile`에 위의 `enable-logging`을 `false`로 지정해주면 된다.

<br />

JPA를 쓰다 보면 예상 밖의 쿼리가 발생하는 경우가 굉장히 많다.

스프링에는 쿼리를 파악하기 좋게 해주는 라이브러리로 p6spy가 있는데, 기본값으로 사용할 경우 단순히 파라미터 바인딩만 보여주는 수준으로 생각보다 가독성이 좋지 않다.

더 큰 문제점은 쿼리가 한번 발생하면 파라미터가 바인딩되지 않은 원본 쿼리와 파라미터를 바인딩한 후의 쿼리, 총 두 개의 쿼리가 나란히 출력되는 것이다.

<br />

이처럼 간단한 쿼리의 경우는 그럭저럭 괜찮을 수 있으나, 통계성 쿼리 같이 복잡하고 수십 줄 이상되는 빅 쿼리가 두 개 연달아 나오면 굉장히 혼란스럽다.

심지어 두 개의 쿼리 중 한 개는 물음표가 가득할 것이다.

이런 문제를 개선하기 위해 p6spy를 커스터마이징 했다.

<br />

## 📕 p6spy 구조

---

1. `DataSource`를 래핑하여 프록시를 만든다.

2. 쿼리가 발생하여 JDBC가 `ResultSet`을 반환하면 이를 만들어둔 프록시로 가로챈다.

3. 내부적으로 `ResultSet`의 정보를 분석하고 p6spy의 옵션을 적용해준다.

4. `Slf4j`를 사용해 로깅한다.

<br />

처음 p6spy가 초기화될 때 쿼리를 포매팅하는 객체가 `MultiLineFormat`이다.

<br />

```java
public class P6SpyProperties {
    private boolean enableLogging = true;
    private boolean multiline = true;
    private P6SpyLogging logging = P6SpyLogging.SLF4J;
    private String logFile = "spy.log";
    private String logFormat;
}
```

<br />

`private boolean multiline = true`이며

`private String logFormat = null`이다.

<br />

```java
if (!initialP6SpyOptions.containsKey("logMessageFormat")) {
            if (p6spy.getLogFormat() != null) {
                System.setProperty("p6spy.config.logMessageFormat", "com.p6spy.engine.spy.appender.CustomLineFormat");
                System.setProperty("p6spy.config.customLogMessageFormat", p6spy.getLogFormat());
            }
            else if (p6spy.isMultiline()) {
                System.setProperty("p6spy.config.logMessageFormat", "com.p6spy.engine.spy.appender.MultiLineFormat");
            }
        }
```

<br />

위 조건으로 인해 `CustomLogMessageFormat`이 아닌 `MultiLineFormat`으로 타고 들어간다.

이후 `MultiLineFormat`의 포맷을 보면,

<br />

```java
public class MultiLineFormat implements MessageFormattingStrategy {
  @Override
  public String formatMessage(int connectionId, String now, long elapsed, String category, String prepared, String sql, String url) {
    return "#" + now + " | took " + elapsed + "ms | " + category + " | connection " + connectionId + "| url " + url + "\n" + prepared + "\n" + sql +";";
  }
}
```

<br />

코드를 보면 알겠지만, 원하는 포맷으로 확장하기 위해서 포매터를 직접 구현하여 지정해주면 된다.

<br />

```java
@Configuration
public class P6spyConfig {
    @PostConstruct
    public void setLogMessageFormat() {
        P6SpyOptions.getActiveInstance().setLogMessageFormat(P6spyPrettySqlFormatter.class.getName());
    }
}
```

<br />

설정 클래스를 생성하여 새로운 `LogFormatter`를 지정해준 후 구현에 들어간다.

<br />

```java
public class P6spyPrettySqlFormatter implements MessageFormattingStrategy {
    @Override
    public String formatMessage(int connectionId, String now, long elapsed, String category, String prepared, String sql, String url) {
        return null;
    }
}
```

<br />

`MessageFormattingStrategy`를 구현한다.

이름 그대로 메시지 포매팅 전략이다.

기본적으로 `SingleLineFormat`, `CustomLineFormat`, `MultiLineFormat`이 구현돼있다.

`CustomLineFormat`은 이름 때문에 약간 헷갈리는데 사용자가 커스터마이징 할 포매터가 아니고, `SingleLineFormat`을 약간 더 손본 포매터다. 

그러므로 이 녀석을 쓰면 안 되고 직접구현해야 한다.

아래는 `CustomLineFormat`의 전체 코드이다. 참고 바람.

<br />

```java
public class CustomLineFormat implements MessageFormattingStrategy {
  private static final MessageFormattingStrategy FALLBACK_FORMATTING_STRATEGY = new SingleLineFormat();

  public static final String CONNECTION_ID = "%(connectionId)";
  public static final String CURRENT_TIME = "%(currentTime)";
  public static final String EXECUTION_TIME = "%(executionTime)";
  public static final String CATEGORY = "%(category)";
  public static final String EFFECTIVE_SQL = "%(effectiveSql)";
  public static final String EFFECTIVE_SQL_SINGLELINE = "%(effectiveSqlSingleLine)";
  public static final String SQL = "%(sql)";
  public static final String SQL_SINGLE_LINE = "%(sqlSingleLine)";
  public static final String URL = "%(url)";

  /**
   * Formats a log message for the logging module
   *
   * @param connectionId the id of the connection
   * @param now          the current ime expressing in milliseconds
   * @param elapsed      the time in milliseconds that the operation took to complete
   * @param category     the category of the operation
   * @param prepared     the SQL statement with all bind variables replaced with actual values
   * @param sql          the sql statement executed
   * @param url          the database url where the sql statement executed
   * @return the formatted log message
   */
  @Override
  public String formatMessage(int connectionId, String now, long elapsed, String category, String prepared, String sql, String url) {
    String customLogMessageFormat = P6SpyOptions.getActiveInstance().getCustomLogMessageFormat();

    if (customLogMessageFormat == null) {
      // Someone forgot to configure customLogMessageFormat: fall back to built-in
      return FALLBACK_FORMATTING_STRATEGY.formatMessage(connectionId, now, elapsed, category, prepared, sql, url);
    }

    return customLogMessageFormat
      .replaceAll(Pattern.quote(CONNECTION_ID), Integer.toString(connectionId))
      .replaceAll(Pattern.quote(CURRENT_TIME), now)
      .replaceAll(Pattern.quote(EXECUTION_TIME), Long.toString(elapsed))
      .replaceAll(Pattern.quote(CATEGORY), category)
      .replaceAll(Pattern.quote(EFFECTIVE_SQL), Matcher.quoteReplacement(prepared))
      .replaceAll(Pattern.quote(EFFECTIVE_SQL_SINGLELINE), Matcher.quoteReplacement(P6Util.singleLine(prepared)))
      .replaceAll(Pattern.quote(SQL), Matcher.quoteReplacement(sql))
      .replaceAll(Pattern.quote(SQL_SINGLE_LINE), Matcher.quoteReplacement(P6Util.singleLine(sql)))
      .replaceAll(Pattern.quote(URL), url);
  }
```

<br />

쿼리가 정확히 어떤 경로를 타고 발생했는지 추적하여 기록해줄 것이다.

<br />

```java
StackTraceElement[] stackTrace = new Throwable().getStackTrace();
for(int i = 0; i < stackTrace.length; i++) {
    System.out.println(stackTrace[i]);
}
```

<br />

`Throwable`을 호출하여 `stack trace`를 쭉 뽑아보면

<br />

```java
io.p6spy.formatter.P6spyPrettySqlFormatter.formatMessage(P6spyPrettySqlFormatter.java:15)
com.p6spy.engine.spy.appender.Slf4JLogger.logSQL(Slf4JLogger.java:50)
com.p6spy.engine.common.P6LogQuery.doLog(P6LogQuery.java:121)
com.p6spy.engine.common.P6LogQuery.doLogElapsed(P6LogQuery.java:91)
com.p6spy.engine.common.P6LogQuery.logElapsed(P6LogQuery.java:203)
com.p6spy.engine.logging.LoggingEventListener.logElapsed(LoggingEventListener.java:107)
com.p6spy.engine.logging.LoggingEventListener.onAfterCommit(LoggingEventListener.java:54)
com.p6spy.engine.event.CompoundJdbcEventListener.onAfterCommit(CompoundJdbcEventListener.java:285)
com.p6spy.engine.wrapper.ConnectionWrapper.commit(ConnectionWrapper.java:172)
org.hibernate.resource.jdbc.internal.AbstractLogicalConnectionImplementor.commit(AbstractLogicalConnectionImplementor.java:86)
org.hibernate.resource.transaction.backend.jdbc.internal.JdbcResourceLocalTransactionCoordinatorImpl$TransactionDriverControlImpl.commit(JdbcResourceLocalTransactionCoordinatorImpl.java:282)
org.hibernate.engine.transaction.internal.TransactionImpl.commit(TransactionImpl.java:101)
org.springframework.orm.jpa.JpaTransactionManager.doCommit(JpaTransactionManager.java:562)
org.springframework.transaction.support.AbstractPlatformTransactionManager.processCommit(AbstractPlatformTransactionManager.java:743)
org.springframework.transaction.support.AbstractPlatformTransactionManager.commit(AbstractPlatformTransactionManager.java:711)
org.springframework.transaction.interceptor.TransactionAspectSupport.commitTransactionAfterReturning(TransactionAspectSupport.java:654)
org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:407)
org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:119)
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
org.springframework.dao.support.PersistenceExceptionTranslationInterceptor.invoke(PersistenceExceptionTranslationInterceptor.java:137)
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
org.springframework.data.jpa.repository.support.CrudMethodMetadataPostProcessor$CrudMethodMetadataPopulatingMethodInterceptor.invoke(CrudMethodMetadataPostProcessor.java:174)
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:97)
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:215)
com.sun.proxy.$Proxy88.save(Unknown Source)
io.p6spy.controller.MainController.run(MainController.java:35)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
java.base/jdk.internal.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
java.base/java.lang.reflect.Method.invoke(Method.java:566)
org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:197)
org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:141)
org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:106)
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:894)
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:808)
org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1063)
org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:963)
org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006)
org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:909)
javax.servlet.http.HttpServlet.service(HttpServlet.java:652)
org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883)
javax.servlet.http.HttpServlet.service(HttpServlet.java:733)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:227)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100)
org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93)
org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201)
org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119)
org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:189)
org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:162)
org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:202)
org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:97)
org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:542)
org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:143)
org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92)
org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78)
org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:357)
org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:374)
org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65)
org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:893)
org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1707)
org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
java.base/java.lang.Thread.run(Thread.java:834)
2021-05-28 21:31:46.316  INFO 2820 --- [nio-8080-exec-2] p6spy                                    :
```

<br />

`Throwable`이 호출된 시점까지의 모든 경로가 출력된다.

여기서 필요한 부분만 추출하면 되는데, 원하는 trace에서 공통점을 찾을 수 있다.

바로 문자열의 시작점이 main 패키지의 경로라는 것이다.

따라서 아래와 같이 코드를 작성해주면 필터링이 된다.

<br />

```java
StackTraceElement[] stackTrace = new Throwable().getStackTrace();
for(int i = 0; i < stackTrace.length; i++) {
        if(stackTrace[i].toString().startsWith("io.p6spy") && !stackTrace[i].toString().contains("P6spyPrettySqlFormatter")) {
            System.out.println(stackTrace[i]);
        }
    }
```

<br />

여기서 `P6spyPrettySqlFormatter`의 trace는 필요 없기 때문에 필터링해 준다.

그리고 이 로그를 더 보기 편하게 역순으로 뒤집어줄 것이다.

Stack을 활용할 것인데, 추출되는 trace를 순서대로 Stack에 `push` 하고, 다시 `pop` 하면 역순으로 뒤집힐 것이다.

<br />

```java
@Override
public String formatMessage(int connectionId, String now, long elapsed, String category, String prepared, String sql, String url) {
    Stack<String> callStack = new Stack<>();
    StackTraceElement[] stackTrace = new Throwable().getStackTrace();

    for(int i = 0; i < stackTrace.length; i++) {
        String trace = stackTrace[i].toString();
        if(trace.startsWith("io.p6spy") && !trace.contains("P6spyPrettySqlFormatter")) {
            callStack.push(trace);
        }
    }

    StringBuilder callStackBuilder = new StringBuilder();
    int order = 1;
    while(callStack.size() != 0) {
        callStackBuilder.append("\n\t\t" + (order++) + ". " + callStack.pop());
    }
    return null;
}
```

<br />

![image](https://user-images.githubusercontent.com/71188307/154077216-08fb570b-c25f-4d25-aa39-f96da87fb31a.png)

<br />

쿼리가 발생한 지점과 클릭하면 즉시 이동할 수 있는 `포탈(🤗)`이 생성됐다.

이제 SQL을 보기 좋게 포매팅할 것이다.

`formatMessage()`에 이런저런 파라미터가 많이 들어오는데

이에 대한 자세한 내용은 p6spy docs를 보면 하기와 같다.

<br />

```text
Params:
connectionId – the id of the connection
now – the current ime expressing in milliseconds
elapsed – the time in milliseconds that the operation took to complete
category – the category of the operation
prepared – the SQL statement with all bind variables replaced with actual values
sql – the sql statement executed
url – the database url where the sql statement executed
```

<br />

이 파라미터들을 적당히 버무려 준다.

<br />

```java
import com.p6spy.engine.logging.Category;
import com.p6spy.engine.spy.appender.MessageFormattingStrategy;
import org.hibernate.engine.jdbc.internal.FormatStyle;

import java.text.MessageFormat;
import java.util.Locale;
import java.util.Objects;
import java.util.Stack;
import java.util.function.Predicate;

import static java.util.Arrays.stream;

public class P6spyPrettySqlFormatter implements MessageFormattingStrategy {
    private static final String NEW_LINE = System.lineSeparator();
    private static final String P6SPY_FORMATTER = "P6spyPrettySqlFormatter";
    private static final String PACKAGE = "io.p6spy";
    private static final String CREATE = "create";
    private static final String ALTER = "alter";
    private static final String COMMENT = "comment";

    @Override
    public String formatMessage(int connectionId, String now, long elapsed, String category, String prepared, String sql, String url) {
        return sqlFormatToUpper(sql, category, getMessage(connectionId, elapsed, getStackBuilder()));
    }

    private String sqlFormatToUpper(String sql, String category, String message) {
        if (Objects.isNull(sql.trim()) || sql.trim().isEmpty()) {
            return "";
        }
        return new StringBuilder()
                .append(NEW_LINE)
                .append(sqlFormatToUpper(sql, category))
                .append(message)
                .toString();
    }

    private String sqlFormatToUpper(String sql, String category) {
        if (isStatementDDL(sql, category)) {
            return FormatStyle.DDL
                    .getFormatter()
                    .format(sql)
                    .toUpperCase(Locale.ROOT)
                    .replace("+0900", "");
        }
        return FormatStyle.BASIC
                .getFormatter()
                .format(sql)
                .toUpperCase(Locale.ROOT)
                .replace("+0900", "");
    }

    private boolean isStatementDDL(String sql, String category) {
        return isStatement(category) && isDDL(sql.trim().toLowerCase(Locale.ROOT));
    }

    private boolean isStatement(String category) {
        return Category.STATEMENT.getName().equals(category);
    }

    private boolean isDDL(String lowerSql) {
        return lowerSql.startsWith(CREATE) || lowerSql.startsWith(ALTER) || lowerSql.startsWith(COMMENT);
    }

    private String getMessage(int connectionId, long elapsed, StringBuilder callStackBuilder) {
        return new StringBuilder()
                .append(NEW_LINE)
                .append(NEW_LINE)
                .append("\t").append(String.format("Connection ID: %s", connectionId))
                .append(NEW_LINE)
                .append("\t").append(String.format("Execution Time: %s ms", elapsed))
                .append(NEW_LINE)
                .append(NEW_LINE)
                .append("\t").append(String.format("Call Stack (number 1 is entry point): %s", callStackBuilder))
                .append(NEW_LINE)
                .append(NEW_LINE)
                .append("----------------------------------------------------------------------------------------------------")
                .toString();
    }

    private StringBuilder getStackBuilder() {
        Stack<String> callStack = new Stack<>();
        stream(new Throwable().getStackTrace())
                .map(StackTraceElement::toString)
                .filter(isExcludeWords())
                .forEach(callStack::push);

        int order = 1;
        StringBuilder callStackBuilder = new StringBuilder();
        while (!callStack.empty()) {
            callStackBuilder.append(MessageFormat.format("{0}\t\t{1}. {2}", NEW_LINE, order++, callStack.pop()));
        }
        return callStackBuilder;
    }

    private Predicate<String> isExcludeWords() {
        return charSequence -> charSequence.startsWith(PACKAGE) && !charSequence.contains(P6SPY_FORMATTER);
    }
}
```

<br />

![image](https://user-images.githubusercontent.com/71188307/154077509-12d8cd08-9b9a-485a-94d1-d1030808e1e5.png)

<br />
