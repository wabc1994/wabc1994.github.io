
---
layout: post
title:  "Spring AOP源码阅读"
date:   2019-7-16 16:50:9
categories: Java
comments: true
tags:
    - Spring   
---

* content
{:toc}

# 前言 

spring AOP 是实现业务代码和其他一些公共行为分割开来，这些公共行为以一种切入的方式注入代码，同时AOP也是后面spring 实现事务的机器。

# Spring AOP
理解spring aop作用机制，主要是在于包括两个步骤
1. 通过ProxyFactoryBean获取一个AopProxy对象，这个代理对象里面包括一个目标对象和拦截器链路
2. 通过代理对象的不同回调接口， 实现回调方法判断是

类之间的总体调用关系

![nArDgA.png](https://s2.ax1x.com/2019/09/03/nArDgA.png)

红色部门是生成代理对象的过程


ProxyCreatorSupport 最终的顶级接口

1. ProxyFactoryBean的getObject方法getObject()
2. initializeAdvisorChain()，chushua 
3. 判断是否是单例模式，其次调用getSingletonInstance（），在getSingletonInstance方法中引入了super中的方法，super是指ProxyCreatorSupport，这里ProxyCreatorSupport是ProxyFactoryBean和ProxyFactory的父类，已经做了很多工作，只需在ProxyFactoryBean的getObject()方法中通过父类的createAopProxy()取得相应的AopProxy
4. 跟踪createAopProxy方法，追踪到了ProxyCreatorSupport中，然后，借助了AopProxyFactory，此时得到的aopProxyFactory，在构造函数中已经定义为DefaultAopProxyFactory
5. 6、进入DefaultAopProxyFactory中，找到createAopProxy方法，在这里判断是调用JDK动态或者CGlib动态中的一种。


还有另一种方式ProxyFactory 创建代理也是也是上述步骤，
1. ProxyFactory.getProxy()
2. 调用ProxyFactory顶级基类ProxyCreatorSupport 的creatorProxy() 获取aopProxy,然后AopProxyFactory  
3. AopProxyFactory()方法调用JDK方式或者CGLIB方法的情况

## ProxyFacotryBean获取获取代理对象

ProxyFactoryBean的getObject获取代理对象

```java
@Override
public Object getObject() throws BeansException {
    //初始化通知器链，为代理对象配置通知器链。
    initializeAdvisorChain();
    //区分SingleTon和ProtoType，生成对应的Proxy
    if (isSingleton()) {
        // 只有SingleTon的Bean才会一开始就初始化，ProtoType的只有在请求的时候才会初始化，代理也一样
        return getSingletonInstance();
    } else {
        if (this.targetName == null) {
            logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
                    "Enable prototype proxies by setting the 'targetName' property.");
        }
        return newPrototypeInstance();
    }
}

```
一般， proxy 对象持有目标对象，
在Aopproxy里面会多一个拦截器链，拦截器里面有通知器
下面是获取拦截器的关键

```java
private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
    if (this.advisorChainInitialized) {
        return;
    }

    if (!ObjectUtils.isEmpty(this.interceptorNames)) {
        if (this.beanFactory == null) {
            throw new IllegalStateException("No BeanFactory available anymore (probably due to serialization) " +
                    "- cannot resolve interceptor names " + Arrays.asList(this.interceptorNames));
        }

        // Globals can't be last unless we specified a targetSource using the property...
        if (this.interceptorNames[this.interceptorNames.length - 1].endsWith(GLOBAL_SUFFIX) &&
                this.targetName == null && this.targetSource == EMPTY_TARGET_SOURCE) {
            throw new AopConfigException("Target required after globals");
        }

        // Materialize interceptor chain from bean names.
        for (String name : this.interceptorNames) {
            if (logger.isTraceEnabled()) {
                logger.trace("Configuring advisor or advice '" + name + "'");
            }

            if (name.endsWith(GLOBAL_SUFFIX)) {
                if (!(this.beanFactory instanceof ListableBeanFactory)) {
                    throw new AopConfigException(
                            "Can only use global advisors or interceptors with a ListableBeanFactory");
                }
                addGlobalAdvisor((ListableBeanFactory) this.beanFactory,
                        name.substring(0, name.length() - GLOBAL_SUFFIX.length()));
            }

            else {
                // If we get here, we need to add a named interceptor.
                // We must check if it's a singleton or prototype.
                Object advice;
                if (this.singleton || this.beanFactory.isSingleton(name)) {
                    // Add the real Advisor/Advice to the chain.
                    advice = this.beanFactory.getBean(name);
                }
                else {
                    // It's a prototype Advice or Advisor: replace with a prototype.
                    // Avoid unnecessary creation of prototype bean just for advisor chain initialization.
                    advice = new PrototypePlaceholderAdvisor(name);
                }
                addAdvisorOnChainCreation(advice, name);
            }
        }
    }

    this.advisorChainInitialized = true;
}
```
拦截器链路初始化是要在xml配置文件里面读取的，比如下面就是配置的一个拦截器, 在拦截器初始化的过程中，获取advisor 通知器

```java
<bean id="testAdvisor" class="com.abc.TestAdvisor" />
<!-- 通知器，实现了目标对象需要增强的切面行为，也就是通知 -->
<!-- 这里我理解的是被代理的bean的scope如果是prototype，那么这个代理Bean就是prototype -->
<bean id="testAop" class="org.springframework.aop.ProxyFactoryBean">
    <property name="proxyInterfaces">
        <value>com.test.AbcInterface</value>
    </property>
    <property name="target">
        <!-- 目标对象 -->
        <bean class="com.abc.TestTarget" />
    </property>
    <property name="interceptorNames">
        <!-- 需要拦截的方法接口，通知器 -->
        <list>
            <value>
                testAdvisor
            </value>
        </list>
    </property>
</bean>

```

上述问题就转换为了DefaultAopProxyFactoryr如何生成AopProxy了，这就涉及到两种不同生成代理对象的方式了

```java
ublic class DefaultAopProxyFactory implements AopProxyFactory, Serializable {
    @Override
    public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
            //获取配置的目标对象
            Class<?> targetClass = config.getTargetClass();
            if (targetClass == null) {
            //如果没有目标对象，抛出异常，提醒AOP应用提供正确的目标配置
                throw new AopConfigException("TargetSource cannot determine target class: " +
                        "Either an interface or a target is required for proxy creation.");
            }
            if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
                return new JdkDynamicAopProxy(config);
            }
            //由于CGLIB是一个第三方类库，所以需要在CLASSPATH中配置
            return new ObjenesisCglibAopProxy(config);
        }
        
        else {
        // 生成代理类的配置文件
            return new JdkDynamicAopProxy(config);
        }
    }
}

```

接下来就是关于如何生成这个AopProxy代理对象的了
分别有两种方式了
1.jdk
2.cglib


>使用AopProxy对象封之后，ProxyFactoryBean的getObject方法得到的对象就不是一个普通的Java对象了，而是一个AopProxy代理对象。

1. ProxyFactoryBean  生成AOPproxy 代理对象， ProxyFactoryBean 本质也是一个FactoryBean, 因此得到AopProxy代理对象也是通过getObject() 方法来获取

Aop  
2. 拦截器- 对代理对象的方法调用之前，先通过拦截器器来执行回调方法，这些回调方法里面就是实现Aop切面方法，然后再调用代理对象代理的目标对象的方法


#  拦截器调用

在上述章节中，既然代理对象已经生成并且初始化拦截链，
那么拦截器是如何被调用的呢？

1. JDK方式生成的proxy代理类被调用时，会调用super.h.method()方法，这里的super一般指的是Proxy，而Proxy中有一个InvocationHandler，即h。所以InvocationHandler的invoke方法会被作为回调函数调用，

2. CGLIB方式，Enhancer.Callback，Callback类似于InvocationHandler，类DynamicAdvisedInterceptor继承了Callback，它的intercept()方法就类似于invoke()，然后匹配通知类型调用通知，最后调用目标方法

对通知和目标方法的增强就是在invoke和interceptz方法中，其中有段代码很重要，
![nE8vbd.png](https://s2.ax1x.com/2019/09/04/nE8vbd.png)

我们看看invoke是如何执行的，保证在目标对象方法调用之前对xml配置

```java

@Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    
    //最后要调用的方法
    
        MethodInvocation invocation;
        Object oldProxy = null;
        boolean setProxyContext = false;
 
        TargetSource targetSource = this.advised.targetSource;
        Class<?> targetClass = null;
        Object target = null;
 
        try {
            if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
                //目标对象未实现equals方法
                return equals(args[0]);
            }
            if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
                //目标对象未实现hashcode方法
                return hashCode();
            }
            if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
                    method.getDeclaringClass().isAssignableFrom(Advised.class)) {
            //这里就是目标对象方法的调用
                return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
            }
 
            Object retVal;
 
            if (this.advised.exposeProxy) {
                // 获得当前的对象
                oldProxy = AopContext.setCurrentProxy(proxy);
                setProxyContext = true;
            }
 
            //目标对象可能源自一个池或者一个简单的方法
          target = targetSource.getTarget();
            if (target != null) {
                targetClass = target.getClass();
            }
            // 得到拦截器链
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
            // 检查是否定义了拦截器方法，如果没有的话直接调用目标方法
            if (chain.isEmpty()) {
 
                retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
            }
            else {
                //我们需要创建一个调用方法
                invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
                // proceed内部实现了递归调用遍历拦截器链
                retVal = invocation.proceed();
            }
            Class<?> returnType = method.getReturnType();
            if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
                    !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
                retVal = proxy;
            }
            else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
                throw new AopInvocationException(
                        "Null return value from advice does not match primitive return type for: " + method);
            }
            return retVal;
        }
        finally {
            if (target != null && !targetSource.isStatic()) {
                // 释放target对象
                targetSource.releaseTarget(target);
            }
            if (setProxyContext) {
                // 替换回原来的proxy
                AopContext.setCurrentProxy(oldProxy);
            }
        }
    }
```

上述方法主要分为三块

1. 拦截器的获取，

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class<?> targetClass) {
        MethodCacheKey cacheKey = new MethodCacheKey(method);
        List<Object> cached = this.methodCache.get(cacheKey);
        if (cached == null) {
            cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                    this, method, targetClass);
            this.methodCache.put(cacheKey, cached);
        }
        return cached;
    }
```

2. 拦截器方法的执行，判断是否配置了拦截器，有多个拦截器的话要逐渐一个个匹配的方式进行，

3. 目标对象方法的执行，使用反射机制来对目标对象的方法进行的

```java
    public static Object invokeJoinpointUsingReflection(Object target, Method method, Object[] args)
            throws Throwable {
        // 通过反射机制来获得相应的方法，并调用invoke
        try {
            ReflectionUtils.makeAccessible(method);
            return method.invoke(target, args);
        }
        catch (InvocationTargetException ex) {
 
            throw ex.getTargetException();
        }
        catch (IllegalArgumentException ex) {
            throw new AopInvocationException("AOP configuration seems to be invalid: tried calling method [" +
                    method + "] on target [" + target + "]", ex);
        }
        catch (IllegalAccessException ex) {
            throw new AopInvocationException("Could not access method [" + method + "]", ex);
        }
    }
```

看看拦截器对象，即ReflectiveMethodInvocation中proceed()方法的调用


```java
invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
retVal = invocation.proceed();
```
invocation是一个new ReflectiveMethodInvocation（）实例，找到ReflectiveMethodInvocation的proceed()方法

```java
private int currentInterceptorIndex = -1;
 
    @Override
    public Object proceed() throws Throwable {
        //  currentInterceptorIndex初始化的长度为-1，下面就就是判断          //interceptorsAndDynamicMethodMatchers长度是否为0
        if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
            return invokeJoinpoint();
        }
 
        Object interceptorOrInterceptionAdvice =
                this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
        if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
            //匹配是否为正确的方法逻辑，（MethodMatcher）可以看出为匹配如果不是就调用下一个
            InterceptorAndDynamicMethodMatcher dm =
                    (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
            if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
                return dm.interceptor.invoke(this);
            }
            else {
                //匹配失败，跳过这个拦截器，继续下一个
                // 递归调用
                return proceed();
            }
        }
        else {
            // 如果是interceptor，则调用invoke方法，这是为了兼容
 
            return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
        }
    }
```

拦截器

![nEQawt.png](https://s2.ax1x.com/2019/09/04/nEQawt.png)


# 参考链接
Spring源码分析
