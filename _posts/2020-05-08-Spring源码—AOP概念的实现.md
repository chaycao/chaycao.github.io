---
title:      Spring源码—AOP概念的实现
date:       2020-05-08
author:     ChayCao
catalog: true
tags:  Spring
---

Spring 作为 Java 中最流行的框架，主要归功于其提供的 IOC 和 AOP 功能。本文将讨论 Spring AOP 的实现。第一节将介绍 AOP 的相关概念，若熟悉可跳过，第二节中结合源码介绍 Spring 是如何实现 AOP 的各概念。

## 1. AOP 概念

### 1.1 JoinPoint

进行织入操作的程序执行点。

常见类型：

- **方法调用（Method Call）**：某个方法被调用的时点。

- **方法调用执行（Method Call Execution）**：某个方法内部开始执行的时点。

  > 方法调用是在**调用对象**上的执行点，方法调用执行是在**被调用对象**的方法开始执行点。

- **构造方法调用（Constructor Call）**：对某个对象调用其构造方法的时点。

- **构造方法执行（Constructor Call Execution）**：某个对象构造方法内部开始执行的时点。

- **字段设置（Field Set）**：某个字段通过 setter 方法被设置或直接被设置的时点。

- **字段获取（Field Get）**：某个字段通过 getter 方法被访问或直接被访问的时点。

- **异常处理执行（Exception Handler Execution）**：某些类型异常抛出后，异常处理逻辑执行的时点。

- **类初始化（Class Initialization）**：类中某些静态类型或静态块的初始化时点。

### 1.2 Pointcut

Jointpoint 的表述方式。

常见表述方式：

- **直接指定 Joinpoint 所在方法名称**
- **正则表达式**
- **特定的 Pointcut 表述语言**

### 1.3 Advice

单一横切关注点逻辑的载体，织入到 Joinpoint 的横切逻辑。

具体形式：

- **Before Advice**：Joinpoint 处之前执行。
- **After Advice**：Joinpoint 处之后执行，细分为三种：
  - **After Returning Advice**：Joinpoint 处正常完成后执行。
  - **After Throwing Advice**：Joinpoint 处抛出异常后执行。
  - **After Finally Advice**：Joinpoint 处正常完成或抛出异常后执行。
- **Around Advice**：包裹 Joinpoint，在 Joinpoint 之前和之后执行，具有 Before Advice 和 After Advice 的功能。
- **Introduction**：为原有的对象添加新的属性或行为。

### 1.4 Aspect

对横切关注点逻辑进行模块化封装的 AOP 概念实体，包含多个 Pointcut 和相关 Advice 的定义。

### 1.5 织入和织入器

织入：将 Aspect 模块化的横切关注点集成到 OOP 系统中。

织入器：用于完成织入操作。

### 1.6 Target

在织入过程中被织入横切逻辑的对象。

将上述 6 个概念放在一块，如下图所示：

![AOP各个概念所处的场景](https://chaycao-1302020836.cos.ap-shenzhen-fsi.myqcloud.com/chaycao%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/2020/2020-05-08-Spring%E6%BA%90%E7%A0%81%E2%80%94AOP%E6%A6%82%E5%BF%B5%E7%9A%84%E5%AE%9E%E7%8E%B0/AOP%E5%90%84%E4%B8%AA%E6%A6%82%E5%BF%B5%E6%89%80%E5%A4%84%E7%9A%84%E5%9C%BA%E6%99%AF.png)

在了解 AOP 的各种概念后，下面将介绍 Spring 中 AOP 概念的具体实现。

## 2. Spring 中的实现

前文提到 AOP 的 Joinpoint 有多种类型，方法调用、方法执行、字段设置、字段获取等。而在 Spring AOP 中，仅支持**方法执行**类型的 Joinpoint，但这样已经能满足 80% 的开发需要，如果有特殊需求，可求助其他 AOP 产品，如 AspectJ。由于 Joinpoint 涉及运行时的过程，相当于组装好所有部件让 AOP 跑起来的最后一步。所以将介绍完其他概念实现后，最后介绍 Joinpoint 的实现。

### 2.1 Pointcut

由于 Spring AOP 仅支持方法执行类别的 Joinpoint，因此 Pointcut 需要定义被织入的方法，又因为 Java 中方法封装在类中，所以 Pointcut 需要**定义被织入的类和方法**，下面看其实现。

Spring 用 ```org.springframework.aop.Pointcut``` 接口定义 Pointcut 的顶层抽象。

```java
public interface Pointcut {
    
    // ClassFilter用于匹配被织入的类   
    ClassFilter getClassFilter();
    
    // MethodMatcher用于匹配被织入的方法
    MethodMatcher getMethodMatcher();
    
    // TruePoincut的单例对象，默认匹配所有类和方法
    Pointcut TRUE = TruePointcut.INSTANCE;
}
```

我们可以看出，`Pointcut` 通过 **`ClassFilter`** 和 **`MethodMatcher`** 的组合来定义相应的 Joinpoint。`Pointcut` 将类和方法拆开来定义，是为了能够**重用**。例如有两个 Joinpoint，分别是 A 类的 `fun()` 方法和 B 类的 `fun()` 方法，两个方法签名相同，则只需一个 `fun()` 方法的 `MethodMatcher` 对象，达到了重用的目的，`ClassFilter` 同理。

下面了解下 `ClassFilter` 和 `MethodMatcher`**如何进行匹配**。

`ClassFilter` 使用**`matches`方法**匹配被织入的类，定义如下：

```java
public interface ClassFilter {
    
   // 匹配被织入的类，匹配成功返回true，失败返回false
   boolean matches(Class<?> clazz);

   // TrueClassFilter的单例对象，默认匹配所有类
   ClassFilter TRUE = TrueClassFilter.INSTANCE;
}
```

`MethodMatcher` 也是使用 **`matches`方法** 匹配被织入的方法，定义如下：

```java
public interface MethodMatcher {
    
   // 匹配被织入的方法，匹配成功返回true，失败返回false
   // 不考虑具体方法参数
   boolean matches(Method method, Class<?> targetClass);
    
   // 匹配被织入的方法，匹配成功返回true，失败返回false
   // 考虑具体方法参数，对参数进行匹配检查
   boolean matches(Method method, Class<?> targetClass, Object... args);
   
   // 一个标志方法
   // false表示不考虑参数，使用第一个matches方法匹配
   // true表示考虑参数，使用第二个matches方法匹配
   boolean isRuntime();

   // TrueMethodMatcher的单例对象，默认匹配所有方法
   MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;

}
```

看到 `matches` 方法的声明，你是否会觉得有点奇怪，在 `ClassFilter` 中不是已经对类进行匹配了吗，那为什么在 `MethodMatcher` 的 `matches` 方法中还有一个 **`Class<?> targetClass`** 参数。请注意，这里的 ```Class<?>``` 类型参数将**不会进行匹配**，而仅是**为了找到具体的方法**。例如：

```java
public boolean matches(Method method, Class<?> targetClass) {
    
    Method targetMethod = AopUtils.getMostSpecificMethod(method, targetClass);
    ...
}
```

在`MethodMatcher`相比`ClassFilter`特殊在有**两个 `matches` 方法**。将**根据 ```isRuntime()``` 的返回结果决定**调用哪个。而`MethodMatcher`因`isRuntime()`分为两个抽象类 **`StaticMethodMatcher`**（返回false，不考虑参数）和 **`DynamicMethodMatcher`**（返回true，考虑参数）。

`Pointcut` 也因 `MethodMathcer` 可分为 **`StaticMethodMatcherPointcut`** 和 **`DynamicMethodMatcherPointcut`**，相关类图如下所示：

![Pointcut相关类图](https://chaycao-1302020836.cos.ap-shenzhen-fsi.myqcloud.com/chaycao%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/2020/2020-05-08-Spring%E6%BA%90%E7%A0%81%E2%80%94AOP%E6%A6%82%E5%BF%B5%E7%9A%84%E5%AE%9E%E7%8E%B0/Pointcut%E7%9B%B8%E5%85%B3%E7%B1%BB%E5%9B%BE.png)

`DynamicMethodMatcherPointcut` 本文将不介绍，主要介绍下类图中列出的三个实现类。

**（1）NameMatchMethodPointcut**

通过指定**方法名称**，然后与方法的名称直接进行匹配，还支持 “*” 通配符。

```java
public class NameMatchMethodPointcut extends StaticMethodMatcherPointcut implements Serializable {
    
    // 方法名称
    private List<String> mappedNames = new ArrayList<>();

    // 设置方法名称
    public void setMappedNames(String... mappedNames) {
        this.mappedNames = new ArrayList<>(Arrays.asList(mappedNames));
    }


    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        for (String mappedName : this.mappedNames) {
            // 根据方法名匹配，isMatch提供“*”通配符支持
            if (mappedName.equals(method.getName()) || isMatch(method.getName(), mappedName)) {
                return true;
            }
        }
        return false;
    }
    
    // ...
}
```

**（2）JdkRegexpMethodPointcut**

内部有一个 Pattern 数组，通过指定**正则表达式**，然后和方法名称进行匹配。

**（3）AnnotationMatchingPointcut**

根据目标对象是否存在指定类型的**注解**进行匹配。

### 2.2 Advice

Advice 为横切逻辑的载体，Spring AOP 中关于 Advice 的接口类图如下所示：

![Advice相关类图](https://chaycao-1302020836.cos.ap-shenzhen-fsi.myqcloud.com/chaycao%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/2020/2020-05-08-Spring%E6%BA%90%E7%A0%81%E2%80%94AOP%E6%A6%82%E5%BF%B5%E7%9A%84%E5%AE%9E%E7%8E%B0/Advice%E7%9B%B8%E5%85%B3%E7%B1%BB%E5%9B%BE.png)

**（1）MethodBeforeAdvice**

横切逻辑将在 **Joinpoint 方法之前执行**。可用于进行资源初始化或准备性工作。

```java
public interface MethodBeforeAdvice extends BeforeAdvice {
    
	void before(Method method, Object[] args, @Nullable Object target) throws Throwable;
    
}
```

下面来实现一个 `MethodBeforeAdvice`，看下其效果。

```java
public class PrepareResourceBeforeAdvice implements MethodBeforeAdvice {
    
   @Override
   public void before(Method method, Object[] args, Object target) throws Throwable {
      System.out.println("准备资源");
   }
    
}
```

定义一个 `ITask` 接口：

```java
public interface ITask {

   void execute();
    
}
```

`ITask` 的实现类 `MockTask`：

```java
public class MockTask implements ITask {
    
   @Override
   public void execute() {
      System.out.println("开始执行任务");
      System.out.println("任务完成");
   }
    
}
```

Main 方法如下，`ProxyFactory`、`Advisor` 在后续会进行介绍，先简单了解下，通过`ProxyFactory`拿到代理类，`Advisor`用于封装 `Pointcut` 和 `Advice`。

```java
public class Main {
    
   public static void main(String[] args) {
      MockTask task = new MockTask();
      ProxyFactory weaver = new ProxyFactory(task);
      weaver.setInterfaces(new Class[]{ITask.class});
      // 内含一个NameMatchMethodPointcut
      NameMatchMethodPointcutAdvisor advisor = new NameMatchMethodPointcutAdvisor();
      // 指定NameMatchMethodPointcut的方法名
      advisor.setMappedName("execute");
      // 指定Advice
      advisor.setAdvice(new PrepareResourceBeforeAdvice());
      weaver.addAdvisor(advisor);
      ITask proxyObject = (ITask) weaver.getProxy();
      proxyObject.execute();
   }
    
}
```
```
output:
准备资源  
开始执行任务  
任务完成  
```

可以看出在执行代理对象 `proxyObject` 的 `execute` 方法时，先执行了 `PrepareResourceBeforeAdvice` 中的 `before` 方法。

**（2）ThrowsAdvice**

横切逻辑将在 **Joinpoint 方法抛出异常时执行**。可用于进行异常监控工作。

ThrowsAdvice 接口未定义任何方法，但约定在实现该接口时，**定义的方法需符合如下规则**：

```java
void afterThrowing([Method, args, target], ThrowableSubclass)
```

前三个参数为 Joinpoint 的相关信息，可省略。`ThrowableSubclass` 指定需要拦截的异常类型。

例如可定义多个 `afterThrowing` 方法捕获异常：

```java
public class ExceptionMonitorThrowsAdvice implements ThrowsAdvice {
    
   public void afterThrowing(Throwable t) {
      System.out.println("发生【普通异常】");
   }

   public void afterThrowing(RuntimeException e) {
      System.out.println("发生【运行时异常】");
   }

   public void afterThrowing(Method m, Object[] args, Object target, ApplicationException e) {
      System.out.println(target.getClass() + m.getName() + "发生【应用异常】");
   }
    
}
```

修改下 `MockTask` 的内容：

```java
public class MockTask implements ITask {
    
   @Override
   public void execute() {
      System.out.println("开始执行任务");
      // 抛出一个自定义的应用异常
      throw new ApplicationException();
      // System.out.println("任务完成");
   }
    
}
```

修改下 `Main` 的内容：

```java
public class Main {
    
   public static void main(String[] args) {
      MockTask task = new MockTask();
      ProxyFactory weaver = new ProxyFactory(task);
      weaver.setInterfaces(new Class[]{ITask.class});
      NameMatchMethodPointcutAdvisor advisor = new NameMatchMethodPointcutAdvisor();
      advisor.setMappedName("execute");
      // 指定异常监控Advice
      advisor.setAdvice(new ExceptionMonitorThrowsAdvice());
      weaver.addAdvisor(advisor);
      ITask proxyObject = (ITask) weaver.getProxy();
      proxyObject.execute();
   }
    
}
```

```
output:
开始执行任务
class com.chaycao.spring.aop.MockTaskexecute发生【应用异常】
```

当抛出 `ApplicationException` 时，被相应的 `afterThrowing` 方法捕获到。

**（3）AfterReturningAdvice**

横切逻辑将在 **Joinpoint 方法正常返回时执行**。可用于处理资源清理工作。

```java
public interface AfterReturningAdvice extends AfterAdvice {
    
   void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target) throws Throwable;
    
}
```

实现一个资源清理的 Advice ：

```java
public class ResourceCleanAfterReturningAdvice implements AfterReturningAdvice {
    
   @Override
   public void afterReturning(Object returnValue, Method method, Object[] args, Object target) throws Throwable {
      System.out.println("资源清理");
   }
    
}
```

修改 `MockTask` 为正常执行成功， 修改 `Main` 方法为指定 `ResourceCLeanAfterReturningAdvice`，效果如下：

```java
output:
开始执行任务
任务完成
资源清理
```

**（4）MethodInterceptor**

相当于 Around Advice，功能十分强大，**可在 Joinpoint 方法前后执行，甚至修改返回值**。其定义如下：

```java
public interface MethodInterceptor extends Interceptor {
    
   Object invoke(MethodInvocation invocation) throws Throwable;
    
}
```

`MethodInvocation` 是对 `Method` 的封装，通过 `proceed()` 对方法进行调用。下面举个例子：

```java
public class AroundMethodInterceptor implements MethodInterceptor {

   @Override
   public Object invoke(MethodInvocation invocation) throws Throwable {
      System.out.println("准备资源");
      try {
         return invocation.proceed();
      } catch (Exception e) {
         System.out.println("监控异常");
         return null;
      } finally {
         System.out.println("资源清理");
      }
   }
   
}
```

上面实现的 invoke 方法，一下子把前面说的三种功能都实现了。

以上 4 种 Advice 会在目标对象类的所有实例上生效，被称为 per-class 类型的 Advice。还有一种 per-instance 类型的 Advice，可为实例添加新的属性或行为，也就是第一节提到的 Introduction。

**（5）Introduction**

Spring 为目标对象**添加新的属性或行为**，需要**声明接口和其实现类**，然后通过**拦截器**将接口的定义和实现类的实现织入到目标对象中。我们认识下 **`DelegatingIntroductionInterceptor`**，其作为拦截器，当调用新行为时，会委派（delegate）给实现类来完成。

例如，想在原 `MockTask` 上进行加强，但不修改类的声明，可声明一个新的接口 `IReinfore`：

```java
public interface IReinforce {
   String name = "增强器";
   void fun();
}
```

再声明一个接口的实现类：

```java
public class ReinforeImpl implements IReinforce {

   @Override
   public void fun() {
      System.out.println("我变强了，能执行fun方法了");
   }

}
```

修改下 Main 方法：

```java
public class Main {
   
   public static void main(String[] args) {
      MockTask task = new MockTask();
      ProxyFactory weaver = new ProxyFactory(task);
      weaver.setInterfaces(new Class[]{ITask.class});
      // 为拦截器指定需要委托的实现类的实例
      DelegatingIntroductionInterceptor delegatingIntroductionInterceptor =
            new DelegatingIntroductionInterceptor(new ReinforeImpl());
      weaver.addAdvice(delegatingIntroductionInterceptor);
      ITask proxyObject = (ITask) weaver.getProxy();
      proxyObject.execute();
      // 使用IReinfore接口调用新的属性和行为
      IReinforce reinforeProxyObject = (IReinforce) weaver.getProxy();
      System.out.println("通过使用" + reinforeProxyObject.name);
      reinforeProxyObject.fun();
   }
   
}
```

```
output:
开始执行任务
任务完成
通过使用增强器
我变强了，能执行fun方法了
```

代理对象 `proxyObject` 便通过拦截器，可以使用 `ReinforeImpl` 实现类的方法。

### 2.3 Aspect

Spring 中用 **`Advisor`** 表示 Aspect，不同之处在于 `Advisor` 通常只持有**一个 `Pointcut`** 和**一个 `Advice`**。`Advisor` 根据 `Advice` 分为 **`PointcutAdvisor`** 和 **`IntroductionAdvisor`**。

#### 2.3.1 PointcutAdvisor

常用的 `PointcutAdvisor` 实现类有：

**（1） DefaultPointcutAdvisor**

最通用的实现类，可以指定**任意类型的 `Pointcut`** 和**除了 `Introduction` 外的任意类型 `Advice`**。

```java
Pointcut pointcut = ...; // 任意类型的Pointcut
Advice advice = ...; // 除了Introduction外的任意类型Advice
DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor();
advisor.setPointcut(pointcut);
advisor.setAdvice(advice);
```

**（2）NameMatchMethodPointcutAdvisor**

在演示 Advice 的代码中，已经有简单介绍过，**内部有一个 `NameMatchMethodPointcut` 的实例**，可持有除 `Introduction` 外的任意类型 `Advice`。

```java
Advice advice = ...; // 除了Introduction外的任意类型Advice
NameMatchMethodPointcutAdvisor advisor = new NameMatchMethodPointcutAdvisor();
advisor.setMappedName("execute");
advisor.setAdvice(advice);
```

**（3）RegexpMethodPointcutAdvisor**

内部有一个 `RegexpMethodPointcut` 的实例。

#### 2.3.2 IntroductionAdvisor

只能支持类级别的拦截，和 `Introduction` 类型的 `Advice`。实现类有 `DefaultIntroductionAdvisor`。

```java
DelegatingIntroductionInterceptor introductionInterceptor =
				new DelegatingIntroductionInterceptor(new ReinforeImpl());
		DefaultIntroductionAdvisor advisor = new DefaultIntroductionAdvisor(introductionInterceptor, IReinforce.class);
```

### 2.4 织入和织入器

在演示 Advice 的代码中，我们使用 **`ProxyFactory`** 作为织入器

```java
MockTask task = new MockTask();
// 织入器
ProxyFactory weaver = new ProxyFactory(task);
weaver.setInterfaces(new Class[]{ITask.class});
NameMatchMethodPointcutAdvisor advisor = new NameMatchMethodPointcutAdvisor();
advisor.setMappedName("execute");
advisor.setAdvice(new PrepareResourceBeforeAdvice());
weaver.addAdvisor(advisor);
// 织入，返回代理对象
ITask proxyObject = (ITask) weaver.getProxy();
proxyObject.execute();
```

`ProxyFactory` 生成代理对象方式有：

- 如果目标类实现了某些接口，默认通过**动态代理**生成。
- 如果目标类没有实现接口，默认通过**CGLIB**生成。
- 也可以**直接设置**`ProxyFactory`生成的方式，即使实现了接口，也能使用CGLIB。

在之前的演示代码中，我们没有启动 Spring 容器，也就是没有使用 Spring IOC 功能，而是独立使用了 Spring AOP。那么 Spring AOP 是如何与 Spring IOC 进行整合的？是采用了 Spring 整合最常用的方法 —— **`FactoryBean`**。

**`ProxyFactoryBean`** 继承了 `ProxyFactory` 的父类 `ProxyCreatorSupport`，具有了创建代理类的能力，同时实现了 `FactoryBean` 接口，当通过 `getObject` 方法获得 Bean 时，将得到代理类。

### 2.5 Target

在之前的演示代码中，我们直接为 `ProxyFactory` 指定一个对象为 Target。在 `ProxyFactoryBean` 中不仅能使用这种方式，还可以通过 **`TargetSource`** 的形式指定。

`TargetSource` 相当于为对象进行了一层封装，`ProxyFactoryBean` 将通过 `TargetSource` 的 `getTarget` 方法来获得目标对象。于是，我们可以通过 `getTarget` 方法来**控制获得的目标对象**。`TargetSource` 的几种实现类有：

**（1）SingletonTargetSource**

很简单，内部只持有一个目标对象，直接返回。和我们直接指定对象的效果是一样的。

**（2）PrototypeTargetSource**

每次将返回一个新的目标对象实例。

**（3）HotSwappableTartgetSource**

运行时，根据特定条件，动态替换目标对象类的具体实现。例如当一个数据源挂了，可以切换至另外一个。

**（4）CommonsPool2TargetSource**

返回有限数目的目标对象实例，类似一个对象池。

**（5）ThreadLocalTargetSource**

为不同线程调用提供不同目标对象

### 2.6 Joinpoint

终于到了最后的 Joinpoint，我们通过下面的示例来理解 Joinpoint 的工作机制。

```java
MockTask task = new MockTask();
ProxyFactory weaver = new ProxyFactory(task);
weaver.setInterfaces(new Class[]{ITask.class});
PrepareResourceBeforeAdvice beforeAdvice = new PrepareResourceBeforeAdvice();
ResourceCleanAfterReturningAdvice afterAdvice = new ResourceCleanAfterReturningAdvice();
weaver.addAdvice(beforeAdvice);
weaver.addAdvice(afterAdvice);
ITask proxyObject = (ITask) weaver.getProxy();
proxyObject.execute();
```

```
output:
准备资源    
开始执行任务     
任务完成    
资源清理
```

我们知道 `getProxy` 会通过动态代理生成一个 `ITask` 的接口类，那么 `execute` 方法的内部是如何先执行了 `beforeAdvice` 的 `before` 方法，接着执行 `task` 的 `execute` 方法，再执行 `afterAdvice` 的 `after` 方法呢？

答案就在生成的**代理类**中。在动态代理中，代理类方法调用的逻辑由 `InvocationHandler` 实例的 `invoke` 方法决定，那答案进一步锁定在 **`invoke` 方法**。

在本示例中，`ProxyFactory.getProxy`  会调用  `JdkDynamicAopProxy.getProxy`  获取代理类。

```java
// JdkDynamicAopProxy
public Object getProxy(@Nullable ClassLoader classLoader) {
   if (logger.isTraceEnabled()) {
      logger.trace("Creating JDK dynamic proxy: " + this.advised.getTargetSource());
   }
   Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
   findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
   return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

在 `getProxy` 中为 `newProxyInstance` 的 `InvocationHandler` 参数传入 `this`，即 `JdkDynamicAopProxy` 就是一个 `InvocationHandler` 的实现，其 `invoke` 方法如下：

```java
// JdkDynamicAopProxy
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   // 通过advised（创建对象时初始化）获得指定的advice
   // 会将advice用相应的MethodInterceptor封装下
   List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

   if (chain.isEmpty()) {
      Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
      retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
   }
   else {
      // 创建一个MethodInvocation
      MethodInvocation invocation =
            new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
      // 调用procced，开始进入拦截链(执行目标对象方法和MethodInterceptor的advice)
      retVal = invocation.proceed();
    }
   return retVal;
}
```

首先获得指定的 advice，这里包含 `beforeAdvice` 和 `afterAdvice` 实例，但会用 `MethodInterceptor` 封装一层，为了后面的拦截链。

再创建一个 `RelectiveMethodInvocation` 对象，最后通过 `proceed` 进入拦截链。

`RelectiveMethodInvocation` 就是 Spring AOP 中 Joinpoint 的一个实现，其类图如下：

![Joinpoint类图](https://chaycao-1302020836.cos.ap-shenzhen-fsi.myqcloud.com/chaycao%E4%B8%AA%E4%BA%BA%E5%8D%9A%E5%AE%A2/2020/2020-05-08-Spring%E6%BA%90%E7%A0%81%E2%80%94AOP%E6%A6%82%E5%BF%B5%E7%9A%84%E5%AE%9E%E7%8E%B0/Joinpoint%E7%B1%BB%E5%9B%BE.png)

首先看下 `RelectiveMethodInvocation` 的构造函数：

```java
protected ReflectiveMethodInvocation(
			Object proxy, @Nullable Object target, Method method, @Nullable Object[] arguments,
			@Nullable Class<?> targetClass, List<Object> interceptorsAndDynamicMethodMatchers) {
      this.proxy = proxy;
      this.target = target;
      this.targetClass = targetClass;
      this.method = BridgeMethodResolver.findBridgedMethod(method);
      this.arguments = AopProxyUtils.adaptArgumentsIfNecessary(method, arguments);
      this.interceptorsAndDynamicMethodMatchers = interceptorsAndDynamicMethodMatchers;
}
```

做了些相关属性的赋值，然后看向 `proceed` 方法，如何调用目标对象和拦截器。

```java
public Object proceed() throws Throwable {
   // currentInterceptorIndex从-1开始
   // 当达到已调用了所有的拦截器后，通过invokeJoinpoint调用目标对象的方法
   if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
      return invokeJoinpoint();
   }
   // 获得拦截器，调用其invoke方法
   // currentInterceptorIndex加1
   Object interceptorOrInterceptionAdvice =
         this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
   return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
}
```

`currentInterceptorIndex` 从 -1 开始，`interceptorsAndDynamicMethodMatchers` 里有两个拦截器，再由于减 1，所有调用目标对象方法的条件是`currentInterceptorIndex` 等于 1。

首先由于 `-1 != 1`，会获得包含了 `beforeAdvice` 的 `MethodBeforeAdviceInterceptor` 实例， `currentInterceptorIndex` 加 1 变为 0。调用其 `invoke` 方法，由于是 Before-Advice，所以先执行 `beforeAdvice` 的 `before` 方法，然后调用 `proceed` 进入拦截链的下一环。

```java
// MethodBeforeAdviceInterceptor
public Object invoke(MethodInvocation mi) throws Throwable {
   this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
   return mi.proceed();
}
```

又回到了 `proceed` 方法，`0 != 1`，再次获得 advice，这次获得的是包含 `afterAdvice` 的 `AfterReturningAdviceInterceptor`实例， `currentInterceptorIndex` 加 1 变为 1。调用其 `invoke` 方法，由于是 After-Returning-Adivce，所以会先执行 `proceed` 进入拦截链的下一环。

```java
// AfterReturningAdviceInterceptor
public Object invoke(MethodInvocation mi) throws Throwable {
   Object retVal = mi.proceed();
   this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
   return retVal;
}
```

再次来到 `proceed` 方法，`1 == 1`，已调用完所有的拦截器，将执行目标对象的方法。 然后 return 返回，回到 `invoke` 中，调用 `afterAdvice` 的 `afterReturning`。

所以在 Joinpoint 的实现中，通过 **`MethodInterceptor`** 完成了 目标对象方法和 Advice 的先后执行。

## 3. 小结

在了解了 Spring AOP 的实现后，笔者对 AOP 的概念更加清晰了。在学习过程中最令笔者感兴趣的是 Joinpoint 的拦截链，一开始不知道是怎么实现的，觉得很神奇 😲 。最后学完了，总结下，好像也很简单，通过拦截器的 `invoke` 方法和`MethodInvocation.proceed` 方法（进入下一个拦截器）的相互调用。好像就这么回事。😛