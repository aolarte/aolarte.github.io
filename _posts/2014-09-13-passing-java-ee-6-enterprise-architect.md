---
permalink: /:categories/:year/:month/:title.html
layout: single
title: Passing the Java EE 6 Enterprise Architect Certified Master Part 2 and 3 (1Z0-865
  and 1Z0-866)
date: '2014-09-13T17:03:00.000-07:00'
tags:
- architecture
- certification
- java
modified_time: '2014-09-14T07:47:33.847-07:00'
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-6836529113934106024
blogger_orig_url: http://www.javaprocess.com/2014/09/passing-java-ee-6-enterprise-architect.html
---

## The "Plan"

I finally got my result for the assignment and essay tests for the Oracle Certified Master, Java EE 6 Enterprise Architect (1Z0-865 and 1Z0-866). I wanted to take this opportunity to share some of my experience on this endeavor. If you want to read about part one of the certification, my notes are available [here](/2014/06/passing-java-ee-6-enterprise-architect.html).  
  
I downloaded my assignment on June 30th, and spent the next few days reading and re-reading the assignment. I set myself a very aggressive (and overly optimistic) timeline to finish the assignment in 4 weeks.  
  

### Week 1

In week one I wanted to brush up on some UML, since this was weak area for me. I used two books that were available in my local public library:

*   "Fast track UML 2.0" by Kendall Scott
*   "Learning UML 2.0" by Miles, Russ

I also wanted to prototype some aspects of the application. Working with J2EE 6 is fast enough that a simple application can be created painlessly. It was also a good opportunity to think how I would create a green field application under "perfect" circumstances, something that is rarely possible in the real world. This prototype would also allow me to validate some of the architectural decisions.  
  

### Week 2

Continuing with the prototype, the idea was to test some of the more complicated aspects that would be required to complete the assignment. Some of these technologies I was comfortable with, while others proved a learning experience:

*   Authentication (using JASPIC)
*   RMI
*   JSF

### Week 3

The plan was to create all of the diagrams during the third week. For the most part the plan was to follow the diagrams as shown in"Sun Certified Enterprise Architect for Java EE Study Guide" by Mark Cade and Humphrey Sheil.  
  
For the UML diagrams I tried free different tools:  
  
#### JDeveloper
  
Pros: Nice looking diagrams. Very good Java round tripping. I was already somewhat familiar with this tool, since I had used it at work to create some basic diagrams.  
Cons: Not suitable to create component and deployment diagrams. A memory hog prone to crashing every so often.  
  
#### StarUML
  
Pros: Simple to use.  
Cons: UML 2.0 support was lacking. Some people mentioned that diagrams could be 1.x UML compliant, but I wanted to focus on 2.0 diagrams. The diagrams were not very aesthetic.  
  
#### Visual Paradigm Community Edition
  
Pros: Nice looking diagrams and plenty to format  
Cons: The Community Edition only support one diagram per type. This only proved to be a problem in the sequence diagrams, in which I had to create one file per diagram.

I ended up using Visual Paradigm, which proved to be the most mature free UML tool I could find.

### Week 4

In week for I was hoping to review everything, create the HTML, and write the top 3 risks, and all of the assumptions.

## How everything went down

All in all, my four week plan took seven weeks. While I was able to meet most of my objectives for the first three weeks (including a mostly complete prototype), I underestimated the time required to do the diagrams. I had to familiarize with a new tool (Visual Paradigm), and learn parts of UML that I had never learned. As other people around the net mention, the sequence diagrams take a lot of time to complete. The last three weeks were spent refining my design, simplifying as much as possible, ensuring that the design was elegant, and not just workable. This required redoing some of the diagrams. I would stop working for a day or two, and then review my design from top to bottom, making changes as needed. The more iterations, the less I was changing, until in week seven I felt comfortable with the finished product. I submitted my assignment and signed up for the essay the very next Saturday.  
As a rough estimate, I would say I spent around 150 hours total preparing the assignment. I could have done it in less time if had not done a prototype, and if I didn't have to try several new UML tools.  
For my solution the sample shown in the Cade and Sheil book was main resource, and I can recommend it as a good concise resource. Even though the book is targeted at SCEA 5 (OCMJEA 5), it's worth nothing that the assignment is the same for OCMJEA 5 & 6.

I also bought the training tool from EPractizeLabs [SCEA 5 Part 2 and 3 Certification Training Lab](http://www.epractizelabs.com/certification/sun/scea-5-part23-exam.html). In my particular case this tool wasn't very valuable, primarily because I bought it once I had done most of the work, so it just helped me verify that there wasn't anything else for me to do. This training lab provides more sample solutions than the single one presented in the Cade and Sheil book, but it becomes evident that in any solution you're always covering the same aspects.  
I finished my certification before the "OCM Java EE 6 Enterprise Architect Exam Guide" by Allen and Bambara was available, so I do not have a comment on how useful their material is.  

## Looking back, my tips and recommendations

*   Read and read and read the assignment until have three things clear:

*   What your application needs to do: the functional requirements.
*   How your application should operate. These are the Non Functional Requirements (NFR), and includes aspects such as performance, security, availability, etc.
*   What systems you have to integrate with, and how you can integrate with them. This includes what protocol to use, async vs. sync integration, etc.
*   Ensure that you implement the business object model as described in the assignment. You can augment it, but do not change it. If it doesn't make sense, read it again until it does. If it still doesn't make sense, add assumptions so that it does make sense. 
*   As you're designing the solution keep a list of risks and assumptions. This will make your life much easier later on. I did not do this so I spent a considerable amount of time going back and forth and trying to remember some of the justifications I had though about
*   Keep in mind all of your NFR, and make sure you address them in your design and your assumption. Even if you think something is obvious, write it down. You're the architect, so everything is either something you have to design, or something you assume is already in place. Either way, document it. I'm talking about things like network connections, databases, external systems, clients, etc. The NFR normally taken into account are performance, scalability, reliability, availability, maintainability, extensibility, manageability, and security. If you need to refresh some of these concepts you can take a look at my [notes for the part 1 exam](/2014/06/passing-java-ee-6-enterprise-architect.html). 
*   If something doesn't feel right, don't be afraid to change it. I redid my diagrams several times. Some changes were very significant, for example changing from one technology framework to another. Other changes were minor, such as polishing naming conventions to make it easier for the instructor to understand what I was doing. The biggest change I did was to convert most of my business logic from Stateless Session Beans to CDI Managed Beans, since EJB provided no useful benefit to solving the problem at hand. I did keep some Message Driven Beans (MDB) and Singleton Session Beans that had specific requirements.
*   When I do an assignment I normally try to do what's required, nothing more, nothing less. In this case in particular I did do a few extra diagrams to explain some of the trickier elements that were not evident in the required diagrams. This included a diagram of the JMS queues, a package diagram to make it easier to understand my class diagram, and an activity diagram detailing how the chosen platform would support one the NFR. Of course this is dependent on your solution, but I feel some extra clarification can be helpful.
*   Don't be afraid to be specific. I chose a specific Application Server (Weblogic in my case), and I justified it since it had particular features to help me meet the particular NFR of my assignment. The same applies to the other aspects of the deployment, server specs, OS specs, database specs, security and firewalls, physical server security, application management. Remember, you're the architect, so every detail in ensuring a successful solution is part of your job. 

## The essay and the wait...

The essay part has a duration of 120 minutes, and be prepared to use all of the time. I'm sure I could have kept going for at least a couple more hours. If you did the assignment correctly, and addressed all of the functional and non functional requirements, you should have more than enough to fill two hours of writing. 

Be ready to justify your choice of technologies and architectures, specially compared to other technologies that could have been used in those cases. This will show that you have a well rounded knowledge of the Java ecosystem, and can make an informed decision between all of the different frameworks out there. This includes technologies such as persistence mechanisms (JPA, Hibernate, JDBC, NoSQL) and web frameworks (JSF, Spring MVC, Struts, etc.). Do you need a full blown EAR vs a simple WAR?

In this part I took special care to justify what I thought was a very controversial part of my design. I will not go into details, suffice to say that I did not have one of the pieces that is almost always present in a modern web application. I had already justified this in my assignment, but I went the extra mile in the essay and described how I would have done it in the "traditional way" if my assumptions were not applicable. I also explained all of the potential downsides of the "traditional way".

The grading took almost three (nerve-wrecking) weeks, but I've read of greatly varying times. I passed with 147 out of 160 possible points, which makes me feel confident of my process. Unfortunately the score report does not shed any light on what I could have done better. Only time will tell how valuable this certification will be for me, but I can at least say that the learning process helped me to structure many of the activities I have already been doing, and solidified some of my weaker areas. 

If you have any questions, feel free to ask.

And if you're on your way to become an OCM, best of lucks!