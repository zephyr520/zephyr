## SpringMVC请求处理流程 ##
参考地址：
[https://jinnianshilongnian.iteye.com/blog/1594806](https://jinnianshilongnian.iteye.com/blog/1594806)

[http://objcoding.com/2017/06/15/SpringMVC-processing-request-process/](http://objcoding.com/2017/06/15/SpringMVC-processing-request-process/)

SpringMVC流程图：
![SpringMVC请求流程图](http://sishuok.com/forum/upload/2012/7/14/57ea9e7edeebd5ee2ec0cf27313c5fb6__2.JPG)

核心架构的具体流程步骤如下：

1、  首先用户发送请求——>DispatcherServlet，前端控制器收到请求后自己不进行处理，而是委托给其他的解析器进行处理，作为统一访问点，进行全局的流程控制；

2、  DispatcherServlet——>HandlerMapping， HandlerMapping将会把请求映射为HandlerExecutionChain对象（包含一个Handler处理器（页面控制器）对象、多个HandlerInterceptor拦截器）对象，通过这种策略模式，很容易添加新的映射策略；

3、  DispatcherServlet——>HandlerAdapter，HandlerAdapter将会把处理器包装为适配器，从而支持多种类型的处理器，即适配器设计模式的应用，从而很容易支持很多类型的处理器；

4、  HandlerAdapter——>处理器功能处理方法的调用，HandlerAdapter将会根据适配的结果调用真正的处理器的功能处理方法，完成功能处理；并返回一个ModelAndView对象（包含模型数据、逻辑视图名）；

5、  ModelAndView的逻辑视图名——> ViewResolver， ViewResolver将把逻辑视图名解析为具体的View，通过这种策略模式，很容易更换其他视图技术；

6、  View——>渲染，View会根据传进来的Model模型数据进行渲染，此处的Model实际是一个Map数据结构，因此很容易支持其他视图技术；

7、返回控制权给DispatcherServlet，由DispatcherServlet返回响应给用户，到此一个流程结束。

### FrameworkServlet如何处理用户请求 ###
```
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
	processRequest(request, response);
}

protected final void doPost(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
	processRequest(request, response);
}
```

```
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

	Throwable failureCause = null;
	// 获取当前线程的LocaleContext
	LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
	LocaleContext localeContext = buildLocaleContext(request);
	// 获取当前线程的RequestAttributes
	RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
	ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
	asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());
	// 初始化ContextHolder
	initContextHolders(request, localeContext, requestAttributes);

	try {
		// 处理用户的请求，抽象方法，交给子类去实现
		doService(request, response);
	}
	// 此处代码胜率

	finally {
		resetContextHolders(request, previousLocaleContext, previousAttributes);
		if (requestAttributes != null) {
			requestAttributes.requestCompleted();
		}
		// 此处代码省略
		publishRequestHandledEvent(request, response, startTime, failureCause);
	}
}
```
```
protected abstract void doService(HttpServletRequest request, HttpServletResponse response) 
				throws Exception;
```

### DispatchServlet处理请求 ###
```
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
	// 此处代码省略

	try {
		// 负责请求真正的处理
		doDispatch(request, response);
	}
	finally {
		if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Restore the original attribute snapshot, in case of an include.
			if (attributesSnapshot != null) {
				restoreAttributesAfterInclude(request, attributesSnapshot);
			}
		}
	}
}
```
```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				//检查是否是请求是否是multipart（如文件上传），如果是将通过MultipartResolver解析
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);
				
				// 将请求通过HandlerMapping进行映射，并包装成一个处理器链(HandlerExecutionChain)
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null || mappedHandler.getHandler() == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// 处理器适配，将处理器包装成相应的适配器(让其支持多种类型的处理器)
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (logger.isDebugEnabled()) {
						logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
					}
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}
				// 执行处理器相关拦截器的预处理操作
				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// 由适配器执行处理器
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}
				
				applyDefaultViewName(processedRequest, mv);
				// 执行处理器的相关拦截器的后置处理
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			// 解析视图并进行视图的渲染 
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```