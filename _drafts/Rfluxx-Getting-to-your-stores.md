---
layout: post
title: RFluXX - React context and the logical tree
tags: rfluxx routing stores nested hierarchy list
---

In this post I want to dive into how to retrieve your RfluXX stores from your UI components when writing complex applications that have a deep hierarchy of stores.

## Disclaimer

I expect you to have some knowlegde about the Rfluxx Framework. Nothing to complicated however. You can have a look at its documentation on [ReadTheDocs](https://rfluxx.readthedocs.io/).

## Managing nested collections with stores

To understand what the problem is that I want to address in this post I first need to explain a setup that causes the problem. 

Imagine a page that presents a list of elements which in turn contain another list of elements. As a concrete example we will look at a car rental management software. The page to manage the cars and its rental to different customers is our focus. It should be able to

- add, edit, and delete cars
- add, edit, and delete rentals for each car

You will easily recognize this page to have a nested collection structure. Something like this:

- Car 1 - VIN: 545KGHFU2343ASD, 10000km, 
  - rented to Henry, Klein, from 01.01. - 14.01.2018, for 1083€
  - rented to Alisa, Big, from 17.01. - 20.01.2018, for 350€
  - ...
- Car 2 - VIN: 897845SDFGFD948, 23000km, 
  - rented to Hugo, Bold, from 01.03. - 15.03.2018, for 1254€
  - ...
- ...

Now what you need to think about is how to structure the stores you want to use.

I want to promote a certain store structure here because I think it has benefits: _write one store per list element on each level_.

This will keep your stores small and easy to understand. It has the additional benefit that one store is responsible for a very small portion of the UI. That means updates due state changes of a store are made to only a fraction of the current UI which is nice due to performance reasons. This is because only the components update that have subscribed the changing store.

So for our example page we would need lets say three stores:

- `CarRentalPageStore`: The root store of the page. In our case it manages loading the list. It could also manage other global stuff on the page.
- `CarStore`: The car store is responsible for managing a single car item in the cars list.
- `RentalStore`: This store manages a single rental in the sublists beneath the different cars.

The hierarchy of stores would look like this:

```
`CarRentalPageStore`
  |- `CarStore`
    |- `RentalStore`
    |- `RentalStore`
    |- `RentalStore`
    |- ...
  |- `CarStore`
    |- `RentalStore`
    |- `RentalStore`
    |- `RentalStore`
    |- ...
  |- ...
```

I personally like this structure because it reflects exactly how the UI looks. No surprises.

Control is very fine granular as you can put all actions relevant for an object inside the proper store for this object.

Now off to answering the big question: _How to setup this hierarchy and how to get to your stores from the UI components_.

## Loading the list

First things first. So how do we get the data. Simple answer: we load it inside our `CarRentalPageStore` when the page loads.

__CarRentalPageStore.ts__:
```typescript

export interface ICarRentalPageStoreState {
    carStores: Map<number, ICarStore>;
}

export class CarRentalPageStore /* ... */ {
    // ...

    private onLoadData() {
        const someUrl = "..."; // wherever the data comes from
        fetch(someUrl).then(respone => response.json())
                      .then(result => {
                          const carStores = new Map();
                          
                          const carStoreList = result.cars.map(car => new CarStore({ car }));

                          for(const carStore of carStoreList) {
                              carStores.set(car.id, car);
                          }
                      })
    }

    // ...
}
```

For each loaded car we create a new store for that car that will manage the UI for each car in this view. It will also provide the actions that can be called from that UI. 

It is beneficial to store everything by id in a map. In this way we can easily find a store that is responsible for a specific car. This can be important e.g. when deleting a car from the list.

# Creating a list of cars

Writing the UI code for this is also pretty easy:

__CarRentalPage.tsx__
```typescript
const CarRentalPage: React.FunctionComponent<{}> = props => {
    const [ storeState, store ] = useStoreStateFromContainerContext<ICarRentalPageStore, ICarRentalPageStoreState>(props);

    return <ul>
    { storeState && Array.from(storeState.carStores.values()).map(carStore => 
        <CarListItem carStore={carStore} />
    <ul>
}
```

Now what remains is to write your `CarListItem` and `CarStore` and repeat the same process for the rentals for each car.

As you can already see you have to hand down a lot of stores between components and even between stores. This can become quite cumbersome as the visual tree (our react components) is usually much more fine granular than the logical tree managing the data (our stores). Instead of handing it down to a single `CarListItem` we suddenly need to pass the store also to the `CarListItemHeader` to `CarListItemDetails` and so on. This multiplies our effort to distribute data in the UI and we effectively have not gained anything above using pure props.

# The logical react context

For our car rental page we have divided the UI between a set of stores. What if we could use the context api to effectively reduce the overhead required to pass down stores between components that are concerned with the same logical data.

__CarContext.tsx__
```typescript
export const CarContext = React.CreateContext<CarStore>(null);

// use the state and store from the current CarContext
export function useCarContext(): [ICarStoreState, ICarStore] {
    const store = React.useContext(CarContext);

    return [useStoreState(store), store];
}
```

We could use this car context like this in the car rental page.

__CarRentalPageWithContext.tsx__
```typescript
const CarRentalPage: React.FunctionComponent<{}> = props => {
    const [ storeState, store ] = useStoreStateFromContainerContext<ICarRentalPageStore, ICarRentalPageStoreState>(props);

    return <ul>
    { storeState && Array.from(storeState.carStores.values()).map(carStore => 
        <CarContext.Provider value={carStores}>
            <CarListItem />
        </CarContext.Provider>
    <ul>
}
```

The result is that we do not have to hand the car store down by hand. Instead in each component that really requires it we can do the following:

__CarListItem.tsx__
```typescript
const CarListItem: React.FunctionComponent<{}> = props => {
    const [state, store] = useCarContext();
    if(!state) {
        return;
    }

    return <li key={state.car.id}>
        <ListItemHeader />
        <ListItemDetails />
    </li>
}
```

This radically reduces prop handover between components. It is easier to refactor your react components, split them up or move them around. The context will always be consumed only in the components that care about the data. No need anymore to hand down the store over multiple visual components to the one logical component that really needs it.

# Escalate your store needs

The interesting thing in a nested list is now that you can easily retrieve the store of your parent logical data element.

Lets say we want to render a car rental. Remember that this is located inside a car list item. And it should be wrapped in a car rental context similar to the car context.

__CarRental.tsx__
```typescript
const CarRentalListItem: React.FunctionComponent<{}> = props => {
    const [rentalStoreState, rentalStore] = useRentalStore();
    const [carStoreState, carStore] = useCarContext();

    const delete = React.useCallback(() => {
        carStore.deleteRental.trigger(rentalStoreState.carRental.id);
    }, [rentalStore, carStore]);

    if(!rentalStoreState && !carStoreState) {
        return;
    }

    return <li key={rentalStoreState.carRental.id}>
        <div>rented to {rentalStoreState.carRental.customer}<>
        <Button onClick={delete}>Delete</Button>
    </li>
}
```

Deletion of something usually needs to be propagated to the parent store because it is responsible for the elements in the list. Only when the parent store is aware of a deletion it can remove the element from the list that renders it.

Being able to retriev the parent store makes it trivial to access actions in any level of the hierarchy that make up our current UI. Even accessing the root store is trivial in this setup. No handing down via props anymore.