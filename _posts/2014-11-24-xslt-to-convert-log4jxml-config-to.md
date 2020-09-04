---
layout: post
title: XSLT to convert log4j.xml config to logback.xml config
date: '2014-11-24T21:07:00.000+03:00'
author: rpuchkovskiy
tags:
- log4j log4j.xml logback logback.xml convert configuration xslt
modified_time: '2017-03-11T18:34:40.767+03:00'
blogger_id: tag:blogger.com,1999:blog-4160252863216482620.post-6502637392876481691
blogger_orig_url: https://rpuchkovskiy.blogspot.com/2014/11/xslt-to-convert-log4jxml-config-to.html
excerpt_separator: <!--more-->
---

[Logback](http://logback.qos.ch/) is a modern logging framework widely used nowadays. It has a drop-in replacement for
a <a href="http://logging.apache.org/log4j/1.2/" target="_blank">log4j 1.2</a> (the previous favorite):
logback-classic. But a little problem arises if you have an application which uses log4j and want to migrate it to
logback: configuration.

<!--more-->

log4j has two configuration formats: log4j.properties and log4j.xml. For the former, everything is fine: there is
<a href="http://logback.qos.ch/translator/" target="_blank">log4j.properties translator</a> script.

But for log4j.xml there doesn't seem to be any convertion tool available, and logback.xml does not understand
the unconverted log4j.xml files.

So <a href="https://github.com/rpuch/log4j2logback/blob/master/log4j-to-logback.xsl" target="_blank">here</a> is an
XSLT which allows to convert log4j.xml files to the corresponding logback.xml configurations.

And here is an example. We have the following log4j.xml file:

```
<?xml version="1.0" encoding="UTF-8"?>

<log4j:configuration xmlns:log4j="http://jakarta.apache.org/log4j/">
    <appender name="default" class="org.apache.log4j.ConsoleAppender">
        <param name="target" value="System.out"/>
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern" value="%d %t %p [%c] - %m%n"/>
        </layout>
    </appender>

    <appender name="log4jremote" class="org.apache.log4j.net.SocketAppender">
        <param name="RemoteHost" value="10.0.1.10"/>
        <param name="Port" value="4712"/>
        <param name="ReconnectionDelay" value="10000"/>
        <layout class="org.apache.log4j.PatternLayout">
            <param name="ConversionPattern"
                   value="[my-host][%d{ISO8601}]%c{1}%n%m%n"/>
        </layout>
        <filter class="org.apache.log4j.varia.LevelRangeFilter">
            <param name="LevelMin" value="ERROR"/>
            <param name="LevelMax" value="FATAL"/>
        </filter>
    </appender>

    <logger name="com.somepackage">
        <level value="INFO"/>
    </logger>

    <root>
        <level value="INFO"/>
        <appender-ref ref="default"/>
    </root>
</log4j:configuration>
```

Using Xalan to convert it:

`java -cp xalan.jar:xercesImpl.jar:serializer.jar:xml-apis.jar org.apache.xalan.xslt.Process -IN log4j.xml -XSL log4j-to-logback.xsl -OUT logback.xml`

And here is the result:

```
<?xml version="1.0" encoding="UTF-8"?><configuration scanPeriod="10 seconds" scan="true">
    <appender name="default" class="ch.qos.logback.core.ConsoleAppender">
        <target>System.out</target>
        <encoder>
            <pattern>%d %t %p [%c] - %m%n</pattern>
        </encoder>
    </appender>

<appender name="log4jremote" class="ch.qos.logback.classic.net.SocketAppender">
        <remoteHost>10.0.1.10</remoteHost>
        <port>4712</port>
        <reconnectionDelay>10000</reconnectionDelay>
        <!-- this is NOT needed tor this logger, so it is commented out -->
        <!--
        <layout>
            <pattern>[my-host][%d{ISO8601}]%c{1}%n%m%n</pattern>
        </layout>
                    -->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
    </appender>

<logger name="com.somepackage" level="INFO"/>

<root level="INFO">
        <appender-ref ref="default"/>
    </root>
</configuration>
```

The script translates the basic loggers (console/email/file/syslog/socket).

Here is the github repository: https://github.com/rpuch/log4j2logback