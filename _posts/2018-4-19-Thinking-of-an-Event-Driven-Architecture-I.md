---
layout: post
title: Thinking of an Event Driven Architecture I
tags: Event Sourcing Driven Messaging Log Kafka
---

With this post I want to start a series of post about event driven architectures. My goal is to sketch an architecture for an event driven microservice architecture using kafka as the central [Log](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying) infrastructure.

In recent years with the advent of [Kafka](https://kafka.apache.org/) and other distributed and by that failsafe event bus systems event driven architectures have become more and more popular. Especially in the area of microservices theses systems start to replace classical REST/HTTP(S) interfaces for inter-service communication. This is because they offer builtin support for downtimes in the service landscape and the different speed of processing of different services.

That is a huge advantage over classical HTTP(S) interfaces as a downtime in one service or a slow service will not compromise the whole service landscape. It also reduces the complexities you usually have when using circuit breakers.

Basic Idea
---

In this section I want to propose a first idea of how to structure the architecture. I also want to explain the advantages and gotchas of the proposes architecture.

The following diagram shows an overview of the architecture:

![EventDrivenArchitecture]({{ "/assets/img/EventDrivenArchitecture.svg" | absolute_url }})

In this architecture I split the whole landscape into different app services. Each app service is made up of:

- Input event queue: This is the queue where all requests that the app service should handle are posted
- Workers: A set of workers that handle all input events
- Materialization DB: A database where the workers put a materialized view of the data as required by the HTTP API of the app service
- HTTP API: An HTTP endpoint for the app service that can for example be used by the UI (of the service), it can read the Materialization DB for presentation of the data and post new events to the input queue to request changes
- Output Event Queue: This is where workers can put events that describe the results of input events, external workers can subscribe this queue

Note that I do not assume this setup to be complete yet. I will explore most common topics during the course of this series and expand the architecture as required.

Materialization
---

Perhaps you know the term materialized view from topics like event sourcing. Similarly I coined the central DB where the results of the events are stored Materialization DB. This does not necessarily mean that you drive down the event sourcing road. It just means that you have a database where you can properly query data for your UI. Because the event bus systems, like Kafka, that are used for the input and output event queues are not meant to be queried efficiently.

So let's say an input event _e<sub>in</sub>_ arrives in the input queue. In some architectures this is named a command, a request to do some work. It is in fact nothing else here but still it is just an event stored in a queue.

The worker will be continuously pulling events from the input queue and will eventually arrive at the said command event _e<sub>in</sub>_ that was put there by some external client. To handle this event the worker will usually perform the following tasks:

1. perform a change in the materialization DB
2. post one or more output events _e<sub>out</sub>_ to the output event queue
3. commit the input event _e<sub>in</sub>_

It is crucial that:

- the update of the materialization DB is indemptotent because when pulling the input events the events _e<sub>in</sub>_ can be handed to us more than once (at least once delivery)
- similarly the output events _e<sub>out</sub>_ can be created multiple times and should be designed in a way that this is not a problem
- commiting the event _e<sub>in</sub>_ after finishing the previous steps is crucial;do not use auto commit like Kafka offers because the database update could fail after a timeout and after the auto commit was performed

Scalability in Materialization
---

Note that I wrote "the worker" because scaling the numbers of workers requires some thought. Nevertheless scalibility is one of the main reasons why we should switch to an event based architecture. Therefore, considering this topic here at least shortly is worth it. Scalability of your workers completely depends on how you can divide your data. Dividing your data comes in two flavors:

- creating data sections, e.g. one set of tables and another set of tables
- creating data shards, e.g. one shard per range of ordered last names of customers 

You can have one worker per data section and shard.

![DataShardsAndSections]({{ "/assets/img/DataShardsAndSections.svg" | absolute_url }})

But why is it important to divide the data like that? And why can you only have one worker per shard and section?

A worker subscribes exactly one event bus queue and will materialize all events in that queue in order. If you attach a second worker to that queue even assuring that each event is only handled once the events in that queue could be handled out of order.

For example think about two consecutive updates to a customer the second one overwriting the first one. But if they are handled in a different order the result will be completely different and the second update will be lost.

Usually thinking about implications of that is hard, so I would rather avoid it altogether. As long as the following condition holds we are save:

Events in a queue are materialized in order.

You can have several workers update different sections of the database because they are not related to each other. And events are handled in order per section.

Putting data into different queues like Kafka offers it can drastically improve your scalability. But you need to make sure that the data is distributed across the queues in a way that the order of events between the queues does not matter. This is because the workers attached to the different queues can work with different speed and reorder events in different queues.

Async Communication
---

As you probably notices we have two different queues:

- Input event queue
- Output event queue

They are required to enable asynchronous communication between the different distributed processes that make up the application. This can be processes of the own app, but also processes of other apps trying to communicate with this app.

For example the input event queue can be used by everyone who want to request something of the app. This can be a different app, the web api used by the UI frontend of the app or another process of the app.

Requests in the input event queue are then handled by the responsible workers which materialize some data and post one or several output events.

The requester naturally must watch this output event queue for the fulfillment of his request. Usually he can track his requests by giving them a unique GUID that is used for tracking the results of that request.

**Example**
>![AsyncCustomerCreation]({{ "/assets/img/AsyncCustomerCreation.svg" | absolute_url }})
>
>Another app wants to create a new customer. It posts a "create customer" event with a unique GUID. The app worker responsible for customers will create the customer and post an event with the customer data and the tracking ID to the output queue. This event in the output queue can then be taken by a worker on the other app to see that the request was fulfilled.

This async communication can be used both by other apps as well as the web UI of the app itself. Although the two cases might differ slightly. The web application could be required to block until the request is fulfilled, while the worker of the other app could just go on. A different worker of that app would then encounter the response event and react to it.

Summary
---

In this post I have sketched out a rough draft of an event driven architecture that uses persistent event queues like the ones provided by Kafka. For the future I plan to release some more post about this topic, describing things like validation or logging.

I hope you enjoyed this post. If you did or have comments and improvements feel free to drop me an email.