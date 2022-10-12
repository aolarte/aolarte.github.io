---
permalink: /:categories/:year/:month/:title.html
layout: single
title: 'My Study notes for  Oracle WebLogic Server 12c: Advanced Administrator II
  (1z0-134)'
date: '2015-03-14T14:40:00.002-07:00'
tags:
- weblogic
- oracle
- certification
- study guide
modified_time: '2015-03-28T06:11:56.113-07:00'
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-1022353190104622599
blogger_orig_url: http://www.javaprocess.com/2015/03/my-study-notes-for-oracle-weblogic.html
---

I recently took the "Oracle WebLogic Server 12c: Advanced Administrator II".  I actually took the beta exam (1z1-134).  This is the exam that will eventually become 1z0-134.  I wanted to share some of my experience.  You will also find my notes down below. As a took, the beta exam, some of the questions I encountered on the test will not on the final production test, so take my advice with that caveat.  
I assume that if you're reading this, you have already achieved the Weblogic OCA level certification, and therefore have basic knowledge.  
  

*   **Practice (a lot of) scripting** For this test, practicing on a real Weblogic instance is a must.  Practice doing routine (and not so routine tasks) using WLST.  Things like monitoring running services, online and offline editing, deploying applications, creating some of the more esoteric Weblogic components. There are plenty of questions regarding scripting.
*   **Learn your JMS** Besides knowing how to setup the different JMS components in Weblogic, review some of the JMS internals, like persistence, acknowledgments, durable subscribers, etc. Review how these different operation methods are reflected in JMS headers. There's a significant focus on JMS. I found this [post](http://developsimpler.blogspot.com/2011/11/jms-performance-tuning-series-part-1.html) useful regarding JMS tuning.
*   **The NodeManager and service migration** There's plenty of questions on different scenarios and the interaction between the NodeManager and migration. I feel that this section requires plenty of work, since most people will only be exposed to only a few of these scenarios in their day to day work.

*   Service vs. whole server migration
*   Script vs. Java NodeManager
*   Consensus vs. Database leasing

*   **WLDF** For me, WLDF (Weblogic Diagnostic Framework), was a fairly weak area.  Make sure you play around with all of the pieces of the framework.
*   **Coherence** Coherence was another weak area for me, having never had practical experience with it.  However, the questions related to Coherence we fairly basic. If you need to get started, this [post](https://coherencedownunder.wordpress.com/2014/05/24/getting-started-with-coherenceweb-in-weblogic-server-12-1-2/) helped me a lot.
*   **Linux** I was surprised to find some basic linux questions.  I would say they were fairly strait forward, and anyone who had done some day to day troubleshooting will have no problem with them.  If you have only worked under Windows, some research will come in handy.

Below you will find my notes.  Most of my notes come the Weblogic documentation found [here](http://docs.oracle.com/middleware/1213/index.html).

  
You can access my notes directly by using this [link](https://docs.google.com/document/d/1XyyuQ0hb_KNqHKnHlNPV18PhdmPDoWS3xXSMOPLK3Ts/pub).  
  
Now the wait is on for the beta period to finish, and to see how well I did. Regardless, this proved to be a very insightful experience, in which I was able to fill quite a few weak areas in my Weblogic knowledge.