---
layout: post
title: spring AOP的流程 
toc: true
cover: media/image-20180722003052030.png
tags: ['spring','java']
category: Java
---

# spring AOP的流程

AOP(Aspect Oriented Programming) 面向切面编程,是一种编程概念. 将非逻辑代码,例如安全监测,监控,日志, 从业务代码中抽出,以非侵入的方式与原来的方法进行协同.  这样做的能让业务代码更关注业务逻辑,代码结构更加清晰.

> AOP 有自己的标准, spring aop遵循相关的标准.

**连接点(JointPoint):**  程序执行的某些点称为 连接点, 例如方法的执行, 异常的处理. 在spring 只支持方法执行的连接点. 
**切点 Pointcut:** 切点用于选择连接点,解析切入点表达式然后将解析的结果和方法进行匹配,匹配到的方法 就是**连接点**.
**通知(Advice)** 抽去出来的非逻辑代码写在通知里面.   

**spring 定义的通知类型:**

+ 前置通知  - 在目标方法调用前调用
+ 后置通知  - 在目标方法完成之后执行
+ 返回通知  - 在目标方法执行成功后调用
+ 异常通知  - 目标方法抛出异常后执行通知
+ 环绕通知  - 在环绕通知中调用目标方法, 可以自定义处理

**切面 Asect**  将通知和切入点 .

##  1.解析spring aop的配置

spring aop 通过解析并处理配置, 最后获得 class 为 `org.springframework.aop.aspectj.autoproxy.AspectJAwareAdvisorAutoProxyCreator`  

name为`org.springframework.aop.config.internalAutoProxyCreator` 的 `beanDefinition` . 

> 如果是使用 注解配置 class则为`AspectJAwareAdvisorAutoProxyCreator` 的子类 `AnnotationAwareAspectJAutoProxyCreator` 

**AspectJAwareAdvisorAutoProxyCreator 的继承体系图:**

![image-20180831214543104](../../../assets/img/2018/08/image-20180831214543104.png)

****

###  1.1 解析配置文件

在`AbstractApplicationContext`的`refresh` 方法中进行 refresh 内部的BeanFactory时进行处理 . 

![image-20180831212248662](../../../assets/img/2018/08/image-20180831212248662.png)



调用堆栈:

![image-20180831203106747](../../../assets/img/2018/08/image-20180831203106747.png)

**spring aop 在xml中的配置:**

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation=" http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans-4.1.xsd
http://www.springframework.org/schema/context
http://www.springframework.org/schema/context/spring-context-4.1.xsd                        http://www.springframework.org/schema/aop                          http://www.springframework.org/schema/aop/spring-aop-4.1.xsd">

    <bean id="userService" class="org.test.service.impl.UserServiceImpl" />
    <bean id="aspectTest" class="org.test.main.AspectTest" />
    <aop:aspectj-autoproxy proxy-target-class="true"  />
</beans>
```

使用spring aop需要引入`http://www.springframework.org/schema/aop` 命名空间, 在spring aop的资源文件**spring.handlers** 中:
```properties
http\://www.springframework.org/schema/aop=org.springframework.aop.config.AopNamespaceHandler
```
可以看出 spring aop是由`AopNamespaceHandler` 来解析的.   

**AopNamespaceHandler类:**

``` java
public class AopNamespaceHandler extends NamespaceHandlerSupport {
	/**
	 * Register the {@link BeanDefinitionParser BeanDefinitionParsers} for the
	 * '{@code config}', '{@code spring-configured}', '{@code aspectj-autoproxy}'
	 * and '{@code scoped-proxy}' tags.
	 */
	@Override
	public void init() {
		registerBeanDefinitionParser("config", new ConfigBeanDefinitionParser());
		registerBeanDefinitionParser("aspectj-autoproxy", new  AspectJAutoProxyBeanDefinitionParser());
		registerBeanDefinitionDecorator("scoped-proxy", new ScopedProxyBeanDefinitionDecorator());
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
	}
}
```

在AopNamespaceHandler中根据不同的配置选取不同的 Parser.

+ 处理`aspectj-autoproxy`的Parser是`AspectJAutoProxyBeanDefinitionParser` 
+ 处理`aop-config`的Parser是   `ConfigBeanDefinitionParser`

### 1.2构造 AspectJAwareAdvisorAutoProxyCreator 对象

使用了`aspectj-autoproxy`的配置 都会使用`AspectJAutoProxyBeanDefinitionParser`来解析

![image-20180831130608788](../../../assets/img/2018/08/image-20180831130608788.png)

所有的**BeanDefinitonParser**都是通过方法`parse`来解析. 

```java
@Override
public BeanDefinition parse(Element element, ParserContext parserContext) {
   //注册 AspectJAnnotationAutoProxyCreator
 AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
    
   // 对于注释中子类进行处理 
   extendBeanDefinition(element, parserContext);
   return null;
}
```

**registerAspectJAnnotationAutoProxyCreatorIfNecessary方法:**

```java
public static void registerAspectJAnnotationAutoProxyCreatorIfNecessary(
      ParserContext parserContext, Element sourceElement) {
  
   // 注册 beanName 为 org.Springframework.aop.config.internalAutoProxyCreator的 BeanDefinition
   BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(
         parserContext.getRegistry(), parserContext.extractSource(sourceElement));

   //对 proxy-target-class 和 expose-proxy 属性的处理
   useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);

   if (logger.isDebugEnabled()){
      logger.debug("对 proxy-target-class 和 expose-proxy 属性的处理");
   }

   //注册组件通知,
   registerComponentIfNecessary(beanDefinition, parserContext);
}
```

### 1.3 创建对象

**AopNamespaceUtils的registerAspectJAnnotationAutoProxyCreatorIfNecessary 方法**

```java
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
   return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}
```

对于注解配置的AOP是通过 `AnnotationAwareAspectJAutoProxyCreator`完成, 根据注解定义的切点自动代理匹配的bean. 


**registerOrEscalateApcAsRequired 方法:**

```java
private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, Object source) {
   Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
   //如果容器中已经存在 自动代理创建类
   if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
      BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
      // AUTO_PROXY_CREATOR_BEAN_NAME="org.springframework.aop.config.internalAutoProxyCreator"
      //如果存在的自动创建器 和现在的不是一个, 根据优先级 选择
      if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
         int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
         int requiredPriority = findPriorityForClass(cls);
         if (currentPriority < requiredPriority) {
            apcDefinition.setBeanClassName(cls.getName());
         }
      }
      //如果是一样的对象 则不需要创建
      return null;
   }

   //创建  beanDefinition
   RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
   beanDefinition.setSource(source);
   beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
   beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
   registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition); // org.springframework.aop.config.internalAutoProxyCreator
   return beanDefinition;
}
```

### 1.4 属性处理

Spring的AOP的实现方式:

- JDK动态代理.  当被代理对象,至少实现了一个接口时,会使用JDK动态代理, 
- CGLIB 动态代理 被代理对象没用实现接口,使用 CGLIB代理.

AOP中配置的  prioxy-target-class 和 expose-proxy 属性 对代理生成有很重要的作用.  

**AopNamespaceUtils.java 的 useClassProxyingIfNecessary 方法:**

```java
private static void useClassProxyingIfNecessary(BeanDefinitionRegistry registry, Element sourceElement) {
   if (sourceElement != null) {
      boolean proxyTargetClass = Boolean.parseBoolean(sourceElement.getAttribute(PROXY_TARGET_CLASS_ATTRIBUTE));
      if (proxyTargetClass) {
         AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
      }
      boolean exposeProxy = Boolean.parseBoolean(sourceElement.getAttribute(EXPOSE_PROXY_ATTRIBUTE));
      if (exposeProxy) {
         AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
      }
   }
}
```

+ prioxy-target-class   负责选择两种代理的实现方式
+ expose-proxy           负责自我调用切面中的增强

> 如果  prioxy-target-class 属性设置为true时 即使被代理对象实现了接口 还是会使用CGLIB代理.



## 2.获取advisor

### 2.1 注册Bean的BeanPostProcessor

在**AbstractApplicationContext**的 **refresh** 方法执行到 `registerBeanPostProcessors` 方法时,会注册所有的 实现了 `BeanPostProcessor` 接口的对象,并添加到BeanFactory的`beanPostProcessors`属性中.    

**AbstractApplicationContext**的 **refresh** 方法:

![image-20180831220459265](../../../assets/img/2018/08/image-20180831220459265.png)

**registerBeanPostProcessors 方法:**

```java
/**
 * Instantiate and invoke all registered BeanPostProcessor beans,
 * respecting explicit order if given.
 * <p>Must be called before any instantiation of application beans.
 */
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
   PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

**PostProcessorRegistrationDelegate.registerBeanPostProcessors :**

![image-20180831222533933](../../../assets/img/2018/08/image-20180831222533933.png)

调用了`registerBeanPostProcessors` 方法进行注册:

```java
	/**
	 * Register the given BeanPostProcessor beans.
	 */
	private static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {

		for (BeanPostProcessor postProcessor : postProcessors) {
			//添加到 beanFactory的BeanPostProcessor中
			beanFactory.addBeanPostProcessor(postProcessor);
		}
	}
```

BeanFactory 添加BeanPostProcessor 过程: 

```java
@Override
public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
   Assert.notNull(beanPostProcessor, "BeanPostProcessor must not be null");
   if (logger.isDebugEnabled()){
      logger.debug("添加 BeanPostProcessor : " + beanPostProcessor);
   }
   this.beanPostProcessors.remove(beanPostProcessor);
   this.beanPostProcessors.add(beanPostProcessor);
   if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
      this.hasInstantiationAwareBeanPostProcessors = true;
   }
   if (beanPostProcessor instanceof DestructionAwareBeanPostProcessor) {
      this.hasDestructionAwareBeanPostProcessors = true;
   }
}
```

 最终 `AnnotationAwareAspectJAutoProxyCreator` 对添加到了 BeanFactory的 `beanPostProcessors` List中.  

### 2.2 在创建对象之前进行PostProcess

 在AbstractApplicationContext的refresh 方法执行到`finishBeanFactoryInitialization`  方法时会实例化所有的非懒加载对象.

![image-20180901141736805](../../../assets/img/2018/08/image-20180901141736805.png)

 在实例化对象的时候 会先调用 BeanFactory的`applyBeanPostProcessorsBeforeInstantiation` 方法.

```java
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
   for (BeanPostProcessor bp : getBeanPostProcessors()) {
      if (bp instanceof InstantiationAwareBeanPostProcessor) {
         InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
         Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
         if (result != null) {
            return result;
         }
      }
   }
   return null;
}
```

 由于AnnotationAwareAspectJAutoProxyCreator 实现了 `InstantiationAwareBeanPostProcessor` 接口,  所以会执行  `postProcessBeforeInstantiation` . 

AnnotationAwareAspectJAutoProxyCreator 的 `postProcessBeforeInstantiation` 在父类 `AbstractAutoProxyCreator` 中

```java
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
   Object cacheKey = getCacheKey(beanClass, beanName);

   if (beanName == null || !this.targetSourcedBeans.contains(beanName)) {
      if (this.advisedBeans.containsKey(cacheKey)) {
         return null;
      }
      if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
         this.advisedBeans.put(cacheKey, Boolean.FALSE);
         return null;
      }
   }

   // Create proxy here if we have a custom TargetSource.
   // Suppresses unnecessary default instantiation of the target bean:
   // The TargetSource will handle target instances in a custom fashion.
   if (beanName != null) {
      TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
      if (targetSource != null) {
         this.targetSourcedBeans.add(beanName);
         Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
         Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
         this.proxyTypes.put(cacheKey, proxy.getClass());
         return proxy;
      }
   }

   return null;
}
```

在调用到 `shouldSkip` 方法时 会进行 获取指定的Advisor.  

###  2.3 构建并缓存Advisor对象.

`shouldSkip`方法在父类AspectJAwareAdvisorAutoProxyCreator中:

```java
@Override
protected boolean shouldSkip(Class<?> beanClass, String beanName) {
   // TODO: Consider optimization by caching the list of the aspect names
   List<Advisor> candidateAdvisors = findCandidateAdvisors();
   for (Advisor advisor : candidateAdvisors) {
      if (advisor instanceof AspectJPointcutAdvisor) {
         if (((AbstractAspectJAdvice) advisor.getAdvice()).getAspectName().equals(beanName)) {
            return true;
         }
      }
   }
   return super.shouldSkip(beanClass, beanName);
}
```

从`shouldSkip`中通过`findCandidateAdvisors` 方法的调用获取 `Advisor`对象的列表 . 

由于当前对象是AnnotationAwareAspectJAutoProxyCreator的实例,所以调用的是这个类的``findCandidateAdvisors` 方法: 

```java
@Override
protected List<Advisor> findCandidateAdvisors() {
   // Add all the Spring advisors found according to superclass rules.
   //调用 父类的 findCandidateAdvisors 方法加载使用xml配置文件的AOP声明
   if (logger.isDebugEnabled()){
      logger.debug("加载xml文件中的 AOP 声明");
   }
   List<Advisor> advisors = super.findCandidateAdvisors();
   // Build Advisors for all AspectJ aspects in the bean factory.
   // 加载使用注解的 AOP
   if (logger.isDebugEnabled()){
      logger.debug("加载注解的 AOP 声明");
   }
   advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
   return advisors;
}
```

在这个方法中,先调用了父类(`AspectJAwareAdvisorAutoProxyCreator`)的findCandidateAdvisors 实现了对XML文件中配置的 advisor解析.  对注解配置的 **Advisor**解析是通过`aspectJAdvisorsBuilder.buildAspectJAdvisors()` 来实现. 

**BeanFactoryAspectJAdvisorsBuilder.java 的 buildAspectJAdvisors 方法:**

```java
public List<Advisor> buildAspectJAdvisors() {
    List<String> aspectNames = this.aspectBeanNames;

    if (aspectNames == null) {
        synchronized (this) {
            aspectNames = this.aspectBeanNames;
            if (aspectNames == null) {
                List<Advisor> advisors = new LinkedList<Advisor>();
                aspectNames = new LinkedList<String>();
                //获取所有在BeanFactory中的beanName
                String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                        this.beanFactory, Object.class, true, false);
                //循环所有的 beanName
                for (String beanName : beanNames) {
                    if (!isEligibleBean(beanName)) {
                        continue;
                    }
                    // We must be careful not to instantiate beans eagerly as in this case they
                    // would be cached by the Spring container but would not have been weaved.
                    Class<?> beanType = this.beanFactory.getType(beanName);
                    if (beanType == null) {
                        continue;
                    }
                    //判断是不是 Aspect 注解
                    if (this.advisorFactory.isAspect(beanType)) {
                        if (logger.isDebugEnabled()) {
                            logger.debug("开始处理 : [" + beanName + "] 的Aspect 注解");
                        }
                        aspectNames.add(beanName);
                        AspectMetadata amd = new AspectMetadata(beanType, beanName);
                        if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                            MetadataAwareAspectInstanceFactory factory =
                                    new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                            // 解析标记了 AspectJ 注解的方法
                            List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                            if (this.beanFactory.isSingleton(beanName)) {
                                if (logger.isDebugEnabled()) {
                                    logger.debug("缓存 [" + beanName + "] 的 Advisor :" + classAdvisors);
                                }
                                this.advisorsCache.put(beanName, classAdvisors);
                            } else {
                                this.aspectFactoryCache.put(beanName, factory);
                            }
                            advisors.addAll(classAdvisors);
                        } else {
                            // Per target or per this.
                            if (this.beanFactory.isSingleton(beanName)) {
                                throw new IllegalArgumentException("Bean with name '" + beanName +
                                        "' is a singleton, but aspect instantiation model is not singleton");
                            }
                            MetadataAwareAspectInstanceFactory factory =
                                    new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
                            this.aspectFactoryCache.put(beanName, factory);
                            advisors.addAll(this.advisorFactory.getAdvisors(factory));
                        }
                    }
                }
                this.aspectBeanNames = aspectNames;
                return advisors;
            }
        }
    }
```

在这个方法中, 会遍历BeanFactory中所有的 Bean对象, 然后判断 Bean的Class对象是否被Aspect注解. 被Aspect注解了的类会交给 AspectJAdvisorFactory 的 getAdvisors 方法处理获得 ` Advisor`列表,然后将,列表缓存起来. 

getAdvisors 方法在 AspectJAdvisorFactory 的子类 `ReflectiveAspectJAdvisorFactory`中实现: 

```java
@Override
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
    Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
    String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
    validate(aspectClass);

    // We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
    // so that it will only instantiate once.
    MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
            new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

    List<Advisor> advisors = new LinkedList<Advisor>();
    //获取所有的注解Advisor的方法
    for (Method method : getAdvisorMethods(aspectClass)) {
        //获取 advisor
        Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }

    // If it's a per target aspect, emit the dummy instantiating aspect.
    if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
        Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
        advisors.add(0, instantiationAdvisor);
    }

    // Find introduction fields.
    for (Field field : aspectClass.getDeclaredFields()) {
        Advisor advisor = getDeclareParentsAdvisor(field);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }

    return advisors;
}
```

 在 getAdvisors 方法中 会遍历类(被Aspect注解)的所有的方法,通过 `getAdvisor`处理 获取Advisor 对象. 

**getAdvisor方法:**

```java
@Override
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory aspectInstanceFactory,
                          int declarationOrderInAspect, String aspectName) {

    validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
    //匹配pointcut
    AspectJExpressionPointcut expressionPointcut = getPointcut(
            candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
    if (expressionPointcut == null) {
        return null;
    }

    if (logger.isDebugEnabled()) {
        logger.debug("处理[" + aspectName + "] 的 expressionPointcut: " + expressionPointcut + " 方法: " + candidateAdviceMethod.getName());
    }

    return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
            this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}
```

  在 getAdvisor 中 会调用`getPointcut` 方法 获取 `Pointcut` , 在获取 `Pointcut`时会判断 方法是否被 `Before.class, Around.class, After.class, AfterReturning.class, AfterThrowing.class` 注解,   注解了的方法,会将 注解中获取的 `PointcutExpression` 封装到 `AspectJExpressionPointcut` 中返回,否则返回`null`.   如果返回的Pointcut 不为空, 则将其和 当前的方法 封装到 `InstantiationModelAwarePointcutAdvisorImpl` 返回.  

获取到的 Advisor 是 `InstantiationModelAwarePointcutAdvisorImpl` 的实例. 

创建`InstantiationModelAwarePointcutAdvisorImpl` 对象时 会调用instantiateAdvice方法 将创建Advice 对象 赋值给 `instantiatedAdvice`  属性. 

`InstantiationModelAwarePointcutAdvisorImpl` 的`instantiateAdvice`方法: 

```java
private Advice instantiateAdvice(AspectJExpressionPointcut pcut) {
   return this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pcut,
         this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
}
```

这个方法通过调用`aspectJAdvisorFactory`的 `getAdvice` 方法获取Advice:

```java
@Override
public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
                        MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {

    Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
    validate(candidateAspectClass);

    AspectJAnnotation<?> aspectJAnnotation =
            AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
    if (aspectJAnnotation == null) {
        return null;
    }

    // If we get here, we know we have an AspectJ method.
    // Check that it's an AspectJ-annotated class
    //检查是不是 Aspect注解
    if (!isAspect(candidateAspectClass)) {
        throw new AopConfigException("Advice must be declared inside an aspect type: " +
                "Offending method '" + candidateAdviceMethod + "' in class [" +
                candidateAspectClass.getName() + "]");
    }

    if (logger.isDebugEnabled()) {
        logger.debug("Found AspectJ method: " + candidateAdviceMethod);
    }

    AbstractAspectJAdvice springAdvice;

    //根据不同的注解,创建不同的 Advice
    switch (aspectJAnnotation.getAnnotationType()) {
        case AtBefore:
            springAdvice = new AspectJMethodBeforeAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;
        case AtAfter:
            springAdvice = new AspectJAfterAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;
        case AtAfterReturning:
            springAdvice = new AspectJAfterReturningAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
            if (StringUtils.hasText(afterReturningAnnotation.returning())) {
                springAdvice.setReturningName(afterReturningAnnotation.returning());
            }
            break;
        case AtAfterThrowing:
            springAdvice = new AspectJAfterThrowingAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
            if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
                springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
            }
            break;
        case AtAround:
            springAdvice = new AspectJAroundAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;
        case AtPointcut:
            if (logger.isDebugEnabled()) {
                logger.debug("Processing pointcut '" + candidateAdviceMethod.getName() + "'");
            }
            return null;
        default:
            throw new UnsupportedOperationException(
                    "Unsupported advice type on method: " + candidateAdviceMethod);
    }

    // Now to configure the advice...
    springAdvice.setAspectName(aspectName);
    springAdvice.setDeclarationOrder(declarationOrder);
    String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
    if (argNames != null) {
        springAdvice.setArgumentNamesFromStringArray(argNames);
    }
    springAdvice.calculateArgumentBindings();
    return springAdvice;
}
```

getAdvice方法,会根据不同的 注解类型,创建不同的Advice. 然后将其封装到Advisor对象中去.



## 3.创建代理对象

###  3.1 创建代理对象的入口

在创建完对象后会执行`AbstractAutowireCapableBeanFactory`的 `applyBeanPostProcessorsAfterInitialization`方法对bean对象进行处理.  调用栈: 

![image-20180903155407567](../../../assets/img/2018/08/image-20180903155407567.png)

 在这个方法中 会获取BeanFactory中所有的BeanPostProcessor 执行其postProcessAfterInitialization 方法 对bean 进行处理. 

AspectJAwareAdvisorAutoProxyCreator的postProcessAfterInitialization 方法在 AbstractAutoProxyCreator中:

```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (!this.earlyProxyReferences.contains(cacheKey)) {
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```

在postProcessAfterInitialization  调用 `wrapIfNecessary` 方法:

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
   if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }

   // Create proxy if we have advice.
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}
```

在 `wrapIfNecessary` 中通过判断调用 `getAdvicesAndAdvisorsForBean` 方法的返回值判断是否创建 代理对象. 

### 3.2 创建代理对象准备步骤 

创建代理对象是通过 `AbstractAutoProxyCreator` 的 `createProxy` 方法来实现

```java
protected Object createProxy(
			Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}
		//创建代理工厂对象
		ProxyFactory proxyFactory = new ProxyFactory();
		//将对象的属性赋值到代理工厂中
		proxyFactory.copyFrom(this);

		//处理代理类的类型
		if (!proxyFactory.isProxyTargetClass()) {
			//检查被代理对象是否自定义了  <aop:scoped-proxy proxy-target-class="false"/> 并将 proxy-target-class 设置为true
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				// 判断被代理的类是否实现了接口, 如果实现了接口将接口添加到ProxyFactory中, 否则设置 proxyTargetClass 属性为 true
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		//将Advisor添加到ProxyFactory中去
		proxyFactory.addAdvisors(advisors);
		//proxyFactiory 设置被代理类
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		//代理过程被配置后是否还允许修改代理配置, 默认为false不允许修改
		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(getProxyClassLoader());
	}protected Object createProxy(
      Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

   if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
      AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
   }
   //创建代理工厂对象
   ProxyFactory proxyFactory = new ProxyFactory();
   //将对象的属性赋值到代理工厂中
   proxyFactory.copyFrom(this);

   //处理代理类的类型
   if (!proxyFactory.isProxyTargetClass()) {
      //检查被代理对象是否自定义了  <aop:scoped-proxy proxy-target-class="false"/> 并将 proxy-target-class 设置为true
      if (shouldProxyTargetClass(beanClass, beanName)) {
         proxyFactory.setProxyTargetClass(true);
      }
      else {
         // 判断被代理的类是否实现了接口, 如果实现了接口将接口添加到ProxyFactory中, 否则设置 proxyTargetClass 属性为 true
         evaluateProxyInterfaces(beanClass, proxyFactory);
      }
   }

   Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
   //将Advisor添加到ProxyFactory中去
   proxyFactory.addAdvisors(advisors);
   //proxyFactiory 设置被代理类
   proxyFactory.setTargetSource(targetSource);
   customizeProxyFactory(proxyFactory);

   //代理过程被配置后是否还允许修改代理配置, 默认为false不允许修改
   proxyFactory.setFrozen(this.freezeProxy);
   if (advisorsPreFiltered()) {
      proxyFactory.setPreFiltered(true);
   }

   return proxyFactory.getProxy(getProxyClassLoader());
}
```

`createProxy` 方法中创建Proxy的步骤:

1. 创建ProxyFactory对象
2. 将 当前`AspectJAwareAdvisorAutoProxyCreator` 对象的属性赋值给`ProxyFactory`对象
3. 处理 proxyTargetClass 属性 
   + 检查被代理对象是否 设置了标签`<aop:scoped-proxy proxy-target-class="true"/>` 
   + 判断被代理的类是否实现了接口, 如果实现了接口将接口添加到ProxyFactory中, 否则设置 proxyTargetClass 属性为 true
4. 将 `Advisor` 添加到 `ProxyFactory`中
5. 设置被代理对象 
6. 自定义ProxyFactory
7. 设置 代理过程被配置后是否还允许修改代理配置, 默认为false不允许修改
8. 调用 ProxyFactory的getProxy 方法 返回被代理的对象

### 3.3 通过ProxyFactory的getProxy 方法 创建代理对象

####  3.3.1确定代理方式

ProxyFactory的getProxy方法:

```java
public Object getProxy(ClassLoader classLoader) {
   return createAopProxy().getProxy(classLoader);
}
```

在这个方法中先创建一个AopProxy对象,然后调用其getProxy方法创建代理对象.

```java
protected final synchronized AopProxy createAopProxy() {
   if (!this.active) {
      activate();
   }
   return getAopProxyFactory().createAopProxy(this);
}
```

AopProxy的createAopProxy方法由其实现类 `DefaultAopProxyFactory` 实现

```java
@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
   //判断使用 jdk代理还是 CGLIB代理
   if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
      Class<?> targetClass = config.getTargetClass();
      if (targetClass == null) {
         throw new AopConfigException("TargetSource cannot determine target class: " +
               "Either an interface or a target is required for proxy creation.");
      }

      if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
         return new JdkDynamicAopProxy(config);
      }
      return new ObjenesisCglibAopProxy(config);
   }
   else {
      return new JdkDynamicAopProxy(config);
   }
}
```

在`createAopProxy` 方法中AopProxyFatory产生不同代理的条件:

1. isOptimize 是否应该执行主动优化 
2. isProxyTargetClass  是否直接代理目标类以及任何接口 
3. hasNoUserSuppliedProxyInterfaces 没有实现代理接口 

如果目标对象实现了接口, 默认情况下会采用JDK动态代理,也可以强制使用CGLIB. 

如果目标对象没有实现接口则必须使用CGLIB . 

#### 3.3.2 jdk动态代理

JDK代理是通过 AopProxy的实现类JdkDynamicAopProxy 来实现

**JdkDynamicAopProxy 的构造方法:**

```java
public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
   Assert.notNull(config, "AdvisedSupport must not be null");
   if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
      throw new AopConfigException("No advisors and no TargetSource specified");
   }
   this.advised = config;
}
```

在构造方法中将 之前创建的 ProxyFactory 复制给对象的 `advised`属性.

在`ProxyFactory`的`getProxy`方法方法中调用了AopPorxy的getProxy方法产生代理对象. 

**JdkDynamicAopProxy 的getProxy方法:**

```java
@Override
public Object getProxy(ClassLoader classLoader) {
   if (logger.isDebugEnabled()) {
      logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
   }
   Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
   findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
   return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

在 getProxy方法中使用了Jdk的代理API创建了代理对象.  在创建代理对象时传入的InvocationHandler 对象是当前对象, 所以在调用方法的时候会调用 当前对象的 `invoke`方法来实现 .

**invoke 方法:**

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   MethodInvocation invocation;
   Object oldProxy = null;
   boolean setProxyContext = false;

   TargetSource targetSource = this.advised.targetSource;
   Class<?> targetClass = null;
   Object target = null;
   try {
      //处理 equal 方法
      if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
         // The target does not implement the equals(Object) method itself.
         return equals(args[0]);
      }
      //hashcode 方法
      else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
         // The target does not implement the hashCode() method itself.
         return hashCode();
      }
      else if (method.getDeclaringClass() == DecoratingProxy.class) {
         // There is only getDecoratedClass() declared -> dispatch to proxy config.
         return AopProxyUtils.ultimateTargetClass(this.advised);
      }
      else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
            method.getDeclaringClass().isAssignableFrom(Advised.class)) {
         // Service invocations on ProxyConfig with the proxy config...
         return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
      }
      Object retVal;

      //目标对象内部调用自己的方法是无法 执行配置的advice方法是, 需要将属性暴露代理
      if (this.advised.exposeProxy) {
         // Make invocation available if necessary.
         oldProxy = AopContext.setCurrentProxy(proxy);
         setProxyContext = true;
      }

      // May be null. Get as late as possible to minimize the time we "own" the target,
      // in case it comes from a pool.
      target = targetSource.getTarget();
      if (target != null) {
         targetClass = target.getClass();
      }

      // Get the interception chain for this method.
      //获得当前方法的拦截链
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

      // Check whether we have any advice. If we don't, we can fallback on direct
      // reflective invocation of the target, and avoid creating a MethodInvocation.
      if (chain.isEmpty()) {
         // We can skip creating a MethodInvocation: just invoke the target directly
         // Note that the final invoker must be an InvokerInterceptor so we know it does
         // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
         //如果执行方法没有 advice 直接调用 切入点方法
         Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
         retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
      }
      else {
         // We need to create a method invocation...
         // 创建一个方法执行的 封装对象
         invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
         // Proceed to the joinpoint through the interceptor chain.
         // 使用 执行调用链
         retVal = invocation.proceed();
      }

      // Massage return value if necessary.
      //返回结果的处理
      Class<?> returnType = method.getReturnType();
      if (retVal != null && retVal == target &&
            returnType != Object.class && returnType.isInstance(proxy) &&
            !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
         // Special case: it returned "this" and the return type of the method
         // is type-compatible. Note that we can't help if the target sets
         // a reference to itself in another returned object.
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
         // Must have come from TargetSource.
         targetSource.releaseTarget(target);
      }
      if (setProxyContext) {
         // Restore old proxy.
         AopContext.setCurrentProxy(oldProxy);
      }
   }
}
```

在invoke方法中, 将通知方法 封装到 `ReflectiveMethodInvocation`中通过执行其 `proceed`  进行调用执行,在这个方法中会形成一个调用链. 

```java
@Override
public Object proceed() throws Throwable {
    // We start with an index of -1 and increment early.
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        //如果拦截器已经执行完了, 执行切入点方法(目标方法)
        return invokeJoinpoint();
    }
    //获取拦截器或者Advice
    Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);

    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        // Evaluate dynamic method matcher here: static part will already have
        // been evaluated and found to match.

        InterceptorAndDynamicMethodMatcher dm =
                (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
            return dm.interceptor.invoke(this);
        } else {
            // Dynamic matching failed.
            // Skip this interceptor and invoke the next in the chain.
            return proceed();
        }
    } else {
        // It's an interceptor, so we just invoke it: The pointcut will have
        // been evaluated statically before this object was constructed.
        //执行拦截器或者Adivce方法
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}
```

  `ReflectiveMethodInvocation` 和Advice调用链关系:

![image-20180905133816076](../../../assets/img/2018/08/image-20180905133816076.png)

####  3.3.3 CGLIB代理

