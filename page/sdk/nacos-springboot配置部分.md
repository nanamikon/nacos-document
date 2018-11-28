#配置例子
官方例子代码如下
```java
@Controller
@RequestMapping("config")
@NacosPropertySource(dataId = "example.properties", autoRefreshed = true)
public class ConfigController {

    @Value("${useLocalCache:false}")
    private boolean useLocalCache;

    public void setUseLocalCache(boolean useLocalCache) {
        this.useLocalCache = useLocalCache;
    }

    @RequestMapping(value = "/get", method = GET)
    @ResponseBody
    public boolean get() {
        return useLocalCache;
    }
}
```

```properties
nacos.config.server-addr=127.0.0.1:8848
```

有几个关键点
1. NacosPropertySource注解用于指定需要加载的配置信息。  其实最好是都放在一个地方，方便管理， 例如放在入口com.alibaba.nacos.example.spring.boot.NacosConfigApplication
2、**必须提供相应的set方法**， 才能实现autoRefreshed，后面会说一下实现细节
3、nacos.config.server-addr配置服务端的地址

#配置注入实现细节
配置注入分为两个部分， 一个是bean初始化的时候，如何注入变量， 一个是配置变化是，如何体现到bean中

先看第一点
```java
public class NacosPropertySourcePostProcessor implements BeanDefinitionRegistryPostProcessor, BeanFactoryPostProcessor,
        EnvironmentAware, Ordered {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

        this.nacosPropertySourceBuilders = beanFactory.getBeansOfType(AbstractNacosPropertySourceBuilder.class).values();
        this.configServiceBeanBuilder = getConfigServiceBeanBuilder(beanFactory);

        String[] beanNames = beanFactory.getBeanDefinitionNames();

        for (String beanName : beanNames) {
            processPropertySource(beanName, beanFactory);
        }

    }
    
    private void processPropertySource(String beanName, ConfigurableListableBeanFactory beanFactory) {

        BeanDefinition beanDefinition = beanFactory.getBeanDefinition(beanName);
        // Build multiple instance if possible
        List<NacosPropertySource> nacosPropertySources = buildNacosPropertySources(beanName, beanDefinition);

        // Add Orderly
        for (NacosPropertySource nacosPropertySource : nacosPropertySources) {
            addNacosPropertySource(nacosPropertySource);
            addListenerIfAutoRefreshed(nacosPropertySource);
        }

    }
    
    private void addNacosPropertySource(NacosPropertySource nacosPropertySource) {

        MutablePropertySources propertySources = environment.getPropertySources();

        boolean first = nacosPropertySource.isFirst();
        String before = nacosPropertySource.getBefore();
        String after = nacosPropertySource.getAfter();

        boolean hasBefore = !nullSafeEquals(DEFAULT_STRING_ATTRIBUTE_VALUE, before);
        boolean hasAfter = !nullSafeEquals(DEFAULT_STRING_ATTRIBUTE_VALUE, after);

        boolean isRelative = hasBefore || hasAfter;

        if (first) { // If First
            propertySources.addFirst(nacosPropertySource);
        } else if (isRelative) { // If relative
            if (hasBefore) {
                propertySources.addBefore(before, nacosPropertySource);
            }
            if (hasAfter) {
                propertySources.addAfter(after, nacosPropertySource);
            }
        } else {
            propertySources.addLast(nacosPropertySource); // default add last
        }
    }
    
    private void addListenerIfAutoRefreshed(final NacosPropertySource nacosPropertySource) {

        if (!nacosPropertySource.isAutoRefreshed()) { // Disable Auto-Refreshed
            return;
        }

        final String dataId = nacosPropertySource.getDataId();
        final String groupId = nacosPropertySource.getGroupId();
        final Map<String, Object> nacosPropertiesAttributes = nacosPropertySource.getAttributesMetadata();
        final ConfigService configService = configServiceBeanBuilder.build(nacosPropertiesAttributes);

        try {
            configService.addListener(dataId, groupId, new AbstractListener() {

                @Override
                public void receiveConfigInfo(String config) {
                    String name = nacosPropertySource.getName();
                    NacosPropertySource newNacosPropertySource = new NacosPropertySource(name, config);
                    newNacosPropertySource.copy(nacosPropertySource);
                    MutablePropertySources propertySources = environment.getPropertySources();
                    // replace NacosPropertySource
                    propertySources.replace(name, newNacosPropertySource);
                }
            });
        } catch (NacosException e) {
            throw new RuntimeException("ConfigService can't add Listener with properties : " + nacosPropertiesAttributes, e);
        }
    }
    ......
}
```

可以看到在NacosPropertySourcePostProcessor中，是通过BeanDefinitionRegistryPostProcessor中的扩展点， 一些扩展点的接口可以参考[这篇文章](https://blog.csdn.net/soonfly/article/details/69480058)
如下图所示
![](https://github.com/nanamikon/nacos-document/blob/master/spring-bean-life-cycle.png)

这个扩展点可以在bean初始化之前， 做一些操作， 而nacos就是在这一步将environment中的propertySources进行修改，后续bean就会使用配置的变量进行初始化。
而从postProcessBeanFactory这个实现可以看出来， 全部的bean都会扫一遍NacosPropertySource注解， 然后添加到environment中

addNacosPropertySource方法看到， 添加到environment的过程， 是按照注解中定义的顺序来进行的；而addListenerIfAutoRefreshed可以看到，需要自动刷新的，会通过sdk来监听变化，改变对应的值。
这里有一个问题是这里的自动刷新功能其实是无法影响到那些已经创建的bean的，因为没有修改对应bean的属性，也没有触发bean重新创建， 那这个功能实际是在哪里触发的呢？


```java
final ConfigService configService = configServiceBeanBuilder.build(nacosPropertiesAttributes);
```
首先这里创建的ConfigService实现对应的是EventPublishingConfigService， 其中addListener的实现如下

```java
    @Override
    public void addListener(String dataId, String group, Listener listener) throws NacosException {
        Listener listenerAdapter = new DelegatingEventPublishingListener(configService, dataId, group, applicationEventPublisher, executor, listener);
        configService.addListener(dataId, group, listenerAdapter);
        publishEvent(new NacosConfigListenerRegisteredEvent(configService, dataId, group, listener, true));
    }

```

DelegatingEventPublishingListener的实现如下
```java
final class DelegatingEventPublishingListener implements Listener {
    @Override
    public void receiveConfigInfo(String content) {
        publishEvent(content);
        onReceived(content);
    }

    private void publishEvent(String content) {
        NacosConfigReceivedEvent event = new NacosConfigReceivedEvent(configService, dataId, groupId, content);
        applicationEventPublisher.publishEvent(event);
    }

    private void onReceived(String content) {
        delegate.receiveConfigInfo(content);
    }
    .....
}
```
可以看到是在正常的onReceived操作之前， 触发了一个事件，对应的事件是NacosConfigReceivedEvent。 这里用到了spring的事件驱动模型， 那就是会有对应的事件处理类 NacosValueAnnotationBeanPostProcessor， 实现如下
```java
public class NacosValueAnnotationBeanPostProcessor extends AnnotationInjectedBeanPostProcessor<NacosValue>
    implements BeanFactoryAware, ApplicationListener<NacosConfigReceivedEvent> {
    @Override
    public void onApplicationEvent(NacosConfigReceivedEvent event) {
        String content = event.getContent();
        if (content != null) {
            Map<Object, List<BeanProperty>> map = new HashMap<Object, List<BeanProperty>>();
            Properties configProperties = toProperties(content);
            for (Object key : configProperties.keySet()) {
                List<BeanProperty> beanPropertyList = placeholderPropertyListMap.get(key.toString());
                if (beanPropertyList == null) {
                    continue;
                }
                for (BeanProperty beanProperty : beanPropertyList) {
                    List<Object> beanList = propertyBeanListMap.get(beanProperty);
                    if (beanList == null) {
                        continue;
                    }
                    for (Object bean : beanList) {
                        beanProperty.setKey((String)key);
                        put2ListMap(map, bean, beanProperty);
                    }
                }
            }
            doBind(map, configProperties);
        }

    }

    private void doBind(Map<Object, List<BeanProperty>> map,
                        Properties configProperties) {
        for (Map.Entry<Object, List<BeanProperty>> entry : map.entrySet()) {
            Object bean = entry.getKey();
            List<BeanProperty> beanPropertyList = entry.getValue();
            PropertyValues propertyValues = resolvePropertyValues(bean, configProperties, beanPropertyList);

            DataBinder dataBinder = new DataBinder(bean);
            dataBinder.bind(propertyValues);
        }
    }
}
```

可以看到，接收到事件后， 将配置内容中配置key对应的bean都找出来， 然后通过DataBinder， 利用BeanWrapper来给对象属性赋值， 而这种方式，需要
有对应的set方法才可以实现。 这样就实现了变量的动态修改

