---
permalink: /:categories/:year/:month/:title.html
layout: single
title: Simple Spring AOP and annotations example
date: '2010-04-15T15:12:00.000-07:00'
tags:
- aop
- spring
- java
modified_time: '2010-04-29T15:37:42.734-07:00'
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-1139297784060030442
blogger_orig_url: http://www.javaprocess.com/2010/04/simple-spring-aop-and-annotations.html
---


Spring seems to be the tool of choice for many new web projects nowadays, and while there's a wealth of examples and documentation out there, one particular use case eluded me quite a bit: AOP with annotations. I don't like xml, I prefer annotations and convections, so here's a step by step example on how to get a basic example up and running. In this very basic example we'll create a logging aspect and wrap it around an object.

First let's get everything setup. We'll start with an empty webproject, and we'll add the following libraries:

* All of the jars from the spring-framework-3.0.1 zip. (from [SpringSource](http://springsource.org))
* asm3.2.jar (from [ASM](http://asm.ow2.org/))
* aspectjrt.jar and aspectjweaver.jar (from [Eclipse](http://www.eclipse.org/aspectj/downloads.php))
  * In this example I'm using aspectj-1.6.8.jar. The jars you need are inside the jar you download.
* cglib-2.2.jar (from [Sourceforge](http://cglib.sourceforge.net/))
* com.springsource.org.aopalliance-1.0.0.jar (from [SpringSource](http://repository.springsource.com/ivy/bundles/external/org.aopalliance/com.springsource.org.aopalliance/1.0.0/com.springsource.org.aopalliance-1.0.0.jar))
* commons-logging-1.1.1.jar (from [Apache Commons](http://commons.apache.org/))
* jstl-api-1.2.jar and jstl-impl-1.2.jar(from [Java site](https://jstl.dev.java.net/download.html)) 
  * Maybe not required, but always useful.

Now let's create a basic web.xml:

{% highlight xml %}
<web-app xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" schemalocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" id="WebApp_ID" version="2.5">
<display-name>spring.test</display-name>
<welcome-file-list>
<welcome-file>index.html</welcome-file>
<welcome-file>index.htm</welcome-file>
<welcome-file>index.jsp</welcome-file>
<welcome-file>default.html</welcome-file>
<welcome-file>default.htm</welcome-file>
<welcome-file>default.jsp</welcome-file>
</welcome-file-list>

<listener>
<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
<context-param>
<param-name>contextConfigLocation</param-name>
<param-value>
        /WEB-INF/spring/web-application-context.xml
    </param-value>
</context-param>

<filter>
<filter-name>encoding-filter</filter-name>
<filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
<init-param>
 <param-name>encoding</param-name>
 <param-value>UTF-8</param-value>
</init-param>
</filter>


<filter-mapping>
<filter-name>encoding-filter</filter-name>
<url-pattern>/*</url-pattern>
</filter-mapping>

<servlet>
<servlet-name>simple-form</servlet-name>
<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
<init-param>
 <param-name>contextConfigLocation</param-name>
 <param-value></param-value>
</init-param>
</servlet>


<servlet-mapping>
<servlet-name>simple-form</servlet-name>
<url-pattern>*.do</url-pattern>
</servlet-mapping>
</web-app>
{% endhighlight %}

The xml simply initializes the spring framework, by passing the location of the main configuration file ( `/WEB-INF/spring/web-application-context.xml` ). We also map our spring actions to `*.do`. Now let's look at the `web-application-context.xml` file:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xmlns:context="http://www.springframework.org/schema/context"
 xsi:schemaLocation="http://www.springframework.org/schema/beans
                        http://www.springframework.org/schema/beans/spring-beans.xsd
                        http://www.springframework.org/schema/context
                        http://www.springframework.org/schema/context/spring-context.xsd">

 <context:component-scan base-package="test.spring.actions" />

 <!-- Imports the configurations of the different infrastructure systems of the application -->
 <import resource="webmvc-context.xml" />
 <import resource="aop-context.xml"/>
</beans>
{% endhighlight %}

This file defines which package should be scanned for components (`test.spring.actions` in this case) and imports two further files. One of these files controls the AOP part of the framework, while the other controls the MVC (Model-View-Controller) part of the framework. Let's look at the `aop-context.xml` file:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:aop="http://www.springframework.org/schema/aop"
  xsi:schemaLocation="
http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">

<aop:aspectj-autoproxy />

<bean id="myAspect" class="test.spring.aspects.Logger"/>
<bean id="myAspectLog" class="test.spring.actions.LogObject"/>
</beans>
{% endhighlight %}

Of note here, is the `aspectj-autoproxy` tag, which is required to enable the creation of proxy object to apply the aspects. Now, lets look at the code. The first class is the logger aspect that we will be applying to one or more classes:

{% highlight java %}
package test.spring.aspects;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;

@Aspect
public class Logger {
    @Around(("execution(* test.spring.actions.LogObject.*(..))"))
    public Object cache(ProceedingJoinPoint pjp) {
        System.out.println("Entering: " + pjp.getSignature().getDeclaringTypeName());
        Object ret=null;
        try {
            ret = pjp.proceed();
        } catch (Throwable e) {
        }        
        System.out.println("Leaving: " + pjp.getSignature().getDeclaringTypeName());
        return ret;
    }
} 
{% endhighlight %}

Here we're using the `@Around` annotation to declare an around join point, since this type gives us more information regarding the class and method we're wrapping. In the same annotation, we also specify that this point cut should be applied to all methods of the `LogObject` class. This is the point cut expression, and it's fairly flexible. More information can be found elsewhere in the web, including the Spring [documentation](http://static.springsource.org/spring/docs/3.0.x/spring-framework-reference/html/aop.html#aop-pointcuts-examples). Other than this, our aspect just logs the entry and exit of the methods.

The actual class we're applying the aspect to is rather uninteresting:

{% highlight java %}
package test.spring.actions;

public class LogObject implements ILogObject{
    @Override
    public String view() {
        return "";
    }
}
{% endhighlight %}

Of course this is where your actual application logic would live, but the purpose here is to provide the simplest example possible. The only notable aspect of this class is that it implements an interface. In our controller we'll use the interface instead of the actual implementation, since Spring will wire a dynamic proxy of the real object. In any case, it is normally good practice to program against interfaces.

Last is our controller, where we tie everything together:

{% highlight java %}
package test.spring.actions;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
@RequestMapping("/test")
public class TestAction {
    @Autowired
    ILogObject l;
    @RequestMapping
    public String view () {
        l.view();
        return "test";
    }
}
{% endhighlight %}


The controller is letting Spring autowire the `ILogObject`, which will be proxied to provide the functionality provided by our logging aspect. So, go ahead and hit the action from your browser, in the console you will see the log messages before and after we execute the `view()` method.

So this is it, I hope this helps people as a basic starting point for using AOP inside spring, without having to write too much xml.