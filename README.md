# @sanity/react-observable

_Hooks and utilities for combining React with RxJS_

This package offers two slightly different utilities for working with RxJS and React:

- :hammer_and_wrench: The _`reactiveComponent()`_ wrapper
- :fishing_pole_and_fish: The _`useObservable()`_ hook

Although they share a lot of similarities, and reactiveComponent is built on top of `useObservable` they are not intended to be used together inside the same component.

---

- [Reactive components](#reactive-components)
- [Observable hooks](#use-observable)
- [Code examples](https://github.com/sanity-io/react-observable/tree/master/packages)

<a name="reactive-components"></a>
## What's a reactive component anyway?

A _reactive component_ is a React component that turns an observable of props into an observable of [values React can render](https://reactjs.org/docs/react-component.html#render) (e.g. element, strings, fragments)

A _reactive component_ enables you to create components that composes different RxJS streams into a single render function, making your view purely a function of state.

The `reactiveComponent()` utility wraps a function component that is very similar to a regular React function component, with two notable differences:

1. Instead of being called with a _props object_, it is called with an _Observable of props_
2. Instead of returning _renderable values_, it returns an _Observable of renderable values_

Here's an example component that fetches some text from the url passed as the `url` prop:

```jsx
import {reactiveComponent} from '@sanity/react-observable'
import {switchMap, map} from 'rxjs'

const Fetch = reactiveComponent(props$ =>
  props$.pipe(
    switchMap(props => fetch(props.url).then(response => response.text())),
    map(responseText => <>Response is {responseText}</>)
  )
)

// Usage
ReactDOM.render(<Fetch url="/some/url)" />, container)
```

Instead of a _function_, `reactiveComponent` can also take an `Observable` as argument as long as it's an observable of [something React can render](https://reactjs.org/docs/react-component.html#render). This is neat when you have component that doesn't take any props, but still depends on values coming from an `Observable`:

```jsx
import {reactiveComponent} from '@sanity/react-observable'

const Counter = reactiveComponent(
  timer(0, 100).pipe(map(counter => <>The number is {counter}</>))
)

// Usage
ReactDOM.render(<Counter />, container)
```

### Using React hooks in a Reactive component

Note: Reactive components will not automatically re render when a value coming from a hook updates, in order achieve that, you need to convert it into an observable. This is done by wrapping it using the `toObservable()` utility before combining it with the returned observable. Here's an example using `React.useState()`:

```jsx
import {reactiveComponent} from '@sanity/react-observable

const Counter = reactiveComponent(props$ => {
  const [count, setCount] = React.useState(0)

  return toObservable(count).pipe(
    map(currentCount => (
      <>
        <div>Click count: {currentCount}</div>
        <button onClick={() => setCount(currentCount + 1)}>Click!</button>
      </>
    )))
})
```

The `toObservable` function makes it possible to consume community provided React hooks inside a reactive component. However, this package also export some utility functions that wraps regular hooks so you can use them with less boilerplate, for example, the `useState` function.

### Local component state

Instead of calling `React.useState()` directly, you can also use the `useState` function exported from this package. Instead of the actual state value it will return an observable representing the state changes, along with a function to update it. Here's the example above rewritten to use `useState` from this package instead.

```jsx
import {reactiveComponent, useState} from '@sanity/react-observable'

const Counter = reactiveComponent(() => {
  const [count$, setCount] = useState(0)

  return count$.pipe(map(currentCount => (
    <>
      <div>Click count: {currentCount}</div>
      <button onClick={() => setCount(currentCount + 1)}>Click!</button>
    </>
  )))
})
```

### Responding to events

Sometimes your state comes as a function of an event triggered by the user. Let's say you want to display the mouse cursors x,y position as the user moves the mouse inside a component. This can be done py using the `useEvent()` utility:

```js
import {reactiveComponent, useEvent} from '@sanity/react-observable'

const MouseTracker = reactiveComponent(() => {
  const [mouseMove$, handleMouseMove] = useEvent()

  return mouseMove$.pipe(
    map(event => (
      <>
        <div onMouseMove={handleMouseMove}>Hover me!</div>
        <pre>x={event.screenX},y={event.screenY}</pre>
      </>
    ))
  )
})

```

#### Gotcha: initial render?

The above example will not produce any DOM before a mousemove event has been triggered, and the mouse event can't be triggered before the DOM has been rendered, which puts us in a catch-22 situation. To cope with it, we can make the observable start by emitting an initial value for the event.

We can do this by using the `startWith` operator that comes with RxJS:

```js
import {reactiveComponent, useEvent} from '@sanity/react-observable'
import {map, startWith} from 'rxjs/operators'

const MouseTracker = reactiveComponent(() => {
  const [mouseMove$, handleMouseMove] = useEvent()
  
  return mouseMove$.pipe(
    startWith(null),
    map(event => (
      <>
        <div onMouseMove={handleMouseMove}>Hover me!</div>
        {event && <pre>x={event.screenX},y={event.screenY}</pre>}
      </>
    ))
  )
})
```

This means the `event` argument now can also be `null` so we also need to guard about that in our render function.

### API

#### `reactiveComponent()`

- `function reactiveComponent<Props>(observable: Observable<Props>): React.FunctionComponent<{}>`
- `function reactiveComponent<Props>(component: Component<Props>): React.FunctionComponent<Props>`

#### `forwardRef()`

The `React.forwardRef()` equivalent for a Reactive component.

- `function forwardRef<Props, Ref>(component: (input$: Observable<Props>,ref: React.RefObject<Ref>) => React.FunctionComponent<Props>)`

#### `useState`

Equivalent to `React.useState()` with the only difference that instead of the stateful value, it returns an observable representing the state changes (e.g. instead of `[T, update(value: T)]`, it returns `[Observable<T>, update(value: T)]`.

- `function useState<T = undefined>(): [Observable<T | undefined>, React.Dispatch<React.SetStateAction<T | undefined>>]`

- `function useState<T>(initial: T | (() => T)): [Observable<T>, React.Dispatch<React.SetStateAction<T>>]`

#### `useContext`

Equivalent to `React.useContext()` with the only difference that instead of a stateful context value, it returns an observable representing the context value changes.

- `function useContext<T>(context: React.Context<T>): Observable<T>`

#### `useEvent`

This has no React API equivalent, but creates a `[Observable<Event>, (Event) => void]` tuple, where the first value is an observable of the arguments the second function is called with.

- function useObservableEvent<Event>(): [Observable<Event>, (event: Event) => void]

#### `toObservable`

Converts any value from a React hook into an observable.

- `function toObservable<T>(value: T): Observable<T>`

TODO:

- useElement: check if it's possible to create an observable of a ref to a dom node

<a name="use-observable"></a>
## :fishing_pole_and_fish: Hooks

### The `useObservable()` hook

Use observables in React components with the `useObservable` hook.

If you need to subscribe to an observable in your component, this hook will give you the current value from it

Example:

```jsx
import {useObservable} from '@sanity/react-observable'
import {interval} from 'rxjs'

function MyComponent(props) {
  const number = useObservable(interval(100), 0)
  
  return <>The number is {number}</>
}
```

The `initialValue` argument is optional. If its omitted, the value returned from `useObservable` may be `null` initially. If the observable emits a value _synchronously_ at subscription time, that value will be used as the initial value, and any `initialValue` passed as argument to `useObservable` will be ignored:

```jsx
import {of} from 'rxjs'
import {useObservable} from '@sanity/react-observable'

// This component will never render "Hello mars!" since the observable emits "world" synchronously.
function MyComponent(props) {
  const planet = useObservable(of('world'), 'mars')

  return <>Hello {planet}!</>
}
```

### The `useObservableState` hook

If you need to represent some piece of state as an observable and also want the ability to change this state during the lifetime of the component, `useObservableState` is for you. It acts like `React.useState()`, only that it returns an observable representing changes to the value instead of the value itself. The callback/setter returned acts like a the regular callback you would otherwise get from `React.useState`. This is useful when you want to compose the state change together with other observables.

Note: this is exact same hook as the `useState()` function described in the section about Reactive Component above, it's only been exported under a different and more explicit name to distinguish better from `React.useState()`.

Here's an example of a component that count numbers at a certain speed, allowing the user to adjust the counting speed by adjusting a slider:

```jsx
import {useObservableState} from '@sanity/react-observable'

function MyComponent(props) {
  const [speed$, setSpeed] = useObservableState(1)

  const count$ = speed$.pipe(
    switchMap(speed => timer(0, 1000 / speed)),
    scan(count => count + 1, 0)
  )

  const currentSpeed = useObservable(speed$)
  const currentCount = useObservable(count$)

  return (
    <div>
      <div>Counter value: {currentCount}</div>
      <div>Counting speed: {Math.round((1 / currentSpeed) * 100) / 100}s</div>
      <input
        type="range"
        value={currentSpeed}
        min={1}
        max={10}
        onChange={event => setSpeed(Number(event.currentTarget.value))}
      />
    </div>
  )
}
```

### The `useObservableEvent` hook

This creates an event handler and an observable that represents calls to that handler.

Here's an example of a component thad displays the current value from a range input:

```jsx
import {useObservableEvent} from '@sanity/react-observable'

const ShowSliderValue = () => {
  const [onSliderChange$, onSliderChange] = useObservableEvent()

  const sliderValue = useObservable(
    onSliderChange$.pipe(
      map(event => event.currentTarget.value),
      map(value => Number(value)),
      startWith(1)
    )
  )

  return (
    <>
      <input type="range" value={sliderValue} min={1} max={10} onChange={onSliderChange} />
      <div>Value is: {sliderValue}</div>
    </>
  )
}
```

### API

#### `useObservable()`

A React hook that returns the latest value from an observable

- `function useObservable<T>(observable$: Observable<T>): T | null`
- `function useObservable<T>(observable$: Observable<T>, initialValue: T): T`

#### `useObservableState()`

Same as `useState()` described in the Reactive Component section above, only exported under a different name for clarity when using in regular React components

#### `useObservableContext()`

Same as `useContext()` described in the Reactive Component section above, only exported under a different name for clarity when using in regular React components

#### `useObservableEvent()`

Same as `useEvent()` described in the Reactive Component section above, only exported under a different name for clarity when using in regular React components

#### `toObservable()`

Same as `toObservable()` described in the Reactive Component section above
