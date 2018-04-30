---
layout: post
title: Thinking of an Event Driven Architecture II
tags: Event Sourcing Driven Messaging Log Kafka
---

Today we want to continue looking at the event driven architecture from my [last post](({% post_url 2018-4-19-Thinking-of-an-Event-Driven-Architecture-I %})). We will focus today on the UI layer and how the communication takes place between it and the event driven backend. As an important ingredient we will discuss validation in more detail.

App Service UI 
---

The UI for our App Service sits on top of the HTTP API as shown below.

![EventDrivenArchitectureUI]({{ "/assets/img/EventDrivenArchitectureUI.svg" | absolute_url }})

It fulfills a very important task indeed, it allows the users of our App Service to

1. watch the data that the App Service manages
2. request changes to the data of the App Service (and beyond)

Visualizing Materialized Data
---

In the first post about this topic I described the Materialization DB in which the workers will put the results of their actions when an event was handled. This data is primarily meant to be easily accessible and visualizable. When talking in [CQRS](https://martinfowler.com/bliki/CQRS.html) terms the data in this database would represent the query model.

It should be relatively simple for the HTTP API to query, filter and aggregate this data so that the UI can visualize it. Therefore, it makes sense to put data into the Materialization DB that is not necessarily in a normalized form. We can define multiple workers for different views of the UI that materialize the events in a way that best fits the queries which a specific view wants to visualize. By doing that the queries can be handled much faster than queries against a strictly normalized model. This is especially important for large amounts of data.

Now you will probably say: "But normalization is important to keep data integrity!"

You are right on wanting to keep the integrity of the data. But in scenarios with large amounts of data you will necessarily be forced to give up transactional integrity of the data. 

Commonly this can start by something simple like implementing an additional thread that handles some sort of requests and puts the result into a normalized data model. I see this a lot in apps that handle large data sets. At some point the CRUD operations are just to slow and therefore an asynchronous processor is introduced that takes requests, handles them and then puts the results back into the database. 

This scenario can quickly evolve into some hell of asynchronous operations that run without any architecture side-by-side. The problem with these asynchronous operations are that your architecture is probably not prepared for them. You are effectively implementing a distributed system without taking advantage of all the architectural ideas, platforms and frameworks that support distributed system development, e.g. swarm/cluster managers, distributed deployment engines. 

And I strongly believe that developing a distributed system is much harder than implementing a classical architecture. You can see this already by the number of additional topics you neet to take care of like:

- managing a lot of servers
- managing network between those servers
- deploying software to those servers
- gathering logging information from all those servers
- handling litteraly hundreds of different processes and services running on different servers
- letting these processes and services communicate without bogging down your system in case of problems with one of those services
- ...

You can see that this list is already intimidating without even trying to complete it.

Updating Data
---

At some point a user wants to change the data he sees in the UI. To do this the UI posts a command object to the HTTP API. The API in turn will convert it into a valid input event which is then posted to the input event queue. The event is handled as usually by the workers which update the Materialization DB. From there the HTTP API can grab the updated data and return it to the UI.

I hope you already see the problem that can arise here: When the HTTP API posts an input event and immediately returns to the UI "OK" the UI will still show the outdated data. This can lead to confusion for the user.

There are actually two solutions for this:

1. Show the user that the command is being executed currently
2. Wait until the updates are completed

In an ideal system you would even implement both of these methods. Remember that HTTP is stateless and a user could requery the page after sending a command. That would leave him with stale data without any indicator about the command status even if we implemented method 2.

I only want to go into the second case here, because most customers are not very flexible about this question.  The question is how to efficiently implement this?

The following image shows a cycle from the request of the UI to the completion indication of the HTTP API:

![AsyncCustomerCreationUI]({{ "/assets/img/AsyncCustomerCreationUI.svg" | absolute_url }})

1. The HTTP API receives the command from the UI that a customer should be created
2. The HTTP API 
    - converts the object into a valid input event (containing an additional tracking ID)
    - subscribes the Output Event Queue for customer creation events
    - post the create customer event to the Input Event Queue
3. workers will receive the event
4. workers will materialize the customer into the Materialization DB
5. workers will post that the customer was created
6. once the the event about customer creation arrives at the HTTP API it can return the OK (200!!) to the UI to indicate completion

What is not shown here is that the UI then refreshes the data by sending the required query to the HTTP API. It will then receive current and fresh data with the command being executed already.

Validation
---

Now that we know how a basic cycle looks in this architecture, we can dive into more advanced topics. The only one I want to handle in this post is validation.

Usually validation is a big upfront thing you do before posting any updates to the database. But this ignores cases where a lot of data needs to be taken into account. In those cases your transaction (you are using transactions, right) should probably fail when any of the data changed since you requested it, because else you need to revalidate. This can reduce performance a lot if your database is busy and your transaction spans a lot of tables.




