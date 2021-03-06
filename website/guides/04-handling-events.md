---
title: Handling Events
---

Most web applications require some measure of interactivity, where the user interface updates according to input. To facilitate this, Crank provides two APIs for listening to events on rendered DOM nodes.

## DOM onevent Props
You can attach event callbacks to host element directly using onevent props. These props start with `on`, are all lowercase, and correspond to the properties as specified according to the DOM’s [GlobalEventHandlers mixin API](https://developer.mozilla.org/en-US/docs/Web/API/GlobalEventHandlers). By combining event props, local variables and `this.refresh`, you can write interactive components.

```jsx
function *Clicker() {
  let count = 0;
  const handleClick = () => {
    count++;
    this.refresh();
  };

  while (true) {
    yield (
      <div>
        The button has been clicked {count} {count === 1 ? "time" : "times"}.
        <button onclick={handleClick}>Click me</button>
      </div>
    );
  }
}
```

## The EventTarget Interface
As an alternative to the onevent props API, Crank contexts also implement the same `EventTarget` interface used by the DOM. The `addEventListener` method attaches a listener to a component’s rendered DOM node or nodes.

```jsx
function *Clicker() {
  let count = 0;
  this.addEventListener("click", () => {
    count++;
    this.refresh();
  });

  while (true) {
    yield (
      <button>I have been clicked {count} {count === 1 ? "time" : "times"}</button>
    );
  }
}
```

The local state `count` is now updated in the event listener, which triggers when the rendered button is actually clicked.

**NOTE:** When using the context’s `addEventListener` method, you do not have to call the `removeEventListener` method if you merely want to remove event listeners when the component is unmounted. This is done automatically.

The context’s `addEventListener` method only attaches to the top-level node or nodes which each component renders, so if you want to listen to events on a nested node, you must use event delegation.

```jsx
function *Clicker() {
  let count = 0;
  this.addEventListener("click", (ev) => {
    if (ev.target.tagName === "BUTTON") {
      count++;
      this.refresh();
    }
  });

  while (true) {
    yield (
      <div>
        The button has been clicked {count} {count === 1 ? "time" : "times"}.
        <button>Click me</button>
      </div>
    );
  }
}
```

Because the event listener is attached to the outer `div`, we have to filter events by `ev.target.tagName` in the listener to make sure we’re not incrementing `count` based on clicks which don’t target the `button` element.

## onevent vs EventTarget
The props-based onevent API and the context-based EventTarget API both have their advantages. On the one hand, using onevent props means you don’t have to filter events by target. You register them on exactly the element you’d like to listen to.

On the other, using the `addEventListener` method allows you to take full advantage of the EventTarget API, which includes registering passive event listeners or listeners which are dispatched during the capture phase. Additionally, the EventTarget API can be used without referencing or accessing the child elements which a component renders, meaning you can use it to listen to components which are passed children, or in utility functions which don’t have access to produced elements.

Crank supports both API styles for convenience and flexibility.

## Dispatching Events
Crank contexts implement the full EventTarget interface, meaning you can use [the `dispatchEvent` method](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/dispatchEvent) and [the `CustomEvent` class](https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent) to dispatch custom events to ancestor components:

```jsx
function MyButton(props) {
  this.addEventListener("click", () => {
    this.dispatchEvent(new CustomEvent("mybuttonclick", {
      bubbles: true,
      detail: {id: props.id},
    }));
  });

  return (
    <button {...props} />
  );
}

function MyButtons() {
  return [1, 2, 3, 4, 5].map((i) => (
    <div> 
      <MyButton id={"button" + i}>Button {i}</MyButton>
    </div>
  ));
}

function *MyApp() {
  let lastId;
  this.addEventListener("mybuttonclick", (ev) => {
    lastId = ev.detail.id;
    this.refresh();
  });

  while (true) {
    yield (
      <div>
        <MyButtons />
        <div>Last pressed id: {lastId == null ? "N/A" : lastId}</div>
      </div>
    );
  }
}
```

`MyButton` is a function component which wraps a `button` element. It dispatches a `CustomEvent` whose type is `"mybuttonclick"` when it is pressed, and whose `detail` property contains data about the ID of the clicked button. This event is not triggered on the underlying DOM nodes; instead, it can be listened for by parent component contexts using event capturing and bubbling, and in the example, the event propagates and is handled by the `MyApp` component. Using custom events and event bubbling allows you to encapsulate state transitions within component hierarchies without the need for complex state management solutions used in other frameworks like Redux or VueX.

The preceding example also demonstrates a slight difference in the way the `addEventListener` method works in function components compared to generator components. With generator components, listeners stick between renders, and will continue to fire until the component is unmounted. However, with function components, because the `addEventListener` call would be invoked every time the component is rerendered, we remove and add listeners for each render. This allows function components to remain stateless while still listening for and dispatching events.

## Form Elements

Form elements like inputs and textareas are stateful and by default update themselves automatically according to user input. JSX libraries like React and Inferno handle these types of elements by allowing their virtual representations to be “controlled” or “uncontrolled,” where being controlled means that the internal DOM node’s state is synced to the virtual representation’s props. These APIs manifest as special “uncontrolled” versions of props like `defaultValue` and `defaultChecked`.

Crank’s approach to this issue is slightly different, in that we do not view the virtual elements as the “source of truth” for the underlying DOM nodes. In practice, this design decision means that renderers do not retain the previously rendered props for host elements. For instance, Crank will not compare old and new props between renders to avoid mutating props which have not changed, and instead attempt to update every prop found in props.

Another consequence is that we don’t delete props which were present in one rendering and absent in the next. In the following example, the checkbox will never uncheck itself.

```jsx
function *App() {
  let checked = false;
  this.addEventListener("click", (ev) => {
    if (ev.target.tagName === "BUTTON") {
      checked = !checked;
      this.refresh();
    }
  });

  this.addEventListener("input", (ev) => ev.preventDefault());

  while (true) {
    if (checked) {
      yield (
        <div>
          <button>Toggle</button>
          <input type="checkbox" checked />
        </div>
      );
    } else {
      yield (
        <div>
          <button>Toggle</button>
          <input type="checkbox" />
        </div>
      );
    }
  }
}
```

While frameworks like React notice the absence of the boolean `checked` prop between renders and mutate the input element, Crank does not. This means that the input element can only ever go from unchecked to checked, and not the other way around. To fix the above example, you would need to make sure the `checked` prop is always passed to the `input` element.

```jsx
function *App() {
  let checked = false;
  this.addEventListener("click", (ev) => {
    if (ev.target.tagName === "BUTTON") {
      checked = !checked;
      this.refresh();
    }
  });

  this.addEventListener("input", (ev) => ev.preventDefault());

  while (true) {
    yield (
      <div>
        <button>Toggle</button>
        <input type="checkbox" checked={checked} />
      </div>
    );
  }
}
```

This design decision means that we now have a way to make the same element prop both “uncontrolled” and “controlled” for an element. Here, for instance, is an input element which is uncontrolled, except that it resets when the button is clicked.

```jsx
function* ResettingInput() {
  let reset = true;
  this.addEventListener("click", ev => {
    if (ev.target.tagName === "BUTTON") {
      reset = true;
      this.refresh();
    }
  });

  while (true) {
    const reset1 = reset;
    reset = false;
    yield (
      <div>
        <button>Reset</button>
        {reset1 ? <input type="text" value="" /> : <input type="text" />}
      </div>
    );
  }
}
```

In the above example, we use the `reset` flag to check whether we need to set the `value` prop of the underlying input DOM element, and we omit the `value` prop when we aren’t performing a reset. Because the prop is not cleared when absent from the virtual element’s props, Crank leaves it alone. Crank’s approach means we do not need special alternative props for uncontrolled behavior, and we can continue to use virtual element rendering over raw DOM mutations in those circumstances where we do need control.
