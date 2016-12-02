---
title: srping-mvc-summary
date: 2016-06-07 16:44:32
tags: spring
---

## MVC
主要解决web开发开发效率,接受请求-->解析请求并包装请求-->分发请求并逻辑处理(C)-->构造返回数据(M)-->构造最终响应(V)-->返回数据

### 接受请求
处理http请求是容器做的事

### 解析请求并包装请求
当http请求读取完毕,根据url分发到具体的逻辑处理代码前需要将请求数据转换为方便代码处理的格式,比如提取参数,将post请求中的数据反序列化为
目标方法参数对象等

### 分发请求并逻辑处理(C)
根据url分发到对应的逻辑处理代码,处理完业务逻辑后构造响应数据

### 构造最终响应(V)
返回的内容可能是JSON也可能是网页,所以需要根据model来构造最终响应数据

### 返回数据
容器完成数据返回

## 配置
spring mvc基于servlet,所以需要在web.xml中配置,servlet 3.0支持使用注解,估计spring也可以不用再web.xml中配置了,先看下使用web.xml如何配置.
在web.xml中配置一个ContextLodaerListener,这是一个ServletContextListener, 在servlet容器启动后,在其生命周期中会调用listener的contextInitialized方法
可以使用javax.servlet.annotation.WebListener就不用在web.xml中配置监听器了

### 加载spring配置文件
ContextLodaerListener完成spring容器初始化,使用XmlWebApplicationContext作为bean factory,<context-param></context-param>指定配置文件位置,加载bean定义,并绑定到servletContex上下文中,
这里加载的context作为mvc容器bean factory的rootContext. 主要加载应用程序逻辑业务bean,包含service,dao等,稍后加载的controller会使用这里加载的bean

### 初始化DispatcherServlet
初始化servlet分发器,执行servlet.init()方法使用web.xml配置初始化servlet,默认加载servletName+servlet.xml作为配置文件名字或者servlet参数配置的文件名,contextLoaderListener已经创建好了webApplicationContext,
使用已经创建好的applicationContext作为父容器创建新的applicationContext. 所有的servlet对应的容器共享contextLoaderListener创建的容器.默认配置文件名采用钩子方法,重写getLocations()实现,如果没有指定mvc配置文件则采用默认文件名字,

servlet加载完spring-servlet.xml配置文件后,开始初始化spring-mvc
http://www.springframework.org/schema/mvc
http://www.springframework.org/schema/mvc/spring-mvc.xsd

## spring-mvc主要组件,主要采用spring标签扩展机制实现
### HandlerMapping
处理请求映射,收到http请求,根据请求去查找对应的Handler,实现有:
1.RequestMappingHandlerMapping(<mvc:annotation-driven>解析器注册,用于@Controller,@RequestMapping,会遍历所有使用前面注解的类(进一步遍历方法),使用url作为key保存,HandlerMethod作为value)
2.BeanNameUrlHandlerMapping(handlerMapping默认实现,根据bean name作为url注册,注册HandlerMapping的地方都会调用注册该HandlerMapping)
3.SimpleUrlHandlerMapping(<resource >解析器注册,用于映射静态资源, 其它标签也可能再次注册使用该类作为class的bean definition,完成其它功能)
当收到一个请求时,会遍历所有HandlerMapping,如果有一个HandlerMapping返回了Handler,则停止遍历

### HandlerAdapter
注册HandlerMapping时,同时注册了几个HandlerAdapter,顺序和HandlerMapping对应
1.RequestMappingHandlerAdapter
2.HttpRequestHandlerAdapter
3.SimpleControllerHandlerAdapter
在HanderMapping中找到对应Handler(类HandlerMethod,或者Controller)后,不是直接进行处理,寻找对应的HandlerAdapter,执行HandlerAdaper.handle(),在内部在调用具体的Handler方法
在HandlerAdapter中并不是直接使用反射调用HandlerMethod中Method,HandlerMethodArgumentResolver负责参数转换,使用HandlerMethodInvoker包装调用,invoker会做额外的工作,比如使用了@RequestBody注解,则该位置的参数需要使用
messageConverters转换请求到对应的参数类型,然后该位置的参数值是根据请求自动构建. messageConverters是在解析<annotation-driven>的时候注入到RequestMappingHandlerAdapter中,


### HandlerExecuteChain
因为执行具体的Handler之前还要执行拦截器,所以采用了链式结构,执行具体Handler之前先执行拦截器,


### HttpMessageConverter
策略接口,根据http请求特征选择不同的converter,例如方法使用了@RequestBody则判断content-type值,遍历messageConverter,寻找能够支持该值的converter并转换





spring-mvc自定义了一套标签,<mvc:annotation-driven>,<mvc:message-converters>等来实现mvc功能,根据spring标签扩展可以肯定一定有个标签解析namespaceHandler
1.<mvc:annotation-driven>,默认注册RequestMappingHandlerMapping来处理注解映射,该类也是InitializingBean子类,在加载完bean definition之后实例化会调用afterPropertiesSet()执行initHandlerMethods(),
该方法遍历bean factory中所有的bean definition的class,根据是否使用了@Controller和@RequestMapping注解,使用了的话继续检查class中method,检查@RequestMapping注解,创建RequestMappingInfo




protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);//a
		initLocaleResolver(context);//b
		initThemeResolver(context);//c
		initHandlerMappings(context);//d
		initHandlerAdapters(context);//e
		initHandlerExceptionResolvers(context);//f
		initRequestToViewNameTranslator(context);//g
		initViewResolvers(context);//h
		initFlashMapManager(context);//i
}

a.默认没有multipartResolver,需要在spring-servlet.xml中配置,使用bean name = multipartResolver, MultipartResolver.class子类
b.默认采用AcceptHeaderLocaleResolver作为LocaleResolver,见文件DispatcherServlet.properties文件
c.默认采用FixedThemeResolver
d.handlerMappings为List,从容器中查找所有实现了HandlerMapping的bean,默认有4个,RequestMappingHandlerMapping,SimpleUrlHandlerMapping...,如果某个handlerMapping能够返回一个handler处理,则停止
d1.RequestMappingHandlerMapping包含一个urlMap,根据请求的url返回RequestMappingInfo,最终找到一个HandlerMethod(包含实例,方法,参数,)




