AMQ使用

下载

ActiveMQ 5.16.3二进制版本  https://activemq.apache.org/components/classic/download/

发布于2021年8月份



环境要求

- 30M磁盘空间
- JRE 1.8及以上
- JAVA_HOME必须设置



启动

```shell
控制台启动
cd xxx/bin
./activemq console

后台进程启动、关闭
./activemq start
./activemq stop
```



日志在xxx/data/activemq.log文件。



启动成功后，管理后台

http://127.0.0.1:8161/admin/  用户名密码: admin/admin



默认监听 61616 端口(tcp)



默认的prefetch数据是1000



