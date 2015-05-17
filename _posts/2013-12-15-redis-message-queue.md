---
title: Building a Message Queue in Redis
author: eric
layout: post
permalink: /redis-message-queue/
categories:
  - distributed computing
  - message queue
  - Redis
---
A few months ago a co-worker sent me <a title="Redis Message Queue" href="http://davidmarquis.wordpress.com/2013/01/03/reliable-delivery-message-queues-with-redis/" target="_blank">this</a> blog post to give me a few ideas for a development message queue system that he wanted. We are planning on using <a title="Kafka" href="https://kafka.apache.org/" target="_blank">Kafka</a> in our production environment, but he wanted something lighter weight (no dependency on zookeeper, etc) so that devs working on our platform could bring up and develop against our message queue library quickly. I borrowed a few ideas from the linked post, but also changed the architecture a bit in order to tailor the system for our needs. This was also a learning exercise in Redis, which is a part of our stack that I wanted to become more familiar with.

## Requirements

The following is a list of requirements for our development message queue system:

  * **Concurrency** &#8211; the ability to read off queues from several different threads/processes without issue.
  * **Topic Replay** &#8211; the ability to replay the entirety of a topic to a consumer. 
      * EDIT: This functionality is less realistic than I originally presumed. The ability to replay live messages on a topic is more accurate. Kafka provides a mechanism for configuring the max amount of time to retain logged messages. A full replay of all historical messages from a topic would involve an outside system.
  * **Language Agnostic** &#8211; the ability to provide APIs for a variety of languages.

## System Design

As I mentioned before, the groundwork for this system was already laid out for me by the linked blog post. I changed a few small details in order to comply with the above requirements.

<img class=" wp-image-58 alignright" alt="redismq" src="/images/redismq-300x141.png" width="400" height="200" />

To account for topic replay, each consumer/topic relationship is represented in Redis as a key with an integer value pointing at the last message that the consumer retrieved. Each message is stored under its own key containing the topic name and index of the message. Since this system is only being used on development machines with a few consumers, the throughput is fairly low. For this reason the TTL is not being set on each message, and if new consumers come into play, they will see the entire message queue from the beginning (since their index will begin at 0 on entry). If this system were to be used in a production environment, the expiration of each key would need to be set, as all keys are stored in memory.

## Sending a Message

Sending a message in this system is very simple. Each message is granted a new key in the format <topic>:messages:<index>. A transaction is used to ensure that if multiple threads/producers are pushing to the same topic, they do not inadvertently overwrite a single index.

Incrementing in Redis is atomic, so a transaction may seem like overkill here. The reason behind this decision was that each time a message is placed into a queue, the nextIndex key should only be incremented if the message was placed into the queue without error. To ensure this extra bit of fault tolerance, we get the nextIndex from Redis and increment it locally. That incremented index is then used to create the key for the message being pushed. The nextIndex increment and message insert (using the locally incremented index value) are both performed in the same transaction to ensure that each operation completes without error. If they do not, then the process is repeated (with a configurable retry limit).

<img class="alignleft  wp-image-60" alt="redismq-produce (1)" src="/images/redismq-produce-1-300x116.png" width="400" height="155" />

In an example scenario, for a topic foo with a nextIndex value of 3 (before sending a message), the nextIndex would be incremented to 4 locally. The message key would be foo:messages:4. Incrementing foo:nextIndex to 4, and setting foo:messages:4 with the given message would occur in one transaction.

## Consuming Messages

Each consumer is granted a key per topic that denotes the last index that was consumed from the topic. Again, a transaction and watches are used to ensure that all portions of the consumption are completed without error. The consumer key is incremented locally to determine if there is a message to consume (the incremented consumer key must be less than the nextIndex key for the producer of the topic). The key is then incremented in Redis, and the corresponding message is retrieved from the producer key which is composed from the topic and the nextIndex value for the consumer/topic.

If a consumer wishes to replay a topic, they can unsubscribe/re-subscribe to the topic which will reset their nextIndex value for that topic to 0. This will effectively replay all live messages on the topic for that consumer.

<img class="alignleft  wp-image-62" alt="redismq-consumer (1)" src="/images/redismq-consumer-1-300x129.png" width="400" height="185" />

In another example, consider a consumer with an ID of bar, and a topic called baz. The consumer last retrieved the message with ID 3 from the baz topic, and baz has messages up to the ID of 12. The consumer&#8217;s nextIndex key will be incremented locally, and the check for a new message from the topic will pass (4 < 12). The increment will then be executed (a watch was placed on the nextIndex key, so that if another thread incremented the index in the meantime, we would try again and get the correct message). If the increment succeeds without error, the message will be retrieved.

## Conclusion

Redis was a treat to work with. It is very easy to pick up, and the set of capabilities out of the box is great for small projects that require inter-process communication and ease of use. I have also implemented a work queue system with Redis that was very quick to put together, without sacrificing any capability. I am hoping that my employer will open-source the code for this message queue, as it is a nice learning tool and it also has potential to become a production-ready queuing system.
