# Recommended

## Use Typescript

If you are building a scalable application, you should use Typescript. You will also get the most out of this method, as it is very typing-friendly, and helps you make sure that as your application grows, you keep easy track of the state data, refactoring and error checking.

## Use @boundMethod

When passing a class method to handlers, Javascript will bind the method to the `this` of the component you are passing it to. However, the `this` of the method needs to be the component, so the handler keeps access to the properties, methods and state of the component when the handler gets called. You can solve this by having a constructor where you bind every handler to the component manually. However that is quite noisy and non-declarative, and may also break typing.

With the [autobind-decorator](https://github.com/andreypopp/autobind-decorator) package, you can simply annotate methods with `@boundMethod`, and this annotation will do this for you. 

## The update method

The standard [`Immer.produce`](https://immerjs.github.io/immer/docs/produce) method works well for updating the state of your component. However it has some inconveniences. By importing the following update method into your component, we can solve these:

```typescript
/**
 * Update the state of a React component using ImmerJS.
 * 
 * Benefits of this wrapper over immer.produce:
 * - returns a promise of the state after performing the update
 * - does not expect a return value, letting you have more compact update calls
 * 
 * Example usages:
 * ```typescript
 * // easily update the state:
 * update(this, state => state.loading = true)
 * 
 * // passing a block of updates with update logic and deep property setting:
 * update(this, state => {
 *   state.loading = true
 *   if(newName) state.user.firstName = newName
 * })
 * 
 * // waiting for the state to be set before reading it:
 * async onButtonClick() {
 *   await update(this, state => state.active = !state.active)
 *   console.log('button active:', this.state.active)
 * }
 * 
 * ```
 * @param component the stateful React component to update the state for. Usually `this`.
 * @param updateFunction a function that receives the ImmerJS draft state, and manipulates it.
 * @returns a promise of the updated state, after the update has asynchronously been performed
 */
export async function update<State>(component: React.Component<any, State>, updateFunction: (draft: State) => void) {
    return new Promise<State>(resolve => {
        component.setState(produce(produceFn), resolve)
    })

    /** This wrapper function is used so if the updateFunction returns a value, this value is discared.
     * This allows usage of the update method like this:
     * ```update(this, state => state.loading = true)```
     */
    function produceFn(state: State) {
        updateFunction(state)
    }

}
```

Its usage is very simple:

```typescript
// perform an update to the state
update(this, state => {
    state.loading = true
})

// allows you to inline your code for more brevity
update(this, state => state.loading = true)

// and allows you to await the change
await update(this, state => state.loading = true)
console.log(this.state.loading) // now true, as it waited for the change
```



