---
permalink: /:categories/:year/:month/:title.html
title: 'Simple Spring and JPA with Hibernate tutorial. Part 3: Simple Security'
date: '2013-10-09T12:06:00.000-07:00'
tags:
- security
- spring-mvc
- spring-security
- spring
- java
modified_time: '2013-10-09T12:34:20.854-07:00'
thumbnail: http://4.bp.blogspot.com/-gqgSA--6Nek/UlWlSuF1LjI/AAAAAAAAAM0/u_C7DtYwr7Y/s72-c/sc2.png
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-4255703040984918461
blogger_orig_url: http://www.javaprocess.com/2013/10/simple-spring-and-jpa-with-hibernate-part3.html
---
Spring security In this blog post we continue adding functionality to our Spring and JPA application. We will be adding basic security using Spring Security (formerly known as Acegi Security). However to achieve this we'll also look at some new elements:  

*   Spring's ContextLoaderListener and hierarchical contexts
*   Entity relationships in JPA
*   Multiple Spring xml files and their organization
*   Maven's quirks with transitive dependencies

This example builds on top of the code that was written for [part 2](/2013/10/simple-spring-and-jpa-with-hibernate-part2.html), but we will rearrange several pieces. Let's start by looking at the pom.xml where we've added two new dependecies:  
 
{% highlight xml %}
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">  
    <modelVersion>4.0.0</modelVersion>  
    <groupId>test.tutorials</groupId>  
    <artifactId>spring-hib-jpa</artifactId>  
    <packaging>war</packaging>  
    <version>1.0-SNAPSHOT</version>  
    <name>spring-hib-jpa Maven Webapp</name>  
  
    <properties>  
        <springVersion>3.2.4.RELEASE</springVersion>  
        <springSecurityVersion>3.1.4.RELEASE</springSecurityVersion> <!-- 1 -->  
    </properties>  
  
    <dependencies>  
    
        <dependency>  
            <groupId>org.springframework</groupId>  
            <artifactId>spring-beans</artifactId>  
            <version>${springVersion}</version>  
        </dependency>  
  
        <dependency>  
            <groupId>org.springframework</groupId>  
            <artifactId>spring-webmvc</artifactId>  
            <version>${springVersion}</version>  
        </dependency>  
  
        <dependency>  
            <groupId>org.springframework</groupId>  
            <artifactId>spring-orm</artifactId>  
            <version>${springVersion}</version>  
        </dependency>  
  
        <dependency> <!-- 2 -->  
             <groupId>org.springframework.security</groupId>  
             <artifactId>spring-security-core</artifactId>  
             <version>${springSecurityVersion}</version>  
        </dependency>  
  
        <dependency>  
             <groupId>org.springframework.security</groupId>  
             <artifactId>spring-security-config</artifactId>  
             <version>${springSecurityVersion}</version>  
        </dependency>  
  
        <dependency>  
            <groupId>org.springframework.security</groupId>  
            <artifactId>spring-security-web</artifactId>  
            <version>${springSecurityVersion}</version>  
        </dependency>  
  
        <dependency> <!-- 3 -->  
            <groupId>org.springframework</groupId>  
            <artifactId>spring-aop</artifactId>  
            <version>${springVersion}</version>  
        </dependency>  
  
        <dependency>  
            <groupId>org.hibernate</groupId>  
            <artifactId>hibernate-entitymanager</artifactId>  
            <version>4.1.9.Final</version>  
        </dependency>  
  
        <dependency>  
            <groupId>jstl</groupId>  
            <artifactId>jstl</artifactId>  
            <version>1.2</version>  
        </dependency>  
  
        <dependency>  
            <groupId>com.h2database</groupId>  
            <artifactId>h2</artifactId>  
            <version>1.3.161</version>  
        </dependency>  
  
        <dependency>  
            <groupId>org.apache.tomcat</groupId>  
            <artifactId>tomcat-dbcp</artifactId>  
            <version>7.0.41</version>  
        </dependency>  
  
    </dependencies>  
</project> 
{% endhighlight %} 

1.  We add a property for the Spring Security version, just like we did for Spring. In version 3, Spring and Spring Security have different release schedules and versions. Both projects should have their versions synchronized by version 4.
2.  We add 3 Spring Security dependencies:
3.  `spring-security-core` provides the basic classes used by the framework
4.  `spring-security-config` provides the elements needed to configure the framework, for example the xsd files used in the xml configuration.
5.  `spring-security-web` provides the hooks to provide URL based security.
6.  Spring security requires the `spring-aop` dependency. However, since the Spring and Spring Security versions are not in sync, it depends on a version of `spring-aop` that is incompatible with Spring 3.2.4 (the one we're using). To solve this, we add an explicit dependency to the right version. When a project depends on two different versions of the same library, Maven will pick the one closer to the root. What we have done is declared the dependency at the very root of the project, to ensure our version is used. Luckily our version of `spring-aop` is compatible with the Spring Security version we're using.

Now let's look the at the web.xml, where we have a few new elements (`src/main/webapp/WEB-INF/web.xml`):  

{% highlight xml %}
<web-app xmlns="http://java.sun.com/xml/ns/javaee" version="2.5">  
  
    <listener>  
         <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class> <!-- 1 -->  
     </listener>  
  
    <context-param>  
        <param-name>contextConfigLocation</param-name>  <!-- 2 -->  
        <param-value>WEB-INF/spring-context.xml</param-value>  
    </context-param>   
  
    <filter>  
        <filter-name>springSecurityFilterChain</filter-name>  
        <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>   <!-- 3 -->  
    </filter>  
  
    <filter-mapping>  
        <filter-name>springSecurityFilterChain</filter-name>   <!-- 4 -->  
        <url-pattern>/*</url-pattern>  
    </filter-mapping>  
  
    <servlet>  
      <servlet-name>spring</servlet-name>  
      <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>  
      <load-on-startup>1</load-on-startup>  
    </servlet>  
     
    <servlet-mapping>  
        <servlet-name>spring</servlet-name>  
        <url-pattern>*.html</url-pattern>  
    </servlet-mapping>  
  
</web-app> 
{% endhighlight %} 

1.  In this new application we introduce another way of configuring the Spring container, through a listener. The listener will create a parent context. We will also create a child context, instantiated by the DispatcherServlet. The child context will have access to the beans of the parent context, but not the other way around.
2.  The location of the xml file to instantiate the parent context is defined with a context parameter named `contextConfigLocation`. We will look at this file in detail. In this parameter you can configure more than one file, but for now
3.  To actually secure the application, we add a new filter. In the spring world, the MVC framework and the security framework are separate mechanisms.
4.  The security filter is normally applied to the whole application, with a mapping of `/*`. We can later tune this apply rules to only some URLs.

Let's start looking at the spring-context.xml, where we configure the parent spring context (`src/main/webapp/WEB-INF/spring-context.xml`):

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
       xmlns:tx="http://www.springframework.org/schema/tx"  
       xmlns:context="http://www.springframework.org/schema/context"  
       xmlns:beans="http://www.springframework.org/schema/beans"  
       xsi:schemaLocation="http://www.springframework.org/schema/beans  
        http://www.springframework.org/schema/beans/spring-beans.xsd  
        http://www.springframework.org/schema/context  
        http://www.springframework.org/schema/context/spring-context.xsd  
        http://www.springframework.org/schema/tx  
        http://www.springframework.org/schema/tx/spring-tx.xsd  
        ">  
  
    <beans:import resource="spring-security.xml"/> <!-- 1 -->  
  
    <context:component-scan base-package="test">  
            <context:exclude-filter expression="org.springframework.stereotype.Controller" type="annotation"/>  <!-- 2 -->  
    </context:component-scan>  
     
    <tx:annotation-driven/> <!-- 3 -->  
  
    <bean id="dataSource" class="org.apache.tomcat.dbcp.dbcp.BasicDataSource">  
        <property name="driverClassName" value="org.h2.Driver"/>  
        <property name="url" value="jdbc:h2:mem:test"/>  
    </bean>  
  
  
    <bean id="entityManagerFactory"  
          class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">  
        <property name="dataSource" ref="dataSource"/>  
        <property name="packagesToScan" value="test.model"/>  
        <property name="jpaVendorAdapter">  
            <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter">  
            </bean>  
        </property>  
        <property name="jpaProperties">  
            <props>  
                <prop key="hibernate.show_sql">true</prop>  
                <prop key="hibernate.dialect">org.hibernate.dialect.H2Dialect</prop>  
                <prop key="hibernate.hbm2ddl.auto">create-drop</prop>  
            </props>  
        </property>  
    </bean>  
  
    <bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager"/>  
  
</beans>  
{% endhighlight %}

1.  In Spring, it is good practice to separate configuration files based on their purpose. Back in the day before annotations, these files could grow enormously. Nowadays they're shorter, but it still helps to keep them separate. The import element will include the contexts of another file, and the corresponding beans will be created in the current context. We will examine the spring-security.xml file later on.
2.  We have copied the component-scan element into this new file. However we're excluding beans that have `@Controller` annotation. Those beans will be registered by the Spring MVC configuration file.
3.  We have moved all of the JPA configuration (from `tx:annotation-driven` to the end of the file) into the main Spring xml file. I could also be possible to create a new file to keep all database/JPA related bean configuration.

Let's look at the security configuration (`src/main/webapp/WEB-INF/spring-security.xml`):

{% highlight xml %}
<beans:beans xmlns="http://www.springframework.org/schema/security"  
    xmlns:beans="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
    xsi:schemaLocation="http://www.springframework.org/schema/beans  
        http://www.springframework.org/schema/beans/spring-beans.xsd  
        http://www.springframework.org/schema/security  
        http://www.springframework.org/schema/security/spring-security-3.1.xsd">  
  
    <http use-expressions="true"> <!-- 1 -->  
        <intercept-url pattern="/**" access="isAuthenticated()" />  <!-- 2 -->  
        <form-login /> <!-- 3 -->  
    </http>  
  
    <authentication-manager> <!-- 4 -->  
        <authentication-provider user-service-ref="userService" />   
    </authentication-manager>  
</beans:beans>  
{% endhighlight %}

1.  The http element provides securing of URLs (Spring Security also allows securing Java methods, but we'll look at that in another post). Of notice here is the attribute `use-expressions="true"`, which allows using Spring EL instead of simply listing allowed roles.
2.  We are securing a single URL, which covers the whole application, and allows access to any use that is successfully authenticated. Multiple urls can be added to fine tune access control, and will be evaluated in the order in which they are declared.
3.  The form-login element provides a basic form based authentication. You can later customized the login page to fit the style of your application. Spring Security provides other ways of authentication, including HTTP BasicAuthentication, and several Single Sign On schemes.
4.  The `authentication-manager` is the bean that will verify the user credentials as well as provide the authorities or roles a user has. We will look at our implementation of the user service later.

Finally, we look at (the now much shorter) `spring-context.xml`. We've removed all of the elements not related to Spring MVC.  

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
       xmlns:context="http://www.springframework.org/schema/context"  
       xmlns:mvc="http://www.springframework.org/schema/mvc"  
       xsi:schemaLocation="http://www.springframework.org/schema/beans  
        http://www.springframework.org/schema/beans/spring-beans.xsd  
        http://www.springframework.org/schema/context  
        http://www.springframework.org/schema/context/spring-context.xsd  
        http://www.springframework.org/schema/mvc  
        http://www.springframework.org/schema/mvc/spring-mvc.xsd  
        ">  
  
    <context:component-scan base-package="test">  
        <context:include-filter expression="org.springframework.stereotype.Controller" type="annotation"/> <!-- 1 -->  
    </context:component-scan>  
    <mvc:annotation-driven/>  
  
    <bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">  
        <property name="prefix" value="/WEB-INF/jsp/"/>  
        <property name="suffix" value=".jsp"/>  
    </bean>  
  
</beans> 
{% endhighlight %}  

1.  Of note in this file is the `context:include-filter` which will only load controllers. Remember that this file is loaded as a subcontext, contrary to the spring-security.xml which is loaded in the main or parent context.

We have also two new entities, `Role` and `Employee`. We'll look at `Role` first (`src/main/java/test/model/Role.java`):

{% highlight java %}
@Entity  
public class Role {  
  
    @Id  
    private Long id;  
    private String name;  
  
    // Getters and setters removed for brevity  
}
{% endhighlight %} 

This is a very basic entity, almost the same as the Person entity we had created in the first part. The most significant difference is that we're missing the `@GeneratedValue` annotation in the id field. This means that if we want to insert new roles, we need to manually specify an id.

Next we'll look at the `Employee` entity (`src/main/java/test/model/Employee.java`):

{% highlight java %}
@Entity  
public class Employee {  
  
  @Id  
  @GeneratedValue  
  private Long id;  
  private String userName;  
  private String password;  
  @ManyToMany  
    
  private List<Role> roles;  
  // Getters and setters removed for brevity  
}
{% endhighlight %}  

This is also a very basic entity, except for the addition of a collection field. 
The `roles` field will contain a list of roles assigned to the employee. 
This is indicated with the `@ManyToMany` annotation. 
In order to map the many-to-many relationship (many employees map to many roles), we need a join table. 
In this case the default join table will be called `Employee_Role`, and it will have two columns `Employee_id` and `roles_id`.
The default can be changed by adding the `@JoinTable` annotation, which allows us to define the table name, as well as the column names.

After seeing the new entities, we can get a better idea of how they will look by looking at the data we're inserting into the database:

Let's look at the data we use to seed the database (`src/main/resources/import.sql`):

{% highlight sql %}
INSERT INTO PERSON (firstName,lastName) VALUES ('Joe','Doe') ;  
INSERT INTO ROLE (id,name) VALUES (1,'ROLE_USER');  
INSERT INTO ROLE (id,name) VALUES (2,'ROLE_ADMIN');  
INSERT INTO EMPLOYEE ( userName, password) VALUES ('user','lion');  
INSERT INTO EMPLOYEE ( userName, password) VALUES ('admin','tiger');  
INSERT INTO Employee_Role (Employee_id , roles_id) VALUES ((SELECT id FROM EMPLOYEE WHERE userName='user'),1);  
INSERT INTO Employee_Role (Employee_id , roles_id) VALUES ((SELECT id FROM EMPLOYEE WHERE userName='admin'),1);  
INSERT INTO Employee_Role (Employee_id , roles_id) VALUES ((SELECT id FROM EMPLOYEE WHERE userName='admin'),2);  
{% endhighlight %}

In this file we're now also inserting two employees, and the corresponding roles for each employee. Since the id of the employee is not know (we could assume they would be 1 and 2, but there's no guarantee), we use a `SELECT` to find the correct Id.

Now let's look at the employee service and DAO. This is the only part that requires some significant Java code. Everything else has been XML and some annotations in a POJO. This shows how much coding you're saving by using such a framework. 

Let's look at the `EmployeeDAO` first (`src/main/java/test/dao/impl/EmployeeDAO.java):

{% highlight java %}
@Repository  
public class EmployeeDAO implements IEmployeeDAO {  
  
    @PersistenceContext  
    private EntityManager em;  
  
    @Override  
    public Employee findByUserName(String userName) {  
        TypedQuery<Employee> query=em.createQuery("SELECT e FROM Employee e WHERE e.userName=:userName", Employee.class); //1  
        query.setParameter("userName",userName); //2  
        return query.getSingleResult(); //3  
    }  
}  
{% endhighlight %}

1.  This class has only one method. In this method we're using the `TypedQuery`, which allows our query results to already have the correct type, and save us some potentially dangerous type casting.
2.  In this query we're using a parameter with a named placeholder `:userName`. We then pass the user name we're looking for with the method `setParameter`.
3.  The `getSingleResult` method will return a single Employee. If no employee is found, a runtime exception of `NoResultException` type will be thrown.

Finally, we look a the user service (`src/main/java/test/service/impl/UserService.java`);
{% highlight java %}
@Service  
@Transactional  
public class UserService implements UserDetailsService {  
  
    @Autowired  
    private IEmployeeDAO userDAO;  
  
    public UserDetails loadUserByUsername(String userName) throws UsernameNotFoundException {  
        Employee employee =userDAO.findByUserName(userName); // 2  
        if (employee ==null) {  
            throw new UsernameNotFoundException("Invalid Employee"); // 3  
        }  
        return createUserDetails(employee); // 4  
    }  
  
    private UserDetails createUserDetails(Employee employee) {  
        List<GrantedAuthority> authorities=new ArrayList<GrantedAuthority>();  
        for (Role role:employee.getRoles()) {  
            authorities.add(new SimpleGrantedAuthority(role.getName())); // 5  
        }  
        User ret=new User(employee.getUserName(),employee.getPassword(),true,true,true,true,authorities); // 6  
        return ret;  
    }  
}  
{% endhighlight %}

1.  The UserService implements the Spring Security interface UserDetailsService. At this point it does not implement any our interfaces, but eventually will, once we add extra functionality that requires finding users.
2.  The employee is found using our DAO.
3.  If the employee is not found, a Spring Security exception of type UsernameNotFoundException is thrown. In this section we could also catch the RuntimeExceptions coming from the DAO, in order to manipulate the errors displayed by Spring Security. If we don't, Spring Security would display the message of the excpetion, which might not be the most user friendly message (for example "No entity found for query" when user is not found).
4.  The actual return value of the method is loadUserByUsername is a Spring Security interface, UserDetails. We use a private method to create the right type of object from the Employee object.
5.  In our private method, each Role object becomes a SimpleGrantedAuthority, another Spring Security object.
6.  Finally we create a User object, based on the information found in the Employee object and the list of GrantedAuthority.

We can then compile and run our small application with:  
  

    mvn tomcat:run  

You can try to run the service but will be greeted with a login screen: <http://localhost:8080/spring-hib-jpa/view.html> :  

[![](http://4.bp.blogspot.com/-gqgSA--6Nek/UlWlSuF1LjI/AAAAAAAAAM0/u_C7DtYwr7Y/s640/sc2.png)](http://4.bp.blogspot.com/-gqgSA--6Nek/UlWlSuF1LjI/AAAAAAAAAM0/u_C7DtYwr7Y/s1600/sc2.png)

  
You can then try to enter a username an password, and unless it matches what we inserted in database, you'll get an error:  

[![](http://2.bp.blogspot.com/-drgRo66xQBc/UlWlShreLkI/AAAAAAAAAMw/vzWzewOuRlU/s640/sc3.png)](http://2.bp.blogspot.com/-drgRo66xQBc/UlWlShreLkI/AAAAAAAAAMw/vzWzewOuRlU/s1600/sc3.png)

  
If you enter a valid username and password you will see main screen of our little test application:  

[![](http://3.bp.blogspot.com/-oclRRb0FUUQ/UlWlSufpMiI/AAAAAAAAAMs/yTdRi3QbrXA/s640/sc4.png)](http://3.bp.blogspot.com/-oclRRb0FUUQ/UlWlSufpMiI/AAAAAAAAAMs/yTdRi3QbrXA/s1600/sc4.png)

  
This is a very basic example, but we've set up security on our application.  And while there was plenty of XML configuration, there were no code changes.  This separation of business logic and security logic should allow for a better and easier to maintain application.  
  

## Source Code

As always the code is available for you to downloand and play with. Clone it from it from github:

git clone https://github.com/aolarte/tutorials.git  

The finished code is contained in directory `spring3_simple_security`.  
  
## Simple Spring and JPA with Hibernate tutorial series: 

*   [Part 1: The basics](/2013/09/simple-spring-and-jpa-with-hibernate.html)
* [Part 2: Forms and persisting entities](/2013/10/simple-spring-and-jpa-with-hibernate-part2.html)
*   Part 3: Simple security