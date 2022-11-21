---
permalink: /:categories/:year/:month/:title.html
title: Using Apache Camel and CDI for creating scalable distributed services backed
  by JMS (part 1)
date: '2015-10-19T16:25:00.004-07:00'
tags:
- integration
- jms
- dependency injection
- cdi
- camel
- java
modified_time: '2016-05-31T17:02:27.894-07:00'
thumbnail: http://4.bp.blogspot.com/-O6DqPU3G4k0/VU95xFoSXOI/AAAAAAAAFMc/yaSvcwYomKg/s72-c/RequestReply.gif
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-7810538677627378569
blogger_orig_url: http://www.javaprocess.com/2015/10/distributed-services-with-apache-camel-part1.html
---

[![](http://camel.apache.org/images/camel-box-small.png)](http://camel.apache.org/images/camel-box-small.png)As an exercise and follow up to my [Spring Integration post](http://www.javaprocess.com/2015/05/spring-integration-services-part1.html), and after a discussion with a coworker, I decided to reimplement the test application using [Apache Camel](http://camel.apache.org/) and CDI (Context and Dependency Injection). I will show some the basic pieces that make the solution work, as well as doing a side by side comparison with the XML from the Spring Integration solution.  
  
The main business logic of the application is an almost exact copy of the Spring Integration example (`Application.java`, `OrderServiceHandler.class`, etc...). The only difference is in the annotations, since we're using JSR-299 CDI annotations like `@Inject` and `@Produces`.  
  
The application has a "client" portion that generates messages that are serviced by a bean. This bean is either located in the same JVM, or remotely accesible through a JMS queue. The objective of this example is to show how easy it is to use this pattern create distributed services that enable spreading particularly resource intensive operations to backend nodes. This is the same operation of the Spring Integration post mentioned previously, therefore the flow can be shown using the exact same diagrams as in previous example:  

[![](http://4.bp.blogspot.com/-O6DqPU3G4k0/VU95xFoSXOI/AAAAAAAAFMc/yaSvcwYomKg/s1600/RequestReply.gif)](http://4.bp.blogspot.com/-O6DqPU3G4k0/VU95xFoSXOI/AAAAAAAAFMc/yaSvcwYomKg/s1600/RequestReply.gif)

  
The main routing logic is found in `CamelContext.java`.  
For anyone used to Spring Integration, one of the most obvious things that jumps out is the abundance of code to wire the Camel and CDI components together. On the other hand, there is basically no XML. The only XML file (`beans.xml`) is basically empty.  
CDI favors using code to wire components. This is done mostly through annotations, and simple producer methods (which are annotated with `@Produces`). This is philosophically very different from Spring. Camel does provide a way to configure its routes using XML, but it requires Spring, so it's not really viable with CDI.  
The general format of a Camel route is to define a source (using `RouteBuilder.from()`) and a destination (using `RouteBuilder.to()`). `RouteBuilder` provides a fluent API, and extra parameters can be set, for example the Exchange Pattern, which is needed to get a response back.  
The format of uri passed to the from and to methods is always prefixed by the component name.  
  
For example the following definition will route requests from a direct endpoint named `order`, to a bean named `orderServiceHandler`:  

{% highlight java %}
from("direct:order").to("bean:orderServiceHandler");
{% endhighlight %}

In this case we use the `direct` component which defines an internal connection, and the `jms` component. The `bean` component provides a container agnostic way to access beans in Dependency Injection container.  
In this case we're including the "camel-cdi" artifact, which allows Camel to locate beans in a JSR-299 (CDI) container such as Weld. Similar modules exist for other containers such as Spring, or a manual Register may be manually maintained.  
The uri also includes the destination name, as well as extra parameters. For example we can define that 5 concurrent listeners (threads) will consume from a single queue:  

{% highlight java %}
jms:queue:order?concurrentConsumers=5
{% endhighlight %}
  
Camel can be hooked into CDI to provide beans that can be consumed by using `@Produces`. This method will produce objects of type `IOrderService`. The resulting object will be a proxy that will be backed by the "direct:order" endpoint.  
  
{% highlight java %}
    @Produces  
    public IOrderService createService() throws Exception {  
        IOrderService service = new ProxyBuilder(camelCtx).endpoint("direct:order").build(IOrderService.class);  
        return service;  
    }  
{% endhighlight %}

A bit of extra logic was change to determine which routes to add based on the functionality desired by the user and passed as command line parameters.  In the Spring example this was controlled based on which files were to be included.  
This does limit the flexibility of the solution, since there are no XML files to tweak after building the application.  
  
It worth noting that CDI only supports mapping one method. In practice the interface can have more than one method, but the invocation of any of those methods will be sent to same endpoint.  
More sophisticated manipulation can be done, but in my experience it's not worth the effort unless absolutely necessary.  
  
Other functions that were defined in XML in the Spring example are now done in code, for example starting an embedded ActiveMQ JMS Broker and JMS connection factory.  These are handled inside `JMSConnectionFactoryProvider`, but in most cases will be handled by a Java EE container.  

## Running the example

This example can be run in the same manner as the Spring Integration one. The code must first be compiled:

    mvn compile

Then it can be run from Maven, passing one (or two) of three parameters. The three possible parameters are:

*   `direct`: Uses a direct connection between the client and the bean handling the requests
*   `server`: Starts a server that will monitor an ActiveMQ queue, and process any requests sent to it. This option will also start an embedded ActiveMQ server, which prevents more than one server from running at once. If using an external ActiveMQ server, no such restriction exists.
*   `client`: Start the distributed client, which will send and receive requests through ActiveMQ. This can be used 

For example :
  
    mvn exec:java -Dexec.mainClass="com.javaprocess.examples.integration.main.Main" -Dexec.args="server client"

Will result in the following output:

    Running in client mode  
    Requesting order processing on thread: 38  
    Requesting order processing on thread: 41  
    Requesting order processing on thread: 39  
    Requesting order processing on thread: 40  
    Requesting order processing on thread: 42  
    Got order with id 100 on thread: 33  
    Got order with id 100 on thread: 32  
    Got order with id 100 on thread: 30  
    Got order with id 100 on thread: 29  
    Got order with id 100 on thread: 31  
    Order was requested by 40  and by processed by thread: 33  
    Order was requested by 42  and by processed by thread: 32  
    Order was requested by 41  and by processed by thread: 29  
    Order was requested by 39  and by processed by thread: 31  
    Order was requested by 38  and by processed by thread: 30  
    Stop Camel

## Side by side comparison with Spring Integration

The following sections compare mostly equivalent Spring Integration XML to their Apache Camel DSL.  

### Directly routing to a bean

{% highlight xml %}
<int:channel id="requestChannel"/>  
<int:service-activator input-channel="requestChannel"  
 ref="orderServiceHandler" method="processOrder" />  
{% endhighlight %}

**vs.**

{% highlight java %}
from("direct:order").to("bean:orderServiceHandler");  
{% endhighlight %}

### Routing to a JMS queue

{% highlight xml %}
<int:channel id="requestChannel"/>  
<int-jms:outbound-gateway request-channel="requestChannel"   
 request-destination="amq.outbound" extract-request-payload="true"/>  
{% endhighlight %}

**vs.**

{% highlight java %}
from("direct:order").to("jms:queue:order").setExchangePattern(ExchangePattern.InOut);  
{% endhighlight %}

### Listening to a JMS queue and routing messages to a bean for service

{% highlight xml %}
<int:channel id="inChannel" />  
<int-jms:inbound-gateway request-channel="inChannel"  
 request-destination="amq.outbound"  
 concurrent-consumers="10"/>  
<int:service-activator  input-channel="inChannel"  
 ref="orderServiceHandler" method="processOrder"/>
{% endhighlight %}

**vs.**

{% highlight java %}
from("jms:queue:order?concurrentConsumers=10").to("bean:orderServiceHandler");  
{% endhighlight %}

### Creating a bean that is a proxy connected to a route

{% highlight xml %}
<int:gateway id="orderService"  
 service-interface="com.javaprocess.examples.integration.interfaces.IOrderService"  
 default-request-channel="requestChannel" />  
{% endhighlight %}

**vs.**

{% highlight java %}
@Produces  
public IOrderService createService() throws Exception {  
 IOrderService service = new ProxyBuilder(camelCtx).endpoint("direct:order").build(IOrderService.class);  
 return service;  
}  
{% endhighlight %}

## Notes about CDI

I'm a big fan of CDI to wire Java EE applications. However, for wiring standalone applications, CDI is lacking.  For a small application I would rather go with something simpler like Google Guice.  
However, to show a comparable application to the standalone Spring Integration example, I have used Weld, the reference implementation for CDI and one of the most popular implementations out there.  
  
One particular challenge was starting a bean eagerly. If running inside a Java EE container, this could have achieved with a single annotation `@Startup`. It is in cases like this that it becomes obvious that CDI is meant to compliment Java EE. However, for my standalone example I had to implement an extension that achieved this behavior. While CDI provides the way to do it, it still not ideal. More information on how this is achieved can be seen in this detailed [post](http://ovaraksin.blogspot.com/2013/02/eager-cdi-beans.html).  

## Conclusion

I hope that this short post has shown the value the Enterprise Integration Patterns, regardless of the implementation. Both Apache Camel and Spring Integration provides a rich set of the Enterprise Integration Patterns, which can be leveraged to solve complex real world problems.  
  

## Source code

You can find the source code for this example in github.  
To check out the code clone the following repository: `https://github.com/aolarte/camel-integration-samples.git`.  

    git clone https://github.com/aolarte/camel-integration-samples.git