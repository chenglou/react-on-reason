__This documentation assumes relative familiarity with ReactJS.__

## Intro Examples

```reason
let component = ReasonReact.statefulComponent "Greeting";

let make ::name _children => {
  let click _event state _self => ReasonReact.Update (state + 1);
  {
    ...component,
    initialState: fun () => 0,
    render: fun state self => {
      let greeting = "Hello " ^ name ^ ". You've clicked the button " ^ (string_of_int state) ^ " time(s)!";
      <button onClick=(self.update click)> (ReasonReact.stringToElement greeting) </button>
    }
  }
};
```

## JSX

Reason comes with the [JSX](http://facebook.github.io/reason/#diving-deeper-jsx) syntax! Reason-React transforms it from an agnostic function call into a library-specific call though a macro. To take advantage of Reason-React JSX, put `{"reason": {"react-jsx": 2}"` in your [`bsconfig.json`](http://bloomberg.github.io/bucklescript/Manual.html#_bucklescript_build_system_code_bsb_code) (schema [here](http://bloomberg.github.io/bucklescript/docson/#build-schema.json)).

### Uncapitalized JSX

```reason
<div foo=bar>child1 child2</div>
````

transforms into

```reason
ReactDOMRe.createElement "div" props::(ReactDOMRe.props foo::bar ()) [|child1, child2|]
```

which compiles to the JS code `React.createElement('div', {foo: bar}, child1, child2)`.

Prop-less `<div />` transforms to `ReactDOMRe.createElement "div" [|child1, child2|]`, which compiles to `React.createElement('div', undefined, child1, child2)`.

### Capitalized JSX

```reason
<MyReasonComponent key=a ref=b foo=bar>child1 child2</MyReasonComponent>
```

transforms to

```reason
ReasonReact.element key::a ref::b (MyReasonComponent.make foo::bar [|child1, child2|])
```

Prop-less `<MyReasonComponent />` transforms to `ReasonReact.element (MyReasonComponent.make [||])`.

## Component Creation

A component template is created through `ReasonReact.statelessComponent "ComponentName"`. The string being passed is for debugging purposes (the equivalent of ReactJS' [`displayName`](https://facebook.github.io/react/docs/react-component.html#displayname)).

As an example, here's a `greeting.re` file's content:

```reason
let component = ReasonReact.statelessComponent "Greeting";
```

Now we'll define the function that's called when other files invoke `<Greeting name="John" />` (without JSX: `ReasonReact.element (Greeting.make name::"John" [||])`): the `make` function. It must return the component spec template we just defined, with functions such as `render` overridden:

```reason
let component = ReasonReact.statelessComponent "Greeting";
let make ::name _children => {
  ...component, /* spread the template's other defaults into here  */
  render: fun () _self => <div> (ReasonReact.stringToElement name) </div>
};
```

**Note**: place `make` and `component` right beside each other! This helps readability and avoiding a corner-case type error for `state`.

### Stateful Components

To turn a component stateful, use `ReasonReact.statefulComponent "ComponentName"` instead. Then, provide the `initialState` method. The state type you return **can be anything**!

```reason
let component = ReasonReact.statefulComponent "Greeting";
let make ::name _children => {
  ...component,
  initialState: fun () => 0, /* here, state is an `int` */
  render: fun state self => {
    let greeting = "Hello " ^ name ^ ". You've clicked the button " ^ (string_of_int state) ^ " time(s)!";
    <div> (ReasonReact.stringToElement greeting) </div>
  }
};
```

_As a matter of fact, `ReasonReact.statelessComponent` is just a convenience helper of `statefulComponent`, with the `state` type being pre-set to `()` (called "unit")_.

### Accepting Props

Props are just the labeled arguments of the `make` function, seen above. They can also be optional and/or have defaults, e.g. `let make ::name ::age=? ::className="box" _children => ...`.

The last prop **must** be `children`. If you don't use it, simply ignore it by naming it `_` or `_children`. Names starting with underscore don't trigger compiler warnings if they're unused.

### Render

`render` needs to return a `ReasonReact.reactElement`: `<div />`, `<MyComponent />`, `ReasonReact.nullElement` (the `null` you'd usually return, but this time of the `reactElement` type), etc. It takes in `state` (which is `()` for stateless components) and `self`, described below.

### `self`

`self` is a record that contains `update` and `handle`, used in conjunctions with callbacks.

### Callback Handlers

In ReactJS, we can do `<div onClick={this.handleClick} />` and in `handleClick`, access the latest `props` and `state` (despite the callback being asynchronous) through `this.props` and `this.state`. We don't use `this` in Reason-React; to access the newest `props`, simply read from `make`'s arguments. To access the newest `state`, wrap the callback with `self.update`!

```reason
let component = ...;
let make ::name _children => {
  let click _event state _self => ReasonReact.Update (state + 1); /* this state is guaranteed fresh */
  {
    ...component,
    initialState: ...,
    render: fun state self => {
      let greeting = ...;
      <button onClick=(self.update click)> (ReasonReact.stringToElement greeting) </button>
    }
  }
};
```

`update` expects a callback that:

- Accepts a payload, the newest state and `self`.
- Returns a `ReasonReact.update 'state`, aka either.
  - `ReasonReact.Update newStateToBeSet`: indicates the handler (e.g. `click`) wants to update the state (in ReactJS, it'd be an imperative `setState` call).
  - `ReasonReact.NoUpdate`: no state update.
  - `ReasonReact.SilentUpdate newStateToBeSet`: like `ReasonReact.Update`, but without triggering a re-render. Useful for `ref` and other instance variables, described later.

and `update` itself returns a function, the actual, ordinary callback passed to the component as prop. When the callback's invoked, `update` will in turn invoke the originally passed in handler, forwarding the payload, the up-to-date state, and `self`.

Often times, you'd return `NoUpdate` from the handler. For convenience, we've exposed a `self.handle`, which is just an `update` that doesn't expect a return value (aka expects just `unit`).

### Ref & Instance Variables

A common pattern in ReactJS is to attach extra variables onto a component's spec:

```js
const Greeting = React.createClass({
  intervalId: null,
  componentDidMount: () => this.intervalId = setInterval(...),
  render: ...
});
```

In reality, this is nothing but a thinly veiled way to mutate a component's "state", without triggering a re-render. Reason-React asks you to correctly put these refs and instance variables into your component's `state`. To simulate updating the references without triggering a re-render, use `SilentUpdate`:

```reason
type state = {intervalId: option int, someOtherVar: option string};
let component = ...; /* remember, `component` needs to be close to `make`, and after `state` type declaration!
let make _children => {
  ...component,
  initialState: fun () => {intervalId: None, someOtherVar: Some "hello"},
  didMount: fun state _self => ReasonReact.SilentUpdate {...state, intervalId: Some (setInterval ...)},
  render: ...
};
```

### Lifecycle Events

ReasonReact supports the usuals:

```reason
willReceiveProps: state => self => state,
didMount: state => self => update state,
didUpdate: previousState::state => currentState::state => self => unit,
willUnmount: state => self => unit,
willUpdate: previousState::state => nextState::state => self => unit,
```

Note:

- We've dropped the `component` prefix from all these.
- `willReceiveProps` asks, for the return type, to be `state`, not `update state` (i.e. `NoUpdate/Update/SilentUpdate`). We presume you'd always want to update the state in this lifecycle.
- `didUpdate`, `willUnmount` and `willUpdate` don't allow you to return a new state to be updated, to prevent infinite loops.
- `didUpdate` and `willUpdate` are labeled now! This reduces confusion.
- `willMount` is unsupported. Use `didMount` instead.

## Interop With Existing JavaScript Components

### ReactJS Using Reason-React


## Common Type Errors

- `The type constructor state would escape its scope`: this probably means you've defined your `state` type _after_ `let component = ...`. Move it before the `let`.

