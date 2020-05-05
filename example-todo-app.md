# Example Todo App



Let’s illustrate the pattern by building a little todo app:

```typescript
TodoApp.tsx:

interface Props {
    // application startup props here
}

interface State {
    todos: Todo[]
}

export interface Todo {
    id: number
    title: string
    done: boolean
}

export class TodoApp extends React.Component<Props, State> {

    constructor(props: Props) {
        super(props)
        this.state = { todos: [] }
    }

  render() {
    return <div>
        <h1>here are your todos:</h1>
        <ul>
        { this.state.todos.map(todo =>
            <TodoEntry todo={todo}/>
        )}
        </ul>
    </div>
  }
}
```

This is our top-level component. It contains our top level `State`, that will render the rest of the app.

It will also contain the actual actions the app can perform. Part of this pattern is making perform the application business logic operations, on the state and to external sources like an API.

### Escalating state property changes to the top

If the user completes a `TodoEntry`, we want to be able to set the `todo.done` to `true` for that entry in the state. We need to add a handler for completing a todo in the `TodoApp`:

```typescript
    // add the onComplete handler:

    { this.state.todos.map(todo =>
        <TodoEntry todo={todo} onComplete={() => this.onCompleteTodo(todo)}/>
    )}

    // add the handler to the TodoApp:

    @boundMethod onCompleteTodo(todo: Todo) {
        this.setState({
            // update the todo in the list
        })
    }
```

_Note: more about the @boundMethod annotation later._

Here we run into a problem. How do we update the correct todo in the list? We need the whole state to be updated. In simple cases it is possible to use the rest \(…\) syntax in Javascript to construct a new object. However this quickly becomes very unwieldy when to have to change a property deeper in the state.

Luckily there is a nice solution now called [Immer](https://immerjs.github.io/immer/docs/introduction). Immer allows us to take the existing state, gives us a draft copy and uses some proxy magic to let us modify it with normal Javascript operations. The result is an updated state, at the top level!

Finally, we said the application component should contain the top level business logic. Updating a todo is an asynchronous action. So let us split this up into an event handler function and an action. This also makes our code more easily testable.

```typescript
    // add the handler to the TodoApp:

    @boundMethod onCompleteTodo(todo: Todo) {
        this.completeTodo(todo)
    }

    // add an action that performs the state update

    async completeTodo(todo: Todo) {
        await update(function(draft: State) {
            // find the todo in the state, and update it
            const foundTodo = draft.todos.find(t => t.id = todo.id)
            if(foundTodo) foundTodo.done = true
        })
    }
```

So what happened here? The `update` function calls `Immer.produce`. You can pass it a function and it will call it with a draft copy of the state of your component. You can then modify that state as you see fit, using normal Javascript. What happens behind the scenes is that every change you make on the draft becomes a modification function of the full state inside Immer, which then puts it all together and updates the full state of the component.

In other words, Immer solves our problem of propagating changes to the top! The code is also fully type-safe and type-checked.

In the above example, we use the find method to look for the todo that we want to set the done property for, and if we found it, we set the todo as done.

So in summary: On marking a `TodoEntry` as done, it will call the passed `onComplete` handler in our app, which will call `completeTodo` in our app. The `completeTodo` action will find the todo that changed and use Immer to set that `todo.done` to `true`. Immer will create a new State object for us and set the new state. That will trigger a re-render of our `TodoApp`, and the user will see the list with the updated done `TodoEntry`.

### Avoiding unnecessary re-renders after application state updates

In the example above, the state change in the app will call the `render` method. **This means all `TodoEntry`’s in the list will get rendered again, even though we only updated a single todo!** If we have a deep component structure in a big app, this can get really expensive.

What we want is that only the `TodoEntry` for the todo we checked gets updated. In order to do this, we need to make each component more responsible for when it updates.

The contract for elements in the browser DOM is that if an element’s properties get changed, you expect the element to update. So you would expect that when React components’ properties change, that is when it re-renders.

However by default, React re-renders the whole DOM tree when `render` gets called. It needs to do this, because otherwise components that get properties that have internal changes might not get updated. For example, if we were to pass a `todo` as a property, and only change the `done` property in the todo, how would the component know it should update? The todo object itself remained the same after all.

For our approach, this is much too aggressive. What we want is to make components more responsible for when they update. That way, we can call render at a higher level, and components not be re-rendered if they do not feel like changing.

There is good news, due to the way ImmerJS works. It updates the state in a very clever way, only updating parts of the object tree that have a child that changed.

For our example, that means that if you pass a todo to a `TodoEntry`, and in the `Todo` you update the `todo.done` property, Immer also updates the `todo` itself, since `done` is a part of the todo, and upward to the full object. That means that even if we only updated the `done` property, the todo itself also gets updated, and our `TodoEntry` knows the property changed, and thus it will re-render as well, showing our done todo!

**All of this means that if we consistently manage state using Immer, we do not need to fully refresh the whole DOM on every update. We only need the components whose properties change to refresh their content.**

There is a relatively new component type that allows this, called `React.PureComponent`. It is much simpler than `React.Component`, and also a lot faster. By making our components extend `React.PureComponent` instead of `React.Component`, we will get the behavior we want.

Let’s implement our `TodoEntry`:

```typescript
TodoEntry.tsx:

interface Props {
    todo: Todo
    onComplete: () => void
}

export class TodoEntry extends React.PureComponent<Props> {

  render() {
    return <div>
        <span>{this.props.todo.title} -</span>
        <checkbox 
            defaultChecked={this.props.todo.done}
            onClick={this.onClickCheckbox}
            />
    </div>
  }

  @boundMethod onClickCheckbox(e) {
      this.props.onComplete()
  }
}
```

In `TodoApp.completeTodo` we only change the `done` property on a todo when we click it. The `TodoApp,render` will then be called. However only the `TodoEntry` whose todo we changed will update now, because it extends `React.PureComponent`. And even though we only updated the `todo` property of the `todo` object we pass to that `TodoEntry`, the `TodoEntry` will still refresh, because Immer also updated the `todo` itself.

Are we done then? If you would run this code, you would see that with every `TodoApp.render`, the whole list still gets updated!

### The pure-closure passing pattern

The reason is this code:

```typescript
    // add the onComplete handler:

    { this.state.todos.map(todo =>
        <TodoEntry todo={todo} onComplete={() => this.onCompleteTodo(todo)}/>
    )}
```

The bad part is the closure we pass. A new closure gets passed every time we render. That means that `TodoEntry` sees a property change every time we render, so it dutifully re-renders. We need a way to pass methods as handlers that remain the same for each `TodoEntry`, so we do not trigger the re-render.

The solution is making the closure a more ‘pure’ function, describing in its parameters what the closure needs from the component to complete.

In our case, what the closure needs to pass back is the task we completed.

This is how the `TodoEntry` would look:

```typescript
TodoEntry.tsx:

interface Props {
    todo: Todo
    onComplete: (Todo) => void // we pass back the todo now
}

export class TodoEntry extends React.PureComponent<Props> {

  render() {
    return <div>
        <span>{this.props.todo.title} -</span>
        <checkbox 
            defaultChecked={this.props.todo.done}
            onClick={this.onClickCheckbox}
            />
    </div>
  }

  @boundMethod onClickCheckbox(e) {
      // pass back the thing we completed
      this.props.onComplete(this.props.todo)
  }
}
```

Let’s rewrite the for `TodoApp` to use it now without the closure:

```typescript
    // add the onComplete handler:

    { this.state.todos.map(todo =>
        <TodoEntry todo={todo} onComplete={this.onCompleteTodo}/>
    )}
```

It could be argued that this is actually a cleaner way to write your code. In any case, this pattern allows us to never need to pass closures, and thus solves our problem!

### In summary, or TLDR;

We are now able to write any scaleable React application without Redux and with only a few clear patterns:

1. Build your application with a top level app component and use that state as your app-level state.
2. Only change state inside your component using Immer
3. Use React.PureComponent instead of React.Component
4. Always use class methods as handlers, and use the pure-closure passing pattern to avoid needing to use closures.

