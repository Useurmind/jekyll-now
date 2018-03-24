---
layout: post
title: RFluX Middleware
---

In my last post we discussed how to setup a very basic flux framework in TypeScript with the help of Rx. Now we will take a look at a feature that will give us some real benefit thanks to the architecture we choose: action middleware.

What is middleware?
---
Redux introduced the concept of middleware. And because we basically do the same, we will use the same terminology here.

Redux middleware represents components that you can hook into the dispatcher that delivers the actions to the stores. By doing that you can implement interesting concepts like an event log or time travel.

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

    this.middleware.forEach(m => 
    {
      action = m.apply(action, actionMetadata);
    });

    return action;
  }
}
```

Once we have working middleware we can just hand all the middleware that should be used to the factory. Then we will use the factory to create all actions. By doing this we will be sure that the middleware is applied everywhere.

Now that it is your decision whether to use metadata for your actions here. Currently we are not forced to do it anywhere.