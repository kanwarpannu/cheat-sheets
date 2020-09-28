# Logging 101 using spring boot  

SLF4J is the mostly common logger used with spring boot. It is an abstract logger, so a runtime logger has to be provided. It's a facade design pattern.  
 
* Logback is the default logger implementation for SLF4J when used with spring boot.  
* Logger is usually directly attached to a class using annotation like `@Slf4j`.  
* Logger emits a logging event to an Appender. Some common appender are console, file/rolling file, or logstash tcp socket. Appender can also apply filters to the logs.  
* Appender are attached to layouts also called `Encoders`. We usually define the pattern of logs in the encoder.  

## Common logging levels  
Common logging levels are (from most verbose to least):  
1. Trace
2. Debug
3. Info
4. Warn
5. Error

## MDC  
MDC is a **USER** provided contextual information to uniquely identify a message.  

Spring sleuth by default adds the following 4 mdc keys to logs(also called sleuth baggage):  
1. AppName  
2. TraceId(constant throughout request)  
3. SpanId(constant in a service)  
4. Exportable(to zipkin)  
  
[Here](http://logback.qos.ch/manual/mdc.html) is link to better explain MDC.  

## Configuring logback with spring boot  
`spring-boot-starter` dependency automatically imports `spring-boot-starter-logging` which in-turn imports `logback-classic`, so no other dependencies are required.  

To customize logback, a config can be added to project by creating file `logback.xml` or `logback-spring.xml` in resources folder.  

[Here](https://gist.github.com/kanwarpannu/3d91cb4768d1922e2b9a52766b64b869) is a sample `logback-spring.xml` for springboot projects.  