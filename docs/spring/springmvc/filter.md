# Filter

Spring MVC提供了一些有用的过滤器, 分别是:

+ Form Data

+ Forwarded Headers

+ Shallow ETag

+ CORS

## Form Data

浏览器只能通过 HTTPGET 或 HTTPPOST 提交表单数据，但非浏览器客户端也可以使用 HTTPPUT、 PATCH 和 DELETE。Servlet API 要求 ServletRequest.getParameter*()方法仅支持 HTTP POST 的表单字段访问。

Spring-web 模块提供 `FormContentFilter` 来拦截 HTTP PUT、 PATCH 和 DELETE 请求，其内容类型为 application/x-www-form-urlencode，从请求体中读取表单数据，并包装 ServletRequest 以使表单数据通过 ServletRequest.getParameter*()系列方法可用。

## Forwarded Headers

当一个请求通过代理（如负载均衡器）时，主机、端口和协议可能会发生变化，这从客户端的角度来看，创建一个指向正确的主机、端口和协议的链接是一个挑战。

RFC 7239定义了 `Forwarded` HTTP头，代理可以使用该头来提供有关原始请求的信息。还有其他非标准头，包括X-Forwarded-Host、X-Forwarded-Port、X-Forwarded-Proto、X-Forwarded-Ssl和X-Forwarded-Prefix。

`ForwardedHeaderFilter` 是一个 Servlet 过滤器，它修改请求的目的是: a)根据 ForwardedHeader 更改主机、端口和方案; b)删除这些 Header 以消除进一步的影响。过滤器依赖于包装请求，因此必须在其他过滤器(如 RequestContextFilter)之前排序，这些过滤器应该处理修改后的请求，而不是原始请求。

由于应用程序无法知道转发的头是由代理添加的，还是由恶意客户机添加的，因此需要考虑安全问题。这就是为什么应该将信任边界上的代理配置为删除来自外部的不受信任的转发头。还可以使用 `RemoveOnly = true` 配置 ForwardedHeaderFilter，在这种情况下，它会删除但不使用标头。

<font style="color: red">为了支持异步请求和错误调度，此过滤器应与`DispatcherType.ASYNC`和`DispatcherType.ERROR`一起映射。如果使用Spring Framework的AbstractAnnotationConfigDispatcherServletInitializer（请参阅Servlet配置），则所有过滤器都会自动为所有调度类型注册。但是，如果通过web.xml或在Spring Boot中通过FilterRegistrationBean注册过滤器，请确保除了DispatcherType.REQUEST外，还包括DispatcherType.ASYNC和DispatcherType.ERROR。</font>

## Shallow ETag

`ShallowEtagHeaderFilter` 过滤器通过缓存写入响应的内容并从中计算 MD5散列来创建“浅”ETag。下一次客户端发送时，它会执行相同的操作，但是它也会将计算值与 If-none-Match 请求头进行比较，如果两者相等，则返回一个304(NOT_MODIFIED)。

这种策略可以节省网络带宽，但不能节省 CPU，因为必须为每个请求计算完整的响应。前面描述的控制器级别的其他策略可以避免计算。请参见 HTTP 缓存。

这个过滤器有一个 `writeWeakETag` 参数，该参数将过滤器配置为写入类似于以下内容的弱 ETtag: W/“02a2d595e6ed9a0b24f027f2b63b134d6”(如 RFC 7232 Section 2.3所定义的)。

为了支持异步请求，这个过滤器必须与 DispatcherType.ASYNC 映射，以便过滤器能够延迟并成功地生成 ETag 到最后一次异步分派的末尾。如果使用 Spring 框架的 AbstractAnnotationConfigDispatcherServletInitializer (参见 Servlet 配置) ，则所有过滤器都会自动注册到所有分派类型。但是，如果通过 web.xml 注册过滤器，或者通过 FilterRegistrationBean 在 Spring Boot 中注册过滤器，请确保包含 DispatcherType.ASYNC。

## CORS

Spring MVC 通过控制器上的注解提供了对 CORS 配置的细粒度支持。但是，当与 Spring Security 一起使用时，我们建议使用内置的 CorsFilter，它的顺序必须在 Spring Security 的过滤器链之前。