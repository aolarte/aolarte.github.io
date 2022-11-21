---
title: ThreadLocal, and how it holds apps together
excerpt: What is ThreadLocal and why is it important? It's one of the key pieces of glue logic in modern Java web applications, so read on.
date: '2017-05-23T07:00:00.000-06:00'
tags:
- java
- threads
commments: true
---

# What is ThreadLocal?

In dry terms, `ThreadLocal` provides per thread variables. What does that mean? If you look at the [API](http://docs.oracle.com/javase/1.5.0/docs/api/java/lang/ThreadLocal.html), you will see it only has 4 methods:

* `get()` To get the value associate with the current thread.
* `set(T value)` To set the value associate with the current thread.
* `remove()` To remove the value associate with the current thread.
* `initialValue()` To return a value when we invoke `get()` without having set a value previously.

Notice a pattern? Basically `ThreadLocal` allows us to store a value that will only be available to the current thread. Other threads will have access to their own values. 
You can also think of `ThreaLocal` as a `Map`, where the key is the current `Thread`. 

# Why is ThreadLocal important?

`ThreadLocal` allows to keep context on a per-thread basis. 
In other words we can use `ThreadLocal` to access information without explicitly passing a "context" object through the call stack.
This aligns very well with the model used by Servlet application servers, in which each request is handled by a single thread. In such cases we can use `ThreadLocal` as a thread global variable.
  
The truth is that even if you have never used `ThreadLocal` directly, there's a good chance that a framework you use depends on `ThreadLocal`. To name a few examples:

* Spring Security
* Mockito
* Transaction management frameworks (think of the code making `@Transactional` work)
* Hibernate (when using `ThreadLocalSessionContext`)

Any of those rings a bell? If so, well, then your application already depends heavily on `ThreadLocal`, and there's an enormous value in understanding its mechanism.

# How to use ThreadLocal

The simplest case in which we use `ThreadLocal` is to provide safe access to non thread-safe objects in a thread-safe way without having to use synchronization.
One of the main culprits is `SimpleDateFormat`, which is relatively expensive to create, and is not thread-safe. With `ThreadLocal` we can access it safely:

~~~java
public class TestController {
  
    private static final ThreadLocal<SimpleDateFormat> dateFormatHolder =
            ThreadLocal.withInitial(() -> new SimpleDateFormat("MM/dd/yyyy hh:mm:ss a"));
    
    public String verify() {
        SimpleDateFormat sdf=dateFormatHolder.get();
        System.out.println("Request received: " + sdf.format(new Date()));
        return "Ok";
    }
}
~~~

In this example `dateFormatHolder` will hold a different instance of `SimpleDateFormat` for each thread. 
Calling `dateFormatHolder.get()` will return the instance of `SimpleDateFormat` already associated to the current thread, or if one doesn't exist, it will be created with the associated `Supplier` lambda.
In this example, no class outside of our TestController will have access to our `dateFormatHolder`. You could also move `dateFormatHolder` to a util class, to allow multiple classes to have access to a thread specific `SimpleDateFormat`. 

## Spring Security

Spring Security provides a series of Filters that wrap every request that the application server (such as Tomcat) receives.
Take a look at the (somewhat abridged) [`SecurityContextPersistenceFilter`](https://github.com/spring-projects/spring-security/blob/master/web/src/main/java/org/springframework/security/web/context/SecurityContextPersistenceFilter.java) from SpringSecurity. 

~~~java
public class SecurityContextPersistenceFilter extends GenericFilterBean {

	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;
		HttpRequestResponseHolder holder = new HttpRequestResponseHolder(request,
				response);
		SecurityContext contextBeforeChainExecution = repo.loadContext(holder); // 1
		try {
			SecurityContextHolder.setContext(contextBeforeChainExecution); // 2
			chain.doFilter(holder.getRequest(), holder.getResponse()); // 3
		}
		finally {
			SecurityContext contextAfterChainExecution = SecurityContextHolder
					.getContext();
			// Crucial removal of SecurityContextHolder contents - do this before anything
			// else.
			SecurityContextHolder.clearContext();	// 4	
		}
	}

	public void setForceEagerSessionCreation(boolean forceEagerSessionCreation) {
		this.forceEagerSessionCreation = forceEagerSessionCreation;
	}
}
~~~

The filter follows a simple order of events:

1. Determine the `SecurityContext` (what roles do I have access to )
2. Store `SecurityContext` in the [`SecurityContextHolder`](https://github.com/spring-projects/spring-security/blob/master/core/src/main/java/org/springframework/security/core/context/SecurityContextHolder.java)
3. Continue the normal flow
4. Ensure we clear the context. We don't want any left over credentials to overflow from the scope of our requrest

We're going to skip [`SecurityContextHolder`](https://github.com/spring-projects/spring-security/blob/master/core/src/main/java/org/springframework/security/core/context/SecurityContextHolder.java), since it basically delegates to a `SecurityContextHolderStrategy`.
Instead we're going to take a look at the default `SecurityContextHolderStrategy `, which is appropriately  named [`ThreadLocalSecurityContextHolderStrategy`](https://github.com/spring-projects/spring-security/blob/master/core/src/main/java/org/springframework/security/core/context/ThreadLocalSecurityContextHolderStrategy.java).

~~~java
final class ThreadLocalSecurityContextHolderStrategy implements
		SecurityContextHolderStrategy {

	private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<SecurityContext>();

	public void clearContext() {
		contextHolder.remove();
	}

	public SecurityContext getContext() {
		SecurityContext ctx = contextHolder.get();
		if (ctx == null) {
			ctx = createEmptyContext();
			contextHolder.set(ctx);
		}
		return ctx;
	}

	public void setContext(SecurityContext context) {
		Assert.notNull(context, "Only non-null SecurityContext instances are permitted");
		contextHolder.set(context);
	}

	public SecurityContext createEmptyContext() {
		return new SecurityContextImpl();
	}
}
~~~

This class is very simple, and for the most part is nothing but an adaptor to map `ThreadLocal` to a `SecurityContextHolderStrategy`.

So how is this used? Imagine we have a service we want to secure: 

~~~java
@Component
public class AdminService {

  @Secured("ADMIN")
  public void admin(Model model) {
    System.out.println("Got new model");
  } 
}
~~~

Annotations like `@Secured` are normally implemented using AOP and proxies, which are examined here and here. 
However the important concept is that there's an interceptor which is going to be executed before the actual `admin(ModelMap model)` method, and it's the interceptors responsability to determine if the user has the "ADMIN" role. 
If the user has the role, execution continues, otherwise an exception is thrown.

Let's look [`AbstractSecurityInterceptor`](https://github.com/spring-projects/spring-security/blob/master/core/src/main/java/org/springframework/security/access/intercept/AbstractSecurityInterceptor.java), 
which is part of Aspect Oriented Programming (AOP) system allows use to the `@Secured` annotation:

~~~java
public abstract class AbstractSecurityInterceptor implements InitializingBean,
		ApplicationEventPublisherAware, MessageSourceAware {

	private AccessDecisionManager accessDecisionManager;

	protected InterceptorStatusToken beforeInvocation(Object object) {	
		Authentication authenticated = authenticateIfRequired(); //1 
		// Attempt authorization
		try {
			this.accessDecisionManager.decide(authenticated, object, attributes); //2
		}
		catch (AccessDeniedException accessDeniedException) {
			publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated, //3
					accessDeniedException));
			throw accessDeniedException;
		}
		//Removed InterceptorStatusToken generation code for brevity
	}

	private Authentication authenticateIfRequired() {
		Authentication authentication = SecurityContextHolder.getContext()
				.getAuthentication();
		//Removed code to try to re-authenticate for brevity
		return authentication;
	}
}
~~~

The Interceptor operates in three steps:
1. Retrieve the `Authentication` object from `SecurityContextHolder`, which will delegate to one the strategies, for example `ThreadLocalSecurityContextHolderStrategy`, which will retrieve the `Authentication` data from `ThreadLocal` as we saw above.
2. The `Authentication` will be checked against the attributes of the method we're are calling (basically who can run this method).
3. If the `accessDecisionManager` throws an `AccessDeniedException`, the interceptor will re throw it, therefore preventing unauthorized execution of the method.

What did we achieve with `ThreadLocal`? We're able to decouple setting the authorization information (which happens in a `Filter`), with actually using this information in our business classes. Pretty cool, isn't it?

# Notes

* There's nothing preventing you from passing your object to another thread manually and causing an unsafe situation, `ThreadLocal` will not protect you in any way.
* In systems that use multiple threads to handle a single request (typically "Actor" systems), `ThreadLocal` won't be useful to pass context around.
* Keep in mind that the scope of the actual `ThreadLocal` object is important. Normally `ThreadLocal` is declared as a `static` field to ensure we keep only a single copy. Using `static` also enables easy access to the field from other classes without the need to access the instance which has the reference to the `ThreadLocal`. 
* The `ThreadLocal` variables become candidates for Garbage Collection after their associated thread is garbage collection.
* Just like with any static variables, `ThreadLocal` code can be difficult to test. If possible, wrap your `ThreadLocal` variable in an managed object and inject it with your Dependency Injection framework.
