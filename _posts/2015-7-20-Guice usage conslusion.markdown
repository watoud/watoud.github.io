---
layout: post
title: Guice依赖注入小结
date: 2015-07-20
comments: true
archive: true
tags: guice
---
Guice是一个轻量级的依赖注入框架。依赖注入是一种设计模式，它的核心思想是行为与依赖解析相分离，从而使得代码模块化并且易于测试。
相比于Spring依赖注入，Guice更轻量级。Guice没有使用XML配置，而是直接使用注解和代码对依赖和绑定关系进行管理。它采用懒加载的方式使得应用启动更快，使用Module对绑定关系进行管理方便了应用的模块化开发。
## 简单示例

```
public interface MessageService
{
	String acquireMsg();
}

public class HelloWorldService implements MessageService
{
	@Override
	public String acquireMsg()
	{
		return "HelloWorld";
	}
}

public class EmptyService implements MessageService
{
	@Override
	public String acquireMsg()
	{
		return "";
	}
}

public class MessageServiceModule extends AbstractModule
{
	@Override
	protected void configure()
	{
		bind(MessageService.class).to(HelloWorldService.class).in(Singleton.class);
	}
}

public static void main(String[] args)
{
	Injector injector = Guice.createInjector(new MessageServiceModule());

	MessageService msgService = injector.getInstance(MessageService.class);
	EmptyService emptyService = injector.getInstance(EmptyService.class);

	System.out.println(emptyService.acquireMsg());
	System.out.println(msgService.acquireMsg());
}
```
main函数执行结果如下：
![](/assets/images/guice-1.png)

## 绑定的类型
### 链式绑定（Linked Binding）
链式绑定把一个类型映射到它的实现，下面这个示例把接口MessageService映射到它的实现HelloWorldService

```
public class MessageServiceModule extends AbstractModule
{
	@Override
	protected void configure()
	{
		bind(MessageService.class).to(HelloWorldService.class).in(Singleton.class);
	}
}
```

于是当我们调用方法```injector.getInstance(MessageService.class)```，或者Injector遇到一个关于```MessageService```的依赖的时候，注解器将使用```HelloWorldService```。

我们可以绑定一个类型到它的任何一个子类，比如它的实现类（implementing class）或者扩展类（extending class），甚至可以绑定一个具体类到它的子类，而且链式绑定还可以顺序链接起来。

```
public class WatoudHelloWorldService extends HelloWorldService
{
	@Override
	public String acquireMsg()
	{
		return "Hello World, I am Watoud.";
	}
}

public class MessageServiceModule extends AbstractModule
{
	@Override
	protected void configure()
	{
		bind(MessageService.class).to(HelloWorldService.class);
		bind(HelloWorldService.class).to(WatoudHelloWorldService.class);
	}
}
```

### 实例绑定（Instance binding）
我们可以绑定一个类型到它的一个特定实例，这种情况仅适合于没有其它任何依赖的类型。

```
bind(String.class).annotatedWith(Names.named("JDBC-URL")).toInstance("jdbc:mysql://localhost/watoud");
bind(Integer.class).annotatedWith(Names.named("login-timeout-seconds")).toInstance(10);
```

接下来可以像下面这样直接获取，

```
injector.getInstance(Key.get(String.class, Names.named("JDBC-URL")));
injector.getInstance(Key.get(int.class,Names.named("login-timeout-seconds")));
```

也可以在其它对象中注入这些值。

```
public class JdbcConfigService
{
	@Inject
	@Named("JDBC-URL")
	private String url;

	@Inject
	@Named("login-timeout-seconds")
	private int loginTimeOutSeconds;

	public String getUrl()
	{
		return this.url;
	}

	public int getLoginTimeoutSeconds()
	{
		return this.loginTimeOutSeconds;
	}
}
```

尽量避免对复杂的对象进行实例绑定，因为这样会降低应用程序的启动速度，这种情况可以使用```@Provides```注解的方法进行替代。
### @Provides方法（@Provides Methods）
@Provides方法必须定义在module里面，并且必须拥有@Provides注解，这个方法返回的类型就是它绑定的类型。当注解器需要这个类型的一个实例时，它将调用这个方法。

```
public class MessageServiceModule extends AbstractModule
{
	@Provides
	MessageService provideMsgService()
	{
		MessageService msgService = new HelloWorldService();
		return msgService;
	}
}
```

如果@Provides方法标有绑定注解，Guice将会绑定注解类型，依赖项可以通过这个方法的参数传递进来。在调用这个方法之前，注解器会先进行绑定。

```
public class JdbcServiceModule extends AbstractModule
{
	@Override
	protected void configure()
	{
		bind(String.class).annotatedWith(Names.named("JDBC-URL")).toInstance(
		        "jdbc:mysql://localhost/watoud");
		bind(Integer.class).annotatedWith(Names.named("login-timeout-seconds")).toInstance(10);
	}

	@Provides
	JdbcConfigService provideJdbcCfg(@Named("JDBC-URL") String url,
	        @Named("login-timeout-seconds") int loginTimeOutSeconds)
	{
		JdbcConfigService jdbcCfg = new JdbcConfigService(url, loginTimeOutSeconds);
		return jdbcCfg;
	}
}
```

### 提供者绑定（Provider Bindings）
当@Provides方法开发变得复杂的时候，可以考虑把它们独立出来形成一个新的类。这个提供者类型实现了Guice的```Provider```接口，这是一个用于提供实例的简单的泛型接口。

```
public interface Provider<T>
{
	T get();
}
```

提供者绑定需要实现```Provider```接口，用于定义返回的类型。它需要的依赖可以通过在构造器上使用@Inject注解来获取。

```
public class JdbcConfigProvider implements Provider<JdbcConfigService>
{

	private String url;

	private int timeOut;

	@Inject
	public JdbcConfigProvider(@Named("JDBC-URL") String url,
	        @Named("login-timeout-seconds") int timeOut)
	{
		this.url = url;
		this.timeOut = timeOut;
	}

	@Override
	public JdbcConfigService get()
	{
		JdbcConfigService jdbcCfg = new JdbcConfigService(this.url, this.timeOut);
		return jdbcCfg;
	}
}
```

最后使用```.toProvider```方法把某个类型绑定到它的提供者。

```
public class JdbcServiceModule extends AbstractModule
{
	@Override
	protected void configure()
	{
		bind(String.class).annotatedWith(Names.named("JDBC-URL")).toInstance(
		        "jdbc:mysql://localhost/watoud");
		bind(Integer.class).annotatedWith(Names.named("login-timeout-seconds")).toInstance(10);

		bind(JdbcConfigService.class).toProvider(JdbcConfigProvider.class);
	}
}
```

### 无目标绑定（Untargetted Binding）
有时我们可能会创建没有明确目标的绑定，对于具体的类以及标注有@ImplementedBy或者@ProvidedBy的类型，无目标绑定是很有用的。

```
	public class MessageServiceModule extends AbstractModule
	{
		@Override
		protected void configure()
		{
			bind(HelloWorldService.class);
		}
	}
```

当指定绑定注解的时候，必须加上绑定目标，即使这个目标是同一个具体类。

```
bind(MyConcreteClass.class).annotatedWith(Names.named("foo")).to(MyConcreteClass.class);
```

### 构造器绑定（Constructor Binding）
有时能够绑定一个类型到它的任何一个构造函数是很有必要的，这种情况发生在```@Inject```注解没法应用在目标类型构造器上的时候。比如，目标类型是第三方的，或者目标类型有多个构造函数都参与了依赖注入。使用```@Provides```注解能够很好的解决这个问题，但是这样做有一个缺陷，那就是手动创建的实例不能使用面向切面编程这一特性。为了解决这个问题，Guice给出了构造器绑定。这需要用户通过反射选择一个目标构造器并且处理这个过程中可能出现的异常。

```
public class JdbcConfigService
{
	private String url;

	private int loginTimeOutSeconds;

	@Inject
	public JdbcConfigService(@Named("JDBC-URL") String jdbcUrl)
	{
		this.url = jdbcUrl;
		this.loginTimeOutSeconds = 10000;
	}

	@Inject
	public JdbcConfigService(@Named("JDBC-URL") String jdbcUrl,
	        @Named("login-timeout-seconds") int loginTimeOutSeconds)
	{
		this.url = jdbcUrl;
		this.loginTimeOutSeconds = loginTimeOutSeconds;
	}

	public String getUrl()
	{
		return this.url;
	}

	public int getLoginTimeoutSeconds()
	{
		return this.loginTimeOutSeconds;
	}
}

public class MessageServiceModule extends AbstractModule
{
	@Override
	protected void configure()
	{
		bind(String.class).annotatedWith(Names.named("JDBC-URL")).toInstance(
		        "jdbc:mysql://localhost/watoud");
		bind(String.class).annotatedWith(Names.named("JDBC-URL-NEW")).toInstance(
		        "jdbc:mysql://localhost/watoud/NEW");
		bind(Integer.class).annotatedWith(Names.named("login-timeout-seconds")).toInstance(10);

		try
		{
			bind(JdbcConfigService.class).toConstructor(
			        JdbcConfigService.class.getConstructor(String.class, int.class));

				//或者绑定到另外一个构造器上
				// bind(JdbcConfigService.class).toConstructor(
				// JdbcConfigService.class.getConstructor(String.class));
		}
		catch (NoSuchMethodException e)
		{
			addError(e);
			e.printStackTrace();
		}
	}
}
```

每一个```toConstructor()```的作用域范围都是独立的，如果创建多个单例绑定，并且每个都绑定到同一个目标构造器，那么每个绑定都将创建它自己的实例。

### 即时绑定（Just-in-time Bindings）
当Injector需要某个类型的实例的时候，它需要一个绑定，那些定义在module中的绑定称为显示绑定。当需要一个类型的实例时，却不存在这个类型的显示绑定，那么Injector将尝试创建一个即时绑定，这种绑定也被称为隐式绑定。

- 符合条件的构造器
Guice通过使用具体类型的可注解构造器为这个类创建绑定。这个具体类型要么拥有一个非私有的、没有参数的构造器，要么拥有一个标有```@Inject```注解的构造器。

- @ImplementedBy
这个注解定义某个类型的默认实现。

```
@ImplementedBy(PayPalCreditCardProcessor.class)
public interface CreditCardProcessor {
  ChargeResult charge(String amount, CreditCard creditCard)
      throws UnreachableException;
}
```

上面的注解的效果同下面的绑定是一样的。

```
 bind(CreditCardProcessor.class).to(PayPalCreditCardProcessor.class);
```

当一个类型同时存在@ImplementedBy和绑定时，绑定优先。@ImplementedBy只是提供一个默认的实现，它能够被绑定覆盖。

- @ProvidedBy

```
@ProvidedBy(DatabaseTransactionLogProvider.class)
public interface TransactionLog {
  void logConnectException(UnreachableException e);
  void logChargeResult(ChargeResult result);
}
```

上面的语句通下面的效果一样。

```
bind(TransactionLog.class).toProvider(DatabaseTransactionLogProvider.class);
```

当一个类型同时存在@ProvidedBy注解和绑定时，绑定优先。


### 内置绑定（Built-in Bindings）

```
bindStage(injector, stage);
bindInjector(injector);
bindLogger(injector);
```

从Guice上述的源码中可以看出，Guice至少内置了以上三个绑定，即stage、logger和injector。

```
public static void main(String[] args)
{
	Injector injector = Guice.createInjector(new AbstractModule()
	{
		@Override
		protected void configure()
		{
		}
	});

	Logger logger = injector.getInstance(Logger.class);
	logger.log(Level.INFO, "This is a test");
}
```

从上面的代码可以看出，logger绑定是内置的。

## 注入的类型
依赖注入模式把行为和依赖解析分离开来。相比于直接或者从工厂方法查找依赖信息，依赖注入建议传入依赖信息。设置对象依赖信息的过程称为注入。
### 构造器注入
构造器注入把实例化和注入结合到一起，使用时在构造器加上@Inject注解。

```
public class RealBillingService implements BillingService
{
	private final CreditCardProcessor processorProvider;
 	private final TransactionLog transactionLogProvider;

  	@Inject
  	public RealBillingService(CreditCardProcessor processorProvider,
      TransactionLog transactionLogProvider)
    {
    	this.processorProvider = processorProvider;
    	this.transactionLogProvider = transactionLogProvider;
  	}
  	......
}
```

如果一个类没有@Inject标注的构造器，如果存在一个public并且没有参数的构造器，那么Guice将使用这个构造器。
### 方法注入
Guice能够对标有@Inject注解的方法进行注入。依赖项表现为方法参数的形式，在调用这个方法之前，注解器将会解析这些依赖项。

```
public class PayPalCreditCardProcessor implements CreditCardProcessor
{
  	private static final String DEFAULT_API_KEY = "development-use-only";
  	private String apiKey = DEFAULT_API_KEY;

  	@Inject
  	public void setApiKey(@Named("PayPal API key") String apiKey)
  	{
    	this.apiKey = apiKey;
  	}
  	......
}
```

### 字段注入
Guice对使用@Inject注解的字段进行注入，这个是最简明的注入，但是也是可测试性最差的注入。

```
public class DatabaseTransactionLogProvider implements Provider<TransactionLog>
{
  	@Inject Connection connection;

  	public TransactionLog get()
  	{
    	return new DatabaseTransactionLog(connection);
  	}
}
```

### 可选注入
有时候当依赖存在的时候就使用它，不存在的就使用不使用它，这种情况下就可以使用可选注入。方法注入和字段注入是可选的，当依赖项不存在的时候，Guice将会忽略它们。使用可选注入，只需要加上注解@Inject(optional=true)即可。

```
public class PayPalCreditCardProcessor implements CreditCardProcessor
{
  	private static final String SANDBOX_API_KEY = "development-use-only";
  	private String apiKey = SANDBOX_API_KEY;

  	@Inject(optional=true)
  	public void setApiKey(@Named("PayPal API key") String apiKey)
  	{
    	this.apiKey = apiKey;
  	}
  	......
}
```

### 按需注入
使用injector.injectMembers API，方法和字段注入能够用于初始化已经存在的实例。

```
public static void main(String[] args)
{
    Injector injector = Guice.createInjector(...);
    CreditCardProcessor creditCardProcessor = new PayPalCreditCardProcessor();
    injector.injectMembers(creditCardProcessor);
    ......
}
```

## 作用域（Scopes）
Guice默认每次返回一个新的实例，这个可以通过作用域来控制，作用域能够让我们在一定的范围内重用实例。

+ @Singleton：应用的整个生命周期
+ @SessionScoped: 一次会话
+ @RequestScoped：一个请求

作用域的配置有两种方法，一种是在类上使用注解```@Singleton```;另一种是在绑定语句中指定作用域，比如
```bind(MessageService.class).to(HelloWorldService.class).in(Singleton.class);```
如果同一个类型的作用域类型发生了冲突，将会使用绑定语句中的作用域。

需要注意一点的是，在链式绑定的作用域配置中，作用域是作用在绑定语句中的源类型上而非目标类型。假设类型Applebees实现了两个接口Bar以及Grill，下面这个绑定将产生两个Applebees实例，一个用于Bar，一个用于Grill。

```
 bind(Bar.class).to(Applebees.class).in(Singleton.class);
 bind(Grill.class).to(Applebees.class).in(Singleton.class);
```

如果希望只产生一个Applebees实例，我们需要添加另一个绑定信息。

```
bind(Applebees.class).in(Singleton.class);
```

## 面向切面编程
Guice在实现依赖注入的同时还支持面向切面编程。面向切面编程适合于横向关注点，比如事物、安全和日志。使用Guice实现面向切面比较简单，只需要选择匹配的类型和方法，创建一个拦截器并且在module中配置就好了。

```
public class MessageIntercepter implements MethodInterceptor
{
	@Override
	public Object invoke(MethodInvocation invocation) throws Throwable
	{
		System.out.println("Before invoke method");
		Object o = invocation.proceed();
		System.out.println("After invoke method");
		return o;
	}
}

protected void configure()
{
	bind(MessageService.class).to(HelloWorldService.class).in(Singleton.class);
	bindInterceptor(Matchers.any(), Matchers.any(), new MessageIntercepter());
}
```

Guice面向切面编程具有一些限制。

- 类必须是public或者package-private
- 类必须是non-final的
- 方法必须是non-final, public, package-private或者protected
- 实例必须由Guice创建

## 参考文献
1.[Guice User Guide:https://github.com/google/guice/wiki/Motivation](https://github.com/google/guice/wiki/Motivation)