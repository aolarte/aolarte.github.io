---
permalink: /:categories/:year/:month/:title.html
title: Using Apache Camel and CDI for creating scalable distributed services backed
  by JMS (part 2)
date: '2015-10-25T16:46:00.001-07:00'
tags:
- integration
- messaging
- jms
- dependency injection
- cdi
- camel
- java
modified_time: '2016-05-31T17:02:27.888-07:00'
thumbnail: http://4.bp.blogspot.com/-cAkq9jncT6c/Vi0KV503a9I/AAAAAAAAHHc/F2v5CnzIFXI/s72-c/camel-box-small.png
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-5523967251946346988
blogger_orig_url: http://www.javaprocess.com/2015/10/distributed-services-with-apache-camel-part2.html
---

[![](http://4.bp.blogspot.com/-cAkq9jncT6c/Vi0KV503a9I/AAAAAAAAHHc/F2v5CnzIFXI/s1600/camel-box-small.png)](http://4.bp.blogspot.com/-cAkq9jncT6c/Vi0KV503a9I/AAAAAAAAHHc/F2v5CnzIFXI/s1600/camel-box-small.png)

Continuing the example shown in [part1](http://www.javaprocess.com/2015/10/distributed-services-with-apache-camel-part1.html) we now add a new feature, the [Claim Check](http://www.enterpriseintegrationpatterns.com/patterns/messaging/StoreInLibrary.html). This is the Apache Camel / CDI equivalent of another [Spring Integration example](http://www.javaprocess.com/2015/06/spring-integration-services-part2.html) shown before.  This pattern, along with most of the groundwork used in enterprise integration, was first codified in "Enterprise Integration Patterns" by Gregor Hohpe and Bobby Woolf. This book is a must read for any one working in this area, regardless of the tool.  

In this example we add the necessary pieces to store data in an external data store while our original message is route throughout our system. The main motivation is to avoid sending and receiving large amounts of data through JMS or other similar systems that are designed to handle low latency communication. Contrary to Spring Integration, Apache Camel does not offer an out of the box implementation of the the Claim Check pattern. The pattern and how to implemented are described [here](http://camel.apache.org/claim-check.html).

There are talks of introducing one for Apache Camel 3.0, but reading the discussions it becomes evident that providing a flexible general case implementation is complicated. Spring Integration provides an implementation, but many times it's insufficient, as it becomes evident that it's necessary to only store parts of the message (most likely binary portions of the message), while keeping most of the message to be processed along the pipeline. The general flow of this pattern can be seen below:  

[![](http://3.bp.blogspot.com/-Apw2c4hQui8/VYD8JIKmyNI/AAAAAAAAF-Y/6Wr3BZ3Zvs0/s1600/StoreInLibrary.gif)](http://3.bp.blogspot.com/-Apw2c4hQui8/VYD8JIKmyNI/AAAAAAAAF-Y/6Wr3BZ3Zvs0/s1600/StoreInLibrary.gif)

## The code

To implement the Claim Check pattern, five different new classes were created. The first two classes provide the actual implementation of the two pieces that make up the Claim Check pattern. 

The two pieces are in essence a [Content Filter](http://www.enterpriseintegrationpatterns.com/patterns/messaging/ContentFilter.html) that removes part of the content (and stores it in separate database), and a [Content Enricher](http://www.enterpriseintegrationpatterns.com/patterns/messaging/DataEnricher.html) that will pull the data from database and back into the message. The code is seen below:  
  
{% highlight java %}
@Named("claimCheckIn")  
public class ClaimCheckIn {  
  
    @Inject  
    private DataStore dataStore;  
  
    public void checkIn(Exchange exchange, @Body Object body) {  
        String id = UUID.randomUUID().toString();  
        // store the message in the data store  
        dataStore.put(id, body);  
        // add the claim check as a header  
        exchange.getIn().setHeader("claimCheck", id);  
        // remove the body from the message  
        exchange.getIn().setBody(null);  
    }  
}  
{% endhighlight %}
  
{% highlight java %}
@Named("claimCheckOut")  
public class ClaimCheckOut {  
  
    @Inject  
    private DataStore dataStore;  
  
    public void checkOut(Exchange exchange, @Header("claimCheck") String claimCheck) {  
        exchange.getIn().setBody(dataStore.get(claimCheck));  
        // remove the message data from the data store  
        dataStore.remove(claimCheck);  
        // remove the claim check header  
        exchange.getIn().removeHeader("claimCheck");  
    }  
}  
{% endhighlight %}

  
  
The data store provides a very simple abstraction to put, get, and delete objects from the database. This current implementation is using JDBC, but shows the basics of how a data store can be implemented:  

{% highlight java %}
public class DataStore {  
  
    @Inject  
    DataSource dataSource;  
  
    public void put(String id, Object body) {  
        try (Connection connection = dataSource.getConnection();  
             PreparedStatement stmt = connection.prepareCall("INSERT INTO DATA (id,data) VALUES (?,?)")) {  
            stmt.setString(1, id);  
            stmt.setObject(2, body);  
            stmt.executeUpdate();  
        } catch (SQLException e) {  
            e.printStackTrace();  
        }  
    }  
  
    public Object get(String id) {  
        Object ret = null;  
        try (Connection connection = dataSource.getConnection();  
             PreparedStatement stmt = connection.prepareCall("SELECT \* FROM  DATA WHERE id=?")) {  
            stmt.setString(1, id);  
            try (ResultSet rs = stmt.executeQuery()) {  
                if (rs.next()) {  
                    ret = rs.getObject("data");  
                }  
            }  
  
        } catch (SQLException e) {  
            e.printStackTrace();  
        }  
        return ret;  
    }  
  
    public void remove(String id) {  
        try (Connection connection = dataSource.getConnection();  
             PreparedStatement stmt = connection.prepareCall("DELETE FROM  DATA WHERE id=?")) {  
            stmt.setString(1, id);  
            stmt.executeUpdate();  
        } catch (SQLException e) {  
            e.printStackTrace();  
        }  
    }  
}  
{% endhighlight %}

  
  
The last two new classes, `H2ServerWrapper` and `DataSourceProvider`, provide the connections to the database, as well as starting the in memory H2 database. Theses classes are normally not needed, since connections are handled by the application server in most Java EE applications.  There were added here to provide similar functionality without an application server. You're welcome to look at the code, but in future post I'll explain in more detail how to provide these services from inside CDI.  
  

## Wiring it all together

Once we have our classes implementing the functionality, the wiring is very simple:  

{% highlight java %}
    RouteBuilder jmsClientCamelRoute = new RouteBuilder() {  
        @Override  
        public void configure() throws Exception {  
            from("direct:order").to("jms:queue:order", "bean:claimCheckOut").setExchangePattern(ExchangePattern.InOut);  
        }  
    };  
  
    RouteBuilder jmsServerCamelRoute = new RouteBuilder() {  
        @Override  
        public void configure() throws Exception {  
            from("jms:queue:order?concurrentConsumers=5").to("bean:orderServiceHandler", "bean:claimCheckIn");  
        }  
    };  
{% endhighlight %}
  
All it takes is adding one destination before and one after.  In Camel, declaring multiple destinations will create a pipeline. This means that the message will be sent to the first destination, then the result of the first destination will be sent to the second destination, and so on. This in contrast to multicast, which will send the same message to all of the destinations.  
  
The first change is the addition of  `bean:claimCheckOut`, after `jms:queue:order`. What this does is to check out the data and putting it back on the message once the response arrives.  Something similar is done on the server side.  First we process the request using the `orderServiceHandler` bean, and then put the response in the data store.  This particular configuration is using the check in for only the response.  
  
It's fairly easy to change it to use the claim check for both the request and the response. It's worth noting that such cases are not common in practice, as normally either the response or the request are large.  

{% highlight java %}
    RouteBuilder jmsClientCamelRoute = new RouteBuilder() {  
        @Override  
        public void configure() throws Exception {  
            from("direct:order").to("bean:claimCheckIn", "jms:queue:order", "bean:claimCheckOut").setExchangePattern(ExchangePattern.InOut);  
        }  
    };  
  
    RouteBuilder jmsServerCamelRoute = new RouteBuilder() {  
        @Override  
        public void configure() throws Exception {  
            from("jms:queue:order?concurrentConsumers=5").to("bean:claimCheckOut", "bean:orderServiceHandler", "bean:claimCheckIn");  
        }  
    };
{% endhighlight %}

In this case, request is also stored.  Since our check in method stores the whole object, this will an instance of [BeanInvokation](https://camel.apache.org/maven/current/camel-core/apidocs/org/apache/camel/component/bean/BeanInvocation.html). This method contains the parameters and other information need to invoke the service method.  Based on your needs, you could of course only extract part of this object, for example one or more of the parameters, while keeping the rest of the message intact.  
  

## Side by side comparison with Spring Integration

The following sections compare mostly equivalent Spring Integration XML to their Apache Camel DSL. I have not included the implementation of the extra infrastructure pieces which Apache Camel requires.  
  

### Routing to a JMS queue and then retrieing the data from the data store

{% highlight xml %}
<int:chain input-channel="requestChannel">  
    <int-jms:outbound-gateway request-destination="amq.outbound" extract-request-payload="true"/>  
    <int:claim-check-out message-store="clientMessageStore"/>  
</int:chain>  
{% endhighlight %}

**vs**  

{% highlight java %}
from("direct:order").to( "jms:queue:order", "bean:claimCheckOut").setExchangePattern(ExchangePattern.InOut);  
{% endhighlight %}
  
### Listening to a JMS queue and routing messages to a bean and storing the result
  
{% highlight xml %}
<int:chain  input-channel="inChannel">   
    <int:service-activator ref="orderServiceHandler" method="processOrder"/>  
    <int:claim-check-in message-store="serverMessageStore"/>   
</int:chain>  
{% endhighlight %}

**vs**  

{% highlight java %}
from("jms:queue:order?concurrentConsumers=5").to( "bean:orderServiceHandler", "bean:claimCheckIn");  
{% endhighlight %}

## Conclusion

From this short tutorial, it's evident that Apache Camel requires quite a bit extra code to accomplish the same tasks we did with Spring Integration. However, most of it becomes irrelevant if running inside an application server, which is the main target for CDI.  The biggest piece that I feel should be provided by Apache Camel is a set of data stores using common data sources such as JDBC, JPA, and some of the more popular NoSQL databases.  However, once that hurdle is overcome, Apache Camel shines in the ease in which complex problems can be solved in clear concise and easy to maintain code. So which one is better? A lot of it depends on what framework you're already using (Spring vs CDI), and if any of the integration platforms provides a particular esoteric feature. But in the end, both tools are very capable and easy to use.  
  

## Source code

You can find the source code for this example in github.  
To check out the code clone the following repository:` https://github.com/aolarte/camel-integration-samples.git`.  

  
    git clone https://github.com/aolarte/camel-integration-samples.git  
    git checkout branches/part2  

This example can be run directly from maven. For example :

    mvn exec:java -Dexec.mainClass="com.javaprocess.examples.integration.main.Main" -Dexec.args="server client"