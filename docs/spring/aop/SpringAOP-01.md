## AOP 概念
### Java OOP 存在哪些局限性？

- 静态化语言： 类结构一旦被定义，不容易被修改（可以通过字节码的处理进行类结构的处理）
- 侵入性扩展：只能通过继承或者组合的方式产生新的类结构

### AOP 常见使用场景

- 日志
    - 诊断上下文  log4j
    - 辅助信息  方法执行时间
- 统计
    - 方法的调用次数
    - 执行异常次数
    - 数据抽样
    - 数值累加
- 安防
    - 熔断   Netflix Hystrix
    - 限流和降级  Alibaba Sentinel
    - 认证和授权  Spring Security
    - 监控  JMX
- 性能
    - 缓存   Spring Cache
    - 超时控制

### AOP 概念

- AOP 定义
    - AspectJ：是一种编程方式，横切某个关注的领域模块的实现
    - Spring：辅助、完善 OOP 编程方式的一种实现，可以通过不改变当前实现的前提下
- Aspect （切面）概念（类似 Java 的 class）
    - AspectJ： 横切关注点的单元模块
    - Spring： 多个类之间关注的模块
- Join point 概念（类似于class内的方法）
    - AspectJ：
    - Sping：执行的程序拦截的一个模块，具体到某一个要执行的动作点上
- Pointcut 概念
    - AspectJ：挑选出某些执行流
    - Sping： 匹配 Join point 的条件
- Advice 概念（类似方法的内容）
    - Sping：目标代码执行前后的动作
        - around
            - 围绕，拦截的方法需要手动执行
        - before
            - 方法执行前
        - after
            - 方法执行后
- Introduction 概念
    - AspectJ： 属于一个内部的声明，声明类的横切类或者组织结构
    - Spring：针对某些场景进行的一些动态的辅助

## Java AOP 设计模式

- 代理模式：静态和动态代理
    - 静态代理：通常通过继承或者组合的方式实现
    - 动态代理：
      - JDK 动态代理 
      - 字节码提升 CGLB
    
- 判断模式：类、方法、注解、参数、异常... （通过反射的模式来匹配拦截的条件）
- 拦截模式：前置、后置、返回、 异常



## Spring AOP 功能概述

- 核心特性
    - 纯 Java 实现、无编译时特殊处理、不修改和控制 ClassLoader
    - 仅支持方法级别的 Join Points
    - 非完整 AOP 实现框架（部分桥接）
    - Spring IoC 容器整合
    - AspectJ 注解驱动整合（非竞争关系）

## Spring AOP 编程模型

- 注解驱动
    - 实现：Enable 模块驱动，@EnableAspectJAutoProxy
    - 注解：
        - 激活 AspectJ 自动代理：@EnableAspectJAutoProxy
        - Aspect ： @Aspect
        - Pointcut ：@Pointcut
        - Advice ：@Before、@AfterReturning、@AfterThrowing、@After、@Around
        - Introduction ：@DeclareParents（允许程序有多个不同的接口实现）
- XML 配置驱动
    - 实现：Spring Extenable XML Authoring
    - XML 元素
        - 激活 AspectJ 自动代理：<aop:aspectj-autoproxy/>
        - 配置：<aop:config/>
        - Aspect ： <aop:aspect/>
        - Pointcut ：<aop:pointcut/>
        - Advice ：<aop:around/>、<aop:before/>、<aop:after-returning/>、<aop:after-throwing/> 和 <aop:after/>
        - 代理 Scope ： <aop:scoped-proxy/>
        - Introduction ：<aop:declare-parents/>
- 底层 API
    - 实现：JDK 动态代理、CGLIB 以及 AspectJ
    - API：
        - 代理：AopProxy
        - 配置：ProxyConfig
        - Join Point：Pointcut
        - Pointcut ：Pointcut
        - Advice ：Advice、BeforeAdvice、AfterAdvice、AfterReturningAdvice、ThrowsAdvice

## Spring AOP 代理实现

- JDK 动态代理实现 - 基于接口代理
  
    通常使用 Proxy 对象来实现，在 AOP 中通过 **JdkDynamicAopProxy** 进行实现，通过其中的 **getProxy** 的方式来创建 Proxy 对象

> 为什么 Proxy.newProxyInstance 会生成新的字节码
>
> 通过对 Proxy.newProxyInstance 源码的分析
>     
>
> 1. 关键方法 getProxyClass0(ClassLoader loader,Class<?>... interfaces) 中，定义了WeakCache，通过 ProxyClassFactory 的 apply 实现进行返回代理类对象
> 2. apply 中 通过 ProxyGenerator.generateProxyClass 静态方法，生成代理对象的字节码数组
> 3. 通过 ClassLoader 机制，用 defineClass0 方法加载类到内存
> 3. 产生的代理对象，实际是实现了我们的原接口，并且继承了 Proxy 类
> 5. 通过调用 Proxy 的构造方法进行实例化，构造方法中默认将 InvocationHandler 传进去，返回实例

- CGLIB 动态代理实现 - 基于类代理（字节码提升）
  
    通过 **CglibAopProxy** 进行实现
    **为什么 Java 动态代理无法满足 AOP 的需要？**
    JDK 动态代理只能适用接口的代理，无法对类的代理
    
    ```java
    public static void main(String[] args) {
            Enhancer enhancer = new Enhancer();
            // 指定拦截父类
            enhancer.setSuperclass(DefaultEchoService.class);
            // 指定拦截的接口
            enhancer.setInterfaces(new Class[]{EchoService.class});
            // 设置拦截方法
            enhancer.setCallback(new MethodInterceptor() {
                @Override
                public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                    // 判断当前的方式来源类是否正确
                    long startTime = System.currentTimeMillis();
                    // 错误使用，这里的method其实是cglib提升后的方法，也就是当前的 intercept 方法，所以后出现循环调用
                    //  Object result = method.invoke(o, objects);
                    Object result = methodProxy.invokeSuper(o, objects);
                    long endTime = System.currentTimeMillis();
                    System.out.println("方法执行时间为：" + (endTime - startTime) + " ms.");
                    return result;
                }
            });
            EchoService echoService = (EchoService) enhancer.create();
            echoService.echo("基于 CGLIB 动态代理");
        }
    ```
    
- AspectJ 适配实现
  
    通过 **AspectJProxyFactory** 进行实现
    
    为什么 Spring 推荐 AspectJ 注解？
    
    AspectJ 有一些特殊的自身的语法，为了减少复杂度，认为AspectJ 实现是完整的