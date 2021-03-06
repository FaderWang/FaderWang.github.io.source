---
title: 使用AOP实现日志切入
date: 2017-12-29 15:24:18
tags: AOP
categories: Java
toc: true
comments: true
---
在Spring中大量的用到了AOP,比如@Transactional.这里我们写一个简单的Demo,使用AOP实现日志的切入。
<!-- more -->
#### 定义`@Cache`注解

```Java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Inherited
public @interface Cache {
    int value() default 0;
}
```

#### 定义日志配置类

```java
@Configuration
@ComponentScan
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class PeopleConfig {
}
```

#### 定义People接口与Man实现类

```Java
public interface People {

    /**
     * @param name
     * say hello
     */
    void sayHello(String name);
}

@Component
public class Man implements People {

    @Override
    @Cache(1)
    public void sayHello(String name) {
        System.out.println("Hello, I am Man, My name is : " + name);
    }
}
```

#### 定义日志切面类

```java
@Component
@Aspect
@Order(1000)
public class LogAspect {

    //定义切入点,使用Cache注解的方法
    @Pointcut("@annotation(Cache)")
    private void cache(){}

    //定义环绕通知
    @Around("cache()")
    private Object logAround(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        Object args = proceedingJoinPoint.getArgs();
        System.out.println("args:" + JSONObject.toJSONString(args));
        return proceedingJoinPoint.proceed();
    }

    public static void main(String[] args) {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(PeopleConfig.class);
        Man man = applicationContext.getBean("man", Man.class);
        man.sayHello("FaderWang");
    }
}
```

#### 测试运行结果

```
args:["FaderWang"]
Hello, I am Man, My name is : FaderWang
```

