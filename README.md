# mobx-react

[![CircleCI](https://circleci.com/gh/mobxjs/mobx-react.svg?style=svg)](https://circleci.com/gh/mobxjs/mobx-react)
[![Join the chat at https://gitter.im/mobxjs/mobx](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/mobxjs/mobx?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![CDNJS](https://img.shields.io/cdnjs/v/mobx-react.svg)](https://cdnjs.com/libraries/mobx-react)

Package with React component wrapper for combining React with MobX.
Exports the `observer` decorator and other utilities.
For documentation, see the [MobX](https://mobxjs.github.io/mobx) project.
This package supports both React and React Native.

## Choosing your version

There are currently two actively maintained versions of mobx-react:

| NPM Version | Supported React versions | Supports hook based components                                                   |
| ----------- | ------------------------ | -------------------------------------------------------------------------------- |
| v6          | 16.8.0 and higher        | Yes                                                                              |
| v5          | 0.13 and higher          | No, but it is possible to use `<Observer>` sections inside hook based components |

The V5 documentation can be found in the [README_v5](README_v5.md).

Both mobx-react 5 and 6 are compatible with mobx 4 and 5

Version 6 is a repackage of the [mobx-react-lite](https://github.com/mobxjs/mobx-react-lite) package + following features from the `mobx-react@5` package added:

-   Support for class based components for `observer` and `@observer`
-   `Provider / inject` to pass stores around (but consider to use `React.createContext` instead)
-   `PropTypes` to describe observable based property checkers (but consider to use TypeScript instead)
-   The `disposeOnUnmount` utility / decorator to easily clean up resources such as reactions created in your class based components.

## Installation

`npm install mobx-react --save`

Or CDN: https://unpkg.com/mobx-react (UMD namespace: `mobxReact`)

```javascript
import { observer } from "mobx-react"
```

This package provides the bindings for MobX and React.
See the [official documentation](http://mobxjs.github.io/mobx/intro/overview.html) for how to get started.

For greenfield projects you might want to consider to use [mobx-react-lite](https://github.com/mobxjs/mobx-react-lite), if you intend to only use function based components. `React.createContext` can be used to pass stores around.

## API documentation

Please check [mobx.js.org](https://mobx.js.org) for the general documentation. The documentation below highlights some specifics.

### `observer(componentClass)`

Function (and decorator) that converts a React component definition, React component class or stand-alone render function into a reactive component, which tracks which observables are used by `render` and automatically re-renders the component when one of these values changes.

When using component classes, `this.props` and `this.state` will be made observables, so the component will react to all changes in props and state that are used by `render`.
Note that `observer` automatically applies `React.memo` to any component you pass to it.

See the [MobX](https://mobxjs.github.io/mobx/refguide/observer-component.html) documentation for more details.

```javascript
import { observer } from "mobx-react"

// ---- ES6 syntax ----
const TodoView = observer(
    class TodoView extends React.Component {
        render() {
            return <div>{this.props.todo.title}</div>
        }
    }
)

// ---- ESNext syntax with decorator syntax enabled ----
@observer
class TodoView extends React.Component {
    render() {
        return <div>{this.props.todo.title}</div>
    }
}

// ---- or just use function components: ----
const TodoView = observer(({ todo }) => <div>{todo.title}</div>)
```

### `Observer`

`Observer` is a React component, which applies `observer` to an anonymous region in your component.
It takes as children a single, argumentless function which should return exactly one React component.
The rendering in the function will be tracked and automatically re-rendered when needed.
This can come in handy when needing to pass render function to external components (for example the React Native listview), or if you
dislike the `observer` decorator / function.

```javascript
class App extends React.Component {
    render() {
        return (
            <div>
                {this.props.person.name}
                <Observer>{() => <div>{this.props.person.name}</div>}</Observer>
            </div>
        )
    }
}

const person = observable({ name: "John" })

React.render(<App person={person} />, document.body)
person.name = "Mike" // will cause the Observer region to re-render
```

In case you are a fan of render props, you can use that instead of children. Be advised, that you cannot use both approaches at once, children have a precedence.
Example

```javascript
class App extends React.Component {
    render() {
        return (
            <div>
                {this.props.person.name}
                <Observer render={() => <div>{this.props.person.name}</div>} />
            </div>
        )
    }
}

const person = observable({ name: "John" })

React.render(<App person={person} />, document.body)
person.name = "Mike" // will cause the Observer region to re-render
```

### `useLocalStore` hook

Local observable state can be introduced by using the `useLocalStore` hook, that runs once to create an observable store. A quick example would be:

```javascript
import { useLocalStore, useObserver } from "mobx-react-lite"

const Todo = () => {
    const todo = useLocalStore(() => ({
        title: "Test",
        done: true,
        toggle() {
            this.done = !this.done
        }
    }))

    return useObserver(() => (
        <h1 onClick={todo.toggle}>
            {todo.title} {todo.done ? "[DONE]" : "[TODO]"}
        </h1>
    ))
})
```

When using `useLocalStore`, all properties of the returned object will be made observable automatically, getters will be turned into computed properties, and methods will be bound to the store and apply mobx transactions automatically. If new class instances are returned from the initializer, they will be kept as is.

It is important to realize that the store is created only once! It is not possible to specify dependencies to force re-creation, _nor should you directly be referring to props for the initializer function_, as changes in those won't propagate.

Instead, if your store needs to refer to props (or `useState` based local state), the `useLocalStore` should be combined with the `useAsObservableSource` hook, see below.

Note that in many cases it is possible to extract the initializer function to a function outside the component definition. Which makes it possible to test the store itself in a more straight-forward manner, and avoids creating the initializer closure on each re-render.

_Note: using `useLocalStore` is mostly beneficial for really complex local state, or to obtain more uniform code base. Note that using a local store might conflict with future React features like concurrent rendering._

### `useAsObservableSource` hook

The `useAsObservableSource` hook can be used to turn any set of values into an observable object that has a stable reference (the same object is returned every time from the hook).
The goal of this hook is to trap React primitives such as props or state (which are not observable themselves) into a local, observable object
so that the `store` or any reactions created by the component can safely refer to it, and get notified if any of the values change.

The value passed to `useAsObservableSource` should always be an object, and is made only shallowly observable.

The object returned by `useAsObservableSource`, although observable, should be considered read-only for all practical purposes.
Use `useLocalStore` instead if you want to create local, observable, mutable, state.

Warning: \_the return value of `useAsObservableSource` should never be deconstructed! So, don't write: `const {multiplier} = useAsObservableSource({ multiplier })`!\_useObservable

The following example combines all concepts mentioned so far: `useLocalStore` to create a local store, and `useAsObservableProps` to make the props observable, so that it can be uses savely in `store.multiplied`:

```typescript
import { observer, useAsObservableSource, useLocalStore } from "mobx-react-lite"

interface CounterProps {
    multiplier: number
}

export const Counter = observer(function Counter(props: CounterProps) {
    const observableProps = useAsObservableSource(props)
    const store = useLocalStore(() => ({
        count: 10,
        get multiplied() {
            return observableProps.multiplier * this.count
        },
        inc() {
            this.count += 1
        }
    }))

    return (
        <div>
            Multiplied count: <span>{store.multiplied}</span>
            <button id="inc" onClick={store.inc}>
                Increment
            </button>
        </div>
    )
})
```

Note that we cannot directly use `props.multiplier` in `multiplied` in the above example, it would not cause the `multiplied` to be invalidated, as it is not observable. Recreating the local store would also not have the desired state, as it would be a shame if it lost its local state such as `count`.

_Performance tip: for optimal performance it is recommend to not use `useAsObservableSource` together on the same component as `observer`, as it might trigger double renderings. In those cases, use `<Observer>` instead._

### Server Side Rendering with `useStaticRendering`

When using server side rendering, normal lifecycle hooks of React components are not fired, as the components are rendered only once.
Since components are never unmounted, `observer` components would in this case leak memory when being rendered server side.
To avoid leaking memory, call `useStaticRendering(true)` when using server side rendering.

```javascript
import { useStaticRendering } from "mobx-react"

useStaticRendering(true)
```

This makes sure the component won't try to react to any future data changes.

### Which components should be marked with `observer`?

The simple rule of thumb is: _all components that render observable data_.
If you don't want to mark a component as observer, for example to reduce the dependencies of a generic component package, make sure you only pass it plain data.

### Enabling decorators (optional)

Decorators are currently a stage-2 ESNext feature. How to enable them is documented [here](https://github.com/mobxjs/mobx#enabling-decorators-optional).

### Should I still use smart and dumb components?

See this [thread](https://www.reddit.com/r/reactjs/comments/4vnxg5/free_eggheadio_course_learn_mobx_react_in_30/d61oh0l).
TL;DR: the conceptual distinction makes a lot of sense when using MobX as well, but use `observer` on all components.

### `PropTypes`

MobX-react provides the following additional `PropTypes` which can be used to validate against MobX structures:

-   `observableArray`
-   `observableArrayOf(React.PropTypes.number)`
-   `observableMap`
-   `observableObject`
-   `arrayOrObservableArray`
-   `arrayOrObservableArrayOf(React.PropTypes.number)`
-   `objectOrObservableObject`

Use `import { PropTypes } from "mobx-react"` to import them, then use for example `PropTypes.observableArray`

### `Provider` and `inject`

_Note: usually there is no need anymore to use `Provider` / `inject` in new code bases; most of its features are now covered by `React.createContext`._

`Provider` is a component that can pass stores (or other stuff) using React's context mechanism to child components.
This is useful if you have things that you don't want to pass through multiple layers of components explicitly.

`inject` can be used to pick up those stores. It is a higher order component that takes a list of strings and makes those stores available to the wrapped component.

Example (based on the official [context docs](https://facebook.github.io/react/docs/context.html#passing-info-automatically-through-a-tree)):

```javascript
@inject("color")
@observer
class Button extends React.Component {
    render() {
        return <button style={{ background: this.props.color }}>{this.props.children}</button>
    }
}

class Message extends React.Component {
    render() {
        return (
            <div>
                {this.props.text} <Button>Delete</Button>
            </div>
        )
    }
}

class MessageList extends React.Component {
    render() {
        const children = this.props.messages.map(message => <Message text={message.text} />)
        return (
            <Provider color="red">
                <div>{children}</div>
            </Provider>
        )
    }
}
```

Notes:

-   It is possible to read the stores provided by `Provider` using `React.useContext`, by using the `MobXProviderContext` context that can be imported from `mobx-react`.
-   If a component asks for a store and receives a store via a property with the same name, the property takes precedence. Use this to your advantage when testing!
-   Values provided through `Provider` should be final, to avoid issues like mentioned in [React #2517](https://github.com/facebook/react/issues/2517) and [React #3973](https://github.com/facebook/react/pull/3973), where optimizations might stop the propagation of new context. Instead, make sure that if you put things in `context` that might change over time, that they are `@observable` or provide some other means to listen to changes, like callbacks. However, if your stores will change over time, like an observable value of another store, MobX will warn you. To suppress that warning explicitly, you can use `suppressChangedStoreWarning={true}` as a prop at your own risk.
-   When using both `@inject` and `@observer`, make sure to apply them in the correct order: `observer` should be the inner decorator, `inject` the outer. There might be additional decorators in between.
-   The original component wrapped by `inject` is available as the `wrappedComponent` property of the created higher order component.

#### Inject as function

The above example in ES5 would start like:

```javascript
var Button = inject("color")(
    observer(
        React.createClass({
            /* ... etc ... */
        })
    )
)
```

A functional stateless component would look like:

```javascript
var Button = inject("color")(
    observer(({ color }) => {
        /* ... etc ... */
    })
)
```

#### Customizing inject

Instead of passing a list of store names, it is also possible to create a custom mapper function and pass it to inject.
The mapper function receives all stores as argument, the properties with which the components are invoked and the context, and should produce a new set of properties,
that are mapped into the original:

`mapperFunction: (allStores, props, context) => additionalProps`

Since version 4.0 the `mapperFunction` itself is tracked as well, so it is possible to do things like:

```javascript
const NameDisplayer = ({ name }) => <h1>{name}</h1>

const UserNameDisplayer = inject(stores => ({
    name: stores.userStore.name
}))(NameDisplayer)

const user = mobx.observable({
    name: "Noa"
})

const App = () => (
    <Provider userStore={user}>
        <UserNameDisplayer />
    </Provider>
)

ReactDOM.render(<App />, document.body)
```

_N.B. note that in this *specific* case neither `NameDisplayer` nor `UserNameDisplayer` needs to be decorated with `observer`, since the observable dereferencing is done in the mapper function_

#### Using `PropTypes` and `defaultProps` and other static properties in combination with `inject`

Inject wraps a new component around the component you pass into it.
This means that assigning a static property to the resulting component, will be applied to the HoC, and not to the original component.
So if you take the following example:

```javascript
const UserName = inject("userStore")(({ userStore, bold }) => someRendering())

UserName.propTypes = {
    bold: PropTypes.boolean.isRequired,
    userStore: PropTypes.object.isRequired // will always fail
}
```

The above propTypes are incorrect, `bold` needs to be provided by the caller of the `UserName` component and is checked by React.
However, `userStore` does not need to be required! Although it is required for the original stateless function component, it is not
required for the resulting inject component. After all, the whole point of that component is to provide that `userStore` itself.

So if you want to make assertions on the data that is being injected (either stores or data resulting from a mapper function), the propTypes
should be defined on the _wrapped_ component. Which is available through the static property `wrappedComponent` on the inject component:

```javascript
const UserName = inject("userStore")(({ userStore, bold }) => someRendering())

UserName.propTypes = {
    bold: PropTypes.boolean.isRequired // could be defined either here ...
}

UserName.wrappedComponent.propTypes = {
    // ... or here
    userStore: PropTypes.object.isRequired // correct
}
```

The same principle applies to `defaultProps` and other static React properties.
Note that it is not allowed to redefine `contextTypes` on `inject` components (but is possible to define it on `wrappedComponent`)

Finally, mobx-react will automatically move non React related static properties from wrappedComponent to the inject component so that all static fields are
actually available to the outside world without needing `.wrappedComponent`.

#### Strongly typing inject

##### With TypeScript

`inject` also accepts a function (`(allStores, nextProps, nextContext) => additionalProps`) that can be used to pick all the desired stores from the available stores like this.
The `additionalProps` will be merged into the original `nextProps` before being provided to the next component.

```typescript
import { IUserStore } from "myStore"

@inject(allStores => ({
    userStore: allStores.userStore as IUserStore
}))
class MyComponent extends React.Component<{ userStore?: IUserStore; otherProp: number }, {}> {
    /* etc */
}
```

Make sure to mark `userStore` as an optional property. It should not (necessarily) be passed in by parent components at all!

Note: If you have strict null checking enabled, you could muffle the nullable type by using the `!` operator:

```
public render() {
   const {a, b} = this.store!
   // ...
}
```

#### Testing store injection

It is allowed to pass any declared store in directly as a property as well. This makes it easy to set up individual component tests without a provider.

So if you have in your app something like:

```javascript
<Provider profile={profile}>
    <Person age={"30"} />
</Provider>
```

In your test you can easily test the `Person` component by passing the necessary store as prop directly:

```
const profile = new Profile()
const mountedComponent = mount(
   <Person age={'30'} profile={profile} />
)
```

Bear in mind that using shallow rendering won't provide any useful results when testing injected components; only the injector will be rendered.
To test with shallow rendering, instantiate the `wrappedComponent` instead: `shallow(<Person.wrappedComponent />)`

### disposeOnUnmount(componentInstance, propertyKey | function | function[])

Function (and decorator) that makes sure a function (usually a disposer such as the ones returned by `reaction`, `autorun`, etc.) is automatically executed as part of the componentWillUnmount lifecycle event.

```javascript
import { disposeOnUnmount } from "mobx-react"

class SomeComponent extends React.Component {
    // decorator version
    @disposeOnUnmount
    someReactionDisposer = reaction(...)
    
    // decorator version with arrays
    @disposeOnUnmount
    someReactionDisposers = [
        reaction(...),
        reaction(...)
    ]


    // function version over properties
    someReactionDisposer = disposeOnUnmount(this, reaction(...))

    // function version inside methods
    componentDidMount() {
        // single function
        disposeOnUnmount(this, reaction(...))

        // or function array
        disposeOnUnmount(this, [
            reaction(...),
            reaction(...)
        ])
    }
}
```

## DevTools

`mobx-react@6` and higher are no longer compatible with the mobx-react-devtools.
That is, the MobX react devtools will no longer show render timings or dependency trees of the component.
The reason is that the standard React devtools are also capable of highlighting re-rendering components.
And the dependency tree of a component can now be inspected by the standard devtools as well, as shown in the image below:

![hooks.png](hooks.png)

## FAQ

**Should I use `observer` for each component?**

You should use `observer` on every component that displays observable data.
Even the small ones. `observer` allows components to render independently from their parent and in general this means that
the more you use `observer`, the better the performance become.
The overhead of `observer` itself is negligible.
See also [Do child components need `@observer`?](https://github.com/mobxjs/mobx/issues/101)

**I see React warnings about `forceUpdate` / `setState` from React**

The following warning will appear if you trigger a re-rendering between instantiating and rendering a component:

```
Warning: forceUpdate(...): Cannot update during an existing state transition (such as within `render`). Render methods should be a pure function of props and state.`
```

-- or --

```
Warning: setState(...): Cannot update during an existing state transition (such as within `render` or another component's constructor). Render methods should be a pure function of props and state; constructor side-effects are an anti-pattern, but can be moved to `componentWillMount`.
```

Usually this means that (another) component is trying to modify observables used by this components in their `constructor` or `getInitialState` methods.
This violates the React Lifecycle, `componentWillMount` should be used instead if state needs to be modified before mounting.
