---
title: Venus
date: 2018-08-25 21:35:39
tags: spider,framework
categories: Java
description: 轻量级爬虫框架Venus,基于组件的形式，可扩展。
---

## 轻量级爬虫框架Venus

[TOC]

### 核心组件

- **Downloader**(网页下载器)
- **Spider**(爬虫处理器)
- **Scheduler**(调度器)
- **Parser**(网页解析器)
- **Pipline**(数据处理器)
- **Engine**(流转引擎)

### 具体介绍

#### **Downloader**

网页下载器，顾名思义，就是从网络上下载数据，也就是爬虫的根本事件。给定一个或一组url,进行http请求，获取我们想要的数据（json、html、xml等）。而编写http请求的过程往往是重复的，所以应当进行封装以通用。

```java
public class Downloader implements Runnable{

    private final Scheduler Scheduler;
    private final Request request;

    public Downloader(Scheduler scheduler, Request request) {
        this.Scheduler = scheduler;
        this.request = request;
    }

	@Override
	public void run() {
        log.info("start request url {}", request.getUrl());
        HttpRequest httpRequest = null;

        if ("get".equalsIgnoreCase(request.getMethod())) {
            httpRequest = HttpRequest.get(request.getUrl());
        } else if ("post".equalsIgnoreCase(request.getMethod())) {
            httpRequest = HttpRequest.post(request.getUrl());
        } else {
            log.error("method {} 无效", request.getMethod());
        }

        InputStream inputStream = httpRequest.contentType(request.contentType())
            .headers(request.headers()).connectTimeout(request.getSpider().getConfig().timeout())
            .readTimeout(request.getSpider().getConfig().timeout()).stream();

        log.info("download has finsihed url {}", request.getUrl());
        Response response = new Response(request, inputStream);
        Scheduler.addResponse(response);
	}
}
```

一个Downloader就是一个线程,`Request`对象封装了请求的信息，包括url、header等，请求完后，获取一个包含了返回数据的流，这里不作处理，直接构造一个`response`对象，然后压入到`Scheduler`（调取器）的返回队列中去。交给后续的爬虫处理器以及解析器来处理。

#### **Scheduler**

调取器，就是调度请求与返回的，在解析器与下载器之间进行流转，解析器解析新的url加入请求队列中，调度器再生成新的下载器进行下载。

```java
private BlockingQueue<Request> pending = Queues.newLinkedBlockingQueue();
private BlockingQueue<Response> processed = Queues.newLinkedBlockingQueue();
```

Scheduler中有两个队列，待爬取的Request和已爬取的Response。Scheduler提供入队和出队操作，供其他组件进行调用。

#### **Spider**

爬虫处理器，用于对爬取的数据进行处理，比如说入库、写文件等。

```java
public abstract class Spider {

    protected Config config;
    protected String name;
    protected List<String> startUrls = Lists.newArrayList();
    protected List<Request> requests = Lists.newArrayList();
    protected List<Pipeline> pipelines = Lists.newArrayList();

    public void setConfig(Config config) {
        this.config = config;
    }

    public Config getConfig() {
        return this.config;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return this.name;
    }

    public List<Pipeline> getPipelines() {
        return this.pipelines;
    }

    public void setPipelines(List<Pipeline> pipelines) {
        this.pipelines = pipelines;
    }

    public Spider() {

    }

    public Spider(String name) {
        this.name = name;
        EventManager.registerEvent(EventManager.VenusEvent.SPIDER_STARTED, this::onStart);

    }

    public List<String> getStartUrls() {
        return this.startUrls;
    }

    public Spider startUrls(String... urls) {
        this.startUrls.addAll(Arrays.asList(urls));
        return this;
    }

    public List<Request> getRequests() {
        return this.requests;
    }

    /**
     * 爬虫启动前执行
     */
    public abstract void onStart(Config config);

    protected <T> Spider addPipline(Pipeline<T> pipeline) {
        this.pipelines.add(pipeline);
        return this;
    }

    /**
     * 构建一个request
     */
    public <T> Request<T> makeRequest(String url) {
        return makeRequest(url, this::parse);
    }

    public <T> Request<T> makeRequest(String url, Parser<T> parser) {
        return new Request<>(this, url, parser);
    }

    /**
     * 解析DOM
     * 子类需要实现此方法
     */
    protected abstract <T> Result<T> parse(Response response);

    protected void resetRequest(Consumer<Request> consumer) {
        this.resetRequest(this.requests, consumer);
    }

    protected void resetRequest(List<Request> requests, Consumer<Request> consumer) {
        requests.forEach(consumer::accept);
    }
}
```

- param-startUrls  抽象类`Spider`维护了一个url集合，用于接受爬虫原始的url，进一步封装成request对象
- method-parse 抽象方法，子类实现该方法定义自己的解析操作
- param-pipeline 管道集合，进行数据处理的类，类似管道

#### **Engine**

整个爬虫流程的核心控制器，流转引擎

```java
public class VenusEngine {

    private List<Spider> spiders;
    private Config config;
    private Scheduler scheduler;
    private ExecutorService executorService;
    private boolean isRunning;

    public VenusEngine(Venus venus) {
        this.spiders = venus.spiders;
        this.config = venus.config;
        this.scheduler = new Scheduler();
        this.executorService = new ThreadPoolExecutor(config.parallelThreads(), config.parallelThreads(), 
            60L, TimeUnit.MILLISECONDS, config.queueSize() == 0 ? new SynchronousQueue<>()
            : (config.queueSize() < 0 ? new LinkedBlockingQueue<>() : new LinkedBlockingQueue<>(config.queueSize())),
            new ThreadFactoryBuilder().setNameFormat("task-thread-%d").build());
    }

    public void start() {
        if (isRunning) {
            throw new RuntimeException("Venus 已经启动");
        }

        isRunning = true;
        log.info("全局启动事件");
        EventManager.fireEvent(VenusEvent.GLOBAL_STARTED, this.config);

        spiders.forEach(spider -> {
            // 使用克隆为每个spider对象设置一个config属性
            Config config = this.config.clone();
            spider.setConfig(config);
            
            List<Request> requests = spider.getStartUrls().stream()
                .map(spider::makeRequest).collect(Collectors.toList());
            
            spider.getRequests().addAll(requests);
            scheduler.addRequest(requests);
            EventManager.fireEvent(VenusEvent.SPIDER_STARTED, this.config);

        });

        // 开启一个线程来不断扫描是否有带爬取的Request
        Thread downloadThread = new Thread(() -> {
            while (isRunning) {
                if (!scheduler.hasRequest()) {
                    VenusUtils.sleep(100);
                    continue;
                }

                Request request = scheduler.nextRequest();
                executorService.submit(new Downloader(scheduler, request));
                VenusUtils.sleep(request.getSpider().getConfig().Delay());
            }
        });

        downloadThread.setDaemon(true);
        downloadThread.setName("download-thread");
        downloadThread.start();

        //消费
        this.complete();
    }

    private void complete() {
        while (isRunning) {
            if (!scheduler.hasResponse()) {
                VenusUtils.sleep(100);
                continue;
            }
            Response response = scheduler.nextResponse();
            Parser parser   = response.getRequest().getParser();
            if (null != parser) {
                Result<?>     result   = parser.parse(response);
                List<Request> requests = result.getRequests();
                if (!VenusUtils.isEmpty(requests)) {
                    requests.forEach(scheduler::addRequest);
                }
                if (null != result.getItem()) {
                    List<Pipeline> pipelines = response.getRequest().getSpider().getPipelines();
                    pipelines.forEach(pipeline -> pipeline.process(result.getItem(), response.getRequest()));
                }
            }
        }
    }
}
```

流转引擎主要分为以下几个步骤：

1. 遍历spider集合，构建request，压入调度器，同时消费爬虫启动事件。
2. 开启一个线程来专门从调取器中获取待处理的request，创建相应的下载器去下载。
3. 消费response,这里分为两步parser和pipeline，解析和处理。

#### **Parser**

网页解析器，对元数据进行解析，提取我们所需的信息，转换成对应的对象。

```java
public interface Parser<T> {

    Result<T> parse(Response response);
    
}
```

实现该方法编写自己的解析逻辑

#### **Pipeline**

数据处理器，pipeline有管道的意思，解析器完成解析后，把解析的结果扔进数据处理管道中进行处理。是入库，写文件还是打印等等。

```java
public interface Pipeline<T> {

    void process(T item, Request<?> Request);
}
```