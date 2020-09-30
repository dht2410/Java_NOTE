## SpringMVC

#### 前端控制器DispatcherServlet

> 继承FramworkServlet <- HttpServletBean <-HttpServlet
>
> 请求一进来来到HttpServlet的doGet或doPost方法
>
> doGet	doPost都在FramworkServlet中被调用，processRequest(HttpServletRequest request, HttpServletResponse response), 链式调用doService(request, response), 链式调用doDispatch(request, response)

##### 处理流程

1. 所有请求过来DispatchServlet来接收
2. 调用doDispatch方法执行

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
            //3、如果没有，则抛异常
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

SpringMVC九大组件（全是接口，接口就是规范），关键位置都是由组件完成：

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
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    
    //重要！！！创建BindingAwareModelMap
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

确定POJO值的三步：

1. 如果隐含模型中有这个key（标了ModelAttribute注解的就是注解指定的value，没标就是参数类型的首字母小写）指定的值，就把这个值赋值给bindObject；
2. 如果是SessionAttributes标注的属性，就从session中拿；
3. 如果都不满足就利用反射创建对象。



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

