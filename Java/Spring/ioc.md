# IOC

内容来源
* 《Spring揭秘》- 王福强

---

## 容器初始化阶段扩展

Spring容器实现功能的过程可以分为两个阶段，在每个阶段可以加入相应的容器扩展点

* 容器启动阶段
* Bean实例化阶段

### 容器启动阶段

* 首先会通过某种方式加载 `Configuration MetaData` 
* 接下来容器会依赖于某些工具类去解析和分析 `Configruation MetaData`
* 将解析后的信息组织为 `BeanDefinition` 
* 最后将 `BeanDefinition` 注册到 `BeanDefinitionRegister`

### Bean实例化阶段

当某个请求方通过容器的 `getBean` 方法明确的请求某个对象时，或者因为依赖关系需要隐式地调用 `getBean` 方法时，就会触发实例化阶段的工作

* 容器首先检查请求的对象之前是否已经初始化
* 如果没有，则会根据注册的 `BeanDefinition` 实例化被请求对象，并为其注入依赖
* 如果被请求对象实现了某些回调接口，则会根据回调接口的要求来装配
* 被请求对象装配完成后，容器立刻将对象返回给请求方

### 插手容器启动

#### 机制

Spring提供了 `BeanFactoryPostProcessor` 接口作为容器初始化的扩展机制  
该机制允许我们在容器实例化相应对象前修改注册到容器的对应的 `BeanDefinition` 

`BeanFactoryPostProcessor` 接口定义

```java
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;

@FunctionalInterface
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

因为容器中可能有多个 `BeanFactoryPostProcessor` ，在执行顺序重要的情况下，可能也需要实现 `Ordered` 接口来保证执行顺序

`Ordered` 接口定义

```java
package org.springframework.core;

public interface Ordered {
    
	int HIGHEST_PRECEDENCE = Integer.MIN_VALUE;
    
	int LOWEST_PRECEDENCE = Integer.MAX_VALUE;
    
	int getOrder();
}
```

---

#### *BeanFactoryPostProcessor*实现类 

因为Spring已经提供了多个 `BeanFactoryPostProcessor` 的实现类，所以大部分情况下我们不需要主动实现

其中常用类
* ~~PropertyPlaceholderConfigurer~~ `PropertySourcesPlaceholderConfigurer`
  * `PropertyPlaceholderConfigurer` 在 Spring 5.2 中弃用
* `PropertyOverrideConfigurer`
* `CustomEditorConfigurer`
  * 可以注册自定义 `PropertyEditor` 来处理配置文件中的数据类型与真正业务对象定义的数据类型之间的转换

##### *PropertySourcesPlaceholderConfigurer*

允许在XML文件中使用占位符 `${}` ，并将这些占位符所代表的资源单独配置到简单的properties文件中来加载  
当 `BeanFactory` 完成第一阶段加载所有配置信息时，保存的对象信息还都是以占位符形式存在  
当该类作为后处理器被应用时，会使用properties文件中的配置信息来替换响应 `BeanDefinition` 中占位符所代表的属性

**注意**  
该处理器不止会从指定的*properties*文件中加载配置项，还会检查 *Java* `System` 类中的 `Properties`  
可以通过调用方法来改变模式选择是否加载`System` 类中的 `Properties` 或者覆盖它  

##### *PropertyOverrideConfigurer* 

该后处理器可以通过指定的 *properties* 文件对 `BeanDefinition` 中任意 *property* 信息进行覆盖替换  
当容器中有多个 `PropertyOverrideConfigurer` 对同一个 *bean* 的同一个 *property* 处理时，最后一个生效  
配置文件规则

```properties
beanName.propertyName=value
```

##### *CustomEditorConfigurer*

辅助性地将后期会用到的信息注册到容器中，对 `BeanDefinition` 没有任何更改

继承 `PropertyEditorSupport` 接口来实现自定义 *PropertyEditor*

---

### Bean的生命周期

`BeanFactory` 的 `getBean` 方法的隐式调用有两种情况
* 当被请求对象的依赖对象还没有实例化时，容器会优先实例化依赖对象，即容器内部调用 `getBean` 方法，这种情况为隐式调用
* `ApplicationContext` 启动之后会实例化所有的Bean定义

`getBean` 方法只有在第一次调用时才会触发实例化阶段活动，第二次调用会返回容器缓存的第一次实例化的对象  
第一次实例化时会调用 `createBean` 方法来进行具体的实例化操作

#### 实例化过程

* 设置对象属性
* 检查Aware相关接口并设置相关依赖
* BeanPostProcessor前置处理
* 检查是否是初始化以决定是否调用 `afterPropertiesSet` 方法
* 检查是否有自定义的 `init-method` 
* BeanPostProcessor后置处理
* 注册必要的 `Destrction` 相关回调接口
* 是否实现 `DisposableBean` 接口
* 是否配置自定义的 `destory` 方法

#### 策略模式实例化Bean

可以通过反射或cglib动态字节码生成来初始化响应的bean实例或者动态生成其子类

实例化策略抽象接口 `InstantiationStrategy`  
接口定义
```java
package org.springframework.beans.factory.support;

import java.lang.reflect.Constructor;
import java.lang.reflect.Method;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.lang.Nullable;

public interface InstantiationStrategy {
    
	Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner)
			throws BeansException;
    
	Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
			Constructor<?> ctor, Object... args) throws BeansException;
    
	Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
			@Nullable Object factoryBean, Method factoryMethod, Object... args)
			throws BeansException;
    
	default Class<?> getActualBeanClass(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner) {
		return bd.getBeanClass();
	}

}
```

其直接子类 `SimpleInstantiationStrategy` 实现了简单的对象实例化功能，可以通过反射来实例化对象  
但不支持方法注入方式的对象实例化

`CglibSubclassingInstantiationStrategy` 继承自 `SimpleInstantiationStrategy` 并且可以动态生成子类满足方法注入的需要  
默认情况下，容器内部采用的是 `CglibSubclassingInstantiationStrategy`  
但是并非直接返回Bean的实例化对象，而是先返回响应的 `BeanWrapper` 实例

#### BeanWrapper

`BeanWapper` 接口一般在Spring框架内部使用，有一个实现类 `BeanWapperImpl` ，主要作用是设置或者获取Bean的属性值  
该接口继承 `PropertyAccessor` 接口，可以以统一方式对对象属性进行访问  
同时也继承 `PropertyEditorRegistrar` 与 `TypeConverter` 接口，在 `CustomEditorConfigurer` 中注册的 `PropertyEditor` 将由 `BeanWapper` 使用  
`BeanWrapper` 完成Bean的依赖装配（即属性设置）


#### Aware接口

Spring会检查Bean对象实例是否实现了某些以 `Aware` 结尾的接口，会将接口定义中规定的依赖注入到Bean实例对象中  
可以用于获取与框架底层相关的对象  

#### BeanPostProcessor

`BeanPostProcessor` 会处理容器内所有符合条件的实例化后的对象实例

**注意**  
`BeanPostProcessor` 存在于Bean实例化阶段  
`BeanFactoryPostProcessor` 存在与容器初始化阶段

`BeanPostProcessor` 接口定义  
```java
package org.springframework.beans.factory.config;

import org.springframework.beans.BeansException;
import org.springframework.lang.Nullable;

public interface BeanPostProcessor {
    
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
    
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}
```
在该接口的定义的两个方法中都传入了Bean实例对象，可以对Bean实例执行任意操作  
接口常见使用场景：处理标记接口实现类，或者为当前对象提供代理实现  
`Aware` 接口注入依赖就是通过 `BeanPostProcessor`进行处理的

***`BeanPostProcessor`是容器提供的对象实例化阶段的强有力扩展点***

#### InitializingBean接口

生命周期标识接口  
接口定义
```java
package org.springframework.beans.factory;

public interface InitializingBean {
    
	void afterPropertiesSet() throws Exception;

}
```
在对象实例化过程调用过 `BeanPostProcessor` 的 `postProcessBeforeInitialization`方法后  
会接着检测当前对象是否实现了 `InitializingBean` 接口，如果是则调用 `afterPropertiesSet` 方法进一步调整实例状态  

但为了框架与应用的解耦，Spring在Bean的配置属性中提供了 `init-method` 属性来指定初始化方法  
还可以在顶层 `beans` 标签中设置 `default-init-method` 属性来统一命名指定初始化方法
不受限于 `InitializingBean` 接口方法  
根据耦合接受度来选择初始化方式

---

## ApplicationContext

### 统一资源加载策略

`java.net.URL` 的资源定位以及查找范围狭隘且功能孱弱  
因此Spring推出了基于 `Resource` 和 `ResourceLoader` 接口的资源加载策略

#### Resource

`Resource` 接口应该可以根据资源的不同类型，或者资源所处的不同场合，给出响应的具体实现
* `ByteArrayResource`
* `ClasspathResource`
* `FileSystemResource`
* `UrlResource`
* `InputStreamResource` 较为少用，应该尝试使用 `ByteArrayResource` 代替  
如果上述实现类不能满足需求，也可以实现 `Resource` 接口或者更简易地继承 `AbstractResource` 抽象类来实现自定义资源类

#### ResourceLoader

该接口是资源查找定位策略的统一抽象

##### DefaultResourceLoader

`ResourceLoader` 接口的默认实现类，资源查找逻辑如下：
* 首先检查资源路径是否以`classpath:`前缀打头，如果是则尝试构造 `ClasspathResource` 返回
* 否则尝试通过URL来定位资源，定位失败则构造一个不存在的 `ClasspathResource` 返回  
该实现类的功能是不健全的，应该考虑使用更具体地实现类

##### ResourcePatternResolver

可以根据指定的资源路径匹配模式，返回多个 `Resource` 实例  
最常用的实现是 `PathMatchingResourcePatternResolver` ，基本上集所有技能于一身  
在构造 `PathMatchingResourcePatternResolver` 时可以指定一个 `ResourceLoader`  
如果不指定，则内部会构造一个 `DefaultResourceLoader`  
最终在查找资源时，会委派给持有的 `ResourceLoader` 对象进行查找
 
##### ApplicationContext与ResourceLoader

`ApplicationContext` 继承了 `ResourcePatternResolver` 
所有的 `ApplicationContext` 继承 `AbstractApplicationContext`  
`AbstractApplicationContext` 继承 `DefaultResourceLoader`  
即 `ApplicationContext` 的实现类将自身作为 `ResourceLoader` 传递给自己持有的 `PathMatchingResourcePatternResolver`  
在执行资源定位查找时将任务委派给自身持有的 `PathMatchingResourcePatternResolver`  




















