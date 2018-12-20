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

 - Defines the relationships and dataflow strategies between entities in a 1:N domain tree (would probably replace everything described in current README - lol)
 - Entity instances can either be composed of arbitrary data or references to other entity instances. This allows for transparent normalization and denormalization of entity instances and their related members.
 - Entity instances are scoped by their parent entity's context. `dispatch` and `commit` actions are delegated via this context.
 - Any `String` value may also be provided as a `Function`, enabling a high degree of flexibility

### Example

```js
new Gooey.Outline({
  @sources: { ... },
  @entities: {
    user: {
      @source: {
        @url: '/v1/user'
      },
      @strategies: {
        @fetch: 'always-fetch',
        @change: 'clear-children',
        @sync: 'replace'
      },
      @entities: {
        libraries: {
          @source: {
            @url: '/v1/libraries'
          },
          @strategies: {
            @fetch: 'expect-or-fetch',
            @sync: 'push'
          },
          @entities: {
            books: {
              @source: {
                @url: ({ parent }) => `/v1/library/${parent.uuid}/books`
              },
              @strategies: {
                @fetch: 'expect-or-fetch',
                @sync: 'push'
              },
              @entities: {
                reviews: {
                  @source: {
                    @url: ({ parent }) => `/v1/books/${parent.uuid}/reviews`
                  },
                  @strategies: {
                    @history: true
                  }
                }
              }
            }
          }
        }
      }
    }
  }
})
```

## Entities

 - Contains unmodified representations of entity instances, from either remote API responses or local representations
 - Entity instances may be provided in either normalized or denormalized forms (via the `normalizer` package)
 - Entity instance states are tracked in a key-value store and relationships between them are tracked via semantic rels
 - Entity instances can be contextually bound to their parent entity instance or to be context-free (which can be achieved by omitting the `@change` strategy)

## Semantics

 - Contains semantics that describe the relationships between entity instances
 - Can also be used to describe a relationship to single enity instance (to itself)

### Rels

@see https://www.iana.org/assignments/link-relations/link-relations.xhtml

 - `parent`
 - `all`
 - `page`

### Example

```
{
  "@rel": "parent",
  "@type": "article"
  "@id": "abcdef-123456",
  "@entities": [
    { "@type": "comment", "@id": "zyxl-123456" }
  ]
}
```

```
{
  "@rel": "page",
  "@type": "comment",
  "@data": { "num": 5 },
  "@entities": [
    { "type": "comment", "@id": "8674934" },
    { "type": "comment", "@id": "5375858" }
  ]
}
```

```
{
  "@rel": "draft",
  "@type": "comment",
  "@id": "abcdef-123456" // this ID links back to a pre-existing entity in the state store
}
```

## Context

Contains a session-based context object for describing collective data between entity instances (maybe try to avoid this, but could be useful re: separation of concerns).

Contexts are in practice scoped by entity instead of entity instance.

Any changes taken to the entity context can be delegated to that entity's instances and child entities.

Allows the original entity instance states to remain prestine.

### Example

```
{
  "@entity": {
    "@type": "article",
    "@id": "12345"
  },
  "@context": {
    "selected": true,
    "viewed": true
  }
}
```

or

```
{
  "@entity": {
    "@type": "article"
  },
  "@context": {
    "selected": 12345
  }
}
```

## Strategies

Reserved and user-provided functionality that determines how an entity should behave whenever any of its entity instance context objects change.

Strategies provided as a `Function` accept the following arguments:

```
strategy ({ store, instance, entity })
```

 - `store`: Allows strategies to invoke actions or mutations based on custom conditions via `dispatch` and `commit`

### Example

The following example specifies that, when loaded for first use, the entity's state should always be fetched from its root source.

It also specifies that all children `@entity` states should be whiped out entirely whenever the entity's context changes.

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

 - `fetch`
 - `create`
 - `update`
 - `replace`
 - `delete`
 - `reset`
 - etc.

All of the reserved actions will be both prefixed with `$` (e.g. `$fetch`) and vanilla (e.g. `fetch`).

By default, the vanilla action simply calls the same action but prefixed with `$`.

This allows the user to either override the reserved action behavior entirely, or to simply wrap it with thier own logic and then invoke the default reserved behavior as they wish.

#### Custom

```
action ({ context, entity, instance }, data)
```

Where `context` is implicitly provided and contains:
 - `commit`: Calls a mutation on a single entity instance
 - `dispatch`: Calls an action on a single entity instance
 - `broadcast`: Calls an action scoped to the entity's context, affecting all entity instances

And `entity` is implicitly provided and contains:
 - `rels`
 - `context`

And `instance` is implicitly provided and contains the canonical normalized representation/state of the entity instance

And `data` is an arbitrary object provided by the user that may be delagated to mutations

## Mutations

Reserved and user-provided functions that implement and commit state transitions to entity instances and contexts

Whenever `context` or `semantics` are mutated, they will invoke the context `@strategies` defined in the parent `Gooey.Outline`.

### Examples

```
mutation ({ entity, instance, rootInstance }, data)
```

Where `entity` is a special object containing:

 - `rels`
 - `context`

and `instance` represents the canonical normalized state of the entity instance and may be safely modfified in mutations.

Any changes made to entity properties that are relationships/links to other entities will be delegated to those related entity instances.

An entity's `context` may also be safely modified in mutations. Changes to this context will be delegated to all entity instances of that type.
 - Actually, that might be dumb. We might want tointroduce a special action-like construct for delegating events to entity contexts. We can call it `broadcast` or something.

Mutations may not trigger other mutations (actions are for grouping together and orchestrating mutations).

## Dataflow

 ---> = 1:1
 -->> = 1:M
 l -> r = Callback

DispatchAction(Rel, Entity, Data) --->
  EntityInstances = FindEntities(Entity, Rel)

  Broadcast(Entity, EntityInstances, ParentEntity?) -->>
    EntitySources = SourcesOf(Entity)

    SyncSubscribers(EntitySources)
    SyncSubscribers(EntityInstances) -->>
      Recurse(Broadcast, [ChildEntity, ChildEntityInstances, ChildEntity => ChildEntity.Parent])
