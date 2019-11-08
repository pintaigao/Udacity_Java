# JAVA 性能优化，处理亿级流量架构笔记

## 一.基本框架

<img src="/Users/hptg/Documents/Project/Java/High_Concurrency_Solution/Resource/image-20191107193530836.png" width="50%"/><img src="/Users/hptg/Documents/Project/Java/High_Concurrency_Solution/Resource/5d5181520001c34019201080.jpg" width="50%"/>

<img src="/Users/hptg/Documents/Project/Java/High_Concurrency_Solution/Resource/image-20191107200051139.png" width="50%"/>



## 二. 压力测试

用性能压力测试来发现系统的瓶颈（有可能是数据库，有可能是写了什么代码拖慢了性能，也有可能是配置问题），**发现容量问题**，目前来说**50**个并发已经是瓶颈了，通常来说这是server端的configuration的问题

解决方法：

1.  **Spring-configuration-metadata.json** （ */2.0.5.RELEASE/spring-boot-autoconfigure-2.0.5.RELEASE.jar!/META-INF/spring-configuration-metadata.json）

    定位到：

    ```json
    {
    	"sourceType": "org.springframework.boot.autoconfigure.web.ServerProperties$Tomcat",
    	"defaultValue": 200, ===> defaultValue 默认200，可以增加到2000或更大
      "name": "server.tomcat.max-threads",
      "description": "Maximum number of worker threads.",
      "type": "java.lang.Integer"
    }
    ```

    所以，默认内嵌Tomact配置

    *   **server.tomcat.accept-count：等待队列长度，默认100**
    *   **server.tomcat.mac-connections：最大可连接数，默认10000**
    *   **server.tomcat.max-threads：最大工作线程数，默认200**
    *   **server.tomat.min-spare-threads：最小工作线程数，默认10**

    默认配置下，连接超过10000后出现拒绝连接情况

    默认配置下，触发的请求超过200+100后拒绝处理，所以服务端上线前一定要改配置，具体更改的方法：

    ```json
    // 在application.properties中：
    server.tomcat.accept-count = 1000
    server.tomcat.max-threads = 800 （4核8G的经验配置）
    server.tomat.min-spare-threads = 100
    ```

2.  **定制化内嵌Tomcat开发，优化keepAlive**

    什么是keepAlive？客户端向server发送http请求的时候，若带上keepAlive的请求头，则表明client希望和server建立keepAlive的连接（long pull request），实现复用连接的目的，用以解决每次http request完成后都要断开再连接所产生的耗时问题。
    
*   keepAliveTimeOut：多少毫秒后若客户端不响应，则断开keepalive连接
    *   maxKeepAliveRequests：对应的这个keepalive，多少次请求后，keepalive断开失效

    使用**WebServerFactoryCustomizer<ConfigurableServletWebServerFactory>**定制化内嵌Tomcat配置

    ```java
    //当Spring容器内没有TomcatEmbeddedServletContainerFactory这个bean时，会吧此bean加载进spring容器中
    @Component
    public class WebServerConfiguration implements WebServerFactoryCustomizer<ConfigurableWebServerFactory> {
        @Override
        public void customize(ConfigurableWebServerFactory configurableWebServerFactory) {
            //使用对应工厂类提供给我们的接口定制化我们的tomcat connector
            ((TomcatServletWebServerFactory) configurableWebServerFactory).addConnectorCustomizers(new TomcatConnectorCustomizer() {
                @Override
                public void customize(Connector connector) {
                    Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
                    //定制化keepalivetimeout,设置30秒内没有请求则服务端自动断开keepalive链接
                    protocol.setKeepAliveTimeout(30000);
                    //当客户端发送超过10000个请求则自动断开keepalive链接
                    protocol.setMaxKeepAliveRequests(10000);
                }
            });
        }
    }
    ```
    
    优点：
    
    *   允许修改除了application.properties暴露出来的properties之外的其他的配置
3.  **容量问题，响应时间变长TPS上不去**
  
  单Web容器上限
  
  *   线程数量：4核cpu 8G内存单进程调度线程数800-1000以上后，即花费巨大的时间在CPU调度上（content switch）
  *   等待队列长度：队列做缓冲池用，但也不能无限长，消耗内存，出队入队也消耗cpu
  
  这部分有关Mysql插入或查询事务优化的问题，下面会讲。
  
    
  

## 三. 分布式扩展

单机容量问题，水平扩展方案引入

*   nginx反向代理负载均衡
*   分布式会话管理
*   使用redis实现分布式会话存储



**Nginx 反向代理负载均衡**

首先，你要有钱，要能买相同的多台服务器（假设买了四台）



## 附录，一些常用的Shell命令


