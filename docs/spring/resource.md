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

InputStreamResource 是 Spring 框架中用于表示给定 InputStream 的资源实现。
它主要用于那些没有更具体资源实现适用的场景。通常，如果可能的话，更推荐使用 ByteArrayResource 或任何基于文件的资源实现。

与其他资源实现相比，InputStreamResource 描述的是一个已经打开的资源。因此，其 isOpen() 方法返回 true。如果你需要将资源描述符保存在某个地方，或者需要多次读取流，那么不要使用 InputStreamResource。

有几个问题需要注意：
+ InputStreamResource 封装了一个已经打开的 InputStream。这意味着在使用它之前，流已经被打开。这可能会导致一些资源管理上的问题，特别是在需要长时间保持资源打开或在多个地方使用同一资源时。
+ 由于 InputStream 只能被读取一次，因此 InputStreamResource 通常也只能被读取一次。如果你试图多次读取它，将会遇到问题。
+ 使用 InputStreamResource 时，需要特别注意资源管理。确保在使用完资源后正确关闭它，以避免资源泄漏。然而，由于 InputStreamResource 封装的是一个已经打开的流，因此关闭资源可能会更加复杂。
+ 如果可能的话，应该优先使用其他更具体的资源实现，如 ByteArrayResource（用于字节数组）或基于文件的资源实现（如 FileSystemResource 或 UrlResource）。这些实现通常提供更好的资源管理和更高的性能。

尽管有上述限制，但在某些特定场景下，InputStreamResource 可能是唯一可行的选择。例如，当你无法将整个资源加载到内存中，或者无法以文件形式访问资源时，可以考虑使用 InputStreamResource。然而，在这些情况下，需要格外小心以确保正确管理资源并避免潜在的问题。

示例：使用InputStreamSource作为下载资源
```java
@RestController  
public class InputStreamResourceController {  
  
    @GetMapping("/download")  
    public ResponseEntity<Resource> downloadData() throws IOException {  
        // 模拟动态生成的数据  
        String data = "This is some dynamic data that the client will download.";  
        InputStream inputStream = new ByteArrayInputStream(data.getBytes());  
  
        // 创建 InputStreamResource  
        InputStreamResource inputStreamResource = new InputStreamResource(inputStream);  
  
        // 设置 HTTP 响应头  
        HttpHeaders headers = new HttpHeaders();  
        headers.add("Content-Disposition", "attachment; filename=data.txt");  
        headers.add("Cache-Control", "no-cache, no-store, must-revalidate");  
        headers.add("Pragma", "no-cache");  
        headers.add("Expires", "0");  
  
        // 返回 ResponseEntity 包含 InputStreamResource 和响应头  
        return ResponseEntity.ok()  
                .headers(headers)  
                .contentType(MediaType.APPLICATION_OCTET_STREAM)  
                .contentLength(data.length())  
                .body(inputStreamResource);  
    }  
}
```

## ByteArrayResource

ByteArrayResource 是 Spring 框架中用于封装给定字节数组的资源实现。它内部使用 ByteArrayInputStream 来提供对字节数组内容的访问。与 InputStreamResource 相比，ByteArrayResource 是可重复读取的，因为字节数组内容在内存中，并且可以多次访问而不需要重新打开或重新创建资源。

示例：
```java
@RestController  
public class ByteArrayResourceController {  
  
    @GetMapping("/download")  
    public ResponseEntity<Resource> downloadData() {  
        // 模拟一些数据  
        String data = "This is some static data that the client will download.";  
        byte[] bytes = data.getBytes();  
  
        // 创建 ByteArrayResource  
        ByteArrayResource byteArrayResource = new ByteArrayResource(bytes);  
  
        // 设置 HTTP 响应头（可选）  
        HttpHeaders headers = new HttpHeaders();  
        headers.add("Content-Disposition", "attachment; filename=data.txt");  
  
        // 返回 ResponseEntity 包含 ByteArrayResource 和响应头  
        return ResponseEntity.ok()  
                .headers(headers)  
                .contentType(MediaType.APPLICATION_OCTET_STREAM)  
                .contentLength(bytes.length) // 设置内容长度是可选的，但有助于客户端了解响应的大小  
                .body(byteArrayResource);  
    }  
}
```

在这个示例中，创建了一个简单的 Spring MVC 控制器，其中包含一个 downloadData() 方法。这个方法将静态字符串数据转换为字节数组，并使用 ByteArrayResource 将其封装起来。然后将 ByteArrayResource 实例与适当的 HTTP 响应头一起作为 ResponseEntity 返回给客户端。

由于 ByteArrayResource 内部使用了 ByteArrayInputStream，客户端可以多次读取响应的内容，而不会出现像使用 InputStreamResource 时可能遇到的问题（比如流已经被关闭或读取到末尾）。此外，因为数据已经加载到内存中，所以不需要担心资源管理问题，比如关闭流或处理文件句柄。这使得 ByteArrayResource 成为处理小到中等大小数据的理想选择。然而，对于非常大的数据集，可能需要考虑使用其他类型的资源实现，以避免消耗过多内存。

# ResourceLoader接口

ResourceLoader接口设计用来获取它所加载的资源（Resource），它的接口定义如下：
```java
public interface ResourceLoader {
    /* classpath: */
	String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;

    /**
     * 返回 Resource 指定资源位置的句柄。
     * 句柄应始终是可重用的资源描述符，允许多次 Resource.getInputStream() 调用。
     * 必须支持完全限定的 URL，例如“file：C：/test.dat”。
     * 必须支持类路径伪 URL，例如“classpath:test.dat”。
     * 应支持相对文件路径，例如“WEB-INF/test.dat”。
     * （这将是特定于实现的，通常由 *ApplicationContext 实现提供。
     * 请注意， Resource 句柄并不意味着现有资源;您需要调用 Resource.exists 来检查是否存在。
     */
	Resource getResource(String location);

    /**
     * 公开此ResourceLoader使用的ClassLoader
     * 需要直接访问的 ClassLoader 客户端可以使用 以统一的方式 ResourceLoader进行访问，而不是依赖于* 线程上下文 ClassLoader。
    */
	@Nullable
	ClassLoader getClassLoader();
}
```
<p style="color: red">所有应用程序上下文都实现 ResourceLoader 接口。</p>

当在特定的应用程序上下文上调用 getResource() ，并且指定的位置路径没有特定的前缀时，将返回一个适合该特定应用程序上下文的 Resource 类型。例如，假设以下代码片段是针对 ClassPathXmlApplicationContext 实例运行的:

```java
Resource template = ctx.getResource("some/resource/path/myTemplate.txt");
```

根据 ClassPathXmlApplicationContext上下文，该代码返回 ClassPathResource。如果对 FileSystemXmlApplicationContext 实例运行相同的方法，它将返回 FileSystemResource。对于 WebApplicationContext，它将返回 ServletContextResource。类似地，它将为每个上下文返回适当的对象。

因此，可以以适合特定应用程序上下文的方式加载资源。

另一方面，也可以通过指定特殊的 `classspath: `前缀来强制使用 ClassPathResource，而不管应用程序上下文类型如何，如下例所示:

```java
Resource template = ctx.getResource("classpath:some/resource/path/myTemplate.txt");
```

类似地，您可以通过指定任何标准 java.net.URL 前缀来强制使用 UrlResource。下面的示例使用文件和 https 前缀:

```java
Resource template = ctx.getResource("file:///some/resource/path/myTemplate.txt");
Resource template = ctx.getResource("https://myhost.com/resource/path/myTemplate.txt");
```

下表总结了将 String 对象转换为 Resource 对象的策略:

| 前缀       | 示例                           | 解释                 |
| ---------- | ------------------------------ | -------------------- |
| classpath: | classpath:com/myapp/config.xml | 从类路径加载         |
| file:      | file:///data/config.xml        | 从文件系统加载为 URL |
| https:     | https://myserver/logo.png      | 作为 URL 加载        |
| (none)     | /data/config.xml               | 依赖特定上下文加载   |

# ResourcePatternResolver接口

ResourcePatternResolver 接口是 ResourceLoader 接口的扩展，该接口定义了将位置模式(例如，Ant 风格的路径模式)解析为 Resource 对象的策略。

```java
public interface ResourcePatternResolver extends ResourceLoader {

    String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

    Resource[] getResources(String locationPattern) throws IOException;
}
```

ResourcePatternResolver 是 Spring 框架中的一个接口，它提供了一种解析资源路径的方式，特别是当这些路径可能包含模式（例如，使用通配符）时。这个接口通常用于加载配置文件、模板或其他资源。该接口还为类路径中的所有匹配资源定义了一个特殊的类路径 * : 资源前缀。

当你想从多个位置（例如，多个JAR文件或目录）加载具有相同模式的资源时，ResourcePatternResolver 特别有用。通过使用 classpath*: 前缀，你可以指示 Spring 查找类路径上的所有匹配资源，而不仅仅是第一个找到的。

该接口实现有如下：

+ PathMatchingResourcePatternResolver：这是一个常用的实现，它使用文件系统路径匹配来解析资源。
+ ClassPathResourcePatternResolver：这个实现专门用于从类路径中加载资源

下面是一个简单的示例，展示如何使用 PathMatchingResourcePatternResolver 从文件系统中加载所有XML配置文件：

```java
PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();  
Resource[] resources = resolver.getResources("classpath*:config/*.xml");  
  
for (Resource resource : resources) {  
    System.out.println(resource.getFilename());  
}
```

PathMatchingResourcePatternResolver 确实是一个可以独立使用的实现，它不依赖于 ApplicationContext。这意味着你可以在不启动整个 Spring 应用上下文的情况下使用它来解析资源路径。此外，ResourceArrayPropertyEditor 类也使用 PathMatchingResourcePatternResolver 来填充 Resource[] 类型的 bean 属性。

这个解析器的关键特性是能够根据指定的资源位置路径解析出一个或多个匹配的 Resource 对象。这个源路径可以是一个简单的、与目标 Resource 具有一对一映射关系的路径，也可以包含特殊的 classpath*: 前缀和/或内部 Ant 风格的正则表达式。

+ classpath*: 前缀告诉解析器要查找类路径上的所有匹配资源，而不仅仅是第一个。这是一个非常有用的功能，特别是在处理打包在多个 JAR 文件中的资源时。
+ Ant 风格的正则表达式允许你使用通配符来匹配多个资源。例如，com/example/config/*.xml 将匹配 com/example/config/ 目录下的所有 XML 文件。

这个解析器内部使用 AntPathMatcher 工具类来匹配这些路径模式，确保你可以灵活地匹配各种资源路径。这使得 PathMatchingResourcePatternResolver 成为在 Spring 应用中加载和解析资源的强大工具。

> <font style="color: red">在任何标准的 ApplicationContext 中，默认的 ResourceLoader 实际上是一个 PathMatchingResourcePatternResolver 的实例，它实现了 ResourcePatternResolver 接口。同样，ApplicationContext 实例本身也实现了 ResourcePatternResolver 接口，并委托给默认的 PathMatchingResourcePatternResolver。
这意味着，当你在 ApplicationContext 中使用资源加载功能时，你实际上是在使用 PathMatchingResourcePatternResolver 的功能。这使得加载资源变得非常方便，因为你可以直接通过 ApplicationContext 实例来加载资源，而无需显式地创建和配置 PathMatchingResourcePatternResolver。</font>

# ResourceLoaderAware接口

ResourceLoaderAware 接口是一个特殊的回调接口，用于标识那些期望被提供 ResourceLoader 引用的组件。当一个类实现 ResourceLoaderAware 接口并被部署到应用上下文中（作为一个由 Spring 管理的 bean）时，应用上下文会识别出这个类实现了 ResourceLoaderAware 接口。然后，应用上下文会调用 setResourceLoader(ResourceLoader) 方法，并将自己作为参数传递（请记住，Spring 中的所有应用上下文都实现了 ResourceLoader 接口）。

该接口定义如下：

```java
public interface ResourceLoaderAware {

    void setResourceLoader(ResourceLoader resourceLoader);
}
```

由于 ApplicationContext 是 ResourceLoader 的一个实现，因此 bean 也可以实现 ApplicationContextAware 接口，并直接使用提供的 ApplicationContext 来加载资源。然而，通常情况下，如果你只需要加载资源，那么使用专门的 ResourceLoader 接口会更好。这样，代码只与资源加载接口（可以认为是一个实用接口）耦合，而不是与整个 Spring ApplicationContext 接口耦合。

在应用组件中，你还可以依赖于 ResourceLoader 的自动装配（autowiring）作为实现 ResourceLoaderAware 接口的替代方案。传统的构造函数和按类型自动装配模式（如“自动装配协作者”中所述）能够为构造函数参数或设置器方法参数提供 ResourceLoader。对于更大的灵活性（包括字段和多个参数方法的自动装配能力），可以考虑使用基于注解的自动装配特性。在这种情况下，只要字段、构造函数或方法带有 @Autowired 注解，并且期望的类型是 ResourceLoader，那么 ResourceLoader 就会被自动装配到该字段、构造函数参数或方法参数中。

> 当你需要加载包含通配符或使用特殊 classpath*: 前缀的资源路径的一个或多个 Resource 对象时，建议将 ResourcePatternResolver 的实例自动装配到你的应用组件中，而不是 ResourceLoader。

# 作为依赖的资源

如果 bean 本身将通过某种动态过程来确定和提供资源路径，那么让 bean 使用 ResourceLoader 或 ResourcePatternResolver 接口来加载资源可能是有意义的。例如，考虑加载某种模板，其中所需的具体资源取决于用户的角色。如果资源是静态的，那么完全不使用 ResourceLoader 接口（或 ResourcePatternResolver 接口）是有意义的，让 bean 暴露它所需的 Resource 属性，并期望它们被注入到 bean 中。

使注入这些属性变得简单的是，所有应用上下文都注册并使用了一个特殊的 JavaBeans PropertyEditor，它可以将字符串路径转换为 Resource 对象。例如，下面的 MyBean 类有一个类型为 Resource 的 template 属性。

```java
public class MyBean {  
  
    private Resource template;  
  
    public Resource getTemplate() {  
        return template;  
    }  
  
    public void setTemplate(Resource template) {  
        this.template = template;  
    }
}
```

在这个例子中，如果你在一个 Spring 配置文件中配置 MyBean，你可以简单地使用字符串路径来设置 template 属性，而不需要直接处理 Resource 对象。Spring 的上下文会利用 PropertyEditor 将这个字符串路径转换为一个 Resource 对象，并自动注入到 template 属性中。

```xml
<bean id="myBean" class="com.example.MyBean">  
    <property name="template" value="some/resource/path/myTemplate.txt"/>  
</bean>
```

请注意，资源路径没有前缀。因此，由于应用程序上下文本身将被用作 ResourceLoader，资源将通过 ClassPathResource、 FileSystemResource 或 ServletContextResource 加载，具体取决于应用程序上下文的确切类型。

如果需要强制使用特定的 Resource 类型，可以使用前缀。下面两个例子展示了如何强制使用 ClassPathResource 和 UrlResource (后者用于访问文件系统中的文件) :

```xml
<property name="template" value="classpath:some/resource/path/myTemplate.txt">
<property name="template" value="file:///some/resource/path/myTemplate.txt"/>
```

如果 MyBean 类被重构用于注解驱动的配置，那么 myTemplate.txt 的路径可以存储在一个名为 template.path 的键下ーー例如，在 Spring Environment 可用的属性文件中。然后可以使用属性占位符通过@Value 注解引用模板路径。Spring 将以字符串的形式检索模板路径的值，而一个特殊的 PropertyEditor 将把字符串转换为要注入到 MyBean 构造函数中的 Resource 对象。如下：

```java
@Component
public class MyBean {

    private final Resource template;

    public MyBean(@Value("${template.path}") Resource template) {
        this.template = template;
    }

    ...
}
```

如果希望支持在类路径的多个位置的同一路径下发现的多个模板ーー例如，在类路径的多个 jar 中ーー我们可以使用特殊的classpath*: 前缀和通配来将 templates.path 键定义为classpath*:/config/template/*.txt。并按照以下方式重新定义 MyBean 类，Spring 将把模板路径模式转换为一个 Resource 对象数组，这些对象可以注入到 MyBean 构造函数中。

```java
@Component
public class MyBean {

    private final Resource[] templates;

    public MyBean(@Value("${templates.path}") Resource[] templates) {
        this.templates = templates;
    }

    // ...
}
```

# 应用程序上下文和资源路径

本节介绍如何使用资源创建应用程序上下文，包括使用 XML 的快捷方式、如何使用通配符以及其他详细信息。

## 构建应用程序上下文

应用程序上下文构造函数(针对特定的应用程序上下文类型)通常将字符串或字符串数组作为资源的位置路径，例如构成上下文定义的 XML 文件。

当这样的位置路径没有前缀时，根据该路径构建并用于加载 bean 定义的特定 Resource 类型取决于并适合于特定的应用程序上下文。例如，考虑下面的示例，它创建了 ClassPathXmlApplicationContext:

```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("conf/appContext.xml");
```

Bean 定义是从类路径加载的，因为使用了 ClassPathResource。但是，请考虑下面的示例，它创建了 FileSystemXmlApplicationContext:

```java
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("conf/appContext.xml");
```

现在 bean 定义是从一个文件系统位置加载的(在本例中，相对于当前的工作目录)。

请注意，在位置路径上使用特殊类路径前缀或标准 URL 前缀会覆盖为加载 bean 定义而创建的 Resource 的默认类型。考虑下面的例子:

```java
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("classpath:conf/appContext.xml");
```

使用 FileSystemXmlApplicationContext 从类路径加载 bean 定义。但是，它仍然是 FileSystemXmlApplicationContext。如果随后将其用作 ResourceLoader，则任何无前缀的路径仍被视为文件系统路径。

## 构建 ClassPathXmlApplicationContext 实例ーー快捷方式

ClassPathXmlApplicationContext 应用上下文用于从类路径（classpath）中加载 XML 配置文件并创建 Spring 应用的上下文。这个类有多个构造函数，允许开发者以不同的方式提供配置文件的路径。

当提供一个 Class 对象给 ClassPathXmlApplicationContext，它会使用这个类的包名作为基础路径来查找 XML 配置文件。例如，如果你提供了一个名为 com.example.MyClass 的类，那么 ClassPathXmlApplicationContext 会在类路径的 com/example/ 目录下查找 XML 文件。

```java
ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext(new String[] {"beans.xml"}, MyClass.class);
```

在这个例子中，ClassPathXmlApplicationContext 会在 com/example/目录下查找名为 beans.xml 的文件。

这种方式的优点是，可以不直接指定 XML 文件的完整路径，而是相对于某个类的包路径来引用它们。这使得代码更加灵活，特别是当项目结构发生变化时，你不需要修改配置文件的路径。

## 应用程序上下文构造器资源路径中的通配符

在 Spring 框架中，ClassPathXmlApplicationContext 和其他 ApplicationContext 实现允许你使用通配符和 Ant 风格的模式来指定资源路径。这提供了很大的灵活性，尤其是在处理多个配置文件或模块化的应用程序时。

当你使用 classpath*: 前缀时，你实际上是告诉 Spring 在类路径下查找所有匹配指定模式的资源。Spring 使用 PathMatcher 实用工具类来解析这些模式，并返回所有匹配的资源。例如，如果你有一个目录结构如下：

```shell
src/  
└── main/  
    ├── java/  
    │   └── com/  
    │       └── example/  
    │           ├── MyApp.java  
    │           └── MyComponent.java  
    └── resources/  
        ├── config/  
        │   ├── app-context.xml  
        │   ├── component-a.xml  
        │   └── component-b.xml  
        └── lib/  
            ├── spring-*.jar  
            └── other-libs/*.jar
```

假设你想加载 config 目录下的所有 XML 配置文件，你可以使用以下代码：

```java
ApplicationContext context = new ClassPathXmlApplicationContext("classpath*:config/*.xml");
```

这里，classpath*:config/*.xml 告诉 Spring 在类路径下查找 config 目录，并加载该目录下所有扩展名为 .xml 的文件。这将会加载 app-context.xml、component-a.xml 和 component-b.xml。

注意，这种通配符机制仅适用于在 ApplicationContext 构造函数中指定的资源路径，或者当你直接使用 PathMatcher 实用工具类层次结构时。它是在构造时解析的，与 Resource 类型本身无关。你不能使用 classpath*: 前缀来构造一个实际的 Resource 对象，因为 Resource 指的是单个资源。

这种机制特别适用于组件风格的应用程序组装，其中各个组件可以将其上下文定义片段发布到众所周知的路径上，然后在创建最终的应用程序上下文时使用带有 classpath*: 前缀的相同路径，从而自动收集所有组件片段。

> 注意点：
在 Spring 中，`classpath:config/*.xml` 和 `classpath*:config/*.xml` 之间的区别在于它们如何解析和加载类路径上的资源。
`classpath:config/*.xml`：这个表达式仅搜索默认的类路径（即不包含任何JAR文件的类路径）。它会在类路径的根目录下的 config 文件夹中查找所有扩展名为 .xml 的文件。这通常用于加载位于类路径根目录中的资源文件。
`classpath*:config/*.xml`：这个表达式搜索整个类路径，包括JAR文件。它会查找所有类路径条目（包括JAR文件和类路径根目录）中的 config 文件夹，并加载其中所有扩展名为 .xml 的文件。这允许你从多个JAR文件或类路径根目录中的 config 文件夹加载配置文件。
简而言之，`classpath`: 仅搜索类路径的根目录，而 `classpath*`: 搜索整个类路径，包括所有JAR文件。
使用 `classpath*`: 的好处是，如果你的应用程序是由多个模块或JAR文件组成的，并且每个模块都有自己的配置文件，你可以很容易地从所有这些模块中加载配置文件，而无需指定每个JAR文件的确切路径。这有助于保持配置的灵活性和模块化。

### Ant风格模式

路径位置可以包含 Ant 样式的模式，如下面的示例所示:

```shell
/WEB-INF/*-context.xml
com/mycompany/**/applicationContext.xml
file:C:/some/path/*-context.xml
classpath:com/mycompany/**/applicationContext.xml
```

当路径位置包含 Ant 样式的模式时，解析器将遵循更复杂的过程来尝试解析通配符。它为直到最后一个非通配符段的路径生成一个 Resource，并从中获得一个 URL。如果此 URL 不是 jar: URL 或特定于容器的变体(如 WebLogic 中的 zip: 、 WebSphere 中的 wsjar 等) ，则为 java.io。文件从中获取，并用于通过遍历文件系统解析通配符。对于 jar URL，解析器要么获得一个 java.net。或者手动解析 jar URL，然后遍历 jar 文件的内容以解析通配符。

### 对便携性的影响

如果指定的路径已经是一个文件 URL (或者隐式地因为基本 ResourceLoader 是一个文件系统 URL，或者显式地因为它是一个文件系统 URL) ，则保证通配符以一种完全可移植的方式工作。

如果指定的路径是一个类路径位置，解析器必须通过调用 Classloader.getResource ()来获取最后一个非通配符路径段 URL。因为这只是路径的一个节点(而不是最后的文件) ，所以实际上(在 ClassLoader javadoc 中)没有定义在这种情况下返回的 URL 的确切类型。实际上，它始终是一个 java.io。表示目录(其中类路径资源解析为文件系统位置)或某种类型的 jar URL (其中类路径资源解析为 jar 位置)的文件。尽管如此，这个操作仍然存在可移植性方面的问题。

如果为最后一个非通配符段获得了 jar URL，解析器必须能够获得 java.net。使用 JarURLConnection 或手动解析 jar URL，以便能够遍历 jar 的内容并解析通配符。这在大多数环境中都可以工作，但在其他环境中却失败了，我们强烈建议在您依赖它之前，应该在您的特定环境中彻底测试来自 jar 的资源的通配符解析。

### classpath*:前缀

在构造基于 XML 的应用程序上下文时，位置字符串可以使用特殊的类路径 * : 前缀，如下面的示例所示:

```java
ApplicationContext ctx =
    new ClassPathXmlApplicationContext("classpath*:conf/appContext.xml");
```

这个特殊的前缀指定必须获取与给定名称匹配的所有类路径资源(在内部，这实际上是通过调用 ClassLoader.getResources (...)实现的) ，然后合并形成最终的应用程序上下文定义。

> 通配符 classpath*: 的行为依赖于底层 ClassLoader 的 getResources() 方法。由于现代的应用服务器通常提供它们自己的 ClassLoader 实现，因此当处理 JAR 文件时，classpath*: 的行为可能会有所不同。这主要是因为不同的应用服务器可能会对 JAR 文件中的资源加载采用不同的策略。
一些应用服务器可能允许你配置 ClassLoader 的行为，例如设置父类加载器的委托模型、是否应该扫描 JAR 文件中的资源等。这些设置可能会影响 classpath*: 通配符的行为。
另外，值得注意的是，ClassLoader 的 getResources() 方法可能会返回类路径上所有匹配的资源，包括那些来自 JAR 文件和类路径根目录的资源。因此，如果你期望只从特定的位置加载资源，你可能需要更精确地指定资源的路径。
总之，当使用 classpath*: 通配符时，了解你的应用服务器如何实现 ClassLoader 以及如何配置它是非常重要的。如果遇到问题，最好查阅相关文档并进行适当的测试，以确保资源被正确加载。

> 可以将 classpath*: 前缀与位置路径中剩余部分的 PathMatcher 模式结合使用，以进一步细化资源加载的范围。例如，classpath*:META-INF/*-beans.xml 将会加载类路径上所有 META-INF 目录下的以 -beans.xml 结尾的文件。
在这种情况下，解析策略相对简单：首先，对最后一个非通配符路径段调用 ClassLoader.getResources() 方法，以获取类加载器层次结构中所有匹配的资源。然后，针对每个资源，使用先前描述的 PathMatcher 解析策略来处理通配符子路径。
这意味着，如果你有一个复杂的目录结构，并且只想加载满足特定模式的资源，你可以使用 classpath*: 前缀结合 PathMatcher 模式来实现。例如，如果你想加载所有位于 META-INF 目录下，并且文件名以 -beans.xml 结尾的 XML 文件，无论这些文件位于类路径的哪个 JAR 包或目录中，你都可以使用上述模式来实现。
这里有一个重要的点需要注意：当使用 classpath*: 前缀时，PathMatcher 模式仅应用于最后一个路径段之后的部分。在前面的例子中，META-INF 是非通配符部分，而 *-beans.xml 是通配符模式，它将会匹配所有以 -beans.xml 结尾的文件。
总之，通过将 classpath*: 前缀与 PathMatcher 模式结合使用，你可以非常灵活地加载类路径上的资源，无论它们位于哪里，只要它们符合指定的模式即可。这为大型应用程序或模块化应用程序中的资源加载提供了很大的便利。

### 与通配符有关的其他要点

注意：涉及到 classpath*: 与 Ant 风格模式结合使用时的一个潜在限制。当你使用 classpath*: 与 Ant 风格模式（如 *.xml）结合时，确保模式前面至少有一个根目录是非常重要的。这是因为在处理 JAR 文件时，ClassLoader.getResources() 方法可能不会返回位于 JAR 文件根部的资源，而只会返回展开目录根部的资源。

这是因为 JAR 文件本质上是一个压缩的归档文件，其中的资源被压缩在内部，并不直接对应文件系统上的位置。因此，当使用 classpath*: 加载 JAR 文件中的资源时，通常需要指定至少一个目录级别的路径，以便 ClassLoader 能够定位到正确的资源。

Spring 框架通过 JDK 的 ClassLoader.getResources() 方法来检索类路径条目。这个方法对于空字符串（表示要搜索的潜在根目录）只返回文件系统位置。Spring 还会评估 URLClassLoader 的运行时配置以及 JAR 文件中的 java.class.path 清单，但这并不保证在所有情况下都能产生可移植的行为。

因此，最佳实践是当使用 classpath*: 与 Ant 风格模式结合时，确保你的模式前面有一个或多个明确的目录或包路径。这样可以提高加载资源的可靠性，并确保在不同的环境和应用服务器中行为一致。如果你需要加载 JAR 文件根部的资源，你可能需要明确指定 JAR 文件的路径，或者考虑将资源移动到可访问的目录路径下。

> 类路径包的扫描需要类路径中存在相应的目录条目。当你使用 Ant 构建 JAR 文件时，不要激活 JAR 任务的仅文件开关，因为这可能会导致类路径上的目录条目不被包含在内。
另外，在某些环境中，基于安全策略，类路径目录可能不会暴露出来。例如，在 JDK 1.7.0_45 及更高版本的独立应用程序中，需要在清单文件中设置 'Trusted-Library' 才能访问类路径目录。这可能会影响 ClassLoader.getResources() 的行为，因为它可能无法找到位于类路径目录中的资源。
在 JDK 9 的模块路径（Jigsaw）上，Spring 的类路径扫描通常可以按预期工作。同样，推荐将资源放入专门的目录中，以避免前面提到的在 JAR 文件根级别搜索时可能出现的可移植性问题。
总之，为了确保资源能够正确加载，你需要注意以下几点：
在构建 JAR 文件时，确保包含必要的目录条目。
避免激活 JAR 任务的仅文件开关。
在需要访问类路径目录的环境中，确保已正确设置安全策略（如 'Trusted-Library'）。
在 JDK 9 的模块路径上，推荐将资源放入专门的目录中。
遵循这些建议可以帮助你避免常见的资源加载问题，并确保你的应用程序在不同的 Java 版本和环境中具有一致的行为。
如果要搜索的根包在多个类路径位置可用，则不能保证使用 classspath: 资源 的 Ant 样式模式能够找到匹配的资源。考虑下面的资源位置示例:

```shell
com/mycompany/package1/service-context.xml
```

现在考虑一个 Ant 风格的路径，有人可能会用它来寻找那个文件:

```shell
classpath:com/mycompany/**/service-context.xml
```

这样的资源可能只存在于类路径中的一个位置，但是当使用前面的示例之类的路径来解析它时，解析器将依赖于 getResource (“com/mycompany”); 返回的(第一个) URL。如果此基本包节点存在于多个 ClassLoader 位置，则所需的资源可能不存在于所找到的第一个位置。因此，在这种情况下，您应该更喜欢使用 classspath* : 具有相同的 Ant 风格模式，该模式搜索包含 com.mycompany 基本包的所有类路径位置: classspath*:com/mycompany/**/service-context.xml。

## FileSystemResource 注意事项

未附加到 FileSystemApplicationContext (即，当 FileSystemApplicationContext 不是实际的 ResourceLoader 时)的 FileSystemResource 按照您预期的方式处理绝对路径和相对路径。相对路径是相对于当前工作目录的，而绝对路径是相对于文件系统的根的。

但是，由于向后兼容性(历史)原因，当 FileSystemApplicationContext 是 ResourceLoader 时，这种情况会发生变化。FileSystemApplicationContext 强制所有附加的 FileSystemResource 实例将所有位置路径视为相对路径，无论它们是否以斜杠开头。在实践中，这意味着下列例子是等价的:

```java
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("conf/context.xml");
```

```java
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("/conf/context.xml");
```

下面的例子也是等价的(即使它们有所不同也是有意义的，因为一种情况是相对的，而另一种情况是绝对的) :

```java
FileSystemXmlApplicationContext ctx = ...;
ctx.getResource("some/resource/path/myTemplate.txt");
```

```java
FileSystemXmlApplicationContext ctx = ...;
ctx.getResource("/some/resource/path/myTemplate.txt");
```

实际上，如果需要真正的绝对文件系统路径，应该避免使用 FileSystemResource 或 FileSystemXmlApplicationContext 的绝对路径，并通过使用 file: URL 前缀强制使用 UrlResource。下面的例子说明了如何做到这一点:

```java
// actual context type doesn't matter, the Resource will always be UrlResource
ctx.getResource("file:///some/resource/path/myTemplate.txt");
```

```java
// force this FileSystemXmlApplicationContext to load its definition via a UrlResource
ApplicationContext ctx =
    new FileSystemXmlApplicationContext("file:///conf/context.xml");
```