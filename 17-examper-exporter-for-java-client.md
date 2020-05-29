## 背景

由于`GBase`只支持`jdbc`连接方式通过数据字典获取监控指标，这就要求外置GBase监控需要基于`prometheus/client_java`，使用java语言构建。exporter的对外提供的能力是以http方式对外提供既定格式(v0.0.4)的metrics。

## 概要

`client_golang`版本使用时，使用`net/http`包提供的http服务对外暴露metrics，而java本身没有提供`http-server`的功能，使用`springboot`等框架又会进入不必要的依赖和功能，增加资源消耗。因此我们可以选择`jetty-servlet。`

## 代码

### Maven配置

prometheus的java client基础依赖包括simpleclient和simpleclient_servlet。

```xml
<dependencies>
    <dependency>
        <groupId>io.prometheus</groupId>
        <artifactId>simpleclient</artifactId>
        <version>0.5.0</version>
    </dependency>
     
    <!-- optional, for default hotspot jvm metrics -->
    <dependency>
        <groupId>io.prometheus</groupId>
        <artifactId>simpleclient_hotspot</artifactId>
        <version>0.5.0</version>
    </dependency>
 
    <dependency>
        <groupId>io.prometheus</groupId>
        <artifactId>simpleclient_servlet</artifactId>
        <version>0.5.0</version>
    </dependency>
 
    <!-- use jetty as http server -->
    <dependency>
        <groupId>org.eclipse.jetty</groupId>
        <artifactId>jetty-servlet</artifactId>
        <version>9.4.11.v20180605</version>
    </dependency>
</dependencies>
```

### 创建指标

```java
package io.sanwishe.metrics;
 
import io.prometheus.client.Counter;
import io.prometheus.client.Gauge;
import io.prometheus.client.Histogram;
import io.prometheus.client.Summary;
 
import java.util.Random;
 
public class Collector {
    private final static String NAME_SPACE = "gbase";
    private final static String SUB_SYSTEM = "business";
 
    // example metrics of Gauge
    private final static Gauge GAUGE = Gauge.build()
            .namespace(NAME_SPACE)
            .subsystem(SUB_SYSTEM)
            .name("up")
            .help("Example metrics for testing")
            .labelNames("version", "node")
            .register();                            // 注册metrics到CollectorRegistry.defaultRegistry
 
 
    // example metrics of Counter
    private final static Counter COUNTER = Counter.build()
            .namespace(NAME_SPACE)
            .subsystem(SUB_SYSTEM)
            .name("scrape_total")
            .help("Example metrics for testing")
            .labelNames("node")
            .register();
 
 
    // example metrics of histogram
    private final static Histogram HISTOGRAM = Histogram.build()
            .namespace(NAME_SPACE)
            .subsystem(SUB_SYSTEM)
            .name("web_request_elapsed")
            .help("Example histogram for testing")
            .labelNames("node")
            .buckets(0.3, 0.5, 1.0, 3.0, 5.0)
            .register();
 
    // example metrics of summary
    private final static Summary SUMMARY = Summary.build()
            .namespace(NAME_SPACE)
            .subsystem(SUB_SYSTEM)
            .name("db_query_elapsed")
            .help("Example summary for testing")
            .quantile(0.5, 0.01)
            .quantile(0.9, 0.01)
            .labelNames("node")
            .register();
 
 
    public void collect() {
        GAUGE.labels("8a", "10.62.127.88").set(1.0);
        COUNTER.labels("10.62.127.88").inc();
        HISTOGRAM.labels("10.62.127.88").observe(new Random().nextDouble() % 10.0);
        SUMMARY.labels("10.62.127.88").observe(new Random().nextDouble());
    }
}
```

### 创建servlet

创建servlet，以响应路径/metrics的请求，servlet继承自HTTPServlet，只需覆盖doGet方法即可。

```java
package io.sanwishe.handle;
 
import io.sanwishe.metrics.Collector;
import io.prometheus.client.CollectorRegistry;
import io.prometheus.client.exporter.common.TextFormat;
 
import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.Writer;
import java.util.Arrays;
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;
 
public class MetricsServlet extends HttpServlet {
 
    private final CollectorRegistry registry;
 
    public MetricsServlet() {
        this.registry = CollectorRegistry.defaultRegistry;
    }
 
    @Override
    protected void doGet(final HttpServletRequest req, final HttpServletResponse resp) throws ServletException, IOException {
        resp.setStatus(HttpServletResponse.SC_OK);                            //响应码200
        resp.setContentType(TextFormat.CONTENT_TYPE_004);                     //格式描述，详见https://wiki.zte.com.cn/pages/viewpage.action?pageId=497064614
 
        // 采集指标
        new Collector().collect();
 
        Writer writer = resp.getWriter();
        try {
            //将注册到CollectorRegistry.defaultRegistry的metrics格式到resp中
            TextFormat.write004(writer, registry.filteredMetricFamilySamples(parse(req)));
            writer.flush();
        } finally {
            writer.close();
        }
    }
 
    private Set<String> parse(HttpServletRequest req) {
        String[] includedParam = req.getParameterValues("name[]");
        if (includedParam == null) {
            return Collections.emptySet();
        } else {
            return new HashSet<String>(Arrays.asList(includedParam));
        }
    }
}
```

### 启动httpserver

```java
package com.sanwishe;
 
import com.sanwishe.handle.DefaultServlet;
import com.sanwishe.handle.MetricsServlet;
import org.eclipse.jetty.server.Server;
import org.eclipse.jetty.servlet.ServletContextHandler;
import org.eclipse.jetty.servlet.ServletHolder;
 
import java.net.InetSocketAddress;
 
public class GbaseExporter {
    public static void main(String[] args) {
        // create http server via jetty server
        Server server = new Server(InetSocketAddress.createUnresolved("0.0.0.0", 9296));
 
        ServletContextHandler contextHandler = new ServletContextHandler();
        contextHandler.setContextPath("/");
//      将上一节中的MetricsServlet绑定到/merices路径
        contextHandler.addServlet(new ServletHolder(new MetricsServlet()), "/metrics");
 
        server.setHandler(contextHandler);
 
//        默认的jvmmetrics，需要使用依赖：simpleclient_hotspot
//        DefaultExports.initialize();
 
        // Start the webserver.
        try {
            server.start();
        } catch (Exception e) {
            e.printStackTrace();
        }
        try {
            server.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
