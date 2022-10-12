---
permalink: /:categories/:year/:month/:title.html
layout: single
title: Measuring coverage for Unit and Integration tests with Maven, Jenkins, and
  SonarQube
date: '2014-02-17T08:46:00.001-08:00'
author: Andres Olarte
tags:
- continuous-integration
- jenkins
- maven
- java
modified_time: '2014-09-13T17:19:47.425-07:00'
thumbnail: http://1.bp.blogspot.com/-6qEkfxC6U-o/UwIjsHSrX6I/AAAAAAAAAYU/kQRwBeD2N5A/s72-c/install+sonar+plugin.png
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-3741338912574598563
blogger_orig_url: http://www.javaprocess.com/2014/02/measuring-coverage-for-unit-and.html
---

This blog post provides a very simple example of integrating Maven, Jenkins, and SonarQube, in particular to properly get coverage metrics of integration tests. 
While the basic integration Maven, Jenkins, and SonarQube was really simple (including code coverage metrics during unit tests, getting the integration data to display in Sonar was more complicated, 
and required me to piece together this configuration from several posts around the internet.  
  
These tools are very easy to install, and provide the basic framework for serious quality development. Hopefully you will find this information useful.  
  
This tutorial assumes that you have installed and setup the following tools:  

*   [Maven](http://maven.apache.org/) A Java build management system.
*   [Jenkins](http://jenkins-ci.org/) A continuous integration (CI) system
*   [SonarQube](http://www.sonarqube.org/) A software quality mangement system.   

If you want some pointers on getting these tools setup, you can find plenty of resources in the web:  

*   [Maven download and installation](http://maven.apache.org/download.cgi)
*   [Jenkins installation](https://wiki.jenkins-ci.org/display/JENKINS/Installing+Jenkins)
*   [Sonar installation](http://docs.codehaus.org/display/SONAR/Setup+and+Upgrade)

  

## Project setup 

We will start looking at the `pom.xml`, which is most critical part of this tutorial. If you're familiar with Jenkins a and Sonar, the rest of this tutorial might be redudant for you.  

{% highlight xml %}
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4\_0\_0.xsd">  
    <modelVersion>4.0.0</modelVersion>  
    <groupId>test.tutorials.multi</groupId>  
    <artifactId>mod2</artifactId>  
    <packaging>jar</packaging>  
    <version>1.0-SNAPSHOT</version>  
    <name>Coverage Maven Application</name>  
  
    <properties>  
        <sonar.jacoco.itReportPath>${env.WORKSPACE}/target/jacoco-integration.exec</sonar.jacoco.itReportPath> <!-- 1 -->  
    </properties>  
  
    <build>  
        <plugins>  
  
            <plugin>  
                <groupId>org.jacoco</groupId>  
                <artifactId>jacoco-maven-plugin</artifactId>  
                <configuration>  
                    <propertyName>jacoco.agent.argLine</propertyName> <!-- 2 -->  
                    <destFile>${project.build.directory}/jacoco-integration.exec</destFile>   
                    <dataFile>${project.build.directory}/jacoco-integration.exec</dataFile> <!-- 3 -->  
                </configuration>  
                <executions>  
                    <execution>  
                        <id>agent</id>  
                        <goals>  
                            <goal>prepare-agent</goal> <!-- 4 -->  
                        </goals>  
                    </execution>  
                </executions>  
            </plugin>  
  
            <plugin>  
                <artifactId>maven-failsafe-plugin</artifactId>  
                <version>2.6</version>  
                <configuration>  
                    <argLine>${jacoco.agent.argLine}</argLine> <!-- 5 -->  
                    <reportsDirectory>${project.build.directory}/surefire-reports</reportsDirectory> <!-- 6 -->                   
                </configuration>  
                <executions>  
                    <execution>  
                        <goals>  
                            <goal>integration-test</goal>   
                            <goal>verify</goal> <!-- 7 -->  
                        </goals>  
                    </execution>  
                </executions>  
            </plugin>             
  
        </plugins>  
    </build>  
  
    <dependencies>  
        <dependency>  
            <groupId>junit</groupId>  
            <artifactId>junit</artifactId>  
            <version>4.11</version>  
            <scope>test</scope>  
        </dependency>  
    </dependencies>  
  
</project>  
{% endhighlight %}

1.  This property lets sonar know where to find JaCoCo reports for the integration tests. The `${env.WORKSPACE}` prefix enables the Sonar plugin to find these files.
2.  The JVM argument required to inject JaCoCo into the integration tests with be stored in a property named `jacoco.agent.argLine`.
3.  The `destFile` and `dataFile` properties indicate where JaCoCo will output its files.
4.  The `prepare-agent` goal will setup the JVM argument that we will reuse later.
5.  Property will inject the required JVM arguments to enable JaCoCo integration. Since JaCoCo does instrumentation at runtime, building and injecting the proper parameters is all that's required to get coverage information.
6.  Changing the `reportsDirectory` property is optional, and will cause the Failsafe plugin to output it's results to the same directory used by the Surefire plugin. The Surefire plugin executes Unit Tests in Maven, and it's results are read by SonarQube, and displayed as "Unit test success". Sending Failsafe's results to the Surefire report directory will trick Sonar into display the information. If you have a CI server such as Jenkins displaying this information, this "trick" might be superfluous.
7.  The Failsafe plugin will run in the `integration-test` and `verify` goals.

  

Other than the `pom.xml`, the test project includes to very basic tests, a unit test, and an integration test. Each one covers 50% of the classes in the projects.  You can get the complete source code [here](https://github.com/aolarte/tutorials.git). You will find a minimal example from which you can work up.

## Setting up a job with Sonar support in Jenkins

There's several ways of running SonarQube against a project:

*   Using the SonarQube plugin for Jenkins
*   Using the stand alone SonarQube Runner
*   Adding it to your Maven (or Ant) build

In this tutorial we will use the first option.  Personally I prefer to keep the pom as clean as possible, and have Jenkins perform extra steps such as analyzing code quality, deployment into package managers, etc.  
  
Make sure that you install the Jenkins SonarQube plugin:  
  

[![](http://1.bp.blogspot.com/-6qEkfxC6U-o/UwIjsHSrX6I/AAAAAAAAAYU/kQRwBeD2N5A/s1600/install+sonar+plugin.png)](http://1.bp.blogspot.com/-6qEkfxC6U-o/UwIjsHSrX6I/AAAAAAAAAYU/kQRwBeD2N5A/s1600/install+sonar+plugin.png)

  
The SonarQube plugin must be configured with one or Sonar installation.  Most shops will only have one Sonar installation.  This can be done in "Manage Jenkins" -> "Configure System"  
  

[![](http://3.bp.blogspot.com/-gw5DMhLgEPQ/UwIjsI2zhCI/AAAAAAAAAYY/x4IYRg-hPMs/s1600/sonar+config.png)](http://3.bp.blogspot.com/-gw5DMhLgEPQ/UwIjsI2zhCI/AAAAAAAAAYY/x4IYRg-hPMs/s1600/sonar+config.png)

  
In this case, we're using the defaults, since our SonarQube installation is running on <http://localhost:9000>, and we're using the embedded database, so there's no need to configure any of the database information.  
  
It is also useful to check "Skip if triggered by SCM Changes" and "Skip if triggered by the build of a dependency", to prevent Sonar running after every commit.  Running Sonar once a night is normally enough for most projects.  This assumes that you have one nightly build scheduled for every project that you want to analyze with SonarQube.  
  
We will then set up a simple Maven job in Jenkins:  

[![](http://1.bp.blogspot.com/-UvmwGTNCero/UwIZYu17D7I/AAAAAAAAAXw/n0bQmtOqFdY/s1600/new+project.png)](http://1.bp.blogspot.com/-UvmwGTNCero/UwIZYu17D7I/AAAAAAAAAXw/n0bQmtOqFdY/s1600/new+project.png)

  
We will check this project out of Source Control Management.  I'm going to use SVN to access a directory inside my git repository.  For real projects, I would recommend having a one git repository per each project:  
  

[![](http://1.bp.blogspot.com/-ZfvzoNCmykQ/UwIZZDHJRKI/AAAAAAAAAYE/EI0CJQ51ovg/s1600/svn.png)](http://1.bp.blogspot.com/-ZfvzoNCmykQ/UwIZZDHJRKI/AAAAAAAAAYE/EI0CJQ51ovg/s1600/svn.png)

  
Finally, we just add a "Post-build Action", to invoke SonarQube.  This will use the configuration for the SonarQube instance we configured before.  
  

[![](http://1.bp.blogspot.com/-NrjfP40DfT4/UwIZYrWCLwI/AAAAAAAAAX0/4AtOHx3a0uk/s1600/sonar.png)](http://1.bp.blogspot.com/-NrjfP40DfT4/UwIZYrWCLwI/AAAAAAAAAX0/4AtOHx3a0uk/s1600/sonar.png)

  
  
Finally you can start a build manually.  This will cause to project to first be built as usual using Maven, and the analyzed by Sonar:  
  

[![](http://2.bp.blogspot.com/-QQynaZ-_5mw/UwIZYpqqV8I/AAAAAAAAAX8/IywMrM2F2ws/s1600/start+a+build.png)](http://2.bp.blogspot.com/-QQynaZ-_5mw/UwIZYpqqV8I/AAAAAAAAAX8/IywMrM2F2ws/s1600/start+a+build.png)

  

## The results

After the build is run successfully by Jenkins, you can see the results on your SonarQube installation.  Make sure that you have the "Integration Tests Coverage" widget in your dashboard.  To add it, log into SonarQube, and click on "Configure widgets".  
  

[![](http://3.bp.blogspot.com/-0GZndfmpS5I/UwI54Xa3jrI/AAAAAAAAAY4/RrpUC3bt6_E/s1600/sonar_result.png)](http://3.bp.blogspot.com/-0GZndfmpS5I/UwI54Xa3jrI/AAAAAAAAAY4/RrpUC3bt6_E/s1600/sonar_result.png)

  
You can see how Test Coverage for Integration Tests is reported separately, and that there's a metric for "Overall Coverage", which joins the coverage for unit tests and integration tests.  If you drill down can see much more detail about your project, so I encourage to click around and see what Sonar has to say about your code.

## Source Code

As always the code is available for you to download and play with. Clone it from it from github:  

    git clone https://github.com/aolarte/tutorials.git  

The finished code is contained in directory `coverage`.  
  

## References

This post was put together with tidbits of information from other postings around the net:  
  
*   [Separating Integration and Unit Tests with Maven, Sonar, Failsafe, and JaCoCo](http://theholyjava.wordpress.com/2012/02/05/separating-integration-and-unit-tests-with-maven-sonar-failsafe-and-jacoco/)
*   [Tracking Integration Test Coverage with Maven and SonarQube](http://davidvaleri.wordpress.com/2013/09/06/tracking-integration-test-coverage-with-maven-and-sonarqube/)
*   [Easy Unit and Integration Code Coverage](http://www.agile-engineering.net/2012/05/easy-unit-and-integration-code-coverage.html)