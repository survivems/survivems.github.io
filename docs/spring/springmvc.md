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

## 应用上下文层次结构

> 待更新。。。
