## 背景

owner-pro活动中，他们使用hibernate连接mysql8，本地测试没有问题，但是部署到服务器上问题频发

- MySQL 8.0更改了默认的认证插件，从`mysql_native_password`更改为`caching_sha2_password`。这可能导致一些客户端工具，如Hibernate（通过其JDBC驱动程序）无法连接到MySQL 8.0服务器，因为它们可能不支持新的认证插件。
- lombok本地可以生效，但是打包时就无法生效，导致一堆方法找不到