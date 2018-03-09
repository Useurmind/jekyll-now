Flux in TypeScript with Rx
=======

In this article I want to show my thoughts on how to implement a simple flux framework for TypeScript apps based on Rx.

I will give you an overview of what flux is, how Facebook implements it in their examples and how we can do better with the help of the TypeScript compiler.

Flux
----

Flux is an frontend application architecture introduced by Facebook in 2014.

It consists of four major parts:
- views: desired state UI components
- actions: pojos that can be handed to stores
- dispatcher: delivers action objects to stores
- stores: update their state based on delivered actions

The state of the stores are subscribed by the different views. The views in turn update themselves to reflect the current state of the stores. They also translate user input into actions and hand them to the dispatcher. Finally the actions arrive at the stores again.

Views are usually implemented with react which allows to design UI components based on the target state and frees the developer from thinking about what changes to apply to get there.

The rest of the architecture is implemented by many different frameworks, e.g.

- [facebook/flux](https://github.com/facebook/flux)
- [redux](https://redux.js.org)

And probably a million more.

Rx
---



What do we need?
---

Let's assume we build our flux framework with views being realized using react.

What else do we need to get a basic framework?

- actions
- stores
- server calls (we need to call a server to do most of the useful stuff)

You could think of some more topics like e.g. dependency injection. But the most basic framework that still works will only contain the three parts above.

String Keys
----
In facebook/flux and also redux actions are plain JavaScript objects. I found the creation of objects with string keys (something natural in JavaScript) in TypeScript to be very unnatural. You need to create:

- a constant string key
- an interface for the action object referencing the string key (mostly named the same as the key)
- create a switch statement that handles all actions of interest by matching the string key

Here is an example of such a switch statement (straight from [facebook/flux](https://github.com/facebook/flux/blob/master/README.md)):

  ```
  switch (action.type) {
    case 'increment': 
      return state + 1; 
    case 'square': 
      return state * state;
    default: return state;
  }
  ```

Note the action.type property which is a string and must be defined by hand for each action.

Personally I tend to stay away from string keys in my apis as far as possible. There are use cases for them especially in untyped languages like JavaScript out in interprocess communication. But if you have a strong compiler like in TypeScript you are not forced to use them.

For example, in C# you can always replace a string key in your libraries api by a lambda expression from which you can derive e.g. a property name.

Sadly such reflection is not yet possible in TypeScript.

Rx: Everything is a stream
-----

What can we take from the previous section to design or own flux framework?

### Actions are streams

So I don't want to create actions as pojos because of the string keys.

So why not define them as streams? True to the mantra of Reactive Programming "everything is a stream" we can define actions as streams of pojos. Each of those pojos represents a single occurrence of someone triggering the action. I call it an **Action Event**.

The difference to dispatching in original flux then becomes this: you don't have a single dispatcher but one stream for each action that can be triggered.

### Stores are streams

If everything is a stream then there is absolutely no reason for stores to not be a stream. The store concept lends itself very well to being implemented as a stream. Because of it's subscribable nature implementing it as a stream makes it subscribable by definition.

But what travels through this stream? This question is easy to answer: the state of the store. 

Each time the state of the store changes a new state item will be available in the stream.

Server calls
----

### Async Actions in Redux

In Redux (but also in some diagrams from facebook/flux) the calls to server side actions are packaged into action creators that create a so called async action.

An async action is basically an action that is handed by the dispatcher to the stores then executes some code and finally calls some other actions (which again are dispatched to the stores).

This pattern is useful to separate state modification logic from async logic calling the server.

So what do you need to implement a server call in Redux:

- three actions (started, success, failure)
- middleware
- some magic code that creates an async action 

See [redux.js.org/**ADVANCED**/async-actions](https://redux.js.org/advanced/async-actions)

I extracted the relevant code below from the link above:

  ```
  export function fetchPosts() {
    return function (dispatch) {
      dispatch(requestPosts(subreddit));

      return fetch(`https://www.reddit.com/r/${subreddit}.json`)
        .then(
          response => response.json(),
          error => console.log('An error occurred.', error)
        )
        .then(json =>
          dispatch(receivePosts(subreddit, json))
        )
    }
  }
  ```

The action creator shown above  is a function returning a function which first dispatches the started action. Then it fetches something from the server. On success it dispatches the success action. On error it handles it somehow.



Putting it all together
----

So we already have the following:

- actions: stream of occurrences
- stores: stream of state
- dispatcher: integrated scheduling in Rx
- views: react UI components

We only need to put it all together!

Because actions are streams you can inject them anywhere using DI or some other technique. But in most cases it makes sense to just create them in a store. 

In either case you can then offer them to the subscribing UI via the interface of the store.

So the following describes a full cycle in the application using this pattern:

- the UI subscribes the store
- the store subscribes the action
- the UI triggers an action
- the store changes it's state in response to the action event
- the UI updates itself in response to the state change of the store

Sounds very much like a flux implementation to me! Even if the terminology and technology is somewhat different in some parts.

Full code example
------

Check out the full example on Github:
https://github.com/Useurmind/FluxInTypescriptWithRx