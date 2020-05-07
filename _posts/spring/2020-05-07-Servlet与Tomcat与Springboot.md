---
layout: article
title: 2020-05-07-Servlet与Tomcat与Springboot
date: 2020/5/7 21:17
categories: [Java, Springboot]
tags: [Java, Springboot]
root-path: ../..
---

# Servlet 相关基础

## Servelt是什么

`java.servlet.Servlet` 接口定义了一套处理网络请求的规范。所有实现servlet的类，都需要实现它那五个方法，其中最主要的是两个生命周期方法 `init()`和`destroy()`，还有一个处理请求的`service()`，也就是说，所有实现servlet接口的类，或者说，所有想要处理网络请求的类，都需要回答这三个问题：

- 1) 初始化时要做什么

- 2) 销毁时要做什么

- 3) 接受到请求时要做什么

servlet不会直接和客户端打交道，tomcat才直接和客户端打交道，tomcat监听了端口，请求过来后，根据url等信息，确定要将请求交给哪个servlet去处理，然后调用那个servlet的service方法，service方法返回一个response对象，tomcat再将这个response返回给客户端。

## Servlet 生命周期

Servlet的生命周期，指Servlet的对象从被创建到被销毁的过程。

Servlet的生命周期方法：

 	1. 构造器
     - Servlet第一次处理请求时，会调用构造器，来创建Servlet实例。
     - 只会调用一次，Servlet是单例模式，他是以多线程的方式调用service()方法.
     - Servlet不是线程安全，所以尽量不要再service()方法中操作全局变量。
 	2. init()方法
     -  构造器调用之后马上被调用，用来初始化Servlet，只会调用一次。
 	3. service()方法
     - Servlet每次处理请求时都会调用service()方法，用来处理请求，会调用多次。
 	4. destroy()方法
     - Servlet对象销毁前(WEB项目卸载时)调用，用来做一些收尾工作，释放资源。

## Servlet 如何工作的

当用户从浏览器向服务器发起一个请求，通常会包含如下信息：[http://hostname:port/contextpath/servletpath]()。

- hostname 和 port 是用来与服务器建立 TCP 连接。
- `/contextpath/servletpath`即URL才是用来选择服务器中的哪个子容器来服务用户的请求。

## Session

- Session的三种工作方式
  - 基于 URL Path Parameter，默认就支持；
  - 基于 Cookie，如果你没有修改 Context 容器和 cookies 标识的话，默认也是支持的；
  - 基于 SSL，默认不支持，只有 `connector.getAttribute(“SSLEnabled”)` 为 TRUE 时才支持
- SSL(Secure Sockets Layer 安全套接层)协议,及其继任者TLS（Transport Layer Security传输层安全）协议，是为网络通信提供安全及数据完整性的一种安全协议。TLS与SSL在传输层对网络连接进行加密，用于保障网络数据传输安全，利用数据加密技术，确保数据在网络传输过程中不会被截取及窃听。SSL协议已成为全球化标准，所有主要的浏览器和WEB服务器程序都支持SSL协议，可通过安装SSL证书激活SSL协议。
   SSL 证书就是遵守 SSL协议的服务器数字证书，由受信任的证书颁发机构（CA机构），验证服务器身份后颁发，部署在服务器上，具有网站身份验证和加密传输双重功能。
   如果是第三种情况的话将会根据 `javax.servlet.request.ssl_session` 属性值设置 Session ID。
- 有了 Session ID 服务器端就可以创建 `HttpSession` 对象了，第一次触发是通过 `request. getSession()` 方法，如果当前的 Session ID 还没有对应的 HttpSession 对象那么就创建一个新的，并将这个对象加到 `org.apache.catalina.Manager` 的 sessions 容器中保存，Manager 类将管理所有 Session 的生命周期，Session 过期将被回收，服务器关闭，Session 将被序列化到磁盘等。只要这个 HttpSession 对象存在，用户就可以根据 Session ID 来获取到这个对象，也就达到了状态的保持。
- Session 的致命弱点是不容易在多台服务器之间共享。

## Servlet中的Listener

- 整个 Tomcat 服务器中 Listener 使用的非常广泛，它是基于【观察者模式】设计的。
   Listener 的设计对开发 Servlet 应用程序提供了一种快捷的手段，能够方便的从另一个【纵向维度控制程序和数据】。

- 目前 Servlet 中提供了 6 种两类事件的观察者接口

  - 4 个 EventListeners 类型：ServletContextAttributeListener、ServletRequestAttributeListener、ServletRequestListener、HttpSessionAttributeListener 和 
  - 2 个 LifecycleListeners 类型：ServletContextListener、HttpSessionListener。

  需要注意的是 ServletContextListener 在容器启动之后就不能再添加新的，因为它所监听的事件已经不会再出现。

  在Springboot可以查看 `org.springframework.boot.web.servlet.ServletListenerRegistrationBean`

## 相关接口

- `javax.servlet.ServletConfig`

  当前Servlet的配置信息，每个Servlet都有与其唯一对应的ServletConfig

- `javax.servlet.ServletContext`

  当前的WEB应用，一个WEB应用对应一个唯一的ServletContext对象，ServletContext对象再项目启动时创建，在项目卸载时销毁。

## GenericServlet

- 通用Servlet的父类、抽象类【继承Servlet和ServletConfig接口】
- 相比Servlet接口，GenericServlet更简单一些，但实际使用的HttpServlet

## HttpServlet

- HttpServlet继承GenericServlet

- HttpServlet重写了service方法

  - 在该方法中将ServletRequest和ServletResponse强转为了HttpServletRequest和HttpServletResponse
  - 调用重载的service方法，并将刚刚强转得到的对象传递到重载的方法中

- 重载`service(HttpServletRequest request , HttpServletResponse response)`

  - 在方法中获取请求的方式（get或post）

  - 在根据不同的请求方式去调用不同的方法：

    ​    如果是GET请求，则调用`doGet(HttpServletRequest request , HttpServletResponse response)`

    ​    如果是post请求，则调用`doPost(HttpServletRequest request , HttpServletResponse response)`

    等等

- 结论

  当通过继承HttpServlet来创建一个Servlet时，我们只需要根据要处理的请求的类型，来重写不同的方法。

  - 处理get请求，则重写`doGet()`
  - 处理post请求，则重写`doPost()`

## HttpServletRequest

- 作用

  浏览器发送给服务器的请求报文。

- 获取

  该对象由Tomcat服务器创建，最终作为参数传递到doGet或doPost方法中，我们可以在这两个方法中直接使用。

- 功能
  - 获取用户发送的请求参数：`request.getParameter("username")`
  - 获取项目的名字(用来设置绝对路径)：`request.getContextPath()`
  - 作为一个域对象，在不同的WEB资源之间共享数据。
  - 请求的转发：`request.getRequestDispatcher("target.html").forward(request, response)`

## HttpServletResponse

- 作用

  服务器发送给浏览器的响应报文。

- 获取

  该对象由Tomcat服务器创建，最终作为参数传递到doGet或doPost方法中，我们可以在这两个方法中直接使用。

- 功能

  - 响应给浏览器一个网页或者是网页片段(设置的是响应报文的响应体)：`response.getWriter("");`
  - 请求的重定向：`response.sendRedirect("target.html");`

- 额外-转发和重定向

  |              | 转发       | 重定向 |
  | ------------ | ---------- | ------ |
  | 请求次数     | 1          | 2      |
  | 发生的位置   | 服务器内部 | 浏览器 |
  | 浏览器地址栏 | 不改变     | 改变   |
  | 浏览器的感知 | 不知道     | 知道   |

## 字符编码

当用户通过表单向Servlet发送中文请求参数时，Servlet获取到内容会产生乱码

当Servlet向浏览器响应中文内容时，也会产生乱码。

浏览器和服务器之间通信时，中文内容时不能直接发送的，需要对中文进行编码。

- 编码：将字符转换为二进制码的过程叫编码。
- 解码：将二进制码转换为普通字符的过程叫解码。

编码和解码所采用的规则我们称为字符集。

- 乱码问题
  - 根本原因：编码和解码所采用的字符集不同。
  - 解决方法：统一编码和解码的字符集为UTF-8。

 常见字符集：ASCII、ISO8859-1、GBK、GB2312、UTF-8

- 请求编码
  - 请求是浏览器发送给服务器的。
  - 浏览器 --> 服务器
  - 浏览器编码：浏览器的会自动使用网页的字符集对参数进行编码
    - UTF-8的张三：%E5%BC%A0%E4%B8%89
    - GBK的张三：%D5%C5%C8%FD

- 服务器解码

  - post请求

    - request解码时默认字符集时iso8859-1，但是iso压根就不支持中文

    - post请求在servlet中解码，所以只需要指定request的字符集即可。

    - 可以通过如下方法，来设置request的字符集：`request.setCharacterEncoding("utf-8");`

      注意：该方法要在`request.getParameter()`第一次调用之前调用

  - get请求

    - get请求是通过url地址传递请求参数，url中的请求参数将会被Tomcat服务器自动解码。
    -  Tomcat的默认编码是iso8859-1，但是iso压根就不支持中文，所以必然乱码。
    - 只需要修改Tomcat的解码的默认字符集，修改配置文件server.xml
    - 在server.xml的Connector标签中（改端口号的那个标签）添加如下属性：`URIEncoding="utf-8"`
    - 修改完配置文件以后，get请求的编码就不用再处理的，但是post请求还是老样子。

- 响应编码

  - 响应是服务器发送给浏览器

  - 服务器 --> 浏览器

  - 服务器 编码

    - 指定服务器的编码字符集为UTF-8。
    - 指定response的字符集：`response.setCharacterEncoding("utf-8");`
    - 虽然已经指定了response的字符集为utf-8，但是浏览器并不是用utf-8解码。浏览器默认使用gb2312解码的，所以依然乱码，只不过没有那么乱。

  - 浏览器 解码

    - 浏览器的解码字符集可以通过浏览器来设置（不靠谱）

    - 可以通过服务器来告诉浏览器，内容的编码格式为utf-8

    - 可以通过一个响应头来告诉浏览器，内容的编码格式：`Content-Type:text/html;charset=utf-8`

    - 通过response的方法，来设置响应头：` response.setHeader("Content-Type", "text/html;charset=utf-8");`

    - 解决方案：

      1) 设置响应头`response.setHeader("Content-Type", "text/html;charset=utf-8");`

      2) 设置response的编码格式 `response.setCharacterEncoding("utf-8");`

      3) 当设置Content-Type这个响应头时，服务器会自动使用响应头中的字符集为内容编码。最终方案：`response.setContentType("text/html;charset=utf-8");`

- 总结

  - post请求：在`request.getParameter()`方法第一次调用之前，调用如下代码：`request.setCharacterEncoding("utf-8");`

  - get请求：修改server.xml配置文件。如果是springboot的配置，可以看看配置类 `org.springframework.boot.autoconfigure.web.ServerProperties`

    ```xml
    <Connector URIEncoding="utf-8" connectionTimeout="20000" port="8080" protocol="HTTP/1.1" redirectPort="8443"/>
    ```

  - 响应：设置一个Content-Type响应头`response.setContentType("text/html;charset=utf-8");`

- 进步一学习

  - Get是URL解码方式。默认解码格式是Tomcat8编码格式。所以URL解码是UTF-8，
    覆盖掉了request容器解码格式
  - Post是实体内容解码方式。默认解码格式是request编码格式。与Tomcat8编码格式无关
  - tomcat服务器中Response容器默认以ISO8859-1的编码解析数据，因此如果需要在参数中解析中文，需要设置``response.setCharacterEncoding(“utf-8”);`
  - Post得到前台数据：（Request容器默认是gbk格式）
    `request.setCharacterEncoding(“utf-8”);`
  - GET得到前台数据：（不需要设置编码格式，默认是按照tomcat服务器的编码格式）
    `System.out.println(request.getParameter(“name”));`
  - GET POST给前台传数据
    `response.setCharacterEncoding(“utf-8”);`
    `response.getWriter().write(“我爱你”);`

## 路径问题

- URI和URL

  - URL是URI的一种实现，也是URI最常见的实现方式。
  - URI有两种实现方式URL和URN，URN用的很少

- URL地址的格式

  [http://主机名:端口号/项目名/资源路径/资源名](http://xn--:-dr6az4pkc1er25kc32a/项目名/资源路径/资源名)

- 相对路径和绝对路径

  - 相对路径

    - 之前使用的路径全都是相对路径：
    - 所谓的相对路径指相对于当前资源所在路径：[http://主机名:端口号/项目名/资源路径/](http://xn--:-dr6az4pkc1er25kc32a/项目名/资源路径/)

    由于转发的出现，相对路径会经常发生变化，容易出现错误的链接，所以**在开发中一般不使用相对路径，而是使用绝对路径。**

  - 绝对路径

    - 绝对路径使用/开头  

    - 由浏览器解析的绝对路径中的/代表的是服务器的根目录：[http://主机名:端口号/](http://xn--:-dr6az4pkc1er25kc32a/)

      注意：需要加上项目名

    - 由服务器解析的绝对路径中的/代表的项目的根目录：[http://主机名:端口号/项目名/](http://xn--:-dr6az4pkc1er25kc32a/项目名/)

      注意：不要加项目名

  - 转发的路径由服务器解析，设置绝对路径时不需要加项目名

  - 重定向的路径由浏览器解析，设置绝对路径时需要加上项目名

- 常见的路径

  - url-pattern 和 转发的路径

    - url-pattern和转发中的路径都是由服务器解析的，根目录是项目的根目录：[http://主机名:端口号/项目名/](http://xn--:-dr6az4pkc1er25kc32a/项目名/)。

      所以这两个路径不需要加项目名

  - 重定向的路径 和 页面中的路径

    -  重定向和页面中的路径（HTML标签中的路径），由浏览器解析的，根目录是服务器的根目录：[http://主机名:端口号/](http://xn--:-dr6az4pkc1er25kc32a/)

      所以这个两个路径必须加上项目名

# Servlet 和 Tomcat



# Tomcat 和 Springboot

# Servlet 和 Springboot