## SpringMVC

![image-20201224132721541](C:\Users\dht24\AppData\Roaming\Typora\typora-user-images\image-20201224132721541.png)

### hello world细节

#### 运行流程：

> 1、客户端点击链接发送请求
> 2、来到tomcat服务器
> 3、SpringMVC的前端控制器收到所有请求
> 4、来看请求地址和@RequestMapping标注的哪个匹配
> 5、前端控制器找到了目标处理器和目标方法，直接利用反射执行目标方法
> 6、方法执行完成后有一个返回值，SpringMVC认为这个返回值就是要去的页面地址
> 7、拿到方法返回值后，用视图解析器进行拼串得到完整的页面地址
> 8、拿到页面地址，前端控制器给转发到页面

#### @RequestMapping

> 默认从当前项目开始
>
> 可以标在类上（基准路径），也可以标在方法上
>
> 属性：
>
> - 默认value
> - method：限定请求方式（RequestMapping.POST），默认都可以
> - params：规定请求参数，只有有某些参数或没有才能处理，params={"username"}代表请求必须带username参数
> - headers：规定请求头，headers={"User-Agent=..."}，请求头里的任意字段
> - consumes：只接受内容类型是哪种的请求，规定请求头中的Content-Type
> - produces：告诉浏览器返回的内同类型是什么，给响应头中设置Content-Type

#### @PathVariable

> 为了支持RESTful风格
>
> 路径上可以有占位符
>
> @RequestMapping("/user/{id}")
> public String getId(@PathVariable("id") String id){
>
> }

#### REST

> Representational State Transfer (资源)表现层状态转化
>
> 万物皆资源，认为客户端都是在向服务端要资源，每种资源对应一个特定的URI
> 发请求，就是通过请求改变资源的状态
>
> 路径里都是资源，不会有deleteBook这样的url
>
> 通过GET, POST, DELETE, PUT来实现增删改查

### 获取请求参数

默认获取请求参数：
1、直接和方法名相同
2、@RequestParam("user") String username 
		等同于username = request.getParameter("user")

#### @RequestParams(注意和PathVariable的区别)

- value
- required
- defaultValue

#### @RequestHeader

​	@RequestHeader("Agent") String Agent       同样有三个属性

#### @CookieValue

​	@CookieValue("JSESSIONID") String jid       同样有三个属性

#### POJO

如果请求参数是POJO，自动进行赋值
从request参数中取出来，按名字，自动封装

#### 原生API

```java
/**
     * 可以传入原生API
     * HttpSession
     * HttpServletRequest
     * HttpServletResponse
     */
@RequestMapping("/handle02")
public String handle02(HttpSession session, HttpServletRequest httpServletRequest){
    session.setAttribute("sessionP","session_hahaha");
    httpServletRequest.setAttribute("requestP","request_hahaha");
    return "success";
}
```

### 数据输出

可以使用原生API

springMVC如何把数据带给页面

```java
/**
     * 可以传入Map Model ModelMap
     * 三者都是被BindingAwareModelMap实现
     * Map(Interface(jdk))
     * Model(Interface(Spring))
     * ModelMap(class) implements LinkedHashMap
     * BindingAwareModelMap(class) extends ModelMap implements Model
     */
@RequestMapping("/response")
public String response(Map<String,Object> map){
    map.put("request","hello world");
    return "success";
}

@RequestMapping("/response2")
public String response2(Model model){
    model.addAttribute("model","model的Value");
    return "success";
}

@RequestMapping("/response3")
public String response3(ModelMap modelMap){
    modelMap.addAttribute("model","modelMap的Value");
    return "success";
}
```

方法的返回值可以变成ModelAndView对象

```java
/**
     * 方法的返回值可以变成ModelAndView
     *  既包含视图页面,也包含模型数据，所有数据都放在了Request域中
     */
@RequestMapping("/response4")
public ModelAndView response4(){
    //传入视图名
    ModelAndView mv = new ModelAndView("success");
    mv.addObject("mv","mv的Value");
    return mv;
}
```

#### @ModelAttribute

> 不修改的字段不提供修改框
> 为了简单，Controller直接在参数位置来写Book对象
> POJO传过来，SpringMVC自动封装book，没有带的值是null
> 如果接下来调用了一个全字段更新的dao操作，会将其他的字段变为null

Book对象的创建：

- SpringMVC创建一个book对象，每个属性默认就是null（这里就会出问题）
- 将请求中所有与book对应的属性一一设置过来
- 调用全字段更新就有问题

SpringMVC从数据库中取出book对象，给它里面设置值。使用这个对象，而不是创建的属性为null的对象。

> @ModellAttribute可以标注于方法和参数上
> 标注于方法：这个方法先于目标方法运行
> 标注于参数：取出map中保存的信息

![image-20201225131132779](C:\Users\dht24\AppData\Roaming\Typora\typora-user-images\image-20201225131132779.png)





```xml
<!--前端控制器-->
<servlet>
    <servlet-name>dispatcherServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <!-- 配置DispatcherServlet的初始化參數：设置文件的路径和文件名称 -->
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:springmvc.xml</param-value>
    </init-param>
    <!--优先级-->
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>dispatcherServlet</servlet-name>
    <!--/会拦截所有请求，jsp访问正常-->
    <!--处理*/jsp是tomcat做的事，所有项目的小web.xml都是继承自tomcat大web.xml
	  1、/表示把大web.xml中的defaultservlet给重写了，/*表示拦截所有
-->
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

#### 如果不指定配置文件

默认找/WEB-INF/前端控制器名-servlet.xml



## 源码

#### 前端控制器DispatcherServlet

> 继承FramworkServlet <- HttpServletBean <-HttpServlet
>
> 请求一进来来到HttpServlet的doGet或doPost方法
>
> doGet	doPost都在FramworkServlet中被调用，processRequest(HttpServletRequest request, HttpServletResponse response), 链式调用doService(request, response), 链式调用DispatcherServlet的doDispatch(request, response)

##### 处理流程

1. 所有请求过来DispatchServlet来接收
2. 调用doDispatch方法执行
   1）getHandler()：根据当前请求地址在HandlerMapping中找到能处理这个请求的目标处理器
   2）getHandlerAdapter()：根据当前处理器类获取能执行这个处理器方法的适配器
   3）使用刚才获得到的适配器执行目标方法
   4）适配器执行目标方法后返回一个ModelAndView对象
   5）根据这个ModelAndView对象的信息转发到具体页面，并可以在请求域中取出ModelAndView对象的模型数据

##### 关键方法

```java
//Actually invoke the handler	控制器（处理器）的方法被调用
//真正的页面跳转
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

```java
//转发的目标页面
processDispatchResult(processedRequest, response, mappedHandler, ex)
```

doDispatch方法：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            //1、检查是否文件上传请求，如果是，包装成新的request
            processedRequest = this.checkMultipart(request);
            multipartRequestParsed = processedRequest != request;
            //2、确定哪个controller来处理这个请求，mappedHandler封装了可以处理请求的对象，细节1
            mappedHandler = this.getHandler(processedRequest);
            //3、如果没有找到哪个controller来处理这个请求，则抛异常
            if (mappedHandler == null || mappedHandler.getHandler() == null) {
                this.noHandlerFound(processedRequest, response);
                return;
            }

         //4、拿到能执行这个类的所有方法的适配器（反射工具 AnnotationMethodHandlerAdapter），细节2
            HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler());
            String method = request.getMethod();
            boolean isGet = "GET".equals(method);
            if (isGet || "HEAD".equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (this.logger.isDebugEnabled()) {
                    String requestUri = urlPathHelper.getRequestUri(request);
                    this.logger.debug("Last-Modified value for [" + requestUri + "] is: " + lastModified);
                }

                if ((new ServletWebRequest(request, response)).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }

            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            try {
                //5、Actually invoke the handler	控制器（处理器）的方法被调用
                //真正的目标方法执行,执行完成后返回ModelAndView对象
                //最终都是用适配器（ha）执行
                //细节3
                mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
            } finally {
                if (asyncManager.isConcurrentHandlingStarted()) {
                    return;
                }

            }

            this.applyDefaultViewName(request, mv);//如果没有视图名，设置默认视图名
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        } catch (Exception var28) {
            dispatchException = var28;
        }

     //6、根据方法最后执行完成后封装的mv对象，转发到对应页面，而且mv对象中的数据可以从请求域中获取，细节4
        this.processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    } catch (Exception var29) {
        this.triggerAfterCompletion(processedRequest, response, mappedHandler, var29);
    } catch (Error var30) {
        this.triggerAfterCompletionWithError(processedRequest, response, mappedHandler, var30);
    } finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            return;
        }

        if (multipartRequestParsed) {
            this.cleanupMultipart(processedRequest);
        }

    }

}
```

**细节1**	mappedHandler = this.getHandler(processedRequest)：

```java
//mappedHandler是HandlerExecutionChain对象
//根据request的中保存的路径，拿到IOC容器中的处理器HandlerExecutionChain handler
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    //HandlerMapping里面属性HandlerMap中保存了路径url对应的处理器对象
    //HandlerMap是一个LinkedHashMap
    //handlerMappings是多个handlerMapping，比如注解形成的，或者配置形成的
    Iterator var2 = this.handlerMappings.iterator();

    HandlerExecutionChain handler;
    do {
        if (!var2.hasNext()) {
            return null;
        }

        HandlerMapping hm = (HandlerMapping)var2.next();
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Testing handler map [" + hm + "] in DispatcherServlet with name '" + this.getServletName() + "'");
        }
		
        //如果找到了，不为空就返回
        handler = hm.getHandler(request);
    } while(handler == null);

    return handler;
}
```

**细节2**	HandlerAdapter ha = this.getHandlerAdapter(mappedHandler.getHandler())

```java
//找到目标处理类的适配器。拿到适配器去执行目标方法
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    //handlerAdapters包含3种
    //HttpRequestHandlerAdaptor
    //SimpleControllerHandlerAdaptor
    //AnnotationMethodHandlerAdaptor(用的是这个)
    Iterator var2 = this.handlerAdapters.iterator();

    HandlerAdapter ha;
    do {
        if (!var2.hasNext()) {
            throw new ServletException("No adapter for handler [" + handler + "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
        }

        ha = (HandlerAdapter)var2.next();
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Testing handler adapter [" + ha + "]");
        }
    } while(!ha.supports(handler));

    return ha;
}
```

SpringMVC九大组件（全是接口，接口就是规范），SpringMVC工作时关键位置都是由组件完成：

```java
private MultipartResolver multipartResolver;	//文件上传解析器
private LocaleResolver localeResolver;	//区域信息解析器，国际化
private ThemeResolver themeResolver;	//主题解析器，主题效果更换，没人用
private List<HandlerMapping> handlerMappings;	//Handler映射信息
private List<HandlerAdapter> handlerAdapters;	//Handler适配器
private List<HandlerExceptionResolver> handlerExceptionResolvers;	//异常解析功能
private RequestToViewNameTranslator viewNameTranslator;	
private FlashMapManager flashMapManager;	//运行重定向携带数据的功能
private List<ViewResolver> viewResolvers;	//视图解析器
```

SpringMVC九大组件的初始化：

> 在Spring源码中的onrefresh方法

```java
/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
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

组件的初始化就是去IOC容器中找这些组件，找不到就是用默认的

```java
//以initHandlerMappings为例
private void initHandlerMappings(ApplicationContext context) {
    this.handlerMappings = null;
    //默认true
    if (this.detectAllHandlerMappings) {
        Map<String, HandlerMapping> matchingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.handlerMappings = new ArrayList(matchingBeans.values());
            OrderComparator.sort(this.handlerMappings);
        }
    } else {
        try {
            HandlerMapping hm = (HandlerMapping)context.getBean("handlerMapping", HandlerMapping.class);
            this.handlerMappings = Collections.singletonList(hm);
        } catch (NoSuchBeanDefinitionException var3) {
            ;
        }
    }

    if (this.handlerMappings == null) {
        this.handlerMappings = this.getDefaultStrategies(context, HandlerMapping.class);
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("No HandlerMappings found in servlet '" + this.getServletName() + "': using default");
        }
    }

}
```

**细节3**	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

反射执行，method.invoke(obj, args), 但args的灵活度非常大

```java
//里面执行方法的是
return invokeHandlerMethod(request,response,handler);
```

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    //获取Method解析器
    AnnotationMethodHandlerAdapter.ServletHandlerMethodResolver methodResolver = this.getMethodResolver(handler);
    //通过Method解析器获得应该执行的方法
    Method handlerMethod = methodResolver.resolveHandlerMethod(request);
    //获取方法执行器
    AnnotationMethodHandlerAdapter.ServletHandlerMethodInvoker methodInvoker = new AnnotationMethodHandlerAdapter.ServletHandlerMethodInvoker(methodResolver);
    //包装原生的request, response
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    
    //重要！！！创建BindingAwareModelMap
    //隐含模型implicitModel
    ExtendedModelMap implicitModel = new BindingAwareModelMap();
    //真正执行目标方法：利用反射执行期间要确定参数值.参见下一部分代码
    Object result = methodInvoker.invokeHandlerMethod(handlerMethod, handler, webRequest, implicitModel);
    ModelAndView mav = methodInvoker.getModelAndView(handlerMethod, handler.getClass(), result, implicitModel, webRequest);
    methodInvoker.updateModelAttributes(handler, mav != null ? mav.getModel() : null, implicitModel, webRequest);
    return mav;
}
```

```java

public final Object invokeHandlerMethod(Method handlerMethod, Object handler, NativeWebRequest webRequest, ExtendedModelMap implicitModel) throws Exception {
    Method handlerMethodToInvoke = BridgeMethodResolver.findBridgedMethod(handlerMethod);

    try {
        boolean debug = logger.isDebugEnabled();
        Iterator var7 = this.methodResolver.getActualSessionAttributeNames().iterator();

        while(var7.hasNext()) {
            String attrName = (String)var7.next();
            Object attrValue = this.sessionAttributeStore.retrieveAttribute(webRequest, attrName);
            if (attrValue != null) {
                implicitModel.addAttribute(attrName, attrValue);
            }
        }

        //找到所有modelAttribute注解标注的方法
        var7 = this.methodResolver.getModelAttributeMethods().iterator();

        while(true) {
            Object[] args;
            String attrName;
            Method attributeMethodToInvoke;
            do {
                if (!var7.hasNext()) {
                    Object[] args = this.resolveHandlerArguments(handlerMethodToInvoke, handler, webRequest, implicitModel);
                    if (debug) {
                        logger.debug("Invoking request handler method: " + handlerMethodToInvoke);
                    }

                    ReflectionUtils.makeAccessible(handlerMethodToInvoke);
                    return handlerMethodToInvoke.invoke(handler, args);
                }

                Method attributeMethod = (Method)var7.next();
                attributeMethodToInvoke = BridgeMethodResolver.findBridgedMethod(attributeMethod);
                //确定modelAttribute执行时需要参数的值，arg是和参数个数一样多的数组
                args = this.resolveHandlerArguments(attributeMethodToInvoke, handler, webRequest, implicitModel);
                if (debug) {
                    logger.debug("Invoking model attribute method: " + attributeMethodToInvoke);
                }

                attrName = ((ModelAttribute)AnnotationUtils.findAnnotation(attributeMethod, ModelAttribute.class)).value();
            } while(!"".equals(attrName) && implicitModel.containsAttribute(attrName));

            ReflectionUtils.makeAccessible(attributeMethodToInvoke);
            //执行目标方法
            Object attrValue = attributeMethodToInvoke.invoke(handler, args);
            if ("".equals(attrName)) {
                Class<?> resolvedType = GenericTypeResolver.resolveReturnType(attributeMethodToInvoke, handler.getClass());
                attrName = Conventions.getVariableNameForReturnType(attributeMethodToInvoke, resolvedType, attrValue);
            }

            if (!implicitModel.containsAttribute(attrName)) {
                implicitModel.addAttribute(attrName, attrValue);
            }
        }
    } catch (IllegalStateException var14) {
        throw new HandlerMethodInvocationException(handlerMethodToInvoke, var14);
    } catch (InvocationTargetException var15) {
        ReflectionUtils.rethrowException(var15.getTargetException());
        return null;
    }
}
```



```java
public final Object invokeHandlerMethod(Method handlerMethod, Object handler,
                                        NativeWebRequest webRequest, ExtendedModelMap implicitModel) throws Exception {

    Method handlerMethodToInvoke = BridgeMethodResolver.findBridgedMethod(handlerMethod);
    try {
        boolean debug = logger.isDebugEnabled();
        for (String attrName : this.methodResolver.getActualSessionAttributeNames()) {
            Object attrValue = this.sessionAttributeStore.retrieveAttribute(webRequest, attrName);
            if (attrValue != null) {
                implicitModel.addAttribute(attrName, attrValue);
            }
        }
        //找到所有@ModelAttribute注解标注的方法，并运行
        for (Method attributeMethod : this.methodResolver.getModelAttributeMethods()) {
            Method attributeMethodToInvoke = BridgeMethodResolver.findBridgedMethod(attributeMethod);
            //确定@ModelAttribute注解标注的方法执行时需要的参数
            Object[] args = resolveHandlerArguments(attributeMethodToInvoke, handler, webRequest, implicitModel);
            if (debug) {
                logger.debug("Invoking model attribute method: " + attributeMethodToInvoke);
            }
            //方法上标注的ModelAttribute注解value值,没标就是空串
            String attrName = AnnotationUtils.findAnnotation(attributeMethod, ModelAttribute.class).value();
            if (!"".equals(attrName) && implicitModel.containsAttribute(attrName)) {
                continue;
            }
            ReflectionUtils.makeAccessible(attributeMethodToInvoke);
            //反射执行ModelAttribute提前运行的方法
            Object attrValue = attributeMethodToInvoke.invoke(handler, args);
            if ("".equals(attrName)) {
                //解析返回值类型（void）
                Class<?> resolvedType = GenericTypeResolver.resolveReturnType(attributeMethodToInvoke, handler.getClass());
                //返回值类型首字母小写
                attrName = Conventions.getVariableNameForReturnType(attributeMethodToInvoke, resolvedType, attrValue);
            }
            //把方法运行后的返回值按照方法上标注的ModelAttribute注解value值为key，放在隐含模型中
            if (!implicitModel.containsAttribute(attrName)) {
                implicitModel.addAttribute(attrName, attrValue);
            }
        }
        
        //解析目标方法参数（一种是标了注解的参数，一种是不标的）
        //标了注解：保存是哪个注解的详细信息
        //没标注解：先看是不是普通参数（是否原生API），再看是否Model或者Map，如果是就传入隐含模型；
        Object[] args = resolveHandlerArguments(handlerMethodToInvoke, handler, webRequest, implicitModel);
        if (debug) {
            logger.debug("Invoking request handler method: " + handlerMethodToInvoke);
        }
        ReflectionUtils.makeAccessible(handlerMethodToInvoke);
        //执行目标方法
        return handlerMethodToInvoke.invoke(handler, args);
    }
    catch (IllegalStateException ex) {
        // Internal assertion failed (e.g. invalid signature):
        // throw exception with full handler method context...
        throw new HandlerMethodInvocationException(handlerMethodToInvoke, ex);
    }
    catch (InvocationTargetException ex) {
        // User-defined @ModelAttribute/@InitBinder/@RequestMapping method threw an exception...
        ReflectionUtils.rethrowException(ex.getTargetException());
        return null;
    }
}
```





确定POJO值的三步：

1. 如果隐含模型中有这个key（标了ModelAttribute注解的就是注解指定的value，没标就是参数类型的首字母小写）指定的值，就把这个值赋值给bindObject；
2. 如果是SessionAttributes标注的属性，就从session中拿；
3. 如果都不满足就利用反射创建对象，将请求中的数据绑定到这个对象中。



确定方法每个参数的值：

1. 标注解：保存注解的信息；最终得到这个注解应该对应解析的值
2. 没标注解：
   1）看类型是否原生API
   2）看是否Model或Map
   3）看是否是简单类型（Integer等基本类型），是的话赋值paramName=“”
   4）给attrName赋值，attrName=“”（参数标了@ModelAttribute("")就是指定的，没标就是“”）;之后用POJO确定值的三步



### SpringMVC视图解析

1. 方法执行后的返回值会作为页面地址参考，转发或重定向到页面
2. 视图解析器可能会进行页面地址的拼串



1. 任何方法的返回值，最终都会被包装成mv对象
2. 

**细节4**  	this.processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);

```java
/**
	 * Handle the result of handler selection and handler invocation, which is
	 * either a ModelAndView or an Exception to be resolved to a ModelAndView.
	 */
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
                                   HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {

    boolean errorView = false;

    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();
        }
        else {
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            mv = processHandlerException(request, response, handler, exception);
            errorView = (mv != null);
        }
    }

    // Did the handler return a view to render?
    if (mv != null && !mv.wasCleared()) {
        //真正渲染的方法
        render(mv, request, response);
        if (errorView) {
            WebUtils.clearErrorRequestAttributes(request);
        }
    }
    else {
        if (logger.isDebugEnabled()) {
            logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +
                         "': assuming HandlerAdapter completed request handling");
        }
    }

    if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
        // Concurrent handling started during a forward
        return;
    }

    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(request, response, null);
    }
}
```

```java
/**
	 * Render the given ModelAndView.
	 * <p>This is the last stage in handling a request. It may involve resolving the view by name.
	 * @param mv the ModelAndView to render
	 * @param request current HTTP servlet request
	 * @param response current HTTP servlet response
	 * @throws ServletException if view is missing or cannot be resolved
	 * @throws Exception if there's a problem rendering the view
	 */
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
    // Determine locale for request and apply it to the response.
    Locale locale = this.localeResolver.resolveLocale(request);
    response.setLocale(locale);

    View view;
    if (mv.isReference()) {
        // We need to resolve the view name.
        // 1、根据viewName得到View对象
        // 方法通过里面的九大组件之视图解析器列表找到视图解析器，配置的是InternalSourceViewResolver
        // View可能得到RedirectView或InternaResourcelView
        view = resolveViewName(mv.getViewName(), mv.getModelInternal(), locale, request);
        if (view == null) {
            throw new ServletException(
                "Could not resolve view with name '" + mv.getViewName() + "' in servlet with name '" +
                getServletName() + "'");
        }
    }
    else {
        // No need to lookup: the ModelAndView object contains the actual View object.
        view = mv.getView();
        if (view == null) {
            throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
                                       "View object in servlet with name '" + getServletName() + "'");
        }
    }

    // Delegate to the View object for rendering.
    if (logger.isDebugEnabled()) {
        logger.debug("Rendering view [" + view + "] in DispatcherServlet with name '" + getServletName() + "'");
    }
    try {
        // 2、调用view对象的render方法
        // mv.getModelInternal()返回的是mv中保存的隐含模型的Map
        view.render(mv.getModelInternal(), request, response);
    }
    catch (Exception ex) {
        if (logger.isDebugEnabled()) {
            logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '"
                         + getServletName() + "'", ex);
        }
        throw ex;
    }
}
```

```java
// 上面的1
protected View resolveViewName(String viewName, Map<String, Object> model, Locale locale,
                               HttpServletRequest request) throws Exception {

    // 根据视图名，用一个视图解析器解析，如果解析出来为null，则换下一个解析器解析
    for (ViewResolver viewResolver : this.viewResolvers) {
        View view = viewResolver.resolveViewName(viewName, locale);
        if (view != null) {
            return view;
        }
    }
    return null;
}
```

**视图解析器只是为了得到视图对象View，视图对象才能真正转发或重定向到页面。**

