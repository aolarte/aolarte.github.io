---
permalink: /:categories/:year/:month/:title.html
layout: single
title: Using Spring Integration for creating scalable distributed services backed
  by JMS (part 2)
date: '2015-06-17T16:49:00.002-07:00'
tags:
- integration
- jdbc
- jms
- esb
- spring
- h2
- spring-integration
- java
modified_time: '2016-05-31T17:02:27.885-07:00'
thumbnail: http://1.bp.blogspot.com/-PSlpvpgeB6Y/VYD9-AKK9BI/AAAAAAAAF-o/0MruKGTHgTA/s72-c/StoreInLibrary.gif
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-3274111061827674944
blogger_orig_url: http://www.javaprocess.com/2015/06/spring-integration-services-part2.html
---
In [part 1](/2015/05/spring-integration-services-part1.html) of this series, we explored how Spring Integration allows us to easily leverage some of the Enterprise Integration Patterns to deploy salable services. In this part we'll take a look at one more pattern that is very helpful, the [claim check pattern](http://www.enterpriseintegrationpatterns.com/StoreInLibrary.html). This patterns allows to store parts of message, to be retrieved at a later stage. This is illustrated on the following diagram:  

[![](http://1.bp.blogspot.com/-PSlpvpgeB6Y/VYD9-AKK9BI/AAAAAAAAF-o/0MruKGTHgTA/s1600/StoreInLibrary.gif)](http://1.bp.blogspot.com/-PSlpvpgeB6Y/VYD9-AKK9BI/AAAAAAAAF-o/0MruKGTHgTA/s1600/StoreInLibrary.gif)

  
In practice, this pattern allows us to avoid having to send very large messages across the JMS broker. Message brokers are designed to handle many small messages, and performance tends to degrade with larger messages. While there are ways of tweaking the different JMS brokers to work better in these kinds of workloads, the "Claim Check" pattern allows us to sidestep the issue entirely, by using a separate storage mechanism to hold parts of our messages. So how big is too big for a message? That's a very subjective matter, but good common sense will tell us that for example most large binary data should not be transferred through a JMS broker.  
Luckily Spring Integration provides out of the box support for this pattern. In fact, we can use the pattern without making any changes to our code. We do require some extra libraries, which we have added to the `pom.xml` file:  

{% highlight xml %}
<dependency>  
    <groupId>org.springframework.integration</groupId>  
    <artifactId>spring-integration-jdbc</artifactId>  
    <version>${spring.version}</version>  
</dependency>  
  
<dependency>  
    <groupId>com.h2database</groupId>  
    <artifactId>h2</artifactId>  
    <version>1.4.187</version>  
</dependency>  
  
<dependency>  
    <groupId>commons-dbcp</groupId>  
    <artifactId>commons-dbcp</artifactId>  
    <version>1.4</version>  
</dependency>
{% endhighlight %}  

We have added:  
  

1.  The Spring Integration JDBC jar. This provides, among other functionality, a JDBC backed message store.
2.  H2, a Java embedded database. In this database we will store our messages.
3.  Apache Commons DBCP, a connection pool to manage our JDBC connections.

  
Now let's look at the spring-int-server.xml file:  

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
       xmlns:jdbc="http://www.springframework.org/schema/jdbc"  
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
       xmlns:int="http://www.springframework.org/schema/integration"  
       xmlns:int-jms="http://www.springframework.org/schema/integration/jms"  
       xmlns:int-jdbc="http://www.springframework.org/schema/integration/jdbc"  
       xmlns:amq="http://activemq.apache.org/schema/core"  
       xsi:schemaLocation="  
   http://www.springframework.org/schema/beans  
   http://www.springframework.org/schema/beans/spring-beans.xsd  
   http://www.springframework.org/schema/jdbc  
   http://www.springframework.org/schema/jdbc/spring-jdbc.xsd  
   http://www.springframework.org/schema/integration  
   http://www.springframework.org/schema/integration/spring-integration.xsd  
   http://www.springframework.org/schema/integration/jms  
   http://www.springframework.org/schema/integration/jms/spring-integration-jms.xsd  
   http://www.springframework.org/schema/integration/jdbc  
   http://www.springframework.org/schema/integration/jdbc/spring-integration-jdbc.xsd  
   http://activemq.apache.org/schema/core  
   http://activemq.apache.org/schema/core/activemq-core.xsd  
  ">  
  
    <amq:broker id="activeMQBroker" useJmx="false" persistent="false">  
        <amq:transportConnectors>  
        <amq:transportConnector uri="tcp://localhost:61616" />  
        </amq:transportConnectors>  
    </amq:broker>  
  
    <bean id="h2DBServer" class="org.h2.tools.Server" <!-- 1 -->  
          factory-method="createTcpServer" init-method="start" destroy-method="stop">  
        <constructor-arg value="-tcp,-tcpAllowOthers,-tcpPort,8043" />  
    </bean>  
  
    <bean id="serverDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close"> <!-- 2 -->  
        <property name="driverClassName" value="org.h2.Driver"/>  
        <property name="url" value="jdbc:h2:tcp://localhost:8043/mem:test"/>  
        <property name="username" value="sa"/>  
        <property name="password" value=""/>  
    </bean>  
  
    <jdbc:initialize-database data-source="serverDataSource"> <!-- 3 -->  
        <jdbc:script location="classpath:org/springframework/integration/jdbc/schema-h2.sql"/>  
    </jdbc:initialize-database>  
  
    <int-jdbc:message-store id="serverMessageStore" data-source="serverDataSource"/> <!-- 4 -->  
  
    <bean id="orderServiceHandler" class="com.javaprocess.examples.integration.impl.OrderServiceHandler"/>  
  
    <int:channel id="inChannel" />  
  
    <int-jms:inbound-gateway request-channel="inChannel"  
                             request-destination="amq.outbound"  
                             concurrent-consumers="10"/>  
    <int:chain  input-channel="inChannel"> <!-- 5 -->  
        <int:service-activator ref="orderServiceHandler" method="processOrder"/>  
        <int:claim-check-in message-store="serverMessageStore"/> <!-- 6 -->  
    </int:chain>  
</beans>  
{% endhighlight %}

Now let's look at the changes:  

1.  We have an in memory instance of an H2 database. H2 is a very useful embedded database, which means that we can run it inside of our JVM process. No need to have a separate database server for this example. Both the server and the client will connect to this instance. Obviously in production you will choose something more robust. But for this example, this setup works well. It's worth noting that the H2 is started with the configuration options needed to accept outside connections over TCP.
2.  A datasource is created. This datasource will point to the database server created in point #1.
3.  A data base initialization element. Spring Integration requires some tables to be created in the database that will be used as a message store. Since our database will reside in memory, this will be run every time the server is started up.
4.  The message store we're using is a JDBC store, using the data source defined in #2.
5.  We now introduce the chain element. The chain element allows multiple filter or transformations to be chained together. In this case we're still delegating to the orderServiceHandler bean for processing as the first step in the chain.
6.  In the second step of the chain, we check in the message. This transformed will take the message (which must be serializable ), store in the message store, and return a UUID (Universally Unique Identifier). This UUID is sent to the client.

Now let's look at the client side. The client side is very similar to server (actually a bit simpler).  

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
       xmlns:int="http://www.springframework.org/schema/integration"  
       xmlns:int-jms="http://www.springframework.org/schema/integration/jms"  
       xmlns:int-jdbc="http://www.springframework.org/schema/integration/jdbc"  
       xsi:schemaLocation="  
   http://www.springframework.org/schema/beans  
   http://www.springframework.org/schema/beans/spring-beans.xsd  
   http://www.springframework.org/schema/integration  
   http://www.springframework.org/schema/integration/spring-integration.xsd  
   http://www.springframework.org/schema/integration/jms  
   http://www.springframework.org/schema/integration/jms/spring-integration-jms.xsd  
   http://www.springframework.org/schema/integration/jdbc  
   http://www.springframework.org/schema/integration/jdbc/spring-integration-jdbc.xsd  
  ">  
  
    <int:channel id="requestChannel"/>  
  
    <bean id="clientDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close"> <!-- 1 -->  
        <property name="driverClassName" value="org.h2.Driver"/>  
        <property name="url" value="jdbc:h2:tcp://localhost:8043/mem:test"/>  
        <property name="username" value="sa"/>  
        <property name="password" value=""/>  
    </bean>  
  
    <int-jdbc:message-store id="clientMessageStore" data-source="clientDataSource" /> <!-- 2 -->  
  
    <int:gateway id="orderService"  
                 service-interface="com.javaprocess.examples.integration.interfaces.IOrderService"  
                 default-request-channel="requestChannel"/>  
    <int:chain input-channel="requestChannel"> <!-- 3 -->  
        <int-jms:outbound-gateway request-destination="amq.outbound" extract-request-payload="true"/>  
        <int:claim-check-out message-store="clientMessageStore"/>  
    </int:chain>  
  
    <bean id="app" class="com.javaprocess.examples.integration.impl.App">  
        <property name="orderService" ref="orderService"/>  
    </bean>  
</beans>  
{% endhighlight %}

Let's look at the changes in this file:  
  
1.  A data source is created, using the same URL as in the server side. There's no need to start up the H2 server here, since we're using the H2 running inside the server process.
2.  A message store is created using the data source created in #1.
3.  A chain is created to handle messages in the `requestChannel`. This chain will receive a UUID, and retrieve the corresponding object from the message store.

To check out the code clone the following repository: <https://github.com/aolarte/spring-integration-samples.git>.

    git clone https://github.com/aolarte/spring-integration-samples.git  
    git checkout branches/part2  

Once you checkout the code, you can run the server and client process using Maven (in different terminals):  

```
mvn exec:java -Dexec.mainClass="com.javaprocess.examples.integration.main.Main" \
  -Dexec.args="server"  
```

```
mvn exec:java -Dexec.mainClass="com.javaprocess.examples.integration.main.Main" \
  -Dexec.args="client"
```

  
You will see the exact same output as in part 1. While that might seem anti-climatic, the important aspect is what is happening behind the scenes. The message is being stored in a database, and a UUID is sent across the wire. What we have implemented is a very simple Enterprise Service Bus. There's plenty of opportunities to use patterns like this.