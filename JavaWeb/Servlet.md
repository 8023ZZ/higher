# Servlet总结
Servlet负责接收用户请求HttpServletRequest，在doGet()，doPost()中进行相应的处理，并将回应的HttpServletResponse返回给用户。
一个Servlet类只会有一个实例，初始化时调用init()方法，销毁时调用destroy()方法。Servlet需要在web.xml中进行配置。
一个Servlet可以设置多个URL访问，Servlet非线程安全。

# Servlet生命周期
Web容器加载Servlet并将其实例化后，Servlet生命周期开始，容器运行其init()方法进行Servlet初始化，请求到达时调用Servlet的service()方法，service()方法会根据需要调用与请求对应的doGet或doPost等方法。当服务器关闭或者项目卸载时服务器会将Servlet实例销毁，此时会调用Servlet的destroy方法。init方法和destroy方法只会执行一次，service方法客户端每次请求Servlet都会执行。

# 转发和重定向
转发是服务器行为，不更改地址栏url，转发页面和转发到的页面可以共享数据，效率高。
重定向是客户端行为，更改地址栏url，不能共享数据，效率较低。

