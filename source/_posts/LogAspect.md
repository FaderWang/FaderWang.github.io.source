---
title: 使用AOP实现日志切入
date: 2017-12-29 15:24:18
tags: AOP
categories: Java
---

如果想要在方法的某个地方输出日志，但是不想要改变原有的代码。这时候可以使用代理、或者切面（原理都是AOP），这里我们使用Aspect切面来实现日志输出。

### 1.定义Cache注解

```Java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Inherited
public @interface Cache {
    int value() default 0;
}
```

### 2.定义日志配置类

```java
@Configuration
@ComponentScan
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class PeopleConfig {
}
```

### 3.定义People接口与Man实现类

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

### 4.定义日志切面类

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

### 5.测试运行结果

```
args:["FaderWang"]
Hello, I am Man, My name is : FaderWang
```

