# 介绍

Java 的标准 Java.net.`URL` 类和各种 URL 前缀的标准处理程序不足以满足所有访问低级资源的需要。例如，没有标准化的 URL 实现可用于访问需要从类路径或相对于 ServletContext 获取的资源。虽然可以为特定的 URL 前缀注册新的处理程序(类似于现有的前缀处理程序，比如 http:) ，但是这通常相当复杂，而且 URL 接口仍然缺乏一些理想的功能，比如检查指向的资源是否存在的方法。

因此，Spring提供了`Resource`接口以及多种资源的实现。
Spring 的资源接口位于 `org.springframework.core.io`包，意味着是一个更有能力的接口，用于抽象对低级资源的访问。下面是`Resource`接口声明如下：

```java
/**
 * 资源描述符的接口，该描述符从基础资源的实际类型（如文件或类路径资源）中抽象出来。
 * 如果 InputStream 以物理形式存在，则可以为每个资源打开它，但只能为某些资源返回 URL 或 File 句柄。* 实际行为是特定于实现的。
 * 参考 #getInputStream()
 * 参考 #getURL()
 * 参考 #getURI()
 * 参考 #getFile()
 * 参考 WritableResource
 * 参考 ContextResource
 * 参考 UrlResource
 * 参考 FileUrlResource
 * 参考 FileSystemResource
 * 参考 ClassPathResource
 * 参考 ByteArrayResource
 * 参考 InputStreamResource
 */
public interface Resource extends InputStreamSource {

	/* 确定此资源是否以物理形式实际存在。
     * 此方法执行确定存在性检查，而句柄的存在 Resource 仅保证描述符句柄有效。
     */
	boolean exists();

	/*
     * 指示是否可以通过 getInputStream()读取此资源的非空内容。
     * true将用于存在的典型资源描述符。请注意，尝试实际内容读取
     * 时仍可能失败。但是，值 false 表示无法读取资源内容。
     */
	default boolean isReadable() {
		return exists();
	}

	/**
	 * 指示此资源是否表示具有开放流的句柄。如果 true，则无法多次读取 InputStream，
     * 必须读取并关闭以避* 免资源泄漏。将 false 用于典型的资源描述符。
	 */
	default boolean isOpen() {
		return false;
	}

	/**
	 * 确定此资源是否表示文件系统中的文件。
     * 值 true 表示 （但不保证）getFile() 调用将成功。
     * 保守地说 false ，这是默认的。
	 */
	default boolean isFile() {
		return false;
	}

	/**
	 * 返回此资源的 URL 句柄
	 */
	URL getURL() throws IOException;

	/**
	 * 返回此资源的 URI 句柄
	 */
	URI getURI() throws IOException;

	/**
	 * 返回此资源的 File 句柄
	 */
	File getFile() throws IOException;

	/**
	 * 返回一个 ReadableByteChannel.
     * 预计每次调用都会创建一个 新 通道。
     * 默认实现返回 Channels.newChannel(InputStream) 结果为 getInputStream()。
	 */
	default ReadableByteChannel readableChannel() throws IOException {
		return Channels.newChannel(getInputStream());
	}

	/**
	 * 确定此资源的内容长度
	 */
	long contentLength() throws IOException;

	/**
	 * 确定此资源的上次修改时间戳
	 */
	long lastModified() throws IOException;

	/**
	 * 创建与此资源相关的资源。
	 */
	Resource createRelative(String relativePath) throws IOException;

	/**
	 * 确定此资源的文件名，即通常是路径的最后一部分：例如，“myfile.txt”。
     * 如果此类型的资源没有文件名，则返回 null 。
	 */
	@Nullable
	String getFilename();

	/**
	 * 返回此资源的说明，用于使用资源时的错误输出。还鼓励实现从其 toString 方法中返回此值。
	 */
	String getDescription();
}

```

从上面`Resource`接口定义，可见其扩展了接口`InputStreamSource`，该接口定义如下：
```java
/**
 * Resource接口的基本接口。对于一次性使用的流，InputStreamResource可以用于任何给定的
 * InputStream。Spring的ByteArrayResource或任何基于文件的Resource实现都可以用作具体实例
 * 允许多次读取底层内容流。这使得该接口作为邮件附件的抽象内容源非常有用。
 */
public interface InputStreamSource {

	/**
	 * 返回基础资源内容的一个InputStream。预计每次调用都会创建一个新流。
     * 当您考虑JavaMail等API时，此要求尤为重要，因为API在创建邮件附件时需要能够多次读取流。
     * 对于此类用例，要求每次getInputStream()调用都返回一个新流。
	 */
	InputStream getInputStream() throws IOException;
}
```

`Resource`接口有一些重要的方法：
+ getInputStream()：定位并打开资源，返回 InputStream 以便从资源中读取。预计每个调用都会返回一个新的 InputStream。调用方负责关闭流。
+ exists()：返回一个布尔值，指示此资源是否实际以物理形式存在。
+ isOpen()：返回一个布尔值，该布尔值指示此资源是否表示具有打开流的句柄。如果为 true，则 InputStream 不能被多次读取，必须只读取一次，然后关闭，以避免资源泄漏。对于所有通常的资源实现返回 false，InputStreamResource 除外
+ getDescription()：返回此资源的说明，用于在使用资源时输出错误。这通常是完全限定的文件名或资源的实际 URL。

其他方法允许获得表示资源的实际 URL 或 File 对象(如果底层实现兼容并支持该功能)。
Resource 接口的一些实现还实现了 `WritableResource` 接口，用于向关联的资源写入内容。

<p style="color: red">Spring 本身广泛地使用了 Resource 抽象，当需要资源时，将其作为许多方法签名中的参数类型。例如各种 ApplicationContext实现的构造函数采用一个String类型的参数代表资源位置，它以未修饰或简单的形式用于创建与该上下文实现相适应的Resource，或者通过String路径上的特殊前缀，让调用者指定必须创建和使用特定的 Resource实现。</p>

尽管Resource接口经常被Spring使用，但是它实际上非常方便地作为一个通用实用类在自己的代码中使用，用于访问资源，即使代码不关心Spring的其他部分。虽然这会将代码与Spring耦合，但它实际上只是将它耦合到一小组实用工具类，这些类可以更好地替代URL，并且可以被认为与任何其他库等价。

> 资源抽象不会替代功能。例如，`UrlResource`包装`URL`并使用包装后的URL执行其工作。

# Resource内置Spring实现

+ UrlResource
+ ClassPathResource
+ FileSystemResource
+ PathResource
+ ServletContextResource
+ InputStreamResource
+ ByteArrayResource

更多介绍可以参考 spring的[javadoc](https://docs.spring.io/spring-framework/docs/5.3.31/javadoc-api/org/springframework/core/io/Resource.html)文档

## UrlResource

UrlResource 包装了 java.net.URL，可以用来访问通常可以通过 URL 访问的任何对象，例如文件、 HTTPS 目标、 FTP 目标等。所有 URL 都有一个标准化的 String 表示形式，以便使用适当的标准化前缀来表示一种 URL 类型与另一种 URL 类型之间的区别。如 file: 前缀用于访问文件系统路径，HTTPS: 前缀用于通过 HTTPS 协议访问资源，FTP: 前缀用于通过 FTP 访问资源等。

UrlResource 是由 Java 代码通过显式使用 UrlResource 构造函数创建的，但是通常在调用接受表示路径的 String 参数的 API 方法时隐式创建。
对于后一种情况，`PropertyEditor` 最终决定创建具体类型的 Resource。
如果路径字符串包含`classspath:`前缀，PropertyEditor（属性编辑器）将为该前缀创建一个适当的专门化 Resource（ClassPathResource）。
但是，如果它不识别前缀，则假定该字符串是标准 URL 字符串并创建一个 UrlResource。

示例：
```java
@Test
@SneakyThrows
public void testUrlResource() {
    // 方式1
    Resource resource = new UrlResource("file:/Users/miaomiao/Desktop/spring5.3.29/target/classes/logback.xml");
    log.info("resource = {}", resource);
    // 大小，单位字节
    long size = resource.contentLength();
    log.info("size: {} byte", size);
    // 最后修改时间，时间戳
    long lastedModified = resource.lastModified();
    Date date = new Date(lastedModified);
    String lastModifiedTime = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(date);
    log.info("last modified time: {}", lastModifiedTime);
    String description = resource.getDescription();
    log.info("description: {}", description);
}
```

输出：
```shell
2024-02-07 15:55:13.474 [main] INFO  cn.chiatso.resource.AppTest - resource 
= URL [file:/Users/miaomiao/Desktop/spring5.3.29/target/classes/logback.xml]
2024-02-07 15:55:13.476 [main] INFO  cn.chiatso.resource.AppTest - size: 1250 byte
2024-02-07 15:55:13.476 [main] INFO  cn.chiatso.resource.AppTest - last modified time: 2024-02-07 15:27:05
2024-02-07 15:55:13.476 [main] INFO  cn.chiatso.resource.AppTest - description:
URL [file:/Users/miaomiao/Desktop/spring5.3.29/target/classes/logback.xml]
```

## ClassPathResource

此类表示从类路径获取的资源。它使用线程上下文类加载器、给定类加载器或给定类来加载资源。

这个 Resource 实现支持将资源解析为`java.io.File`对象。如果类路径资源存在文件系统中，
而不是在尚未解压缩的jar中。
<font style="color: red">如果类路径上的资源位于JAR文件中，并且没有解压缩（例如，由Servlet引擎或其他环境展开到文件系统），则ClassPathResource实现不能将其解析为java.io.File对象。</font>

JAR文件是Java的归档文件，用于将多个Java类文件、相关的元数据和资源（如文本、图像等）打包到一个文件中。

为了解决这个问题，各种 Resource 实现始终支持作为 java.net.URL 的解析。

java.net.URL代表一个统一资源定位符，它指向一个抽象或物理资源的名称。无论资源是在文件系统中还是在JAR文件中，都可以被解析为java.net.URL。

ClassPathResource 是由 Java 代码通过显式地使用 ClassPathResource 构造函数创建的，但是通常是在调用一个 API 方法时隐式地创建的，该方法接受表示路径的 String 参数。对于后一种情况，`PropertyEditor` 识别字符串路径上的特殊前缀 classspath: ，并在该情况下创建 ClassPathResource。

示例：
```java
@Test
@SneakyThrows
public void testClassPathResource() {
    Resource resource = new ClassPathResource("/logback.xml");
    log.info("resource = {}", resource);
    // 获取资源大小，单位字节
    long size = resource.contentLength();
    log.info("size of resource: {} byte", size);
    URL url = resource.getURL();
    log.info("url = {}", url);
    URI uri = resource.getURI();
    log.info("uri = {}", uri);
    String description = resource.getDescription();
    log.info("description is {}", description);
    long lasted = resource.lastModified();
    log.info("last modified time is {}", new Date(lasted));
    boolean readable = resource.isReadable();
    log.info("resource is readable: {}", readable);
    boolean open = resource.isOpen();
    log.info("is open : {}", open);
    File file = resource.getFile();
    log.info("file: {}", file);
}
```

结果：
```shell
2024-02-07 16:33:26.726 [main] INFO  cn.chiatso.resource.AppTest - 
resource = class path resource [logback.xml]
2024-02-07 16:33:26.727 [main] INFO  cn.chiatso.resource.AppTest - size of resource: 1250 byte
2024-02-07 16:33:26.727 [main] INFO  cn.chiatso.resource.AppTest - 
url = file:/Users/miaomiao/Desktop/spring5.3.29/target/classes/logback.xml
2024-02-07 16:33:26.727 [main] INFO  cn.chiatso.resource.AppTest - 
uri = file:/Users/miaomiao/Desktop/spring5.3.29/target/classes/logback.xml
2024-02-07 16:33:26.727 [main] INFO  cn.chiatso.resource.AppTest - 
description is class path resource [logback.xml]
2024-02-07 16:33:26.727 [main] INFO  cn.chiatso.resource.AppTest - last modified time is 2024-02-07 15:27:05
2024-02-07 16:33:26.727 [main] INFO  cn.chiatso.resource.AppTest - resource is readable: true
2024-02-07 16:33:26.727 [main] INFO  cn.chiatso.resource.AppTest - is open : false
2024-02-07 16:33:26.727 [main] INFO  cn.chiatso.resource.AppTest - 
file: /Users/miaomiao/Desktop/spring5.3.29/target/classes/logback.xml
```

## FileSystemResource

此实现是一个标准的java.io.File句柄的资源实现，除了java.io.File，其还支持java.nio.file.Path。
当使用Path句柄时，这种资源实现会应用Spring的基于字符串的路径转换标准。
尽管应用了Spring的路径转换，但这种资源实现的所有操作都是通过java.nio.file.Files API来执行的。Files类提供了用于操作文件系统的静态方法。
如果需要纯基于java.nio.path.Path的支持，应该使用`PathResource`而不是FileSystemResource。
<p style="color: red">这意味着PathResource可能是专门为Path对象设计的，而FileSystemResource则同时支持File和Path，但可能更多地依赖于File相关的功能。</p>

FileSystemResource支持将资源解析为File和URL。这意味着，无论资源是文件系统中的文件还是JAR文件内的资源，FileSystemResource都可以提供对这些资源的访问。

示例：
```java
@Test
@SneakyThrows
public void testFileSystemResource() {
    FileSystemResource resource = new FileSystemResource("/Users/miaomiao/Desktop/凭证处理中心.md");
    log.info("resource: {}", resource);
    String path = resource.getPath();
    log.info("path: {}", path);
    String filename = resource.getFilename();
    log.info("filename: {}", filename);
    long size = resource.contentLength();
    log.info("size: {} byte", size);
}
```

输出：
```shell
2024-02-07 16:45:08.033 [main] INFO  cn.chiatso.resource.AppTest - resource: 
file [/Users/miaomiao/Desktop/凭证处理中心.md]
2024-02-07 16:45:08.034 [main] INFO  cn.chiatso.resource.AppTest - 
path: /Users/miaomiao/Desktop/凭证处理中心.md
2024-02-07 16:45:08.034 [main] INFO  cn.chiatso.resource.AppTest - filename: 凭证处理中心.md
2024-02-07 16:45:08.035 [main] INFO  cn.chiatso.resource.AppTest - size: 65 byte
```

## PathResouce

PathResource是一个专门用于处理java.nio.file.Path句柄的资源实现。java.nio.file.Path是Java NIO（New I/O）包中的一个接口，用于表示文件系统中的路径。PathResource通过Path API执行所有操作和转换，支持将资源解析为File和URL。

PathResource还实现了扩展的`WritableResource`接口。这表明PathResource不仅是一个用于读取资源的实现，还提供了写入资源的能力。

PathResource是FileSystemResource的一个纯基于java.nio.path.Path的替代方案，但具有不同的相对路径创建行为。这意味着PathResource和FileSystemResource在如何处理相对路径方面可能存在差异。使用PathResource时，相对路径的解析和创建可能会按照java.nio.file.Path的语义进行，而不是java.io.File的语义。

PathResource是一个专门为java.nio.file.Path设计的资源实现，它使用java.nio.file包中的新API进行所有操作，并提供对资源的读写访问。它是FileSystemResource的一个替代方案，尤其是在需要纯Path支持和/或不同的相对路径行为时。

示例：
```java
 @Test
@SneakyThrows
public void testPathResource() {
    Path path = Paths.get("/Users/miaomiao/Desktop/凭证处理中心.md");
    PathResource resource = new PathResource(path);
    long size = resource.contentLength();
    log.info("size: {} byte", size);
    File file = resource.getFile();
    log.info("file: {}", file);
    String filename = resource.getFilename();
    log.info("filename: {}", filename);
    // 是否可写
    boolean writable = resource.isWritable();
    log.info("writable: {}", writable);
    if (writable) {
        OutputStream outputStream = resource.getOutputStream();
        outputStream.write("Hello PathResource".getBytes(StandardCharsets.UTF_8));
        outputStream.close();
    }
}
```

输出：
```shell
2024-02-07 17:02:35.653 [main] INFO  cn.chiatso.resource.AppTest - size: 65 byte
2024-02-07 17:02:35.654 [main] INFO  cn.chiatso.resource.AppTest - 
file: /Users/miaomiao/Desktop/凭证处理中心.md
2024-02-07 17:02:35.654 [main] INFO  cn.chiatso.resource.AppTest - filename: 凭证处理中心.md
2024-02-07 17:02:35.655 [main] INFO  cn.chiatso.resource.AppTest - writable: true
```

> 提示：
> 写入内容时，是以追加方式写入，还是覆盖写入！！！

## ServletContextResource

ServletContextResource 能够解析相对于 Web 应用程序根目录的路径。对于在 Web 应用程序中定位资源非常有用，例如读取配置文件、静态资源等。

它总是支持流访问和 URL 访问。对于文件访问，ServletContextResource 的行为取决于 Servlet 容器的实现。如果 Web 应用程序归档文件（如 WAR 文件）被展开，并且资源实际存在于文件系统中，那么可以通过 java.io.File 访问该资源。但是，如果资源被直接存储在 JAR 文件中，或者从数据库等其他位置访问，那么可能不支持 java.io.File 访问。

是否展开 Web 应用程序归档文件以及资源是如何访问的（直接从 JAR，还是通过文件系统，或者其他方式），这实际上取决于使用的 Servlet 容器的实现。不同的 Servlet 容器可能会有不同的行为。

ServletContextResource 是一个灵活的资源访问机制，它能够处理 Web 应用程序中的各种资源，但是否支持特定类型的访问（如文件访问）可能取决于 Servlet 容器的实现和资源的实际存储方式。

## InputStreamResource

