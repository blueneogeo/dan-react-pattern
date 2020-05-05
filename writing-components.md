# Writing components

## Using React.PureComponent

In a normal `React.Component`, when a parent component calls `render`, it will also call `render` on every child component. The application is the top level state. So if we were to use normal React components, every time the application state changed would call render on all of its child components. This would be really inefficient.

Instead we want to only have the children update when their properties have changed. We make their props their only source of truth. So if their props remain the same, the component stays the same.

In React we can achieve this by using [`React.PureComponent`](https://reactjs.org/docs/react-api.html#reactpurecomponent) instead of `React.Component`.

`React.PureComponent` is a simpler version of `React.Component`. It only updates when its properties or state change. This is what we want, and makes components very predictable to reason about. We can simply pass properties down to child components.

However `React.PureComponent` has a limitation. To check the properties it does a shallow properties check, meaning it only updates if its properties have changed on the surface. Changes inside the properties are not seen however. This means that left unchecked, we may miss updates. For example, given this component:

```typescript
// TodoEntry.tsx

type Todo = {
    done: boolean
}

interface Props {
    todo: Todo
}

export class TodoEntry extends React.PureComponent<Props> {  
    ..
}
```

Now if we were to call this component and pass a Todo:

```typescript
someTodo = { done: false }

...

<TodoEntry todo={someTodo}/>
```

If we now do this in the state of the component that calls the component:

```typescript
someTodo.done = true
```

The TodoEntry will not update. The done property is wrapped by an object, and that object was not updated. The shallow compare means the change will not get detected.

## Immer to the rescue

On the [application component page](the-top-level-application-component.md#application-logic) you already saw how to use Immer. However Immer also does something really clever we need. When it updates a property, it goes up step by step in the object tree whose deep property you changed, and updates all components up to the top, but everything else remains the same object. Only the branch you changed a leaf in gets fully updated.

What this means for us is that as long as you mutate the state using Immer, changing any property inside of an object will also update the wrapping object for that object. In the case of the todo, changing the .done property will also update the todo object itself, meaning our TodoEntry would see the change and update when the todo.done is changed.

**Takeaway: Always use the update method to update state in any of your components.**

## When should your component keep state?

React.PureComponent can keep state like a normal component. However most often you will not need to.

Any child component of the application is purely for presentation. Most will not keep state, but some may. For example, a dropdown component needs to know if it is open. 

When deciding if a presentation component needs state, simply consider if it is business data or presentation data. If the data has to do with the application itself, the data should be in the application component state. If it has to do ONLY with presenting the interface state, it should be part of the presentation component state.

## Pure event handlers

It is common to pass a closure as an event handler, which then calls some action in your component. For example:

```typescript
<ItemCard onClick={() => this.onItemClicked(item)}>click me</ItemCard>
```

If we use this in pure components, there is a problem. A pure component will re-render whenever a property changes. This code will create a new closure for every time `render` gets called. That means that the pure component will see a change, and it will always re-render.

What we need is a way to always pass the same handler, so the component does not see a change:

```typescript
<ItemCard onClick={this.onItemClicked}>click me</ItemCard>
```

However now our handler no longer receives the item we wanted to pass.

The solution is to change the responsibility of who fulfills the event. In the above case, the `ItemCard` is the fulfiller of the event. We need to make sure that `ItemCard` has all the information it needs to 

The solution is to make the eventhandlers themselves pure. They need to know what to receive, and our components such as `ItemCard` in the example needs to pass the item. `ItemCard` becomes responsible to pass what it knows about the event, instead of the component around it.

To make this work in our example, the `ItemCard` needs to be passed the `item` to return in the event handler:

```typescript
<ItemCard item={item} onClick={this.onItemClicked}>click me</ItemCard>
```

```typescript
// ItemCard.tsx

interface Props {
    item: Item
    onClick: (Item) => void
}

export class ItemCard extends React.PureComponent<Props> {

    render() {
        return <div href="#" onClick={onCardClicked}>
            // card content
        </div>
    }
    
    @boundMethod
    onCardClicked(e) {
        this.props.onClick(this.props.item)
    }

}
```

Now the event handlers are always the same, and the `ItemCard` will not unnecessarily re-render.

