# Implementation

## Stores

```js
import gooey from 'gooey'
import config from './config.gooey.json'

const stores = new gooey.Module(config)

export const store = new gooey.St
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
