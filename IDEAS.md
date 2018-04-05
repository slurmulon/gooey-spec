# Implementation

## Stores

```js
import gooey from 'gooey'
import config from './config.json'

export const module = new gooey.Module(config)

export const products = module.store('products')
export const cart = module.store('cart')
```

## State Transactions

Load every element

```js
this.store.dispatch({
  state: 'products/all',
  action: 'fetch'
})
```

Load every element and pipe only to certain source APIs

```js
this.store.dispatch({
  state: 'products/all',
  action: 'fetch',
  sources: ['memory.ram', 'memory.local']
})
```

Clear every element in a module and cascade the operation to its entity's dependents

```js
this.store.dispatch({
  state: 'products/all/**/*',
  action: 'delete'
})
```

Clear a single element in a module and cascade the operation to its entity's dependents

```js
this.store.dispatch({
  state: 'products/one/**/*',
  action: 'delete'
})
```

Select a single element as `one`

```js
this.store.dispatch({
  state: 'products/one',
  action: 'update',
  data: { id: 123, name: 'Kale' kind: 'vegetable' }
})
```

Go back to a previous entity state

```js
this.store.dispatch({
  state: 'cart/one',
  action: 'history.past',
  data: -1 // make this the default value
})
```

Go forward through history to another entity state

```js
this.store.dispatch({
  state: 'cart/all',
  action: 'history.future',
  data: 2
})
```
