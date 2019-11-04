# picoapp
🐣 Tiny no-framework component toolkit. **900b gzipped.**

> tl;dr - this library automatically instantiates JavaScript modules on specific
> DOM elements in a website if they exist on the page. This is helpful for
> projects like Shopify or Wordpress that aren't using a framework like React or
> Vue. `picoapp` also contains functionality that make it a great companion to
> any PJAX library – like
> [operator](https://github.com/estrattonbailey/operator) – where page
> transitions can make conventional JS patterns cumbersome.

**Note**: You’ll need to polyfill `Promise` for browsers that don’t support it.

## Install
```
npm i picoapp --save
```

# Usage
Define data attributes on the DOM nodes you need to bind to:
```html
<button data-component='button'>I've been clicked 0 times</button>
```

Create a corresponding component:
```javascript
// button.js
import { component } from 'picoapp'

export default component((node, ctx) => {
  let count = 0

  node.onclick = () => {
    node.innerHTML = `I've been clicked ${++count} times`
  }
})
```

Import your component and create a `picoapp` instance:
```javascript
import { picoapp } from 'picoapp'
import button from './button.js'

const app = picoapp({ button })
```

To bind your component to the DOM node, call `mount()`:
```javascript
app.mount() // returns a promise which resolves when all components successfully mount
```

## Adding Components
If you need to add components – maybe asynchronously – you can use `add`:
```javascript
app.add({
  modal: component(context => {})
})

app.mount() // bind the newly added component to the DOM
```

Additionally, you can pass arbitrary promises that resolve to a component:
```javascript
import { picoapp } from 'picoapp'
import button from './button.js'

const components = { button }

// check for native image lazy-loading
if (!'loading' in HTMLImageElement.prototype) {
  const lazyImage = import('lazy-image-fallback.mjs').then(module => module.default)
  components.push(lazyImage)
}

const app = picoapp(components) // components.lazyImage is a promise
```

```javascript
// lazy-image-fallback.mjs
export default component((node, ctx) => {
…
})
```

## State & Events
`picoapp` uses a very simple concept of state, which is shared and updated using
events or `hydrate` helpers. Internally, picoapp uses
[evx](https://github.com/estrattonbailey/evx), so check that library out for
more info.

You can define initial state:
```javascript
const app = picoapp({ button }, { count: 0 })
```

And consume it on the `context` object passed to your `component`:
```javascript
export default component((node, ctx) => {
  // ctx.getState().count
})
```

To interact with state, you will primarily use events. Passing an `object` when
emitting an event will *merge* that object into the global `state`. Event
listeners are then passed the entire `state` object for consumption.
```javascript
export default component((node, ctx) => {
  ctx.on('incremenent', state => {
    node.innerHTML = `I've been clicked ${state.count} times`
  })

  node.onclick = () => {
    ctx.emit('increment', { count: ctx.getState().count + 1 })
  }
})
```

You can also pass a function to an emitter in order to reference the previous
state:
```javascript
ctx.emit('increment', state => {
  return {
    count: state.count + 1
  }
})
```

Just like [evx](https://github.com/estrattonbailey/evx), `picoapp` supports
multi-subscribe, wildcard, and property keyed events as well:
```javascript
ctx.on([ 'count', 'otherProp' ], state => {}) // fires on `count` & `otherProp`
ctx.on('*', state => {}) // fires on all state updates
ctx.on('someProp', ({ someProp }) => {}) // fires on all someProp updates
```

If you need to update state, but don't need to fire an event, you can use
`ctx.hydrate`:
```javascript
export default component((node, ctx) => {
  ctx.hydrate({ count: 12 })
})
```

## Un-mounting
`picoapp` components are instantiated as soon as they're found in the DOM after
calling `mount()`. Sometimes you'll also need to *un-mount* a component, say to
destroy a slideshow or global event listener after an AJAX page transition.

To do so, return a function from your component:
```javascript
import { component } from 'picoapp'

export default component((node, ctx) => {
  ctx.on('incremenent', state => {
    node.innerHTML = `I've been clicked ${state.count} times`
  })

  function handler (e) {
    ctx.emit('increment', { count: ctx.getState().count + 1 })
  }

  node.addEventListener('click', handler)

  return (node) => {
    node.removeEventListener('click', handler)
  }
})
```

And then, call `unmount()`. All [evx](https://github.com/estrattonbailey/evx) event subscriptions within your component will be destroyed automatically.
```javascript
app.unmount()
```

`unmount()` is also synchronous, so given a PJAX library like
[operator](https://github.com/estrattonbailey/operator), you can do this *after*
every route transition:
```javascript
router.on('after', state => {
  app.unmount() // cleanup
  app.mount() // init new components
})
```

If your component does not define an `unmount` handler, the component will remain mounted after calling `unmount` (including all [evx](https://github.com/estrattonbailey/evx) event subscriptions within the component). This is useful for components that persist across AJAX page transitions such as global navigation or even a WebGL canvas.

## Other Stuff
The `picoapp` instance also has access to start and the event bus:
```javascript
app.emit('event', { data: 'global' })
app.on('event', state => {})
```

So you can add arbitrary state to the global `state` object directly:
```javascript
app.hydrate({ count: 5 })
```

And then access it from anywhere:
```javascript
app.getState() // { count: 5 }
```

If `data-component` isn't your style, or you'd like to use different types of
"components", pass your attributes to `mount()`:

Given the below, `picoapp` will scan the DOM for both `data-component` and
`data-util` attributes and init their corresponding JS modules:
```javascript
app.mount([
  'data-component',
  'data-util'
])
```

## License
MIT License © [Eric Bailey](https://estrattonbailey.com)
