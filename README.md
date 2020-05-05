---
description: An alternative approach to handling React application State
---

# The DAN React Pattern

The DAN React pattern is a method organizing your application state without needing to introduce new methods of organizing state, such as stores or other external libraries. It allows your application state to be like any other component state. For this to work, you need to adhere to a pattern .

The DAN React pattern comes down to the following:

* Have a top-level application component whose state is your application state
* Treat component props as dependencies, using them to pass down data and to handle events
* Make reasoning about components easier, by having their behavior to be only determined by their props and their state
* Have all component state changes done using [Immer](https://immerjs.github.io/immer/docs/introduction)
* Make all component event handlers must be fixed, using the pure handlers pattern
* Avoid property-drilling by passing the application as a property when necessary

Strongly recommended with this are:

* Use [Typescript](https://www.typescriptlang.org)
* Use `@boundMethod` from the [autobind-decorator](https://github.com/andreypopp/autobind-decorator) package
* Use the provided `update` method to wrap Immer's `produce` method

## What is state?

State is simply the data you store at a given moment. Application state is data about your application, such as users, and notes in a notes app. It is the data your would for example store in a database. UI state is state purely used for presenting the interface. Is a dropdown open, what text entries to display, etc.

## Scaling state is hard

The hardest part when scaling your React application is how to deal with the state amongst your many components. If the state that determines how your application reacts is scattered amongst many little parts, it becomes harder and harder to predict your applications' behavior. Your components will tend to become very interdependent, which makes maintenance and scaling more and more difficult.

The now accepted pattern is to separate your state between application level state and interface/UI state. 

The application state is centralized, and connected to the business logic of your app. The interface state is purely about what is needed to present a UI component and should be in the component that manages that UI element.

## Centralized application state

What we need then, is a way to do the first: have a single source of truth for the state of the business data of the app. There are already many approaches for this, the most common storing all this data in a centralized [Redux](https://redux.js.org) store. 

However using Redux comes with a lot of complexity and new patterns to learn. This guide will describe the alternative approach used at DAN, leveraging what has become possible due to new techniques like Immer.

## How it works

If you keep all of your state in a top level component, and that component re-renders, it will trigger the re-render of all of the components in the component tree, which is expensive.

This pattern borrows loosely from how Flutter updates its tree. It only updates those components whose properties have updated, by using [`React.PureComponent`](https://reactjs.org/docs/react-api.html#reactpurecomponent)instead of React.Component. 

The pattern leverages the feature from Immer that it updates up the object tree from any deeper object leaf you modify. This makes components update even if a subproperty of a property was updated. 

By having fixed handlers, you avoid passing a new handler at each render, since this would trigger unnecessary re-renders. 

Finally by passing the application itself as a property, you clearly declare that a component depends on the application, and avoid passing a lot of properties across the tree.

## The result

By adhering to this pattern, you can reason about application state like any normal React component state. You are writing normal, unremarkable components. This also means you no longer need to declare a store in your code, and an application-level component can be the child of any other component. Finally typing your code becomes easy and fun again!

