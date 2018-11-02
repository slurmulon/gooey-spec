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

Update state for entities matching either a JSON Path or a Function

```js
this.store.dispatch({
  state: 'cart/all',
  action: 'delete',
  only: entity => entity.price > 100
})
```

JSON Schema states that are implicitly validated

```js
export default {
  state: {
    user: require('schemas/user.json'),
    email: require('schemas/email.json')
  },

  actions: {
    setEmail (state, email) {
      state.email = email // will throw a TypeError if the `email` doesn't pass the `email.json` validation
    }
  }
}
```

### Syncronization Strategies

Provide simple string configurations for common data syncronization strategies. Can be overridden with a custom `Function` instead.

```js
this.store.dispatch({
  state: '#/cart/all',
  sync: 'push', // default (i.e. append/push cart/all with results - grows indefinitely)
  action: 'fetch'
})
```
---

```js
this.store.dispatch({
  state: 'cart'/all'
  sync: 'replace', // replace cart/all with results
  action: 'fetch'
})
```

## Actions

Static/literal

```js
actions: {
  setEmail (state, email) {
    state.email = email
  }
}
```

Dyanmic/interpreted

(hmm, maybe don't need this if `actions` pertain to an entity instance instead of just an entity

```js
actions: {
  setEmail: (context) => (state, email) => {
    
  }
}
```

# Architecture

## Outlines

 - Defines the relationships and flow strategies between entities in your domain (would probably replace everything described in current README - lol)

### Example

```js
new Gooey.Outline({
  user: {
    @source: '/v1/user',
    @strategies: {
      @fetch: 'always-fetch',
      @change: 'clear-children'
    },
    @children: {
      farms: {
        @source: '/v1/farms',
        @strategies: {
          @fetch: 'always-fetch'
        },
        @children: {
          crops: {
            @source: ({ parent }) => `/v1/farms/${parent.uuid}/crops`
            @streategies: {
              @fetch: 'expect-or-fetch'
            }
          }
        }
      }
    }
  }
})
```

## Entities

 - Contains unmodified representations of entity instances, from either API responses or in-memory representations

## Semantics

 - Contains meta-descriptive semantics for entity instances

### Example

```
{
  "@rel": "parent",
  "@type": "Post"
  "@id": "abcdef-123456",
  "@children": [
    { "@type": "Comment", "@id": "zyxl-123456" }
  ]
}
```

```
{
  "@rel": "draft",
  "@type": "Comment",
  "@id": abcdef-123456,
  "data": {
    "body": "Hi there,"
  }
}
```

## Meta

 - Contains session-based meta data for entity instances (try to avoid this, but could be useful re: separation of concerns)

### Example

```
{
  "@entity": {
    "@type": "Post",
    "@id": "12345"
  },
  "@meta": {
    "selected": true
  }
}
```

## Dataflow

 ---> = 1:1
 -->> = 1:M
 l -> r = Callback

DispatchAction(Data) ---> Broadcast(EntitySources, ParentEntity?) -->> SyncSubscribers(EntitySources) -->> Recurse(Broadcast, [ChildEntitySources, ChildEntity => ChildEntity.Parent])

