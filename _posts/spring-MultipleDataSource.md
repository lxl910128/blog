---
title: spring配置多数据源
date: 2020/12/2 18:41:10
categories:
- spring
tags:
- spring
keywords:
- spring, 多数据源, AbstractRoutingDataSource
---
# 前言

在使用spring开发业务系统时，经常会遇到一个项目需要连接多个数据库配置多个数据源的问题。比如数据库做了读写分离，读写操作访问不同数据库。本文主要介绍借助spring提供的`AbstractRoutingDataSource`实现多数据源配置，用AOP和注解灵活切换数据源，实现读写分离。
<!--more-->
# 思路

读写分离的基本出发点是，为了减轻数据库压力，让主数据库尽量只处理增删改操作，从数据库只处理查询操作。其中主从数据库使用mysql自带的主从同步机制保证数据一致。在数据库配置好主从同步后，核心问题就是业务系统如何在使用的过程中根据需要切换使用的数据源。本文实现多数据源切换的主要思路是，我们借助spring提供的`AbstractRoutingDataSource`类实现可配置多数据源的`DataSource`。然后使用注解和AOP，当发现Service层的方法使用了只读的注解，则选择从库读取数据，如果没有注解则使用主库。

# AbstractRoutingDataSource

根据上述思路，多数据源的切换主要是借助spring的`AbstractRoutingDataSource`，下面主要介绍下这个类。源码200多行，下面我们先从类的说明入手：

```java
/**
 * Abstract {@link javax.sql.DataSource} implementation that routes {@link #getConnection()}
 * calls to one of various target DataSources based on a lookup key. The latter is usually
 * (but not necessarily) determined through some thread-bound transaction context.
 */
public abstract class AbstractRoutingDataSource extends AbstractDataSource implements InitializingBean {
}

```

首先，该抽象类继承了`AbstractDataSource`，也就是是他本质是`DataSource`接口的抽象实现。其次，在`getConnection()`时会基于一个key，选择出实际的`DataSource`。最后官方建议这个key可是通过线程上线文（`ThreadLocal`）传递。接着看`getConnection()`方法。

```java
@Nullable
	private Map<Object, DataSource> resolvedDataSources;

@Override
	public Connection getConnection() throws SQLException {
    // 从determineTargetDataSource中获取实际DataSource
		return determineTargetDataSource().getConnection();
	}
	/**
	 * Retrieve the current target DataSource. Determines the
	 * {@link #determineCurrentLookupKey() current lookup key}, performs
	 * a lookup in the {@link #setTargetDataSources targetDataSources} map,
	 * falls back to the specified
	 * {@link #setDefaultTargetDataSource default target DataSource} if necessary.
	 * @see #determineCurrentLookupKey()
	 */
	protected DataSource determineTargetDataSource() {
		Assert.notNull(this.resolvedDataSources, "DataSource router not initialized");
    // 从determineCurrentLookupKey中获取 key
		Object lookupKey = determineCurrentLookupKey();
    // 使用该key从resolvedDataSources中获取Datasource
		DataSource dataSource = this.resolvedDataSources.get(lookupKey);
		if (dataSource == null && (this.lenientFallback || lookupKey == null)) {
      // 未拿到取默认
			dataSource = this.resolvedDefaultDataSource;
		}
		if (dataSource == null) {
			throw new IllegalStateException("Cannot determine target DataSource for lookup key [" + lookupKey + "]");
		}
		return dataSource;
	}
	/**
	 * Determine the current lookup key. This will typically be
	 * implemented to check a thread-bound transaction context.
	 * <p>Allows for arbitrary keys. The returned key needs
	 * to match the stored lookup key type, as resolved by the
	 * {@link #resolveSpecifiedLookupKey} method.
	 */
	@Nullable
	protected abstract Object determineCurrentLookupKey();
```

从源码中能看出来，实际上该类通过`determineTargetDataSource()`方法选择实际执行的`DataSource`。在`determineTargetDataSource()`方法中通过`determineCurrentLookupKey()`方法获取到key，根据这个key从`resolvedDataSources`这个Map中取出对应的`DataSource`。如果没有取到，则使用默认的DataSource。其中取key的`determineCurrentLookupKey()`方法是需要用户自己实现的。

看到这里其实思路已经很明显了，我们需要继承实现`AbstractRoutingDataSource`，主要要干的事有：

1. 实现`determineCurrentLookupKey`方法，该方法需要返回1个key，这个key可以从`resolvedDataSources`中获取一个`datasource`
2. 构建`resolvedDataSources`这个map

针对第一个问题`determineCurrentLookupKey`方法的实现，官方已经给出了明确的提示`从线程上线文中获取key`，因此考虑调用方法时从`ThreadLocal`中拿指定ken值。下面是第二个问题，如果优雅的构造`resolvedDataSources`这个Map。相关代码如下：

```java
	@Override
	public void afterPropertiesSet() {
		if (this.targetDataSources == null) {
			throw new IllegalArgumentException("Property 'targetDataSources' is required");
		}
		this.resolvedDataSources = new HashMap<>(this.targetDataSources.size());
		this.targetDataSources.forEach((key, value) -> {
			Object lookupKey = resolveSpecifiedLookupKey(key);
			DataSource dataSource = resolveSpecifiedDataSource(value);
			this.resolvedDataSources.put(lookupKey, dataSource);
		});
		if (this.defaultTargetDataSource != null) {
			this.resolvedDefaultDataSource = resolveSpecifiedDataSource(this.defaultTargetDataSource);
		}
	}
	/**
	 * Specify the map of target DataSources, with the lookup key as key.
	 * The mapped value can either be a corresponding {@link javax.sql.DataSource}
	 * instance or a data source name String (to be resolved via a
	 * {@link #setDataSourceLookup DataSourceLookup}).
	 * <p>The key can be of arbitrary type; this class implements the
	 * generic lookup process only. The concrete key representation will
	 * be handled by {@link #resolveSpecifiedLookupKey(Object)} and
	 * {@link #determineCurrentLookupKey()}.
	 */
	public void setTargetDataSources(Map<Object, Object> targetDataSources) {
		this.targetDataSources = targetDataSources;
	}
```

通过观察可以发现该抽象类还实现了`InitializingBean`接口，在`afterPropertiesSet()`方法中，使用`targetDataSources` 的内容构建的`resolvedDataSources`。`targetDataSources` 有公有的set方法。这样看针对第二个问题的思路也很明显了。

我们在创建`AbstractRoutingDataSource`的 Bean时需要对`targetDataSources`进行赋值，在交个spring容器管理时,`targetDataSources`的内容会被付给`resolvedDataSources`。在赋值是还能通过`resolveSpecifiedLookupKey`和`resolveSpecifiedDataSource`方法对key和DataSource进行统一的转换。相关实现代码如下：

```java
@Slf4j
public class DynamicDataSource extends AbstractRoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        return DatabaseContextHolder.getDbType().orElse("master");
    }
}


```

```java
// ThreadLocal
public class DatabaseContextHolder {
    private static ThreadLocal<String> contextHolder = new ThreadLocal<String>();
    
    public static void setDbType(String dbType) {
        contextHolder.set(dbType);
    }
    
    public static Optional<String> getDbType() {
        return Optional.ofNullable(contextHolder.get());
    }
    
    public static void clearDbType() {
        contextHolder.remove();
    }
}

```

```java
@Configuration
public class DataSourceConfiguration {
  @Bean
    public DataSource getDataSource() {
        // 主数据库连接池
        HikariConfig masterConfig = new HikariConfig();
        masterConfig.setJdbcUrl(masterUrl);
        masterConfig.setUsername(masterUser);
        masterConfig.setPassword(masterPwd);
        masterConfig.addDataSourceProperty("connectionTimeout", "1000"); // 连接超时：1秒
        masterConfig.addDataSourceProperty("idleTimeout", "60000"); // 空闲超时：60秒
        masterConfig.addDataSourceProperty("maximumPoolSize", "10"); // 最大连接数：10
        DataSource master = new HikariDataSource(masterConfig);
        // 从数据库连接池
        HikariConfig slaveConfig = new HikariConfig();
        slaveConfig.setJdbcUrl(slaveUrl);
        slaveConfig.setUsername(slaveUser);
        slaveConfig.setPassword(slavePwd);
        slaveConfig.addDataSourceProperty("connectionTimeout", "1000"); // 连接超时：1秒
        slaveConfig.addDataSourceProperty("idleTimeout", "60000"); // 空闲超时：60秒
        slaveConfig.addDataSourceProperty("maximumPoolSize", "10"); // 最大连接数：10
        DataSource slave = new HikariDataSource(slaveConfig);
        
        Map<Object, Object> dsMap = new HashMap<>();
        dsMap.put("master", master);
        dsMap.put("slave", slave);
        DynamicDataSource dataSource = new DynamicDataSource();
        // 设置TargetDataSources
        dataSource.setTargetDataSources(dsMap);
        // 设置默认数据源
        dataSource.setDefaultTargetDataSource(master);
        return dataSource;
    }
}
```

# 业务实现

在上一部分我们清楚了`AbstractRoutingDataSource`本质是DataSource的抽象类，我们主要工作是实现这个抽象类并交给spring使用。那么在业务进行中我们需要做的主要就是在合适的时机设置`ThreadLocal`。本文提供的思路是借助注解和切面，注解合适的`Service`方法，在调用该方法时根据注解往`ThreadLocal`中保存合适的Key。这里逻辑比较清晰，直接展示相关代码

```java
@Aspect
@Component
@Slf4j
public class ServiceAOP {
    @Pointcut("@annotation(club.gaiaproject.homework.source.common.ReadOnly)")
    public void readOnly() {
    }
    
    @Before("readOnly()")
    public void doBefore() {
        DatabaseContextHolder.setDbType("slave");
    }
    
}
```

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface ReadOnly {
}
```

使用方法就是在只有读业务的`Service`方法上添加`@readOnly`注解即可。

# 总结

业务中实现读写分离是比较常见的场景，他可以有效的减轻数据库的压力，横向扩展数据库能力。本文介绍了在业务端手动实现切换数据源。其实，有一些插件已经封装了该功能，比如Apache的开源项目——`ShardingSphere`，它提供了两种实现方式。有侵入代码的`ShardingSphere-jdbc`，他会接管你的数据库连接，在此处做读写分类。非侵入代码的`ShardingSphere-proxy`，他会接管你的数据库，你像连接普通数据库一样连接`ShardingSphere-proxy`就可享受读写分离、数据分片、分布式事务等功能（ShardingSphere-jdbc也可实现）。有兴趣的小伙伴可以自行了解。