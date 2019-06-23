# 1、执行DispatcherServlet的doService方法
  1、存储applicationContext（以及localeResolver国际化、themeResolver页面主题、flashMapManager页面重定向缓存等现在很少用的属性）到request中，方便获取。  
  2、doDispatch路由
 
  
# 2、doDispatch
  - 1、获取异步任务管理器WebAsyncManager。  
  ```java
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
```  
   如果由异步任务，在主任务执行完`mv = ha.handle(processedRequest, response, mappedHandler.getHandler())`后，会再次走到doDispatch。  
   如果没有异步任务，处理返回结果。  
   ```java
        if (asyncManager.isConcurrentHandlingStarted()) {
            return;
        }
```   
   - 2、检查是不是multipartRequest（例如上传文件等）,如果是，那么进行处理任务的HttpServletRequest替换成对应的MultipartRequest。  
   ```java
    processedRequest = checkMultipart(request);
```  
  对应mutipartRequest，请求处理完成后会清理掉资源，例如删除上传的文件。
  ```java
    if (multipartRequestParsed) {
        cleanupMultipart(processedRequest);
    }
```  
  - 3、获取请求对应的Handler（也就是Controller,存放在handlerMappings变量中）,`mappedHandler = getHandler(processedRequest);`  
  - 4、获取顶球对应的HandlerAdapter  
 ```java
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());  
```        
   - 5、处理last-modified请求头，如果最后修改时间没有变，请求结束。  
   - 6、处理拦截器中的`HandlerInterceptor`类中的preHandle方法，如果preHandle方法返回false，那么请求结束。
   ```java
    if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
    }
```  
   - 7、执行Controller方法的主体  
   ```java
    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```  
   - 8、设置返回的视图。Spring历史版本中的ModelAndView  
   ```java
     applyDefaultViewName(processedRequest, mv);
```  
   - 9、执行拦截器的postHandle方法，类似第6步骤。  
   ```java
    mappedHandler.applyPostHandle(processedRequest, response, mv);
```  
   - 10、处理返回结果，并进行试图渲染等。processDispatchResult。  
     现在一般直接返回json对象，不再返回view了。  
       
# 3、具体分析第二步中每个步骤的执行过程
   - 1、核心的方法执行过程。（第7步）  
      - 1、执行RequestMappingHandlerAdapter（假设第二步中的第4步获取的是RequestMappingHandlerAdapter）的handleInternal方法  
	       - 1、检查supportedMethods（方法Post、Get方法等是否执行），以及requireSession（是否需要session）。（WebContentGenerator中的变量）  
		   - 2、执行invokeHandlerMethod方法，执行前检查session是否需要锁住（synchronizeOnSession变量控制），因为多个请求处理同一个session时，session并不线程安全。  
		   - 3、处理缓存。如果方法没有设置Cache-control，检查注解SessionAttributes，如果有注解，设置缓存过期时间
		   ```java
              if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
					if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
						applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
					}
					else {
						prepareResponse(response);
					}
				}
        ```    
     - 2、invokeHandlerMethod执行过程 
        - 1、生成InitBinder注解的处理对象WebDataBinderFactory。获取Controller里的InitBinder注解方法以及@ControllerAdvice里被InitBinder注释的方法，生成WebDataBinderFactory。  
        生成SessionAttribute、ModelAttribute注解（包含ControllerAdvice中的ModelAttribute注解）的处理对象SessionAttributesHandler。（存放在sessionAttributesHandlerCache、modelAttributeCache变量中）。  
        ```java
         WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
         ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
        ```  
        - 2、创建invocableMethod，该对象会执行请求体。首先初始化该对象，设置参数解析器、返回值解析器、步骤1中生成的initBinder处理对象WebDataBinderFactory、参数名称转换的parameterNameDiscoverer。   
        ```java
       ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
				if (this.argumentResolvers != null) {
					invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
				}
				if (this.returnValueHandlers != null) {
					invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
				}
				invocableMethod.setDataBinderFactory(binderFactory);
			 	invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
        ```   
        参数解析器：请求数据中，请求头、body等有各种各样的数据，这些数据的解析放在参数argumentResolvers（类型HandlerMethodArgumentResolverComposite，在类RequestMappingHandlerAdapter中）中，该参数在spring启动时初始化，见方法RequestMappingHandlerAdapter.getDefaultArgumentResolvers。也可以继承HandlerMethodArgumentResolver自定义一个解析器。   
        返回值解析器：类似参数解析器，getDefaultReturnValueHandlers，其值均存放在变量returnValueHandlers中，也可以通过继承HandlerMethodReturnValueHandler自定返回值解析器。   
        - 3、创建ModelAndViewContainer对象，字面意思是Model和View的容器，主要职责是维护model和view，记录HandlerMethodArgumentResolver 和 HandlerMethodReturnValueHandler（参数和返回值解析器）执行handler时，model和view的变化。 
        ```java
            ModelAndViewContainer mavContainer = new ModelAndViewContainer();
            mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
            modelFactory.initModel(webRequest, mavContainer, invocableMethod);
            mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
         ```  
        - 4、处理异步请求。（待续。。）  
     	- 5、执行请求方法。参数为request和ModelAndViewContainer对象。（待续）
     		```java
                invocableMethod.invokeAndHandle(webRequest, mavContainer);
            ```
     		- 1、 获取参数
     		- 2、 执行方法
     		- 3、 处理返回值	
     	- 6、返回被请求方法处理过的ModelAndView。  
     	```java
          return getModelAndView(mavContainer, modelFactory, webRequest);
        ```
     					