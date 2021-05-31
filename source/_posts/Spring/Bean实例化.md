---
title: Bean实例化
categories: 
- Spring
---

**原文：** https://juejin.cn/post/6883143694917042183

**「个人公众号：月伴飞鱼，欢迎关注」**

创建Spring Bean实例化是Spring Bean生命周期的第一阶段

Bean的生命周期主要有如下几个步骤：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3232408b65b44717a642765fdafce427~tplv-k3u1fbpfcp-watermark.image)

**「详细介绍：Spring In Action是这样讲的：」**

> ❝
>
> 实例化Bean对象，这个时候Bean的对象是非常低级的，基本不能够被我们使用，因为连最基本的属性都没有设置，可以理解为连Autowired注解都是没有解析的；
>
> 填充属性，当做完这一步，Bean对象基本是完整的了，可以理解为Autowired注解已经解析完毕，依赖注入完成了；
>
> 如果Bean实现了BeanNameAware接口，则调用setBeanName方法；
>
> 如果Bean实现了BeanClassLoaderAware接口，则调用setBeanClassLoader方法；
>
> 如果Bean实现了BeanFactoryAware接口，则调用setBeanFactory方法；
>
> 调用BeanPostProcessor的postProcessBeforeInitialization方法；
>
> 如果Bean实现了InitializingBean接口，调用afterPropertiesSet方法；
>
> 如果Bean定义了init-method方法，则调用Bean的init-method方法；
>
> 调用BeanPostProcessor的postProcessAfterInitialization方法；当进行到这一步，Bean已经被准备就绪了，一直停留在应用的上下文中，直到被销毁；
>
> 如果应用的上下文被销毁了，如果Bean实现了DisposableBean接口，则调用destroy方法，如果Bean定义了destory-method声明了销毁方法也会被调用。
>
> ❞

在实例化Bean之前在BeanDefinition里头已经有了所有需要实例化时用到的元数据，接下来Spring只需要选择合适的实例化方法以及策略即可。

**「BeanDefinition」**

Spring容器启动的时候会定位我们的配置文件，加载文件，并解析成Bean的定义文件BeanDefinition

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99d5e61bae1547679ea4d4c971179a9e~tplv-k3u1fbpfcp-watermark.image)

右边的Map里存储这bean之间的依赖关系的定义BeanDefinition，比如OrderController依赖OrderService这种

实例化方法有两大类分别是工厂方法和构造方法实例化，后者是最常见的。其中Spring默认的实例化方法就是无参构造函数实例化。

如我们在xml里定义的`<bean id="xxx" class="yyy"/>`以及用注解标识的bean都是通过默认实例化方法实例化的

# 实例化方法

**「使静态工厂方法实例化」**

```
public class FactoryInstance {

    public FactoryInstance() {
        System.out.println("instance by FactoryInstance");
    }
}
public class MyBeanFactory {

    public static FactoryInstance getInstanceStatic(){
        return new FactoryInstance();
    }
}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <bean id="factoryInstance" class="spring.service.instance.MyBeanFactory" 
          factory-method="getInstanceStatic"/>
</beans>
```

**「使用实例工厂方法实例化」**

```
public class MyBeanFactory {

    /**
     * 实例工厂创建bean实例
     *
     * @return
     */
    public FactoryInstance getInstance() {
        return new FactoryInstance();
    }
}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
    <!-- 工厂实例 -- >   
    <bean id="myBeanFactory" class="MyBeanFactory"/>
    <bean id="factoryInstance" factory-bean="myBeanFactory" factory-method="getInstance"/>
</beans>
```

**「使用无参构造函数实例化（默认的）」**

```
public class ConstructorInstance {

    public ConstructorInstance() {
        System.out.println("ConstructorInstance none args");
    }

}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
    <bean id="constructorInstance" class="spring.service.instance.ConstructorInstance"/>
</beans>
```

**「使用有参构造函数实例化」**

```
public class ConstructorInstance {

    private String name;
    
    public ConstructorInstance(String name) {
        System.out.println("ConstructorInstance with args");
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
       
   <bean id="constructorInstance" class="spring.service.instance.ConstructorInstance">
        <constructor-arg index="0" name="name" value="test constructor with args"/>
    </bean>
</beans>
```

# 源码阅读

直接来看看doCreateBean方法

具体实现在AbstractAutowireCapableBeanFactory类里面。

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dfb7d18b794b4669a7ee498d4384a00f~tplv-k3u1fbpfcp-watermark.image)

我们这里只需关注第一步创建bean实例的流程即可

```
instanceWrapper = createBeanInstance(beanName, mbd, args);
```

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b60eda97cac84496a7cadc4b88aaeaf2~tplv-k3u1fbpfcp-watermark.image)

上面代码就是spring 实现bean实例创建的核心代码。这一步主要根据BeanDefinition里的元数据定义决定使用哪种实例化方法，主要有下面三种：

- instantiateUsingFactoryMethod 工厂方法实例化的具体实现
- autowireConstructor 有参构造函数实例化的具体实现
- instantiateBean 默认实例化具体实现（无参构造函数）

**「实例化策略（cglib or 反射）」**

> ❝
>
> 工厂方法的实例化手段没有选择策略直接用了反射实现的，所以这个实例化策略都是对于构造函数实例化而言的
>
> ❞

下面选一个instantiateBean的实现来介绍

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e4d95e76ca8e403399ab743271806f10~tplv-k3u1fbpfcp-watermark.image)

上面说到的两构造函数实例化方法不管是哪一种都会选一个实例化策略进行，到底选哪一种策略也是根据BeanDefinition里的定义决定的。

下面这一行代码就是选择实例化策略的代码

```
beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
```

**「选择使用反射还是cglib」**

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4a29d9fdaf748d6a0f057ad7732ba28~tplv-k3u1fbpfcp-watermark.image)

先判断如果`beanDefinition.getMethodOverrides()`为空也就是用户没有使用replace或者lookup的配置方法，那么直接使用反射的方式，简单快捷

但是如果使用了这两个特性，在直接使用反射的方式创建实例就不妥了，因为需要将这两个配置提供的功能切入进去，所以就必须要使用动态代理的方式将包含两个特性所对应的逻辑的拦截增强器设置进去，这样才可以保证在调用方法的时候会被相应的拦截器增强，返回值为包含拦截器的代理实例-----Spring源码深度解析

```
<bean id="constructorInstance" class="spring.service.instance.ConstructorInstance" >
        <lookup-method name="getName" bean="xxx"/>
        <replaced-method name="getName" replacer="yyy"/>
    </bean>
```

如果使用了lookup或者replaced的配置的话会使用cglib，否则直接使用反射。

```
public static final String LOOKUP_METHOD_ELEMENT = "lookup-method";

public static final String REPLACED_METHOD_ELEMENT = "replaced-method";
```

参考：

Spring源码深度解析

Spring In Action

https://url.ms/owy8p