---
layout: post
title: Guice与web以及数据库的整合
date: 2015-07-25
comments: true
archive: true
---
### Guice与web的整合
1. 创建web工程    
首先在eclipse中创建一个maven工程，修改工程中的pom.xml文件，在project节点下面添加```<packaging>war</packaging>```。然后在src/main目录下依次创建webapp/WEB-INF目录，再在WEB-INF目录下创建web.xml文件。这样一个使用maven进行依赖管理的web工程就搭建起来了，工程创建好后的目录结构如下图所示。

![](/assets/images/guice-web.png)

2.在pom.xml中添加maven依赖配置

```
<dependencies>
  	<dependency>
	    <groupId>com.google.inject.extensions</groupId>
	    <artifactId>guice-servlet</artifactId>
	    <version>4.0</version>
	</dependency>

	<dependency>
		<groupId>javax.servlet</groupId>
		<artifactId>javax.servlet-api</artifactId>
		<version>3.1.0</version>
		<scope>provided</scope>
	</dependency>

	<dependency>
	    <groupId>com.google.inject</groupId>
	    <artifactId>guice</artifactId>
	    <version>4.0</version>
	</dependency>
</dependencies>
```

3.在web.xml中添加GuiceFilter配置

```
<filter>
    <filter-name>guiceFilter</filter-name>
    <filter-class>com.google.inject.servlet.GuiceFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>guiceFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

4.编写servlet类

```
@Singleton
public class HelloWorldServlet extends HttpServlet
{
	private final String contents;

	@Inject
	public HelloWorldServlet(@Named("contents") String contents)
	{
		this.contents = contents;
	}

	@Override
	public void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException,
	        IOException
	{
		PrintWriter out = resp.getWriter();

		out.println("<html>");
		out.println("<head><title>WEB开发</title></head>");
		out.println("<body>");
		out.println("<h1>" + contents + "</h1>");
		out.println("</body>");
		out.println("</html>");
		out.close();
	}
}
```

5.编写servlet module以及其它module

```
public class GuiceServletModule extends ServletModule
{
	@Override
	protected void configureServlets()
	{
		serve("*").with(HelloWorldServlet.class);
	}
}

public class ServiceModule extends AbstractModule
{
	@Override
	protected void configure()
	{
		bind(String.class).annotatedWith(Names.named("contents"))
		        .toInstance("Hello world...");
	}
}
```

6.配置ServletContextListener    
首先继承GuiceServletContextListener配置所需的module，

```
public class MyGuiceServletConfig extends GuiceServletContextListener
{
	@Override
	protected Injector getInjector()
	{
		return Guice
		        .createInjector(new GuiceServletModule(), new ServiceModule());
	}
}
```

接着在web.xml中添加配置项。

```
<listener>
  		<listener-class>com.watoud.learn.guice.demo.integration.MyGuiceServletConfig</listener-class>
</listener>
```

7.打包并运行
使用maven命令打包，把war放到tomcat的webapps目录下，然后启动tomcat，使用浏览器访问，就能在浏览器中看到如下所示结果了。
![](/assets/images/guice-web-result.png)

### guice与数据库的整合
1.在web.xml中添加如下maven依赖配置

```
	<dependency>
	    <groupId>org.mybatis</groupId>
	    <artifactId>mybatis-guice</artifactId>
	    <version>3.6</version>
	</dependency>

	<dependency>
	    <groupId>com.google.inject.extensions</groupId>
	    <artifactId>guice-multibindings</artifactId>
	    <version>4.0</version>
	</dependency>

	<dependency>
	    <groupId>org.apache.commons</groupId>
	    <artifactId>commons-dbcp2</artifactId>
	    <version>2.1</version>
	</dependency>

	<dependency>
	    <groupId>mysql</groupId>
	    <artifactId>mysql-connector-java</artifactId>
	    <version>5.1.36</version>
	</dependency>

	<dependency>
	    <groupId>org.mybatis</groupId>
	    <artifactId>mybatis</artifactId>
	    <version>3.3.0</version>
	</dependency>
```

2.添加数据库配置信息
在src/mian/resources目录下添加配置文件guice.properties，在文件中添加如下数据库配置信息。

```
jdbc_username=watoud
jdbc_passwd=watoud
jdbc_driver=com.mysql.jdbc.Driver
jdbc_url=jdbc:MySQL://localhost:3306/watoud
```

3.提供DataSource类

```
public class DataSourceProvider implements Provider<DataSource>
{
	private BasicDataSource dataSource;

	@Inject
	public DataSourceProvider(@Named("jdbc_url") String url,
	        @Named("jdbc_username") String username, @Named("jdbc_passwd") String passwd, @Named("jdbc_driver") String driver)
	{
		dataSource = new BasicDataSource();
		dataSource.setUrl(url);
		dataSource.setUsername(username);
		dataSource.setPassword(passwd);
		dataSource.setDriverClassName(driver);
	}

	@Override
	public DataSource get()
	{
		return dataSource;
	}
}
```

4.创建数据库表以及编写数据库映射mapper
在数据库中创建user表，并插入数据。

```
create table user(id int(20),name varchar(20),addr varchar(100));
insert into user(id,name,addr) values(001,'watoud','every where');
```

编写User数据模型类以及映射类

```
public class User
{
	private int user;
	private String name;
	private String addr;

	public int getUser()
	{
		return user;
	}

	public void setUser(int user)
	{
		this.user = user;
	}

	public String getName()
	{
		return name;
	}

	public void setName(String name)
	{
		this.name = name;
	}

	public String getAddr()
	{
		return addr;
	}

	public void setAddr(String addr)
	{
		this.addr = addr;
	}
}

public interface UserMapper
{
	@Select("Select * from user")
	List<User> selectUsers();
}
```

5.编写并配置数据库module以及其它module

```
public class JdbcModule extends MyBatisModule
{

	@Override
	protected void initialize()
	{
		bindDataSourceProviderType(DataSourceProvider.class);
		bindTransactionFactoryType(JdbcTransactionFactory.class);
		addMapperClass(UserMapper.class);
	}
}
```
在ServiceModule类中添加如下代码。

```
@Override
	protected void configure()
	{
		......

		bind(String.class).annotatedWith(Names.named("mybatis.environment.id")).toInstance("test");

		loadGuiceConfig();
	}

	private void loadGuiceConfig()
	{
		Properties properties = new Properties();
		InputStream input = ServiceModule.class.getResourceAsStream("/guice.properties");
		if (input != null)
		{
			try (Reader reader = new InputStreamReader(input))
			{
				properties.load(reader);
			}
			catch (IOException e)
			{
				e.printStackTrace();
			}
		}

		Set<Object> keys = properties.keySet();
		for (Object key : keys)
		{
			bindConstant().annotatedWith(Names.named(key.toString())).to(
			        properties.get(key).toString());
		}
	}

```

修改MyGuiceServletConfig如下。

```
public class MyGuiceServletConfig extends GuiceServletContextListener
{
	@Override
	protected Injector getInjector()
	{
		return Guice
		        .createInjector(new JdbcModule(), new GuiceServletModule(), new ServiceModule());
	}
}
```

6.修改HelloWorldServlet，从数据库中获取用户信息

```
@Singleton
public class HelloWorldServlet extends HttpServlet
{
	private static final long serialVersionUID = 1L;
	private final String contents;

	private final UserMapper mapper;

	@Inject
	public HelloWorldServlet(@Named("contents") String contents, UserMapper mapper)
	{
		this.contents = contents;
		this.mapper = mapper;
	}

	@Override
	public void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException,
	        IOException
	{
		PrintWriter out = resp.getWriter();

		out.println("<html>");
		out.println("<head><title>WEB开发</title></head>");
		out.println("<body>");
		out.println("<h1>" + contents + ", " + this.mapper.selectUsers().get(0).getName() + "</h1>");
		out.println("</body>");
		out.println("</html>");
		out.close();
	}
}
```

7.打包并运行
使用maven命令打包并在tomcat中运行，就能在浏览器中看到如下所示结果了。
![](/assets/images/guice-mybatis.png)

### 参考资料
Guice与web整合:[https://github.com/google/guice/wiki/Servlets](https://github.com/google/guice/wiki/Servlets)    
Guice与mybatis整合:[http://mybatis.github.io/guice/core.html](http://mybatis.github.io/guice/core.html)
