---
title: post请求流的使用
date: 2018-09-21 14:01:50
tags: SpringBoot
categories: Java
toc: true
description: 解决Http POST请求request.getInputStream()数据为空的问题
---

*post*方法 request.getInputStream()为空解惑



#### 前言 

在SpringMVC web应用中，对于一个rest接口，获取请求参数我们一般使用`@requestParam`、`@requestBody`等注解 。对于表单类型的请求参数，有一下几种获取方式

1. @requestParam注解方式
2. request.getParameter(String name)
3. request.getInputStream()

前两种方式其实是一种方式，@requestParam底层就是利用request.getParameter的原理。这两种方式有一个弊端就是只能一个个获取，而且必须知道对方传过来的参数的key值，如果想要一次性获取，可以使用request.getInputStream方法获取一个inputStream对象，然后读取流里面的数据。

```html
//获取到的数据格式key=value以‘&’分隔的形式
age=20&name=faderw
```

#### 问题

但在实际过程中，我们会发现通过request.getInputStream()方式获取的数据为空。

> 根据Servlet规范，如果同时满足下列条件，则请求体(Entity)中的表单数据，将被填充到request的parameter集合中（request.getParameter系列方法可以读取相关数据）

1. 这是一个HTTP/HTTPS请求 
2. 请求方法是POST（querystring无论是否POST都将被设置到parameter中） 
3. 请求的类型（Content-Type头）是application/x-www-form-urlencoded 
4. Servlet调用了getParameter系列方法

这里的表单数据已经被填充到parameterMap中，不能再通过getInputStream获取。

如何解决这个问题呢。

#### 实现

在javax.servlet.http包下面有一个装饰器类`HttpServletRequestWrapper`，利用这个装饰器类，我们可以重新包装一个HttpServletRequest对象。

```java
public class HttpServletRequestWrapper extends ServletRequestWrapper implements
        HttpServletRequest {
```

定义一个装饰器继承`HttpServletRequestWrapper`,`streamBody`字节变量用来保存读取的数据，以便于多次读取。

```java
public class InputStreamHttpServletRequestWrapper extends HttpServletRequestWrapper{


    private final byte[] streamBody;
    private static final int BUFFER_SIZE = 4096;

   
    public InputStreamHttpServletRequestWrapper(HttpServletRequest request) throws IOException {
        super(request);
        byte[] bytes = inputStream2Byte(request.getInputStream());
        if (bytes.length == 0 && RequestMethod.POST.name().equals(request.getMethod())) {
            //从ParameterMap获取参数，并保存以便多次获取
            bytes = request.getParameterMap().entrySet().stream()
                    .map(entry -> {
                        String result;
                        String[] value = entry.getValue();
                        if (value != null && value.length > 1) {
                            result = Arrays.stream(value).map(s -> entry.getKey() + "=" + s)
                                    .collect(Collectors.joining("&"));
                        } else {
                            result = entry.getKey() + "=" + value[0];
                        }

                        return result;
                    }).collect(Collectors.joining("&")).getBytes();
        }

        streamBody = bytes;
    }

    private byte[] inputStream2Byte(InputStream inputStream) throws IOException {
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        byte[] bytes = new byte[BUFFER_SIZE];
        int length;
        while ((length = inputStream.read(bytes, 0, BUFFER_SIZE)) != -1) {
            outputStream.write(bytes, 0, length);
        }

        return outputStream.toByteArray();
    }


    @Override
    public ServletInputStream getInputStream() throws IOException {
        ByteArrayInputStream inputStream = new ByteArrayInputStream(streamBody);

        return new ServletInputStream() {
            @Override
            public boolean isFinished() {
                return false;
            }

            @Override
            public boolean isReady() {
                return false;
            }

            @Override
            public void setReadListener(ReadListener listener) {

            }

            @Override
            public int read() throws IOException {
                return inputStream.read();
            }
        };
    }

    @Override
    public BufferedReader getReader() throws IOException {
        return new BufferedReader(new InputStreamReader(getInputStream()));
    }
}
```

声明一个带有HttpServletRequest入参的构造器，从该参数对象的流中解析数据，如果没有则继续从parameterMap中获取，然后以key=value&key=value形式拼接。用streamBody接收。然后我们重写getInputStream方法，以后每次调用getInputStream方法，其实是重新利用streamBody重新new一个流，所以可以多次读取。

有了装饰器后，我们就要装饰目标对象。我们都知道SpringMVC的一次请求会被一个个过滤器层层调用，也就是我们常说的责任链模式。利用`Filter`我们就可以在某个特定的位置装饰HttpServletRequest对象。

```java
public class InputStreamWrapperFilter extends OncePerRequestFilter{

    @Override
    protected void doFilterInternal(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, FilterChain filterChain) throws ServletException, IOException {
        ServletRequest servletRequest = new InputStreamHttpServletRequestWrapper(httpServletRequest);

        filterChain.doFilter(servletRequest, httpServletResponse);
    }
}
```

`OncePerRequestFilter`这个过滤器能够保证一次请求只经过一次过滤器，所以我们直接继承该类就行了。

```java
@Bean
@Order(1)
public FilterRegistrationBean inputStreamWrapperFilterRegistration() {
    FilterRegistrationBean registrationBean = new FilterRegistrationBean();
    registrationBean.setFilter(new InputStreamWrapperFilter());
    registrationBean.setName("inputStreamWrapperFilter");
    registrationBean.addUrlPatterns("/*");

    return registrationBean;
}
```

然后注册该过滤器，设置优先级为1。Spring Boot 会按照order值的大小，从小到大的顺序来依次过滤。

#### 测试

我们写一个简单的rest接口测试下

```java
@PostMapping(produces = MediaType.APPLICATION_JSON_UTF8_VALUE)
public Object inputStreamTest(HttpServletRequest request) throws Exception {
    String bs = IOUtils.toString(request.getInputStream(), "UTF-8");
    Map<String, String> map = Maps.newHashMapWithExpectedSize(1);
    map.put("data", bs);

    return map;
}
```

curl命令

```shell
curl -X POST \
  http://127.0.0.1:9003/home \
  -H 'Cache-Control: no-cache' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -H 'Postman-Token: bb6e680c-5142-4d27-b930-6efb118a505a' \
  -d 'age=20&name=wangyuxin'
```

结果

```html
{
    "data": "age=20&name=wangyuxin"
}
```