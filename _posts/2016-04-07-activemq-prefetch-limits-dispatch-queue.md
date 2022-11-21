---
permalink: /:categories/:year/:month/:title.html
title: ActiveMQ, prefetch limits, the Dispatch Queue and transactions
date: '2016-04-07T07:28:00.000-07:00'
tags:
- jms
- activemq
- java
modified_time: '2016-04-07T07:30:58.932-07:00'
thumbnail: https://1.bp.blogspot.com/-CvHgQMNb-BY/VwZoJdhrbeI/AAAAAAAAHO4/kbcg_cgRnIce_zif6SQzUtoL-waO6rTmg/s72-c/activemq-logo.png
blogger_id: tag:blogger.com,1999:blog-3306197464901287625.post-8955958985713510673
blogger_orig_url: http://www.javaprocess.com/2016/04/activemq-prefetch-limits-dispatch-queue.html
---

[![](https://1.bp.blogspot.com/-CvHgQMNb-BY/VwZoJdhrbeI/AAAAAAAAHO4/kbcg_cgRnIce_zif6SQzUtoL-waO6rTmg/s200/activemq-logo.png)](http://1.bp.blogspot.com/-CvHgQMNb-BY/VwZoJdhrbeI/AAAAAAAAHO4/kbcg_cgRnIce_zif6SQzUtoL-waO6rTmg/s1600/activemq-logo.png)The objective of this article is to describe the interaction of ActiveMQ and its consumers, and how message delivery is affected by the use of transactions, acknowledgement modes, and the prefetch setting.  This information is particularly useful if you have messages that could potentially take a long time to process.  

## The Dispatch Queue

The Dispatch Queue contains a set of messages that ActiveMQ has destined to be sent to a particular consumer.  These messages are not available to be sent to any other consumers, unless their target consumer runs into an error (such as being disconnected). These messages are streamed to the consumer, to allow faster processing.  This is also referred to as "pushing" messages to the consumer. This is in contrast to consumer polling or "pulling" messages when it's available to process a new one.  
The prefetch limit is defined by the ActiveMQ [documentation](http://activemq.apache.org/what-is-the-prefetch-limit-for.html) as "how many messages can be streamed to a consumer at any point in time. Once the prefetch limit is reached, no more messages are dispatched to the consumer until the consumer starts sending back acknowledgements of messages (to indicate that the message has been processed)".  Basically, the prefetch limit defines the maximum number of messages to assign to the dispatch queue of a consumer. This can be seen in the following diagram (the dispatch queue of each consumer is depicted by the dotted line):  

[![](https://4.bp.blogspot.com/-NxHlDe0UDkU/VwQt0F504rI/AAAAAAAAHMQ/O4F7N6Q7m4YuIeDpCIPHYsSFHEUqyD94A/s1600/dispatch%2Bqueue1.png)](https://4.bp.blogspot.com/-NxHlDe0UDkU/VwQt0F504rI/AAAAAAAAHMQ/O4F7N6Q7m4YuIeDpCIPHYsSFHEUqyD94A/s1600/dispatch%2Bqueue1.png)

Dispatch queue with a prefetch limit of 5 and transactions enabled in the consumer
  
Streaming multiple messages to a client is a very significant performance boost, specially when messages can be processed quickly.  Therefore the defaults are quite high:  

*   persistent queues (default value: 1000)
*   non-persistent queues (default value: 1000)
*   persistent topics (default value: 100)
*   non-persistent topics (default value: `Short.MAX_VALUE -1`)

The prefetch values can be configured at the connection level, with the value reflected in all consumers using that connection. The value can be overridden in a per consumer basis. Configuration details can be found in the [ActiveMQ documentation](http://activemq.apache.org/what-is-the-prefetch-limit-for.html).  

Normally messages are distributed somewhat evenly, but by default ActiveMQ doesn't guarantee balanced loads between the consumers (you can plug in your own DispatchQueue policy, but in most cases that would overkill). Some cases in which the messages are unevenly distributed might be caused by:  

*   A new consumer connects after all of the available messages have already been committed to the Dispatch Queue of the consumers already connected.
*   Consumers that have different priorities.
*   If the number of messages is small, they might all be assigned to a single consumer.

Tuning these numbers is normally not necessary, but if messages take (or could potentially take) a significant long time to process, it might be worth the effort to tune.  For example, you might want to ensure a more even balancing of message processing across multiple consumers, to allow processing in parallel.  While the competing consumer pattern is very common, the ActiveMQ's Dispatch Queue could get in your way. Particularly, one of the consumers can have all of the pending messages (up to the prefetch limit) assigned to its Dispatch Queue. This would leave other consumers idle.  Such a case can be seen below:  

[![](https://3.bp.blogspot.com/-K1TGxj2_yCM/VwZsR38D6SI/AAAAAAAAHPE/hhZDgtxaJoQTJBss8VlhBMmuHgaY8icgA/s1600/dispatch%2Bqueue1-5.png)](http://3.bp.blogspot.com/-K1TGxj2_yCM/VwZsR38D6SI/AAAAAAAAHPE/hhZDgtxaJoQTJBss8VlhBMmuHgaY8icgA/s1600/dispatch%2Bqueue1-5.png)

Dispatch queue with a prefetch limit of 5 and transactions enabled in the consumer

 
This is normally not a big issue if messages are processed quickly.  However, if the processing time of a message is significant, tweaking the prefetch limit is an option to get better performance.  
  

## Queues with low message volume and high processing time

While it's a best practice to ensure your consumer can process messages very quickly, that's not always possible. Sometimes you have to call a third party system that might be unreliable, or the business logic just keeps growing without much thought about the real world implications.  
For consumers with very long processing times, or very variable processing time, it is recommended to reduce the prefetch queue.  A low prefetch limit prevents messages from "backing up" in the dispatch queue, earmarked for a consumer that is busy:  

[![](https://1.bp.blogspot.com/--dkyA60BfYU/VwQyjGjumlI/AAAAAAAAHNI/fBk_tBJSmZkRxaeHkfwZBA4lRQ2_Ex3cQ/s400/dispatch%2Bqueue2-5.png)](http://1.bp.blogspot.com/--dkyA60BfYU/VwQyjGjumlI/AAAAAAAAHNI/fBk_tBJSmZkRxaeHkfwZBA4lRQ2_Ex3cQ/s1600/dispatch%2Bqueue2-5.png)

Dispatch queue with a prefetch limit of 5 and transactions enabled in the consumer

  
This behavior can be seen in the ActiveMQ console with a symptom most people describe as "stuck" messages, even though some of the consumers are idle.  If this is the case, it's worth examining the consumers:[![](https://3.bp.blogspot.com/-VLfv_4M6zY8/VwUin5D8n2I/AAAAAAAAHOY/NC_VxyfCRLsz1xx2cFqUbHx8ikkYAlOWA/s1600/Screen%2BShot%2B2016-04-06%2Bat%2B9.50.45%2BAM.png)](http://3.bp.blogspot.com/-VLfv_4M6zY8/VwUin5D8n2I/AAAAAAAAHOY/NC_VxyfCRLsz1xx2cFqUbHx8ikkYAlOWA/s1600/Screen%2BShot%2B2016-04-06%2Bat%2B9.50.45%2BAM.png)  

The "Active Consumers" view can help shed light into what is actually happening:  

[![](https://4.bp.blogspot.com/-8EM0iWVKBRE/VwUjLAc4PsI/AAAAAAAAHOk/-zIsdwOYYnQ9quQ0kVwcHAJlLXBuh8T3g/s1600/Screen%2BShot%2B2016-04-06%2Bat%2B9.48.24%2BAM.png)](http://4.bp.blogspot.com/-8EM0iWVKBRE/VwUjLAc4PsI/AAAAAAAAHOk/-zIsdwOYYnQ9quQ0kVwcHAJlLXBuh8T3g/s1600/Screen%2BShot%2B2016-04-06%2Bat%2B9.48.24%2BAM.png)

Screenshot showing a consumer with 5 messages in its Dispatch Queue. The prefetch limit is set at 5 for this consumer.

  
  
To address the negative effects of such cases, a prefetch limit of 1 will ensure maximum usage of all available consumers:  

[![](https://3.bp.blogspot.com/-zNI9pOEgk6c/VwQx9PQotQI/AAAAAAAAHM8/7RuHuJS1vPkw8KjB7qlaGUBzWLYwLxRWg/s400/dispatch%2Bqueue2.png)](http://3.bp.blogspot.com/-zNI9pOEgk6c/VwQx9PQotQI/AAAAAAAAHM8/7RuHuJS1vPkw8KjB7qlaGUBzWLYwLxRWg/s1600/dispatch%2Bqueue2.png)

Dispatch queue with a prefetch limit of 1 and transactions enabled in the consumer

  
This will negate some of the efficiencies of streaming a large number of messages to a consumer, but this is negligible in cases where processing each message takes a long time.  

## The Dispatch Queue and transactions (or lack thereof)

When the consumer is set to use `Session.AUTO_ACKNOWLEDGE`, the consumer will automatically acknowledge a message as soon as it receives it, and then it will start actually processing the message.  In this scenario, ActiveMQ has no idea if the consumer is consumer is busy processing a message or not, and will therefore not take that message into account for the dispatch queue delivery.  Therefore, it's possible for a **SECOND** message to be queued for a busy consumer, even if there is another consumer idle:  
  

[![](https://4.bp.blogspot.com/-ooxQxY7dAQ8/VwQxmKBNsxI/AAAAAAAAHM0/M4mQNFYKIN0JcYmVa72DbJ4idz7bHsNBg/s400/dispatch%2Bqueue3.png)](http://4.bp.blogspot.com/-ooxQxY7dAQ8/VwQxmKBNsxI/AAAAAAAAHM0/M4mQNFYKIN0JcYmVa72DbJ4idz7bHsNBg/s1600/dispatch%2Bqueue3.png)

Dispatch queue with a prefetch limit of 1 and auto acknowledge enabled

If Consumer 1 takes a long time processing its message, the second message could will take a long time to even start being process.  This could have significant impact on the performance issues.  Normal troubleshooting might a few discrepancies  
  

*   We have messages waiting to be picked up in ActiveMQ
*   We have idle consumers

How can we this situation be prevented? For such cases, one option is to disable the dispatch queue altogether, by setting the prefetch limit to zero.  This cause force consumers to have to fetch a message every time they're idle, instead of waiting for messages to be pushed to them.  This will further degrade performance of the JMS delivery, so it should be used will care.  However this will ensure that all available consumer are kept busy:

  

[![](https://3.bp.blogspot.com/-ac1Q0XLTeqA/VwQ2qx1Ji9I/AAAAAAAAHNY/hCtAa7zVpQYMgVN4hXdpXIgXv_bfrMvgA/s400/dispatch%2Bqueue4.png)](http://3.bp.blogspot.com/-ac1Q0XLTeqA/VwQ2qx1Ji9I/AAAAAAAAHNY/hCtAa7zVpQYMgVN4hXdpXIgXv_bfrMvgA/s1600/dispatch%2Bqueue4.png)

Consumer with no dispatch queue (prefetch limit set to zero)

  

## Final thoughts and considerations

While the prefetch limit default is good enough for most applications, a good understanding of what happening under the covers can go a long way in tuning a system.