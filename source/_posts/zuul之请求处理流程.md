title: zuul之请求处理流程
author: Mr.G
date: 2018-11-23 10:26:26
tags:
---
#### 目录
 - [zuul之请求处理流程](https://guofazhan.github.io/2018/11/23/zuul%E4%B9%8B%E8%AF%B7%E6%B1%82%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B/)
 -  [zuul之动态过滤器Filter类文件管理加载](https://guofazhan.github.io/2018/11/23/zuul%E4%B9%8B%E5%8A%A8%E6%80%81%E8%BF%87%E6%BB%A4%E5%99%A8Filter%E7%B1%BB%E6%96%87%E4%BB%B6%E7%AE%A1%E7%90%86%E5%8A%A0%E8%BD%BD/)
 -  [zuul之过滤器Filter管理](https://guofazhan.github.io/2018/11/23/zuul%E4%B9%8B%E8%BF%87%E6%BB%A4%E5%99%A8Filter%E7%AE%A1%E7%90%86/)

#### 概述 

> zuul 是netflix开源的一个API Gateway 服务器, 本质上是一个web servlet应用，下面通过源码分析zuul的请求处理流程
<!--more -->
##### Zuul请求的生命周期
> 如图，该图详细描述了各种类型的过滤器的执行顺序

![image](https://github.com/guofazhan/image/blob/master/zuul.png?raw=true)


> 通过上图可以看到zuul通过拦截servlet应用http请求后进入zuul定义的过滤器链表进行处理路由；

> 一般我们开发web应用需要在请求进来前进行统一处理,通常有两种方式
> 1. 定义一个servlet Filter来在请求进来前拦截处理，例如会话校验，请求header校验等
> 2. 定义一个统一servlet，所有请求进来都走此servlet，一般mvc框架通过此方式实现，例如springMvc的DispatcherServlet

> zuul也不例外，zuul分别实现了这两种拦截方式: ZuulServletFilter与ZuulServlet。

#### ZuulServletFilter
> 所在包:com.netflix.zuul.filters

```java

public class ZuulServletFilter implements Filter {

    //zuul核心运行器
    private ZuulRunner zuulRunner;

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

        String bufferReqsStr = filterConfig.getInitParameter("buffer-requests");
        boolean bufferReqs = bufferReqsStr != null && bufferReqsStr.equals("true") ? true : false;

        //创建
        zuulRunner = new ZuulRunner(bufferReqs);
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        try {
            //初始化当前请求zuul的上下问环境信息
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);
            //执行前置过滤器列表
            try {
                preRouting();
            } catch (ZuulException e) {
                //执行错误过滤器列表
                //执行后置路由过滤器列表
                error(e);
                postRouting();
                return;
            }
            
            //判断响应是否已经发送，发送完成时不需要执行下面路由列表流程
            // Only forward onto to the chain if a zuul response is not being sent
            if (!RequestContext.getCurrentContext().sendZuulResponse()) {
                filterChain.doFilter(servletRequest, servletResponse);
                return;
            }
            
            //执行路由中过滤器列表
            try {
                routing();
            } catch (ZuulException e) {
                //执行错误过滤器列表
                //执行后置路由过滤器列表
                error(e);
                postRouting();
                return;
            }
            //执行后置路由过滤器列表
            try {
                postRouting();
            } catch (ZuulException e) {
                //执行错误过滤器列表
                error(e);
                return;
            }
        } catch (Throwable e) {
            //执行错误过滤器列表
            error(new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_FROM_FILTER_" + e.getClass().getName()));
        } finally {
            RequestContext.getCurrentContext().unset();
        }
    }

    void postRouting() throws ZuulException {
        //zuul核心运行器执行后置过滤器列表
        zuulRunner.postRoute();
    }

    void routing() throws ZuulException {
        zuulRunner.route();
    }

    void preRouting() throws ZuulException {
        zuulRunner.preRoute();
    }

    void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {
        zuulRunner.init(servletRequest, servletResponse);
    }

    void error(ZuulException e) {
       //设置异常的上下文环境中
        RequestContext.getCurrentContext().setThrowable(e);
        zuulRunner.error();
    }
}

```
#### ZuulServlet
> 所在包：com.netflix.zuul.http

```java
public class ZuulServlet extends HttpServlet {

    private static final long serialVersionUID = -3374242278843351500L;
    private ZuulRunner zuulRunner;


    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);

        String bufferReqsStr = config.getInitParameter("buffer-requests");
        boolean bufferReqs = bufferReqsStr != null && bufferReqsStr.equals("true") ? true : false;

        zuulRunner = new ZuulRunner(bufferReqs);
    }

    @Override
    public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
        try {
            init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);

            // Marks this request as having passed through the "Zuul engine", as opposed to servlets
            // explicitly bound in web.xml, for which requests will not have the same data attached
            RequestContext context = RequestContext.getCurrentContext();
            context.setZuulEngineRan();

            try {
                preRoute();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                route();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                postRoute();
            } catch (ZuulException e) {
                error(e);
                return;
            }

        } catch (Throwable e) {
            error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
        } finally {
            RequestContext.getCurrentContext().unset();
        }
    }

    /**
     * executes "post" ZuulFilters
     *
     * @throws ZuulException
     */
    void postRoute() throws ZuulException {
        zuulRunner.postRoute();
    }

    /**
     * executes "route" filters
     *
     * @throws ZuulException
     */
    void route() throws ZuulException {
        zuulRunner.route();
    }

    /**
     * executes "pre" filters
     *
     * @throws ZuulException
     */
    void preRoute() throws ZuulException {
        zuulRunner.preRoute();
    }

    /**
     * initializes request
     *
     * @param servletRequest
     * @param servletResponse
     */
    void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {
        zuulRunner.init(servletRequest, servletResponse);
    }

    /**
     * sets error context info and executes "error" filters
     *
     * @param e
     */
    void error(ZuulException e) {
        RequestContext.getCurrentContext().setThrowable(e);
        zuulRunner.error();
    }
}
```
> ZuulServlet与ZuulServletFilter完成了相同的功能
> 1. 初始化时创建ZuulRunner
> 2. 请求进入调用ZuulRunner.init初始化当前请求的上下文环境
> 3. 按规则执行zuul的过滤器列表:
> - 正常情况 :preRoute()->route()->postRoute()
> - preRoute异常：preRoute()->error()->postRoute()
> - route异常：route()->error()->postRoute()
> - postRoute异常：postRoute()->error()


####  ZuulRunner
>ZuulRunner将servlet请求和响应初始化到请求上下文中，并封装过滤器处理器调用preRoute()、Router（）、postRoute（）和Error（）方法


```java
public class ZuulRunner {

    private boolean bufferRequests;

    /**
     * Creates a new <code>ZuulRunner</code> instance.
     */
    public ZuulRunner() {
        this.bufferRequests = true;
    }

    /**
     *
     * @param bufferRequests - whether to wrap the ServletRequest in HttpServletRequestWrapper and buffer the body.
     */
    public ZuulRunner(boolean bufferRequests) {
        this.bufferRequests = bufferRequests;
    }

    /**
     * sets HttpServlet request and HttpResponse
     *
     * @param servletRequest
     * @param servletResponse
     */
    public void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {

        RequestContext ctx = RequestContext.getCurrentContext();
        if (bufferRequests) {
            ctx.setRequest(new HttpServletRequestWrapper(servletRequest));
        } else {
            ctx.setRequest(servletRequest);
        }

        ctx.setResponse(new HttpServletResponseWrapper(servletResponse));
    }

    /**
     * executes "post" filterType  ZuulFilters
     *
     * @throws ZuulException
     */
    public void postRoute() throws ZuulException {
        FilterProcessor.getInstance().postRoute();
    }

    /**
     * executes "route" filterType  ZuulFilters
     *
     * @throws ZuulException
     */
    public void route() throws ZuulException {
        FilterProcessor.getInstance().route();
    }

    /**
     * executes "pre" filterType  ZuulFilters
     *
     * @throws ZuulException
     */
    public void preRoute() throws ZuulException {
        FilterProcessor.getInstance().preRoute();
    }

    /**
     * executes "error" filterType  ZuulFilters
     */
    public void error() {
        FilterProcessor.getInstance().error();
    }
}
```
> ZuulRunner 是提供给请求的核心处理类，主要完成下面事件
> 1. 初始化request，response到上下问环境
> 2. 将 FilterProcessor的过滤器调用暴露出去

#### FilterProcessor
> FilterProcessor是执行过滤器的核心类

```java
public class FilterProcessor {

    static FilterProcessor INSTANCE = new FilterProcessor();
    protected static final Logger logger = LoggerFactory.getLogger(FilterProcessor.class);

    private FilterUsageNotifier usageNotifier;


    public FilterProcessor() {
        usageNotifier = new BasicFilterUsageNotifier();
    }

    /**
     * @return the singleton FilterProcessor
     */
    public static FilterProcessor getInstance() {
        return INSTANCE;
    }

    /**
     * sets a singleton processor in case of a need to override default behavior
     *
     * @param processor
     */
    public static void setProcessor(FilterProcessor processor) {
        INSTANCE = processor;
    }

    /**
     * Override the default filter usage notification impl.
     *
     * @param notifier
     */
    public void setFilterUsageNotifier(FilterUsageNotifier notifier) {
        this.usageNotifier = notifier;
    }

    /**
     * 按顺序执行类型为post的过滤器列表（后置过滤器）
     * runs "post" filters which are called after "route" filters. ZuulExceptions from ZuulFilters are thrown.
     * Any other Throwables are caught and a ZuulException is thrown out with a 500 status code
     *
     * @throws ZuulException
     */
    public void postRoute() throws ZuulException {
        try {
            runFilters("post");
        } catch (ZuulException e) {
            throw e;
        } catch (Throwable e) {
            throw new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_IN_POST_FILTER_" + e.getClass().getName());
        }
    }

    /**
     * 按顺序执行类型为error的过滤器列表（错误过滤器）
     * runs all "error" filters. These are called only if an exception occurs. Exceptions from this are swallowed and logged so as not to bubble up.
     */
    public void error() {
        try {
            runFilters("error");
        } catch (Throwable e) {
            logger.error(e.getMessage(), e);
        }
    }

    /**
     * 按顺序执行类型为route的过滤器列表（路由过滤器）
     * Runs all "route" filters. These filters route calls to an origin.
     *
     * @throws ZuulException if an exception occurs.
     */
    public void route() throws ZuulException {
        try {
            runFilters("route");
        } catch (ZuulException e) {
            throw e;
        } catch (Throwable e) {
            throw new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_IN_ROUTE_FILTER_" + e.getClass().getName());
        }
    }

    /**
     * 按顺序执行类型为pre的过滤器列表（前置过滤器）
     * runs all "pre" filters. These filters are run before routing to the orgin.
     *
     * @throws ZuulException
     */
    public void preRoute() throws ZuulException {
        try {
            runFilters("pre");
        } catch (ZuulException e) {
            throw e;
        } catch (Throwable e) {
            throw new ZuulException(e, 500, "UNCAUGHT_EXCEPTION_IN_PRE_FILTER_" + e.getClass().getName());
        }
    }

    /**
     * 按顺序执行给定类型的过滤器列表
     * 通过FilterLoader加载所有给定类型type的过滤器
     * 循环遍历过滤器列表依次执行
     * runs all filters of the filterType sType/ Use this method within filters to run custom filters by type
     *
     * @param sType the filterType.
     * @return
     * @throws Throwable throws up an arbitrary exception
     */
    public Object runFilters(String sType) throws Throwable {
        if (RequestContext.getCurrentContext().debugRouting()) {
            Debug.addRoutingDebug("Invoking {" + sType + "} type filters");
        }
        boolean bResult = false;
        List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
        if (list != null) {
            for (int i = 0; i < list.size(); i++) {
                ZuulFilter zuulFilter = list.get(i);
                Object result = processZuulFilter(zuulFilter);
                if (result != null && result instanceof Boolean) {
                    bResult |= ((Boolean) result);
                }
            }
        }
        return bResult;
    }

    /**
     * Processes an individual ZuulFilter. This method adds Debug information. Any uncaught Thowables are caught by this method and converted to a ZuulException with a 500 status code.
     *
     * @param filter
     * @return the return value for that filter
     * @throws ZuulException
     */
    public Object processZuulFilter(ZuulFilter filter) throws ZuulException {

        RequestContext ctx = RequestContext.getCurrentContext();
        boolean bDebug = ctx.debugRouting();
        final String metricPrefix = "zuul.filter-";
        long execTime = 0;
        String filterName = "";
        try {
            long ltime = System.currentTimeMillis();
            filterName = filter.getClass().getSimpleName();
            
            RequestContext copy = null;
            Object o = null;
            Throwable t = null;

            if (bDebug) {
                Debug.addRoutingDebug("Filter " + filter.filterType() + " " + filter.filterOrder() + " " + filterName);
                copy = ctx.copy();
            }
            
            ZuulFilterResult result = filter.runFilter();
            ExecutionStatus s = result.getStatus();
            execTime = System.currentTimeMillis() - ltime;

            switch (s) {
                case FAILED:
                    t = result.getException();
                    ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
                    break;
                case SUCCESS:
                    o = result.getResult();
                    ctx.addFilterExecutionSummary(filterName, ExecutionStatus.SUCCESS.name(), execTime);
                    if (bDebug) {
                        Debug.addRoutingDebug("Filter {" + filterName + " TYPE:" + filter.filterType() + " ORDER:" + filter.filterOrder() + "} Execution time = " + execTime + "ms");
                        Debug.compareContextState(filterName, copy);
                    }
                    break;
                default:
                    break;
            }
            
            if (t != null) throw t;

            usageNotifier.notify(filter, s);
            return o;

        } catch (Throwable e) {
            if (bDebug) {
                Debug.addRoutingDebug("Running Filter failed " + filterName + " type:" + filter.filterType() + " order:" + filter.filterOrder() + " " + e.getMessage());
            }
            usageNotifier.notify(filter, ExecutionStatus.FAILED);
            if (e instanceof ZuulException) {
                throw (ZuulException) e;
            } else {
                ZuulException ex = new ZuulException(e, "Filter threw Exception", 500, filter.filterType() + ":" + filterName);
                ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
                throw ex;
            }
        }
    }

    /**
     * Publishes a counter metric for each filter on each use.
     */
    public static class BasicFilterUsageNotifier implements FilterUsageNotifier {
        private static final String METRIC_PREFIX = "zuul.filter-";

        @Override
        public void notify(ZuulFilter filter, ExecutionStatus status) {
            DynamicCounter.increment(METRIC_PREFIX + filter.getClass().getSimpleName(), "status", status.name(), "filtertype", filter.filterType());
        }
    }
}
```
> FilterProcessor 通过FilterLoader加载给定类型的过滤器列表，然后按顺序依次执行过滤器，根据过滤器的类型包含如下方法：
> - preRoute：按顺序执行类型为pre的过滤器列表（前置过滤器）
> - route:按顺序执行类型为route的过滤器列表（路由过滤器）
> - postRoute:按顺序执行类型为post的过滤器列表（后置过滤器）
> - error:按顺序执行类型为error的过滤器列表（错误过滤器）

> 到此，zuul拦截servlet应用http请求后通过过滤器列表进行处理路由的整个流程以及代码都查看完毕，对后期我们根据业务改造还是自定义过滤器有很大的指导意义。

---

**备注** 过滤器分类：
>- pre：前置过滤器
>- route：路由过滤器
>- post：后置过滤器
>- error：错误过滤器