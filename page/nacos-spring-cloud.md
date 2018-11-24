## nacos官方介绍
https://nacos.io/zh-cn/docs/what-is-nacos.html


## 整体例子跑通步骤

1. [下载地址](https://github.com/alibaba/nacos/releases) 下载最新的release包， 解压后执行bin目录下面的 startup.cmd (mac or linux 下执行startup.sh)

1. 服务默认以8848端口启动， 访问 http://localhost:8848/nacos 看是否正常
    ![](https://github.com/nanamikon/nacos-document/blob/master/YY%E5%9B%BE%E7%89%8720181124155303.jpg)
    
1. 在nacos的配置管理页面， 添加一个新的配置， data-id为example.properties, 配置内容如下
    ```yaml
    useLocalCache:true
    ```
    配置格式为properties， 其他都不需要改，用默认即可
1. [下载地址](https://github.com/nacos-group/nacos-examples)checkout代码，导入到ide中

1. com.alibaba.nacos.example.spring.cloud.NacosConfigApplication是入口main方法所在地， 通过他来启动

1. 请求http://localhost:8080/config/get,  看到返回**true**，说明配置生效了。 这时候在nacos配置页面修改useLocalCache的配置值，就可以立刻获取到最新值

## 实现细节分析

整体设计和spring-cloud-config类似, 也是通过bootstrap.properties，在context初始化之前， 从远处配置服务获取配置。
data-id就是配置文件的名字， 内容就是具体配置的键值对 

参考例子中的bootstrap.properties配置。
```properties
spring.cloud.nacos.config.server-addr=127.0.0.1:8848
spring.application.name=example
```

spring.cloud.nacos.config中相关的属性配置可以参考org.springframework.cloud.alibaba.nacos.NacosConfigProperties中的说明
spring.application.name则是指定了工程的应用名。

nacos-spring-cloud实现了自己的PropertySourceLocator实现org.springframework.cloud.alibaba.nacos.client.NacosPropertySourceLocator
其中最关键的是下面三个方法

```java
@Override
	public PropertySource<?> locate(Environment env) {

		ConfigService configService = nacosConfigProperties.configServiceInstance();

		if (null == configService) {
			logger.warn(
					"no instance of config service found, can't load config from nacos");
			return null;
		}
		long timeout = nacosConfigProperties.getTimeout();
		nacosPropertySourceBuilder = new NacosPropertySourceBuilder(configService,
				timeout);

		String name = nacosConfigProperties.getName();

		String nacosGroup = nacosConfigProperties.getGroup();
		String dataIdPrefix = nacosConfigProperties.getPrefix();
		if (StringUtils.isEmpty(dataIdPrefix)) {
			dataIdPrefix = name;
		}

		String fileExtension = nacosConfigProperties.getFileExtension();

		CompositePropertySource composite = new CompositePropertySource(
				NACOS_PROPERTY_SOURCE_NAME);

		loadApplicationConfiguration(composite, nacosGroup, dataIdPrefix, fileExtension);

		return composite;
	}

	private void loadApplicationConfiguration(
			CompositePropertySource compositePropertySource, String nacosGroup,
			String dataIdPrefix, String fileExtension) {
		loadNacosDataIfPresent(compositePropertySource,
				dataIdPrefix + DOT + fileExtension, nacosGroup, fileExtension);
		for (String profile : nacosConfigProperties.getActiveProfiles()) {
			String dataId = dataIdPrefix + SEP1 + profile + DOT + fileExtension;
			loadNacosDataIfPresent(compositePropertySource, dataId, nacosGroup,
					fileExtension);
		}
	}
	
    private void loadNacosDataIfPresent(final CompositePropertySource composite,
            final String dataId, final String group, String fileExtension) {
        NacosPropertySource ps = nacosPropertySourceBuilder.build(dataId, group,
                fileExtension);
        if (ps != null) {
            composite.addFirstPropertySource(ps);
        }
    }
```

可以看到在nacos-spring-cloud的设计中，  data-id的命名规则如下
(${spring.cloud.nacos.config.prefix} | ${spring.application.name})-(foreach ${spring.profiles.active}).${spring.cloud.nacos.config.fileExtension:properties} 

结合例子的配置文件， 最终为
example.properties

总体来说用法和spring-cloud-config差不多， 区别在于配置的文件名（data-id）不需要完全遵从spring的规范， 可以灵活的做区分。
同时也支持不同profile有不同的配置文件， 基本从spring-cloud-config切换过去不会有大的问题

## RefreshScope实现
原本spring cloud热更新的功能，是需要集成Spring Cloud Bus ， 然后统一通过 /refresh 这个Endpoint来接收Spring Cloud Bus发送过来的endpoint来实现热更新的。
nacos-spring-cloud的做法有点不同， 代码如下

org.springframework.cloud.alibaba.nacos.refresh.NacosContextRefresher
```java
public class NacosContextRefresher implements ApplicationListener<ApplicationReadyEvent> {
	@Override
	public void onApplicationEvent(ApplicationReadyEvent event) {
		this.registerNacosListenersForApplications();
	}

	private void registerNacosListenersForApplications() {
		if (refreshProperties.isEnabled()) {
			for (NacosPropertySource nacosPropertySource : nacosPropertySourceRepository
					.getAll()) {
				String dataId = nacosPropertySource.getDataId();
				registerNacosListener(dataId);
			}
		}
	}
	
	private void registerNacosListener(final String dataId) {
    
    		Listener listener = listenerMap.computeIfAbsent(dataId, i -> new Listener() {
    			@Override
    			public void receiveConfigInfo(String configInfo) {
    				String md5 = "";
    				if (!StringUtils.isEmpty(configInfo)) {
    					try {
    						MessageDigest md = MessageDigest.getInstance("MD5");
    						md5 = new BigInteger(1, md.digest(configInfo.getBytes("UTF-8")))
    							.toString(16);
    					} catch (NoSuchAlgorithmException | UnsupportedEncodingException e) {
    						logger.warn("unable to get md5 for dataId: " + dataId, e);
    					}
    				}
    				refreshHistory.add(dataId, md5);
    				contextRefresher.refresh();
    			}
    
    			@Override
    			public Executor getExecutor() {
    				return null;
    			}
    		});
    
    		try {
    			configService.addListener(dataId, properties.getGroup(), listener);
    		} catch (NacosException e) {
    			e.printStackTrace();
    		} 
	}
	......
}
```

可以看到是在context ready的时候，  将所有nacos-spring-cloud管理的PropertySource取出来， 然后通过sdk， 监听配置的变化， 实现热更新。
中间如何刷新最关键的点是 contextRefresher.refresh()， 代码如下

```java
	public synchronized Set<String> refresh() {
		Map<String, Object> before = extract(
				this.context.getEnvironment().getPropertySources());
		addConfigFilesToEnvironment();
		Set<String> keys = changes(before,
				extract(this.context.getEnvironment().getPropertySources())).keySet();
		this.context.publishEvent(new EnvironmentChangeEvent(context, keys));
		this.scope.refreshAll();
		return keys;
	}
```

原生spring cloud热更新也是通过这个方法， 从新走一次环境初始化流程， 获取到最新的配置和目前配置的差异， 修改过配置的key会触发事件，最后才会进行对有**RefreshScope注解**的bean进行统一的刷新 


## 配置优先级问题

总体来说配置的优先级[官方文档](https://docs.spring.io/spring-boot/docs/2.0.4.RELEASE/reference/htmlsingle/#boot-features-external-config)有介绍

而org.springframework.cloud.alibaba.nacos.client.NacosPropertySourceLocator的实现和加载方式如下

```java
@Order(0)
public class NacosPropertySourceLocator implements PropertySourceLocator {
    .....
}
```

```java
@Configuration
@EnableConfigurationProperties(PropertySourceBootstrapProperties.class)
public class PropertySourceBootstrapConfiguration implements
		ApplicationContextInitializer<ConfigurableApplicationContext>, Ordered {
            	@Override
            	public void initialize(ConfigurableApplicationContext applicationContext) {
            		CompositePropertySource composite = new CompositePropertySource(
            				BOOTSTRAP_PROPERTY_SOURCE_NAME);
            		AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
            		boolean empty = true;
            		ConfigurableEnvironment environment = applicationContext.getEnvironment();
            		for (PropertySourceLocator locator : this.propertySourceLocators) {
            			PropertySource<?> source = null;
            			source = locator.locate(environment);
            			if (source == null) {
            				continue;
            			}
            			logger.info("Located property source: " + source);
            			composite.addPropertySource(source);
            			empty = false;
            		}
            		if (!empty) {
            			MutablePropertySources propertySources = environment.getPropertySources();
            			String logConfig = environment.resolvePlaceholders("${logging.config:}");
            			LogFile logFile = LogFile.get(environment);
            			if (propertySources.contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
            				propertySources.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
            			}
            			insertPropertySources(propertySources, composite);
            			reinitializeLoggingSystem(environment, logConfig, logFile);
            			setLogLevels(applicationContext, environment);
            			handleIncludedProfiles(environment);
            		}
            	}
            	.....
		}
```

可以看到按照自然顺序的话， nacos的配置是排第一位， 优先级比其他都高， 可以认为远端的配置最优先，而在远端的配置中， 优先级最高的profile对应的配置也是最高的，这些和spring-cloud-config是一致的