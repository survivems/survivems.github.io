# 概述

Spring MVC，全称Spring Web MVC，是一种基于ServletAPI的轻量级Web框架，实现了MVC（Model-View-Controller）设计模型。它是Spring框架的一部分，特别是作为Spring Framework后续产品的一部分。

Spring MVC的特点包括清晰的角色划分、灵活的配置功能、提供了大量的控制器接口和实现类，以及真正做到了与View层的实现无关（如JSP、Velocity、Xslt等）。此外，它还支持国际化，并采用了面向接口编程的方式。Spring MVC简化了Web开发，提供了Web应用开发的一整套流程，使得开发者可以更加高效地进行Web开发。

简而言之，Spring MVC是一种强大且灵活的Web框架，它基于MVC设计模式，通过将Web层进行解耦，使得开发者可以更加专注于业务逻辑的实现，提高了开发效率和代码质量。

## DispatchServlet

和其他web框架类似，Spring MVC围绕一个叫做 `DispatchServlet` 的Servlet的前端控制器进行设计，该Servlet为请求处理提供了一个共享的算法，而<font style="color: red">实际的工作则是由可配置的代理组件来完成的</font>。这种模型非常灵活，支持多种工作流程。

下面是基于 Java的配置注册和初始化 DispatcherServlet 的示例，它由 Servlet 容器自动检测:

```java
public class ServletConfig implements WebApplicationInitializer {

  @Override
  public void onStartup(ServletContext servletContext) throws ServletException {
    // 创建web上下文，可以选择 WebApplicationContext实现
    AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
    // 注册配置类，该配置类主要创建spring mvc组件，作用与 spring-web.xml类似
    context.register(SpringWebMvcConfigBean.class);
    // 创建 DispatchServlet前端控制器，用于监听用户请求
    DispatcherServlet dispatcherServlet = new DispatcherServlet(context);
    // 注册servlet（该特性由Servlet3.0及以后支持）
    ServletRegistration.Dynamic registration = servletContext.addServlet("dispatchServlet", dispatcherServlet);
    registration.setLoadOnStartup(1);
    registration.addMapping("/");
  }
}
```

`WebApplicationInitializer`是Spring提供的一个接口，用于配置Servlet 3.0以后的配置，<font style="color: red">其主要目的是替代传统的web.xml文件的作用</font>，实现这个接口的类会自动被SpringServletContainerInitializer获取到。

> 在Servlet3.0规范中，引入了一种新机制，称为Servlet容器初始化器（Servlet Container Initializer, SCI）。这种机制允许开发者通过Java SPI机制提供自定义的Servlet容器初始化器以替代传统的web.xml配置方式

**<font style="color: red">Spring MVC通过SPI机制自动发现WebApplicationInitializer</font>**

SPI机制是一种服务发现机制，Spring MVC实现该机制分为以下步骤：

1. Servlet容器提供了服务发现接口：`ServletContainerInitializer`，该接口由Spring MVC实现，该实现类是：`SpringServletContainerInitializer`，其声明如下：
```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {

	@Override
	public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

		List<WebApplicationInitializer> initializers = Collections.emptyList();

		if (webAppInitializerClasses != null) {
			initializers = new ArrayList<>(webAppInitializerClasses.size());
			for (Class<?> waiClass : webAppInitializerClasses) {
				// 如果类类型不是接口并且不是抽象类，同时与WebApplicationInitializer是相同类型，则注册并实例化
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						initializers.add((WebApplicationInitializer)
								ReflectionUtils.accessibleConstructor(waiClass).newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

        // 如果没有找到合适的 WebApplicationInitializer 给出提示并返回
		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");

        // 根据声明的顺序，依次调用 onStartup 方法进行启动初始化工作
		AnnotationAwareOrderComparator.sort(initializers);
		for (WebApplicationInitializer initializer : initializers) {
			initializer.onStartup(servletContext);
		}
	}
}
```

2. 在META-INF/services目录下注册服务实现：SPI机制通过扫描META-INF/services目录下的配置文件来发现服务实现。对于ServletContainerInitializer，有一个名为javax.servlet.ServletContainerInitializer的文件，文件中列出了所有ServletContainerInitializer实现类的全限定名。

> 注意到 `SpringServletContainerInitializer`接口上有一个注解`@HandlesTypes(WebApplicationInitializer.class)`声明，该注解由javax所提供，在Servlet容器启动时，会在类路径查找指定类型的实现类类型，上述接口中将找到的实现类类型传递给`webAppInitializerClasses`参数。存在多个类型可以通过Ordered接口声明初始化顺序。

下面是基于 web.xml的注册和配置 DispatcherServlet：

```xml
<web-app>

    <!--应用启动监听器-->
    <listener>
        <listener-class>
            org.springframework.web.context.ContextLoaderListener
        </listener-class>
    </listener>

    <!--应用配置文件路径（根容器）-->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/app-context.xml</param-value>
    </context-param>

    <!--DispatcherServlet前端控制器注册-->
    <servlet>
        <servlet-name>dispatcherServlet</servlet-name>
        <servlet-class>
            org.springframework.web.servlet.DispatcherServlet
        </servlet-class>
        <init-param>
            <!--与该DispatcherServlet控制器关联的spring 组件配置文件-->
            <param-name>contextConfigLocation</param-name>
            <!--
                必须配置这个参数，但是值可以不配置，如果不配置该参数，会有一个默认值，可能
                可能导致配置文件找不到
            -->
            <param-value></param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <!--前端控制器映射-->
    <servlet-mapping>
        <servlet-name>dispatcherServlet</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>

</web-app>
```

> 注意：上述基于web.xml的配置文件中，注册了名为`ContextLoaderListener`的监听器，它是一个`ServletContextListener`，其主要作用是在Web应用程序加载时启动Spring容器。具体来说，ContextLoaderListener在Web应用程序启动时负责创建ApplicationContext对象，并将其存储在ServletContext中。进而其他组件（如控制器、过滤器等）就可以通过ServletContext获取ApplicationContext，从而访问Spring的功能。<font style="color: red; font-weight: bolder">ContextLoaderListener还负责初始化和销毁ApplicationContext。它在Web应用程序启动时调用ApplicationContext的refresh()方法进行初始化。此外，ContextLoaderListener的主要作用还包括读取在contextConfigLocation中定义的xml文件，自动装配ApplicationContext的配置信息，并产生WebApplicationContext对象。然后，它将这个对象放置在ServletContext的属性里，以便其他组件可以通过ServletContext获取并使用这个对象，从而利用Spring容器管理的bean。ContextLoaderListener在Spring MVC中扮演着至关重要的角色，它确保了Spring容器的正确初始化和配置，使得其他组件能够顺利地访问和使用Spring的功能。</font>

> 感兴趣可以参考ContextLoaderListener父类 ContextLoader初始化源码！！！

### 应用上下文层次结构

DispatcherServlet需要一个特定的上下文环境（`WebApplicationContext`）来配置自己，WebApplicationContext是ApplicationContext的一个扩展，它除了拥有ApplicationContext的所有功能外，还与ServletContext和与其关联的Servlet有链接。WebApplicationContext被绑定到ServletContext，这意味着应用程序可以使用`RequestContextUtils`的静态方法来查找WebApplicationContext。例如，可以使用RequestContextUtils.findWebApplicationContext(request)来获取当前的WebApplicationContext。

> `RequestContextUtils`是Spring框架提供的一个工具类，它<font style="color: red">主要用于从HttpServletRequest上下文中获取特定的对象。这些对象可能包括`WebApplicationContext`、`LocaleResolver`、`Locale`、`ThemeResolver`、`Theme`以及`MultipartResolver`等。</font>更多参考javadoc文档。

<font style="color: red">对于多数应用程序来说，拥有一个单独的 WebApplicationContext 是足够的。此外，还可以配置一个有层次结构的上下文，其中一个根 WebApplicationContext 在多个 DispatcherServlet (或其他 Servlet)实例之间共享，每个实例都有自己的子 WebApplicationContext 配置。</font>

根 WebApplicationContext 通常包含基础设施 bean，例如需要跨多个 Servlet 实例共享的数据存储库（Dao）和业务服务(Service)。这些 bean 被有效地继承，并且可以在 Servlet 特定的子 WebApplicationContext 中重写(即重新声明) ，该子 WebApplicationContext 通常包含给定 Servlet 的本地 bean。下图显示了这种关系:

![Alt text](imgs/f61b808b88df49149931dd68d193f413.png)

> 特定的子容器获取不到，可以从根容器获取需要的组件（bean）

下面示例配置了一个有层次结构的上下文：

```java
public class MvcConfig extends AbstractAnnotationConfigDispatcherServletInitializer {

  // 配置根容器配置类，该配置类就是一个基于注解配置的根容器
  @Override
  protected Class<?>[] getRootConfigClasses() {
    return new Class[]{RootWebApplicationConfig.class};
  }

  // 配置子容器配置类，该配置类是一个基于注解配置的spring mvc配置容器
  @Override
  protected Class<?>[] getServletConfigClasses() {
    return new Class[]{DispatcherServletConfig.class};
  }

  // 配置DispatcherServlet映射
  @Override
  protected String[] getServletMappings() {
    return new String[]{"/"};
  }
}
```

> <font style="color: red">如果不需要层次结构上下文，可以将子容器配置类（getServletConfigClasses）返回null即可，只保留根容器！</font>

上面基于注解配置的层次接口上下文可以采用下面的方式基于xml配置：

```xml
<web-app>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <!--指定跟容器的spring配置文件，默认实现是 XmlWebApplicationContext-->
        <param-value>/WEB-INF/root-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>dispatcherServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <!--指定子容器的spring配置文件，默认实现是 XmlWebApplicationContext-->
            <param-value>/WEB-INF/dispatcherServlet.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <!--DispatcherServlet拦截路径-->
    <servlet-mapping>
        <servlet-name>dispatcherServlet</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```
> <font style="color: red">同样，如果不需要子容器，将DispatcherServlet的contextConfigLocation参数设置为空，不能省略!!!</font>

除了上述可以通过`AbstractAnnotationConfigDispatcherServletInitializer`自定义层次上下文，还可以继承其父类`AbstractDispatcherServletInitializer`配置层次上下文，如下示例：

```java
public class MvcConfigV2 extends AbstractDispatcherServletInitializer {

  // 配置子容器，创建WebApplicationContext类型容器
  @Override
  protected WebApplicationContext createServletApplicationContext() {
    XmlWebApplicationContext context = new XmlWebApplicationContext();
    // 在 spring配置文件中定义web组件
    context.setConfigLocation("/WEB-INF/dispatcherServlet.xml");
    return context;
  }

  // 配置DispatcherServlet映射
  @Override
  protected String[] getServletMappings() {
    return new String[]{"/"};
  }

  // 创建根容器，创建WebApplicationContext类型容器
  @Override
  protected WebApplicationContext createRootApplicationContext() {
    XmlWebApplicationContext context = new XmlWebApplicationContext();
    // 在spring配置文件中定义公共组件
    context.setConfigLocation("/WEB-INF/application.xml");
    return context;
  }
}
```

> <font style="color: red">注意，当只配置一个容器时，容器的类型必须是WebApplicationContext类型，因为web模块中的Controller(控制器)只有在web上下文才会被扫描。</font>，示例如下：

```java
/**
 * 只配置根容器，特定的DispatcherServlet子容器返回null
 */
public class MvcConfig extends AbstractAnnotationConfigDispatcherServletInitializer {

  @Override
  protected Class<?>[] getRootConfigClasses() {
    return new Class[]{RootWebApplicationConfig.class};
  }

  // 子容器返回null
  @Override
  protected Class<?>[] getServletConfigClasses() {
    return null;
  }

  @Override
  protected String[] getServletMappings() {
    return new String[]{"/"};
  }
}
```

```java
@ComponentScan("cn.chiatso.mvc")
@Configuration
@EnableWebMvc
// 由于只配置根容器，不会扫描Controller，因此实现接口 `WebMvcConfigurer` 并开启WebMvc配置
public class RootWebApplicationConfig implements WebMvcConfigurer {
}
```

### Spring MVC中特殊类型Bean

DispatcherServlet 委托“特殊的bean” 来处理请求并呈现适当的响应。这里的“特殊 bean”是指实现框架契约的 Spring 管理的 bean 实例，这些内置特殊bean可以通过配置自定义扩展。Spring MVC内置的特殊bean有以下：

+ HandlerMapping：<font style="color: red">在Spring MVC中，HandlerMapping的作用是将请求映射到处理器以及与之相关的拦截器列表中，具体因HandlerMapping实现不同有差异。其主要职责是解析Http请求，并将其映射到处理器（Controller定义的方法），该映射过程可以包括一些前置和后置处理的拦截器，用于在处理请求的不同阶段执行额外逻辑。</font>
> Spring MVC提供了2中HandlerMapping的实现，分别是：`RequestMappingHandlerMapping` 和 `SimpleUrlHandlerMapping`。它们之间的差异如下：
> 1. RequestMappingHandlerMapping：<font style="color: red">支持使用 `@RequestMapping`注解将请求映射到方法，基于注解自动发现和处理映射关系，无需在配置文件申明映射关系，并且它会自动检测带有 `Controller` 和 `RestController`注解类，并将其方法映射到响应的URL</font>，此外 RequestMappingHandlerMapping 还可以与接口 HandlerInterceptor 实现一起工作，以定义前置或后置拦截器用于处理请求不同阶段的额外逻辑。 
> 2. SimpleUrlHandlerMapping：<font style="color: red">用于显式地在配置中定义URL路径到处理器的映射关系，通常通过XML配置文件来定义映射规则，它不支持基于注解的自动映射</font>。SimpleUrlHandlerMapping适用于简单的URL映射场景，或者当开发者想要完全控制映射逻辑时。同样，它也可以与HandlerInterceptor接口的实现类一起使用，以定义前置和后置拦截器。
在配置HandlerMapping时，可以定义一系列拦截器，这些拦截器将在请求处理的不同阶段被调用。

+ HandlerAdapter：<font style="color: red">HandlerAdapter的主要职责是帮助DispatcherServlet调用与请求映射的处理器。</font>HandlerAdapter提供了一个通用的接口，使得DispatcherServlet不需要知道处理器是如何被实际调用的。这有助于解耦DispatcherServlet和处理器之间的关系，使得DispatcherServlet可以专注于请求的分发，而HandlerAdapter负责处理请求的实际执行。

+ HandlerExceptionResolver：<font style="color: red">异常解析器，主要负责识别异常，提供响应以及自定义异常处理。</font>Spring MVC提供的默认实现有 `ExceptionHandlerExceptionResolver` （用于处理使用@ExceptionHandler注解的方法）和 `ResponseStatusExceptionResolver`（用于处理带有 @ResponseStatus 注解的异常）和 `DefaultHandlerExceptionResolver`（用于处理标准的 Spring MVC 异常）

+ ViewResolver：<font style="color: red">视图解析器，负责将逻辑视图名称转换为具体的视图实现</font>，如JSP、Thymeleaf模板等。视图解析器通常通过实现ViewResolver接口来定义，该接口包含一个方法resolveViewName，它接收一个视图名称（通常是一个字符串），并返回一个View对象。Spring MVC提供了几个默认的视图解析器实现，如InternalResourceViewResolver（用于解析JSP视图）和ThymeleafViewResolver（用于解析Thymeleaf模板）。

+ LocaleResolver, LocaleContextResolver：在Spring MVC中，<font style="color: red">LocaleResolver 和 LocaleContextResolver 接口都与国际化（i18n）和本地化（l10n）相关，它们负责确定用户的区域设置（locale），以便在Web应用程序中提供适当的本地化内容</font>。
> LocaleResolver：<font style="color: red">它负责解析HTTP请求中的区域设置信息，并将这些信息存储在LocaleContextHolder中，以便在应用程序的后续请求处理过程中使用。LocaleResolver 的实现通常会在每次请求时根据请求头中的信息（如Accept-Language）来解析出用户的区域设置。</font>Spring MVC提供了以下LocaleResolver实现：
> + FixedLocaleResolver：使用固定的区域设置；
> + AcceptHeaderLocaleResolver：根据HTTP请求头的Accept-Language来解析区域设置；
> + CookieLocaleResolver：从客户端的cookie中读取区域设置信息；
> + SessionLocaleResolver：从用户的session中读取区域设置信息。
> 
> LocaleContextResolver：是Spring 3.1之后引入的一个更通用的接口，扩展了LocaleResolver接口，<font style="color: red">不仅解析Locale，还解析TimeZone等其他与区域相关的上下文信息。</font>
LocaleContextResolver的实现通常也会根据请求中的信息来解析区域上下文，并将其存储在LocaleContextHolder中。LocaleContextHolder是一个线程局部存储，它存储了当前线程（即当前请求）的区域上下文信息。LocaleContextResolver的一个常见实现是FixedLocaleContextResolver，它类似于FixedLocaleResolver，但提供了更多的上下文信息。

+ ThemeResolver：<font style="color: red">用于解析和管理用户的主题设置。</font>主题通常指的是应用程序的整体样式和风格，包括 CSS 样式、图片和其他静态资源。通过 ThemeResolver，Spring MVC 允许开发者根据用户的请求或偏好动态地切换应用程序的主题。
> 在 Spring MVC 中，主题通常与一组静态资源相关联，这些资源被打包在一个或多个主题属性文件（通常是 .properties 文件）中。这些属性文件包含了主题的名称以及该主题所使用的资源路径。ThemeResolver 的主要作用是根据请求的上下文信息（如用户的会话、请求参数等）来确定应该使用哪个主题。一旦确定了主题，ThemeResolver 就会将这些资源路径提供给 ThemeSource，后者负责加载和提供这些资源。Spring MVC 提供了几种内置的 ThemeResolver 实现：
> + FixedThemeResolver：始终使用固定的主题，不会根据请求上下文进行变化。
> + SessionThemeResolver：根据用户的会话信息来确定主题。通常，主题名称会作为会话属性存储。
> + CookieThemeResolver：根据用户的 Cookie 信息来确定主题。主题名称被存储在 Cookie 中。
> 主题可以通过实现 `ThemeResolver`接口自定义

+ MultipartResolver：<font style="color: red">用于处理文件上传的组件。它是Spring对文件上传处理流程在接口层次的抽象。当接收到类型为multipart/form-data的请求时，MultipartResolver会解析这个请求，将文件数据解析成MultipartFile对象，并将其封装在MultipartHttpServletRequest对象中。Spring MVC提供了内置实现`CommonsMultipartResolver。</font>

+ FlashMapManager：<font style="color: red">FlashMapManager是Spring MVC框架中的一个组件，用于管理FlashMap。</font>FlashMap主要用于在页面重定向（redirect）时传递参数。
> <font style="color: red">FlashMap就是一个特殊的Map，用于保存需要传递的数据，并且这些数据只在重定向过程中有效。一旦重定向完成，这些数据就会被自动删除。FlashMapManager的主要职责是创建、保存和检索FlashMap对象。当需要在重定向前设置一些信息，并在重定向后获取这些信息时，就可以使用FlashMapManager来实现。在重定向之前，将数据放入FlashMap中，并通过FlashMapManager将其保存。在重定向之后，再通过FlashMapManager从session中找到对应的FlashMap，从而获取到之前保存的数据。</font>需要注意的是，FlashMap中的数据是存储在session中的，因此在使用时需要谨慎处理，避免因为大量存储而导致session溢出。同时，由于FlashMap只在重定向过程中有效，所以在使用完毕后应该及时清理，以避免不必要的资源浪费。

### Web MVC配置

Web应用程序可以声明处理请求所需的“特殊的bean”组件用于处理请求，DispatcherServlet会检查WebApplication中每一个“特殊bean组件（前一节提到）”，如果没有找到，则会从`DispatcherServlet.properties`加载并创建组件处理请求。

在大多数情况下，MVC 配置是最好的起点。它用 Java 或 XML 声明所需的 bean，并提供一个更高级的配置回调 API 来定制它。

> SpringBoot 依赖于 MVCJava 配置来配置 SpringMVC，并提供了许多额外的便利选项。

### Servlet配置

在 Servlet 3.0及以后环境中，可以以编程方式配置 Servlet 容器，作为替代方案或与 web.xml 文件组合使用。下面的示例注册一个 DispatcherServlet:

```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    /** 
     * 依赖于 Servelt容器的SPI机制自动发现 SpringServletContainerInitializer，并将扫描到的
     * WebApplicationInitializer实现作为类对象创建调用其 onStartup方法
     */
    @Override
    public void onStartup(ServletContext container) {
        // 创建XmlWebApplicationContext
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        // spring web配置文件路径
        appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

        // 注册 DispatcherServlet以及映射
        ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
```

`WebApplicationInitializer` 是 Spring MVC 提供的一个接口，它确保检测到您的实现并自动用于初始化任何 Servlet3容器。WebApplicationInitializer 的抽象基类实现 `AbstractDispatcherServletInitializer` 通过覆盖指定 servlet 映射和 DispatcherServlet 配置位置的方法，使注册 DispatcherServlet 变得更加容易。对于使用基于 Java 的 Spring 配置的应用程序，建议如下配置:

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    // 返回根容器配置类
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    // 返回子容器配置类
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { MyWebConfig.class };
    }

    // 映射DispatcherServlet
    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

如果使用基于 XML 的 Spring 配置，应该直接从 AbstractDispatcherServletInitializer 进行扩展，如下面的示例所示:

```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    // 创建父容器
    @Override
    protected WebApplicationContext createRootApplicationContext() {
        return null;
    }

    // 创建子容器
    @Override
    protected WebApplicationContext createServletApplicationContext() {
        XmlWebApplicationContext cxt = new XmlWebApplicationContext();
        cxt.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
        return cxt;
    }

    // 映射DispatcherServlet
    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

AbstractDispatcherServletInitializer 还提供了一种方便的方法来添加 Filter 实例，并将它们自动映射到 DispatcherServlet，如下面的示例所示:

```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    // ...

    // 添加过滤器
    @Override
    protected Filter[] getServletFilters() {
        return new Filter[] {
            new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
    }
}
```

每个过滤器都根据其具体类型添加一个默认名称，并自动映射到 DispatcherServlet。AbstractDispatcherServletInitializer 的 isAsyncSupport 方法提供了一个单独的位置，以便在 DispatcherServlet 和映射到它的所有过滤器上启用异步支持。默认情况下，此标志设置为 true。

最后，如果需要进一步自定义 DispatcherServlet 本身，可以重写 createDispatcherServlet 方法。

### DispatcherServlet处理流程

> 待更新...