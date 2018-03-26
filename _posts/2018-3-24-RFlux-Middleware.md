---
layout: post
title: RFluX Middleware
tags: Rx, TypeScript, RFluX, Flux, Middleware
---

In my [last post]({% post_url 2018-3-19-RFluX-Flux-in-Typescript-with-Rx %}) we discussed how to setup a very basic flux framework in TypeScript with the help of Rx. Now we will take a look at a feature that will give us some real benefit thanks to the architecture we choose: action middleware.

What is middleware?
---
[Redux](https://redux.js.org/) introduced the concept of middleware. And because we basically do the same, we will use the same terminology here.

[Redux middleware](https://redux.js.org/advanced/middleware) represents components that you can hook into the dispatcher that delivers the actions to the stores. By doing that you can implement interesting concepts like an event log or time travel.

In our architecture the middleware is not used inside a dispatcher but rather directly subscribing or wrapping all the actions we create.

Rx Middleware
---

Let's define an interface for our middleware:

```typescript
interface IActionMiddleware {
  apply<TActionEvent>(
    action: IObservableAction<TActionEvent>, 
    actionMetadata: IActionMetadata)
    : IObservableAction<TActionEvent>;
} 
```

This again is a simple but very flexible interface. The middleware receives the action itself and can return a new action that should be used instead. In addition it receives some metadata describing the action. Initially it could only contain the name of the action (it could even be empty):

```typescript
interface IActionMetadata {
  name: string;
}
```

Action Factory
---
So now we know how our middleware should look like. But how do we apply the middleware to all actions.

To solve this problem we use the factory pattern and create an action factory that will apply all middleware.

```typescript
class MiddlewareActionFactory implements IActionFactory {
  constructor(private middleware: IActionMiddleware[]) 
  {
  }

  public create<TActionEvent>(actionMetadata?: IActionMetadata)
    : IObservableAction<TActionEvent> 
  {
    var action: IObservableAction<TActionEvent>;
    action = new Action<TActionEvent>();

    actionMetadata = actionMetadata ? actionMetadata : { name: "none" };

    // apply the middleware to all actions
    this.middleware.forEach(m => 
    {
      action = m.apply(action, actionMetadata);
    });

    return action;
  }
}
```

Once we have working middleware we can just hand all the middleware that should be used to the factory. Then we will use the factory to create all actions. By doing this we will be sure that the middleware is applied everywhere.

Note that it is your decision whether to use metadata for your actions here. Currently we are not forced to do it anywhere.

Subscribing Actions with a Logging Middleware
---
There are two basic ways in which middleware can intercept actions. The first and simpler one is subscribing actions.

As an example for a Subscribing Middleware we will write a middleware that prints all action events to the console.

```typescript
class ConsoleLoggingMiddleware 
  implements IActionMiddleware 
{
  public apply<TActionEvent>(
    action: IObservableAction<TActionEvent>, 
    actionMetadata: IActionMetadata)
    : IObservableAction<TActionEvent> 
  {
    // subscribe the action
    action.subscribe(actionEvent => {
      let actionJson = JSON.stringify(actionEvent);
      let actionMetadataJson = JSON.stringify(actionMetadata);

      // log the action event
      console.info(`Action ${actionMetadataJson} executed: ${actionJson}`);
    })

    // return the incoming action, we do not want to change its behaviour
    return action;
  }
}
```

When applied this middleware subscribes all incoming actions and prints all action events of those actions to the console.

If we have applied meaningful metadata we can better distinguish between the different actions in the logging output.

Wrapping Actions with a Blocking Middleware
---

Now that we can write simple middleware to subscribe actions we will go one step further and try to turn off all actions.
We want to tell our Blocking Middleware to just block the execution of all actions. Nothing should happen anymore in the app if we tell it to do so.

Now you could say: "Subscribing was easy but how the hell do we wrap an action?"

Well we first need some additonal functionality to do that. The very first thing we need is a way to dynamically create an action that we can put together as we like. I'll call it a wrapper. Let me present you the options I would like it to have.

```typescript
interface IActionWrapperOptions<T> {
    observe?: (action: IObservableAction<T>) => Observable<T>;
    subscribe?: (action: IObservableAction<T>, next: (parameter: T) => void) => Subscription;
    trigger?: (action: IObservableAction<T>, actionEvent: T) => void;
}
```
Every property in this interface is a function of the action interface that in addition to the original parameters
 receives the original action that should be wrapped. The rest of the signature is the same. In this way you can effectively wrap the action.

In summary you can:

  1. return an arbitrary observable for the `observe` method
  2. do whatever you like for the `subscribe` method
  3. do whatever you like for the `trigger` method

We still need to apply these options somehow. Let's define a function which should wrap an existing action given the options above.

```typescript
function wrapAction<T>(
  action: IObservableAction<T>, 
  options: IActionWrapperOptions<T>): IObservableAction<T> 
{
  // use the different options only if they are really set
  let observeFunction = options.observe 
                            ? () => options.observe(action)
                            : () => action.observe();
  let subscribeFunction = options.subscribe 
                            ? (next: (parameter: T) => void) => options.subscribe(action, next)
                            : (next: (parameter: T) => void) => action.subscribe(next);
  let triggerFunction = options.trigger 
                            ? (actionEvent: T) => options.trigger(action, actionEvent)
                            : (actionEvent: T) => action.trigger(actionEvent);

  // we still don't know this
  return new LambdaAction<T>(
    observeFunction, 
    subscribeFunction, 
    triggerFunction);
}
```

We are missing just one last piece namly the `LambaAction` that was used above. This class is a way to create an action with arbitrary functionality.

```typescript
class LambdaAction<T> implements IObservableAction<T> {
  constructor(
    private observeFunction: () => Observable<T>,
    private subscribeFunction: (next: (parameter: T) => void) => Subscription,
    private triggerFunction: (actionEvent: T) => void) 
  {      
  }

  public observe(): Observable<T> {
    return this.observeFunction();
  }

  public subscribe(next: (parameter: T) => void): Subscription {
    return this.subscribeFunction(next);
  }

  public trigger(actionEvent: T): void {
    this.triggerFunction(actionEvent);
  }
}
```

Finally, we can define our blocking middleware using the `wrapAction` method:

```typescript
class BlockingMiddleware 
  implements IActionMiddleware 
{
  private block: boolean = false;

  // if we call this with setBlocking(true) the middleware will start 
  // blocking the execution of all actions
  public setBlocking(block: boolean): void 
  {
    this.block = block;
  }

  public apply<TActionEvent>(
    action: IObservableAction<TActionEvent>, 
    actionMetadata: IActionMetadata)
    : IObservableAction<TActionEvent> 
  {
    return wrapAction(action,
    {
      trigger: (action, actionEvent) => {
        // only trigger action if we have not blocked execution
        if(!this.block) {
          action.trigger(actionEvent);
        }
      }
    });
  }
}
```

We now have a middleware that we could use to block all action execution. But why should we need such a thing?
At the moment this example may look a little bit contrived but we will need this functionality once we implement time travel.

Conclusion
---
Implementing action middleware in [RFluX](https://github.com/Useurmind/RFluX) was not as hard as I initially thought it to be. The hardest part surely was to wrap an action.
You can decide for yourself if you found that complex. But in the end it was very little functional code.

I hope you have enjoyed this post and join me in one of my next posts where I want to show you how time travel can be implemented with
the framework we currently have.

If you liked my post or have any suggestions please drop me an email.