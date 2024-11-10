
## 概述


SpringMVC 中的 MVC 即模型\-视图\-控制器，该框架围绕一个 DispatcherServlet 改计而成，DispatcherServlet 会把请求分发给各个处理器，并支持可配置的处理器映射和视图渲染等功能


SpringMVC 的工作流程如下所示：


![](https://img2024.cnblogs.com/blog/1759254/202409/1759254-20240903152022415-129722252.png)


1. 客户端发起 HTTP 请求：客户端将请求提交到 DispatcherServlet
2. 寻找处理器：DispatcherServlet 控制器查询一个或多个 HandlerMapping，找到处理该请求的 Controller
3. 调用处理器：DispatcherServlet 将请求提交到 Controller
4. 调用业务处理逻辑并返回结果：Controller 在调用业务处理逻辑后，返回 ModelAndView
5. 处理视图映射并返回模型：DispatcherServlet 查询一个或多个 ViewResolver 视图解析器，找到 ModelAndView 指定的视图
6. HTTP 响应：视图负责将结果在客户端浏览器上谊染和展示


## DispatcherServlet


在 Java 中可以使用 Servlet 来处理请求，客户端每次发出请求，Servlet 会调用 service 方法来处理，SpringMVC 通过创建 DispatchServlet 来统一接收请求并分发处理


#### 1\. 创建 DispatcherServlet


在 Tomcat 中创建 DispatcherServlet 的方式有两种：


第一种方式是通过 web.xml，Tomcat 会在启动时加载根路径下 /WEB\-INF/web.xml 配置文件，根据其中的配置加载 Servlet，Listener，Filter 等，下面是 SpringMVC 的常见配置：



```
<servlet>
    <servlet-name>dispatcherservlet>
    <servlet-class>org.springframework.web.servlet.DispatcherServletservlet-class>
    <init-param>
        
        <param-name>contextConfigLocationparam-name>
        <param-value>/WEB-INF/applicationContext.xmlparam-value>
        
        <load-on-startup>1load-on-startup>
    init-param>
servlet>

<servlet-mapping>
    <servlet-name>dispatchservlet-name>
    <servlet-pattern>/*servlet-pattern>
servlet-mapping>

```

第二种方式是通过 WebApplicationInitializer，简单来说就是 Tomcat 会探测并加载 ServletContainerInitalizer 的实现类，并调用他的 onStartup 方法，而 SpringMVC 提供了对应的实现类 SpringServletContainerInitializer。而 SpringServletContainerInitializer 又会探测并加载 ClassPath 下 WebApplicationContextInitializer 的实现类，调用它的 onStartUp 方法


因此我们可以继承 WebApplicationContextInitializer 实现 onStartUp 方法，在其中以代码的方式配置 DispatchServlet



```
public class MyWebAppInitializer implements WebApplicationInitializer {
 
    @Override
    public void onStartup(ServletContext container) {

        // 创建 dispatcher 持有的上下文容器
        AnnotationConfigWebApplicationContext dispatcherContext = new AnnotationConfigWebApplicationContext();
        dispatcherContext.register(DispatcherConfig.class);

        // 注册、配置 dispatcher servlet
        ServletRegistration.Dynamic dispatcher = container.addServlet("dispatcher", new DispatcherServlet(dispatcherContext));
        dispatcher.setLoadOnStartup(1);
        dispatcher.addMapping("/*");
    }
}

```

在创建 DispatcherServlet 时，其内部会创建一个 Spring 容器 WebApplicationContext，目的是通过 Bean 的方式管理 Web 应用中的对象


#### 2\. DispatcherServlet 初始化


DispatcherServlet 是 Servlet 的实现类，Servlet的生命周期分为三个阶段：初始化、运行和销毁。初始化阶段会调用 init() 方法，DispatcherServlet 经过一系列封装，最终会调用 initStrategies 方法进行初始化，在这里我们重点关注 initHandlerMappings 和 initHandlerAdapters



```
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}

```

initHandlerMappings 方法负责加载 HandlerMappings 也就是处理器映射器，如果程序员没有配置，那么 SpringMVC 也有默认提供的 HandlerMapping。每个 HandlerMapping 会以 Bean 的形式保持在容器，并执行各自的初始化方法。


默认的 HandlerMapping 有以下两种：


* RequestMappingHandlerMapping：根据请求 URL 映射到对应 @RequestMapping 方法
* BeanNameUrlHandlerMapping：根据请求 URL 映射到对应的 Bean 的名称（如该 Bean 的名称为 /test），这个 Bean 会提供一个处理请求逻辑的方法


RequestMappingHandlerMapping 在初始化的过程中会从处理器 bean（即被 @Controller 注解）中找出所有的处理方法（即被 @RequestMapping 注解），把处理方法的 @RequestMapping 注解解析成 RequestMappingInfo 对象，再把处理方法对象包装成 HandlerMethod 对象。然后把 RequestMappingInfo 和 HandlerMethod 对象以 map 的形式缓存起来，key 为 RequestMappingInfo，value 为 HandlerMethod，日后将请求映射到处理器时会使用到


BeanNameUrlHandlerMapping 在初始化的过程中会扫描 Spring 容器中所有的 bean，获取每个 bean 的名称以及对应的 Bean 保持起来。将每个 bean 的名称与请求的 URL 路径进行匹配，如果 bean 的名称与 URL 路径匹配（忽略大小写），那么就以匹配的 Bean 作为处理该请求的处理器。匹配 Bean 的实现如下：



```
@Componet("/welcome*")
public class WelcomeController implements Controller {

    @Override
    public void handleRequest(HttpServletRequest request, HttpServletResponse response)   {
        ...
    }
}

```

或者



```
@Componet("/welcome*")
public class WelcomeController implements HttpRequestHandler {

    @Override
    public void handleRequest(HttpServletRequest request, HttpServletResponse response)   {
        ...
    }
}

```

initHandlerAdapters 方法负责加载适配器，同样以 Bean 的形式保持在容器并执行初始化方法。如果程序员没有配置，那么 SpringMVC 也有默认提供的 HandlerAdapter。处理请求时，根据请求找到对应的处理器对象后，就会适配得到一个 HandlerAdapter，由 HandlerAdapter 执行处请求


SpringMVC 默认的适配器有：


* RequestMappingHandlerAdapter：适配处理器是 HandlerMethod 对象
* HandlerFunctionAdapter：适配处理器是HandlerFunction对象
* HttpRequestHandlerAdapter：适配处理器是 HttpRequestHandler 对象
* SimpleControerHandlerAdapter：适配处理器是 Controller 对象


## 父子容器


前面提到过，初始化 DispatcherServlet 时其内部会跟着创建一个 Spring 容器，那如果在 web.xml 中配置了两个不同的 DispatcherServlet，那么就会有两个分属不同 DispatcherServlet 的 Spring 容器



```

<servlet>
    <servlet-name>app1servlet>
    <servlet-class>org.springframework.web.servlet.DispatcherServletservlet-class>
    <init-param>
        <param-name>contextConfigLocationparam-name>
        <param-value>/WEB-INF/spring1.xmlparam-value>
        <load-on-startup>1load-on-startup>
    init-param>
servlet>

<servlet-mapping>
    <servlet-name>app1servlet-name>
    <servlet-pattern>/app1/*servlet-pattern>
servlet-mapping>


<servlet>
    <servlet-name>app2servlet>
    <servlet-class>org.springframework.web.servlet.DispatcherServletservlet-class>
    <init-param>
        <param-name>contextConfigLocationparam-name>
        <param-value>/WEB-INF/spring2.xmlparam-value>
        <load-on-startup>1load-on-startup>
    init-param>
servlet>

<servlet-mapping>
    <servlet-name>app2servlet-name>
    <servlet-pattern>/app2/*servlet-pattern>
servlet-mapping>

```

出现多个 DispatcherServlet 一般是解决多版本的问题，比如有一个 TestV1Controller 在 app1 这个 DispatcherServlet，现在多了一个升级版 TestV2Controller，就可以放在 app2，使用不同的映射路径


而有时候我们只希望区分不同的 Controller，而通用的 Service 并不需要在每个容器都保存一份，就可以配置父容器，将 Service 放在父容器。DispatcherServlet 初始化时会自动寻找是否存在父容器。



```
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListenerlistener-class>
    listener>

    <context-param>
        <param-name>contextConfigLocationparam-name>
        <param-value>/WEB-INF/root-spring.xmlparam-value>
    context-param>

    <servlet>
        <servlet-name>app1servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServletservlet-class>
        <init-param>
            <param-name>contextConfigLocationparam-name>
            <param-value>/WEB-INF/spring1.xmlparam-value>
        init-param>
        <load-on-startup>1load-on-startup>
    servlet>

    <servlet-mapping>
        <servlet-name>app1servlet-name>
        <url-pattern>/app1/*url-pattern>
    servlet-mapping>

web-app>

```

ContextLoaderListener 被配置到监听器列表，ServletContext 初始化时会使用 context\-param 中参数名为 contextConfigLocation 设置的配置文件初始化父容器


## SpringMVC 处理请求


SpringMVC 处理请求流程可分如下步骤：


1. 根据路径找到对应的 Handler
2. 解析参数并绑定
3. 执行方法
4. 解析返回值


#### 1\. 根据请求寻找 Handler


请求到来会执行 DispatcherServlet 的 getHandler 方法。遍历所有 HanlderMapping，每个 HandlerMapping 都是根据请求寻找 Handler，但寻找的方式不一样，比如 RequestMappingHandlerMapping 就是根据请求路径寻找 HandlerMethod， BeanNameUrlHandlerMapping 则是将请求路径映射到对应的 Bean 的名称。通过遍历 HandlerMapping，直到请求能找到对应的 Handler


不同的 HanlderMapping 所对应的 Handler 类型也不同，因此要找到对应类型的适配器。遍历所有 HandlerAdapter，如果找对适配的 HandlerAdapter 就返回，执行适配器的 handle 方法


#### 2\. 解析参数并执行方法


以 RequestMappingHandlerMapping 为例，Handler 的实际类型是 HandlerMethod，适配的是 RequestMappingHandlerAdapter。执行 invokeHandlerMethod 方法，解析 @initBinder 注解的方法并保存，解析 @SessionAttributes 注解设置的键值对，解析 @ModelAttribute 注解的方法，上述解析的结果将保存在 ModelFactory 对象，ModelFactory 用来初始化 Model 对象，初始化时将 @SessionAttributes 和 @ModelAttribute 设置的值保存到 Model 对象


接下来是创建参数解析器 argumentResolvers 和返回值解析器 returnValueHandlers。解析器有多种类型，对应不同的场景，例如使用 @PathVariable 注解传参就使用 PathVariableMethodArgumentResolver 解析器对象，返回值是 ModelAndView 对象则用 ModelAndViewMethodReturnValueHandler 解析器对象


获取方法参数，方法参数的类型是 MethodParameter，不仅包含了参数的名称，还包括参数的信息，比如是否有 @ReqeustParam 注解。遍历方法参数，并逐一用参数解析器遍历，找到适用的解析器进行解析，再根据参数名称从请求中获取参数值。如果定义了类型转换器，那就对参数类型进行转换。最后使用反射执行真正的方法逻辑


#### 3\. 解析返回值


拿到返回值后也是遍历寻找合适的返回值解析器进行处理，比如开发中经常会使用 @ResponseBody 注解返回 json，就会使用 RequestResponseBodyMethodProcessor 处理器进行处理，该处理器同时还承担了参数解析的作用。解析的过程中需要用到消息转换器 HttpMessageConverter，其作用是将方法的返回值转换为接收端（如浏览器）能接受的响应类型，SpringMVC 同样提供了默认的转换器。比如使用 @ResponseBody 注解的方法返回了 String 类型的返回值，那么就会遍历判断哪个消息转换器能处理 String 类型的返回值，在 RequestResponseBodyMethodProcessor 处理器中默认使用 StringHttpMessageConverter。接下来是内容协商，即是找到客户端能接受并且服务端能提供的内容类型，比如客户端希望优先返回 text/plain 类型的内容，而 StringHttpMessageConverter 能支持该类型，那么就使用 StringHttpMessageConverter 将方法返回值写入响应报文返回给客户端。如果我们希望方法直接返回对象类型并自动序列化为 json，那么就需要自定义消息转换器，此时 SpringMVC 将不再提供默认的转换器而是直接使用自定义的转换器，比如引入 MappingJackson2HttpMessageConverter 便能支持对象类型返回值转换为 json 并返回给客户端


如果不使用 @ResponseBody 注解，那么就会使用 ModelAndView 保存视图路径和数据。SpringMVC 同样提供了默认的视图解析器 ViewResolver，它会根据方法返回的 url 在 tomcat内部（不经由 DispatcherServlet 转发，而是使用原生 Servlet）进行一次转发请求到对应的视图文件如 jsp。如果 url 带有前缀 forward: 就表示这是一次转发请求，比如 forward:/app/test，SpringMVC 会去掉该前缀，使用 /app/test 重新交由 DispatcherServlet 转发交由对应处理器处理。如果 url 带有前缀 redirect:，比如 redirect:/test，SpringMVC 会去掉该前缀，给客户端的响应写上重定向头以及重定向地址即 /test，客户端会重新发送请求。转发和重定向的区别在于：转发请求是同一个，重定向则每次都是新的请求。转发时由于经过 DispatcherServlet，所以每次都会新建 Model，而重定向则会自动将 Model 中的参数拼接到重定向的 url


## @EnableWebMvc


使用 @EnableWebMvc 注解可以帮助我们在代码中自定义 SpringMVC 配置，比如添加拦截器。使用 @EnableWebMvc 注解的配置类必须继承 WebMvcConfigurer 类


需要注意的是，@EnableWebMvc 是较旧的配置 SpringMVC 的方式。如果使用 SpringBoot，它提供了自动配置，通常不需要显式使用 @EnableWebMvc，只需要在配置文件配置即可



```
@Configuration
@EnableWebMvc
public class AppConfig implements WebMvcConfigurer {

    @Autowired
    private BeforMethodInteceptor beforMethodInteceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {    
        // 注册自定义拦截器，添加拦截路径和排除拦截路径
        registry.addInterceptor(beforMethodInteceptor) //添加拦截器
                   .addPathPatterns("/**") //添加拦截路径
                   .excludePathPatterns(  //添加排除拦截路径
                           "/index",
                           "/login",
                           ...
                           );
        super.addInterceptors(registry);        
    }

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        // 配置视图解析器
        InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
        viewResolver.setPrefix("");
        viewResolver.setSuffix(".html");
        viewResolver.setCache(false);
        viewResolver.setContentType("text/html;charset=UTF-8");
        viewResolver.setOrder(0);        
        registry.viewResolver(viewResolver);
        super.configureViewResolvers(registry);
    }

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        // 定义静态资源位置和 URL 映射规则
        // 例如，将所有以 /static/ 开头的 URL 映射到 /resources/ 目录下的静态资源
        registry.addResourceHandler("/static/**")
                .addResourceLocations("/resources/");
    }

    @Override
    public void configureMessageConverters(List> converters) {
        // 添加 JSON 消息转换器
        converters.add(new MappingJackson2HttpMessageConverter());
    }

   @Override
    public void addCorsMappings(CorsRegistry registry) {
        // 跨域配置 
        registry.addMapping("/**")  // 配置允许跨域的路径
                .allowedOrigins("*")  // 配置允许访问的跨域资源的请求域名
                .allowedMethods("PUT,POST,GET,DELETE,OPTIONS")  // 配置允许访问该跨域资源服务器的请求方法
                .allowedHeaders("*"); // 配置允许请求 header 的访问
        super.addCorsMappings(registry);
    }
}

```

@EnableWebMvc 注解导入了 DelegatingWebMvcConfiguration 配置类，该类会将所有 WebMvcConfigurer 接口的实现类找到并保存起来。DelegatingWebMvcConfiguration 配置类还实现了 Aware 回调接口，因此会在 Spring 容器生命周期过程中调用回调接口，从而实现自定义配置


 \_\_EOF\_\_

       - **本文作者：** [低吟不作语](https://github.com)
 - **本文链接：** [https://github.com/Yee\-Q/p/18431349](https://github.com)
 - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com):[FlowerCloud机场](https://hanlianfangzhi.com)我。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
