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

We might also want to allow the user to provide the `entity` `rel` in which this action is relevant to

# Architecture

## Outlines

 - Defines the relationships and dataflow strategies between entities in your domain (would probably replace everything described in current README - lol)
 - Any `String` value may also be provided as a `Function`, enabling a high degree of flexibility

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
 - May be provided in either normalized or denormalized forms (via the `normalizer` package)
 - Allow entity instances to be contextual bound to their parent entity instance or to be context-free

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
  "@id": "abcdef-123456" // this ID links back to a pre-existing entity in the state tree
}
```

## Meta

Contains session-based meta data for entity instances (try to avoid this, but could be useful re: separation of concerns)

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

## Strategies

Reserved and user-provided functionality that determines how entities should behave whenever their overarching context changes.

### Example

The following example specifies that, when loaded for first use, the entity's state should always be fetched from its root source.

It also specifies that all `@children` entity states should be whiped out entirely whenever the entity's context changes.

```
@strategies: {
  @fetch: 'always-fetch',
  @change: 'clear-children'
},
```

## Actions

Reserved and user-provided functions that organize and coordinate state transitions

### Examples

#### Reserved

 - `fetch`,
 - `delete`,
 - `create`,
 - `replace`,
 - `reset`
 - etc.

All of the reserved actions will be both prefixed with `$` (e.g. `$fetch`) and vanilla (e.g. `fetch`).

By default, the vanilla action simply calls the same action but prefixed with `$`.

This allows the user to either override the reserved action behavior entirely, or to simply wrap it with thier own logic and then invoke the default reserved behavior as they wish.

#### Custom

```
action (context, entity, data)
```

Where `context` is implicitly provided and contains:
 - `commit`
 - `dispatch`

And `entity` is implicitly provided and contains:
 - `data`
 - `meta`
 - `semantics`

And `data` is an arbitrary object provided by the user

## Mutations

Reserved and user-provided functions that implement and commit state transitions

Whenever `meta` or `semantics` are mutated, they will invoke the context `@strategies` defined in the parent `Gooey.Outline`.

### Examples

```
mutation ({ state, meta, semantics }, entity)
```

Where `entity` is a special object containing:

 - `data`
 - `meta`
 - `semantics`

## Dataflow

 ---> = 1:1
 -->> = 1:M
 l -> r = Callback

DispatchAction(Data) ---> Broadcast(EntitySources, ParentEntity?) -->> SyncSubscribers(EntitySources) -->> Recurse(Broadcast, [ChildEntitySources, ChildEntity => ChildEntity.Parent])

