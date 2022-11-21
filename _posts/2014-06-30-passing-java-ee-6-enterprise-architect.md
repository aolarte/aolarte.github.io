---
permalink: /:categories/:year/:month/:title.html
title: Passing the Java EE 6 Enterprise Architect Certified Master Part 1 (Exam 1Z0-807)
date: '2014-06-30T10:34:00.003-07:00'
tags:
- architecture
- certification
- java
modified_time: '2014-09-13T17:17:25.936-07:00'
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-5426010454744815823
blogger_orig_url: http://www.javaprocess.com/2014/06/passing-java-ee-6-enterprise-architect.html
---
Last Saturday I passed the 1Z0-807 test, and I wanted to share some of my experience, specially since there seems to be little information about this test (compared to other Java certification tests at least). You can also take a look at my study notes. You can see the my [notes for part 2 & 3 here](/2014/09/passing-java-ee-6-enterprise-architect.html).  
  
My total time to prepare was around 6 weeks. It was a slow time at the office, so I think I was doing around 15-20 hours of reading / studying per week.  
  
Going into the studying phase I felt I was very weak in the EJB and JSF parts of the material, since it's not something that I actively use at work, while on the other technologies I felt I had a moderate to strong grasp on the material.  
  
I think 6 weeks was more than enough time to prepare if you have a background working with database driven Java web applications (even if not using the full Java EE stack as in my case). Your pace may vary, and honestly I signed up for the test before I had started studying, so the time constraint was my motivation.  
  
Books that I read to prepare for this test:  
  

*   "The Java EE 6 Tutorial" by Eric Jendrock et al. (you can get it [here](http://docs.oracle.com/javaee/6/tutorial/doc/))
*   "Real World Java EE Pattern" by Adam Bien (the second edition)
*   "Sun Certified Enterprise Architect for Java EE Study Guid" by Mark Cade and Humphrey Sheil
*   "Bitter Java" and "Bitter EJB" by Bruce Tate
*   "SOA in Practice" by Nicolai Josuttis

Books that I had read in the past that helped:  
  

*   "Design Patterns: Elements of Reusable Object-Oriented Software" Erich Gamma,Richard Helm,Ralph Johnson,John Vlissides
*   "Enterprise Integration Patterns" by Gregor Hohpe

  

The biggest issue I had was the lack of a proper study guide for this test. The study guides I found were for the previous version of the test, based on Java EE 5, however most of the material is relevant. I found the Java EE 6 tutorial very useful to make sure I figured what were the gaps that I needed to fill. An specific guide for this test is supposed to be available very soon, so you might have better luck than I did. If and when it becomes available, by all means read it to guide yourself.  
  
In general I recommend getting through the material as described by the outline once, and then practicing and practicing some more. Get as many practice questions as you can. Some of those questions have subjective answers, but the more you practice the better chance you will have. Currently the only test simulator is from [EPracticeLabs](http://www.epractizelabs.com/), and while not awesome I think is more than enough to make it through.  
  
Relevant things that I found are not directly mentioned in the outline but are significant in the exam:  
  

*   Design patterns and anti-patterns (covered by the books listed above)
*   Security (specially risks and how to mitigate them)
*   When to use a technology. For example when to use EJB or just a web container. When to use DAOs, JDBC, or JPA. When to use durable subscribers in JMS. For this I would recommend you practice until you can detect the hints that the give in the questions regarding portability, scalability, etc.

Regarding the exam itself I can point out:  
  

*   The exam was what I expected based on the practice questions I had seen. No surprise here.
*   There were many scenario questions with incomplete information. So it get's down to eliminating obvious wrong answers and then some luck picking from what's left.
*   Only multiple choice questions (pick one, or pick X). No drag and drop or anything like that.
*   Time is plentiful. The give you 150 minutes (two and a half hours). I'm fast at taking tests, but it took me thirty minutes to do my first pass, and probably ten minutes to review.
*   Some questions can get ambiguous if you over think them. For example, one-tier is a mainframe application, two-tier is thick client, three tier is web based application. Don't over think about a contrived three tier application with thick clients talking and sharing business logic with EJBs unless specifically told so on the questions.
*   Finally, they didn't give me result right away. They said I would get an email in 30 minutes with the result, but it arrived in a matter of five minutes or so after finishing the test. I haven't taken an Oracle test in more than 3 years, but I don't remember this being the case. 

  
  

You can take a look my study notes below. They do borrow material from some Java EE 5 notes that are floating around in the internet:

*   http://java.boot.by/scea5-guide/
*   http://rieck.dyndns.org/architecture/scea5.html

  

  

  

  
  
Here's the direct link to the [document](https://docs.google.com/document/d/19wRpcFyLH309H_W1jFxTN-YOd7szCe4yilXtPuS9jKs/pub).  
  
If you have any questions, feel free to add a comment and I'll try to help you.  
  
You can now see the my [notes for part 2 & 3 here](/2014/09/passing-java-ee-6-enterprise-architect.html).