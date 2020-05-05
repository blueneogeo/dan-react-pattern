# The top-level application component

Your application component sits at the top of the tree. It is a good convention to combine the name of the application with the word "application" or "app". Eg: "FooApp".

Your application is "just a component". It should have a **Props** and **State** interface that describe each. For example:

```typescript
interface Props {
    apiBaseURL: string
}

interface State {
    loading: boolean
    data?: SomeData
}

export class MyTestApp extends React.Component<Props, State> {

    constructor(props: Props) {
        super(props)
        this.state = {
            loading: false
        }
    }

}
```

This makes it clear what your application needs to run, and what state it keeps.

## Using Immer

When you update state in a component, you would normally use `Component.setState`. However if you change a property inside of the state, that will not trigger a re-render, even though you made a change. By using Immer, you can make changes inside of the state, and Immer will propagate these changes up the state, giving you a new state object.

The way you use Immer is that you pass it a function that gives you a draft state which you can modify. Immer then updates the actual state for you. You never actually modify the state directly, the draft state Immer provides you with is a proxy with some clever magic to make it seem like a normal object as you manipulate it. It is also fully type-safe.

For example:

```typescript
import { produce } from 'immer'

// inside a component function:

produce(state => {
    state.loading = true
})
```

However we recommend you use the update wrapper from DAN:

```typescript
update(this, state => {
    state.loading = true
})

// allows you to do this as well
update(this, state => state.loading = true)

// and allows you to await the change
await update(this, state => state.loading = true)
console.log(this.state.loading) // now true, as it waited for the change
```

## Application logic

Your application is a central place for application-level logic. Any action that changes the application state should be a method in your application component. The child components should only pass up events, and the application component receives these with handler methods, which in turn call methods that perform actions. For example:

```typescript
export class MyTestApp extends React.Component<Props, State> {

    ...

    render() {
        return <button onClick={this.onButtonClick}>Click me</button>
    }
    
    @boundMethod
    onButtonClick() {
        this.reloadContent()
    }
    
    async reloadContent() {
        update(this, state => state.loading = true)
        const newData = await this.api.loadData()
        update(this, state => {
            state.loading = false
            state.data = newData
        })
    }

}
```

Have subcomponents not pass HTML event properties, but data that represents the event from the perspective of the subcomponent. The onClick event of a Button returns an HTMLInputEvent, but your UserPanel.onUserClick would pass back the user who clicked.

By doing so, your application logic is centralized and easy to reason about.

## Creating a dedicated application render component

As your application file becomes large, you may want to split off your application rendering from the logic. You can do so simply by creating an external function that is just the render template for your code. For example:

```typescript
import { template } from 'MyTestAppTemplate'

export class MyTestApp extends React.Component<Props, State> {

    ...
    
    render() {
        return template(this)
    }

}

// MyTestAppTemplate

export function template(app: MyTestApp) {
    return ... complex render code here ...
}
```

## Splitting up your application logic

If the business logic in your app becomes very large, you can create helper methods in external files. These helper methods receive the component, so they can perform the updates on the state. For example:

```typescript
import { loadContent } from 'apphelper'

export class MyTestApp extends React.Component<Props, State> {
    
    ...
    
    @boundMethod
    onButtonClick() {
        loadContent(this, true)
    }

}

// apphelper.ts

export function loadContent(app: MyTestApp, foo: boolean) {
    update(this, state => state.loading = true)
    const newData = await this.api.loadData()
    update(this, state => {
        state.loading = false
        state.data = newData
    })
}
```



