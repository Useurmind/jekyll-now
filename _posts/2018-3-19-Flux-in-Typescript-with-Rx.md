---
layout: post
title: Flux in TypeScript with Rx
---

In this article I want to show my thoughts on how to implement a simple flux framework for TypeScript apps based on Rx.

I will give you an overview of what flux is, how common frameworks are implemented and how we can implement a framework based on Rx that takes advantage of the TypeScript intellisense and compiler.

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

Rx is a very flexible implementation of the observer pattern. It handles event streams, their transformation and combination, and scheduling of the events to different receivers.

It is a very mature framework available across a wide range of languages like C#, Java and TypeScript/JavaScript just to name a few.

If you want some in depth introductions take a look at the following links:

- [reactivex.io/intro](http://reactivex.io/intro.html)
- [The introduction to Reactive Programming you've been missing](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)
- [introtorx](http://www.introtorx.com)

Why Rx for flux?
----

You probably ask yourself why we should use Rx to implement a flux framework.

The point is that Rx already implements all technical components required for flux. These are:

- observable objects (like the stores)
- dispatching of event streams (like the actions)

In addition it allows a very flexible combination of these components that you will miss in other frameworks.

What do we need?
---

Let's assume we build our flux framework with views being realized using react.

What else do we need to get a basic framework?

- actions
- stores
- server calls (we need to call a server to do most of the useful stuff)

You could think of some more topics like e.g. dependency injection or automatic state merging for react views. But the most basic framework that still works will only contain the three parts above.

String Keys
----
In facebook/flux and also redux actions are plain JavaScript objects. I found the creation of objects with string keys (something natural in JavaScript and required to identify the type of action in redux) in TypeScript to be very unnatural. You need to create:

- a constant string key
- an interface for the action object referencing the string key (mostly named the same as the key)
- create a switch statement that handles all actions of interest by matching the string key

Here is an example of such a switch statement (straight from [facebook/flux](https://github.com/facebook/flux/blob/master/README.md)):

```JavaScript
switch (action.type) {
  case 'increment': 
    return state + 1; 
  case 'square': 
    return state * state;
  default: return state;
}
```

Note the action.type property which is a string and must be defined by hand for each action.

Personally I tend to stay away from string keys in my apis as far as possible. There are use cases for them especially in untyped languages like JavaScript or in interprocess communication. But if you have a strong compiler like in TypeScript you are not forced to use them.

Rx: Everything is a stream
-----

What can we take from the previous section to design our own flux framework?

### Actions are streams

So I don't want to create actions as pojos because of the string keys.

So why not define them as streams? True to the mantra of Reactive Programming "everything is a stream" we can define actions as streams of pojos. Each of those pojos represents a single occurrence of someone triggering the action. I call it an **Action Event**.

The difference to dispatching in original flux then becomes this: you don't have a single dispatcher but one stream for each action that can be triggered.

So let's first think about what an action should be able to do.

The primary thing we can do with an action is trigger a new event:

```TypeScript
interface IAction<TActionEvent> {
  trigger(eventData: TActionEvent): void;
}
```

The second thing we want to do is to observe the action (usually from a store):

```TypeScript
interface IObservableAction<TActionEvent> 
  extends IAction<TActionEvent> {
  observe(): Rx.Observable<TActionEvent>;
}
```

As actions are stateless event streams the straight forward choice for their implementation is then an Rx subject.

```TypeScript
class Action<TActionEvent> 
  implements IObservableAction<TActionEvent> {
  private subject: Rx.Subject<TEventData>;

  trigger(eventData: TActionEvent): void {
    this.subject.next(eventData);
  }

  observe(): Rx.Observable<TActionEvent> {
    return this.subject;
  }
}
```

This is a very simple and straight forward implementation thanks to Rx. Just call `next` on the subject to trigger it.

Returning an observable for observing the events in the action allows us to harness the power of all the available Rx operators.
This is useful for example when we not only want to observe the action but also the state of some other store and combine both into one handler.

### Stores are streams

If everything is a stream then there is absolutely no reason for stores to not be a stream. The store concept lends itself very well to being implemented as a stream. Because of it's subscribable nature implementing it as a stream makes it subscribable by definition.

But what travels through this stream? This question is easy to answer: the state of the store. 

Each time the state of the store changes a new state item will be available in the stream.

And what is the primary ability we want from a store? Correct, we want to subscribe it's state from the views.

```TypeScript
interface IStore<TState> {
  subscribe(next: (state: TState) => void): Rx.Subscription;

  observe(): Rx.Observable<TState>;
}
```

I also added the same observe call we know from the actions because there will be situations when we want to harness the full power of Rx to subscribe multiple stores or actions at once.

Because stores are stateful we will not use the default subject. Instead we use the [behavior subject](http://reactivex.io/documentation/subject.html). This subject repeats the last state or a default state when new subscribers arrive. This assures that the complete state of the application is always available.

```TypeScript
class Store<TState> implements IStore<TState> {
  private subject: Rx.BehaviorSubject<TState>;

  constructor(defaultState: TState) {
    this.subject = new Rx.BehaviorSubject<TState>(defaultState);
  }

  protected get state(): TState {
    return this.subject.getValue();
  }

  protected setState(state: TState): void {
    if (state)
    {
      this.subject.next(state);
    }
  }

  public subscribe(next: (state: TState) => void): Rx.Subscription {
    return this.observe().subscribe(next);
  }

  public observe(): Rx.Observable<TState> {
    return this.subject;
  }
}
```

This forms a very rudimentary base class for our stores but it works with this level of implementation already. 

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

```JavaScript
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

The action creator shown above is a function returning a function which first dispatches the started action. Then it fetches something from the server. On success it dispatches the success action. On error it handles it somehow.

### Async Actions with Rx

When looking at what Redux does you have to decide between two things.

1. Being explicit and verbose with separate actions for async start, success and error
2. Being implicit and just using the async API, e.g. fetch, for starting and handling success, error

There is a little tradeoff (little bit more code) to possibility number one, but belief me it is worth the effort.
Only if every state change is triggered through a separate action, you can implement interesting features like time travel or action event logs.
And for async actions start, success and error will most probably all change the state.

Therefore I encourage you to create one action per state change.

Putting it all together
----

So we already have the following:

- actions: stream of action events
- stores: stream of state
- dispatcher: integrated scheduling in Rx
- views: react UI components

We only need to put it all together!

![FluxRx]({{ "/assets/img/FluxRx.svg" | absolute_url }})

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

Check out the full example in my [Flux Rx Github repository](https://github.com/Useurmind/FluxInTypescriptWithRx):

[Flux Rx Counter Example](https://github.com/Useurmind/FluxInTypescriptWithRx/tree/master/example/counter)

Conclusion
---
Implementing a Flux framework with Rx for usage in the TypeScript language is not that complex.
The hard part will be to reduce the boilerplate code and implement advanced features like timetravel or event logs for a better debugging experience.

Join me in my next posts concerning this topic and contact me if you have any suggestions.