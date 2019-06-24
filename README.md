[in progress]

## Aggregating logs of Spring Boot applications running on Docker with Elastic Stack

This post describes how to aggregate logs of Spring Boot applications running on Docker with Elastic Stack.

## What are logs and what are they meant for?

The [Twelve-Factor App methodology][12factor], a set of best practices for building _software as a service_ applications, define logs as _a stream of aggregated, time-ordered events collected from the output streams of all running processes and backing services_ which _provide visibility into the behavior of a running app._

These best practices recommend that logs should be treated as _event streams_:

> A twelve-factor app never concerns itself with routing or storage of its output stream. It should not attempt to write to or manage logfiles. Instead, each running process writes its event stream, unbuffered, to `stdout`. During local development, the developer will view this stream in the foreground of their terminal to observe the app’s behavior.
>
> In staging or production deploys, each process’ stream will be captured by the execution environment, collated together with all other streams from the app, and routed to one or more final destinations 
for viewing and long-term archival. These archival destinations are not visible to or configurable by the app, and instead are completely managed by the execution environment.

With that in mind, the log event stream for an application can be routed to a file or watched via realtime `tail` in a terminal or, preferrably, sent to a log indexing and analysis system such as Elastic Stack.

## What is Elastic Stack?

Elastic Stack is a group of open source products from Elastic designed to help users take data from any type of source and in any format and search, analyze, and visualize that data in real time. The solution was formerly known as ELK Stack, in which the letters in the name stood for the products in the group: Elasticsearch, Logstash and Kibana. A fourth product, Beats, was subsequently added to the stack, rendering the potential acronym unpronounceable. 

Let's have a quick look at each component of the Elastic Stack.

### Elasticsearch

Elasticsearch is a real-time, distributed storage, JSON-based search, and analytics engine designed for horizontal scalability, maximum reliability, and easy management. It can be used for many purposes, but one context where it excels is indexing streams of semi-structured data, such as logs or decoded network packets.

### Kibana

Kibana is an open source analytics and visualization platform designed to work with Elasticsearch. Kibana can be used to search, view, and interact with data stored in Elasticsearch indices. You can easily perform advanced data analysis and visualize your data in a variety of charts, tables, and maps.

### Beats

Beats are open source data shippers that can be installed as agents on servers to send operational data directly to Elasticsearch or via Logstash, where it can be further processed and enhanced. There's a number of Beats for different purposes:

- Filebeat: Log files
- Metricbeat: Metrics
- Packetbeat: Network data
- Heartbeat: Uptime monitoring

See the full list in the Beats page (https://www.elastic.co/products/beats). As we intend to ship log files, we'll use Filebeat.

### Logstash

Logstash is a powerful tool that integrates with a wide variety of deployments. It offers a large selection of plugins to help you parse, enrich, transform, and buffer data from a variety of sources. If the data requires additional processing that is not available in Beats, then Logstash must be added to the deployment.

### Putting the pieces together

The following diagram illustrates how the components of Elastic Stack interact with each other:

![Elastic Stack][img.elastic-stack]

- File beat will collect data from the log files and will ship it to Logststash. Logstash will enhance the data and send it to Elasticsearch for storage and indexing. Finally, the data can be visualized in Kibana.

## Overview of our micro services

For this example, let's consider two micro services:

![Movie and review services][img.services]

The `movie-service` manages information related to movies while the `review-service` manages information related to the reviews of each movie. For simplicity, we'll support only `GET` requests.

## Tracing the requests

Unlike in a monolithic application, a single business operation is split across a number of services. When something goes wrong, it's important to be able to trace each operation across the services.

To trace the request across multiple services, we'll use [Spring Cloud Sleuth][spring-cloud-sleuth]. Implements a distributed tracing solution for Spring Cloud. For us, application developers, Sleuth is invisible, and all your interactions with external systems should be instrumented automatically.

Spring Cloud Sleuth implements a distributed tracing solution for Spring Cloud.

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-sleuth</artifactId>
            <version>${spring-cloud-sleuth.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-sleuth</artifactId>
    </dependency>
</dependencies>
```

[to be continued]

## Creating the log appender

See https://github.com/logstash/logstash-logback-encoder

Using `LogstashEncoder`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <appender name="jsonConsoleAppender" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>

    <root level="INFO">
        <appender-ref ref="jsonConsoleAppender"/>
    </root>
    
</configuration>
```

If you want greater flexibility in the JSON format and data included in logging, we can use the LoggingEventCompositeJsonEncoder.

These encoders/layouts are composed of one or more JSON providers that contribute to the JSON output. No providers are configured by default in the composite encoders/layouts. You must add the ones you want.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <springProperty scope="context" name="springAppName" source="spring.application.name"/>

    <appender name="jsonConsoleAppender" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <logLevel/>
                <threadName/>
                <loggerName/>
                <pattern>
                    <pattern>
                        {
                          "application_name": "${springAppName:-}",
                          "tracing": {
                            "trace_id": "%X{X-B3-TraceId:-}",
                            "span_id": "%X{X-B3-SpanId:-}",
                            "parent_span_id": "%X{X-B3-ParentSpanId:-}",
                            "exportable": "%X{X-Span-Export:-}"
                          },
                          "pid": "${PID:-}",
                          "class": "%logger{40}"
                        }
                    </pattern>
                </pattern>
                <message/>
                <stackTrace/>
            </providers>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="jsonConsoleAppender"/>
    </root>
    
</configuration>
```xml


## Elastic Stack with Docker

To get up and running quicker, we'll run our application in Docker containers. As we need multiple containers, we'll use Docker Compose.

With Docker Compose, we use a YAML file to configure our application’s services.

![Elastic Stack][img.elastic-stack-docker]

### Building the Spring Boot applications

- Change to the `review-service` folder: `cd review-service`
- Build the application and create a Docker image: `mvn clean install`

```java
...
[INFO] Successfully tagged cassiomolin/review-service:latest
[INFO] 
[INFO] Detected build of image with id e69b67fcd80a
[INFO] Building jar: /Users/cassiomolin/Projects/log-aggregationt/review-service/target/review-service-1.0-SNAPSHOT-docker-info.jar
[INFO] Successfully built cassiomolin/review-service:latest
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  11.236 s
[INFO] Finished at: 2019-06-20T22:37:33+01:00
[INFO] ------------------------------------------------------------------------
```

- Change to the parent folder: `cd ..`
- Change to the `movie-service` folder: `cd movie-service`
- Build the application and create a Docker image: `mvn clean install`

```java
...
[INFO] Successfully tagged cassiomolin/movie-service:latest
[INFO] 
[INFO] Detected build of image with id 0cc25953d5ec
[INFO] Building jar: /Users/cassiomolin/Projects/log-aggregation/movie-service/target/movie-service-1.0-SNAPSHOT-docker-info.jar
[INFO] Successfully built cassiomolin/movie-service:latest
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  10.045 s
[INFO] Finished at: 2019-06-20T22:39:48+01:00
[INFO] ------------------------------------------------------------------------
```

### Spinning up the containers

- Change to the parent folder: `cd ..`
- Start Docker Compose: `docker-compose up`

### Checking the logs in Kibana

- Perform a `GET` request to the `movie-service`: `http://localhost:8001/movies/2`
- Open Kibana: `http://localhost:5601`
  - Click the management icon
  - Create an index pattern
  - Visualize the logs
  
  
  [img.services]: /misc/img/diagrams/services.png
  [img.elastic-stack]: /misc/img/diagrams/elastic-stack.png
  [img.elastic-stack-docker]: /misc/img/diagrams/services-and-elastic-stack-with-docker.png


  [12factor]: https://12factor.net
  [spring-cloud-sleuth]: https://spring.io/projects/spring-cloud-sleuth
