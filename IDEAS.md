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

Load every element and pipe only to specific resources

```js
this.store.dispatch({
  state: 'products/all',
  action: 'fetch',
  resources: ['memory', 'storage.local']
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
  action: 'history-past',
  data: -1 // make this the default value
})
```

Go forward through history to another entity state

```js
this.store.dispatch({
  state: 'cart/all',
  action: 'history-future',
  data: 2
})
```

Update state for entities matching either a JSON Path or a Function

```js
this.store.dispatch({
  entity: 'cart',
  rel: 'all'
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
  cascade: true,
  action: 'fetch'
})
```

## Actions

### Props

 - `"@rel"` (`Array | String`): Dispatches the action to only those entities/entity-instances with a matching `@rel`.
 - `"@data" (`Object`): Arbitrary data to dispatch to each subscriber entity/entity-instance.
 - `"@entity"` ('String`): Unique name of an entity. Scopes the action to the entity context when provided.
 - `"@instance"` (`String | Object`): Unique string identifier or literal representation of entity instance. Scopes the action to the instance and its related instances when `@cascade` returns `true`.
 - `"@context"` (`Object'):
 - `"@resources"` (`Array`): Limits any state transactions as a result of this action to the provided state resources.
 - `"@cascade"` (`Boolean | Function`): Cascades the action to all subscribing entities/entity-instances.
 - `"@sync"` (`String | Function`): Specifies the behavior of the mutation state transactions (via `commit`) immediately resulting from this action.
   * Raises the question though - should this cascade to any child actions called?
 - `"@only"`(`Function`): 

### 

# Architecture

## Outlines

 - Defines the relationships and dataflow strategies between entities in a domain represented by a directed rooted tree.
 - Entity instances can either be composed of arbitrary data or references to other entity instances. This allows for transparent normalization and denormalization of entity instances and their related members.
 - Entity instances are scoped by their parent entity's context. `dispatch` and `commit` actions are delegated via this context.
 - Entity instances may have multiple states (handled by `@context` objects). Only the diffs between states are stored in order to enable efficient history storage and iteration.
 - Any `String` value may also be provided as a `Function`, enabling a high degree of flexibility.

### Example

```js
new Gooey.Outline({
  @origins: {
    // ...
  },
  @resolvers: {
    @id: ({ instance }) => instance.uuid, // Determines how unique entity instance IDs are generated
    @hash: ({ instance, time, core }) => core.hasher(instance, time), // Determines how hashes are generated for each entity instance state
    @delta ({ prev, next, core }) => core.delta(prevInstance, nextInstance) // Determines the delta between two instance states
  },
  // @pipeline: ['api', ['
  @entities: {
    user: {
      @type: 'object',
      @origins: {
        api: '/v1/user'
      },
      @entities: {
        libraries: {
          @type: 'array',
          @origins: {
            api: '/v1/libraries'
          },
          @entities: {
            books: {
              @type: 'array',
              @origins: {
                api: ({ parent }) => `/v1/library/${parent.instance.uuid}/books`,
                object: ({ parent }) => parent.instance.books
              },
              @entities: {
                reviews: {
                  @type: 'array',
                  @origins: {
                    api: ({ parent }) => `/v1/books/${parent.instance.uuid}/reviews`
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

```js


new Gooey.Outline({
  
})
```

## Entities

 - Contains unmodified representations of entity instances, from either remote API responses or local representations
 - Entity instances may be provided in either normalized or denormalized forms (via the `normalizer` package)
 - Entity instance states are tracked in a key-value store and relationships between them are tracked via semantic rels
 - Entity instances can be contextually bound to their parent entity instance or to be context-free

## Semantics

 - Contains semantics that describe the relationships between entity instances
 - Can also be used to describe a relationship to single enity instance (to itself)

### Rels

Defines semantic relationships between entity instances

Internally stored, maintained and synchronized with each defined `@origin`

Adhere to the URN specification `urn:gooey:*`

@see https://www.iana.org/assignments/link-relations/link-relations.xhtml
@see https://tools.ietf.org/html/rfc8141

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

Contains a session-based state object that describes additional data between one or more entity instances (maybe try to avoid this, but could be useful re: separation of concerns).

Contexts are, in practice, scoped by either and entity or an entity instance.

Any changes made via an action to an entity context can be cascaded to that entity's instances and child entities.

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

> **Note**
>
> Can/should probably remove the need for this by allowing each entity to specify the priority/ordering of their instance state `@origins`

Reserved and user-provided functionality which determines how an entity should behave whenever any of its entity instance context objects change.

Reserved strategies provided as an `Object` describe a natively supported semantic action (indexed by `@ref`):

```
{
  "@ref: "expect-or-fetch",
  "@sync": "push"
}
```

Strategies provided as a `Function` accept the following arguments:

```
strategy ({ store, instance, entity })
```

 - `store`: Allows strategies to invoke actions or mutations based on custom conditions via `dispatch` and `commit`
 - `instance`: Reference to entity instance
 - `entity`: Parent entity to the entity instance

### Example

The following example specifies that, when loaded for first use, the entity's state should always be fetched from its root resource.

It also specifies that all children `@entity` states should be whiped out entirely whenever the entity's context changes.

```
"@strategies": {
  "@fetch": "always-fetch",
  "@change": "clear-children"
},
```

### Core

## Actions

Reserved and user-provided functions that organize and coordinate state transitions

Actions are internally provided an ephemeral `transactions` tree that tracks every single action that was called in relation

All state transactions performed on enntity instances will be implicitly normalized, such that each normalized child entity instance found in the state transaction will automatically inherit the updated state of their parent

Actions can be optionally cascaded to all subscribing child entities and instances

Actions may be atomic but are non-atomic by default (need a flag for this)

### Examples

#### Reserved

 - `fetch`
 - `create`
 - `update`
 - `delete`

All of the reserved actions will be both prefixed with `$` (e.g. `$fetch`) and vanilla (e.g. `fetch`).

By default, the vanilla action simply calls the same action but prefixed with `$`.

This allows the user to either override the reserved action behavior entirely, or to simply wrap it with thier own logic and then invoke the default reserved behavior as they wish.

#### Custom

```
action ({ store, entity, instance }, data)
```

Where `store` is implicitly provided and contains:
 - `commit`: Calls a mutation on a single entity instance
 - `dispatch`: Calls an action on a single entity instance
 - `broadcast`: Calls an action scoped to the entity's context, affecting all entity instances
 - `cascade`: Calls an action scoped to either a single entity instance or an entity's context and delgates to all children

And `entity` is implicitly provided and contains:
 - `rels`
 - `context`

And `instance` is implicitly provided and contains the canonical normalized representation/state of the entity instance

And `data` is an arbitrary object provided by the user that may be delagated to mutations

## Mutations

Reserved and user-provided functions that implement and commit state transitions to entity instances and contexts

Under the hood, mutations are invoking and performing state synchronizations.

Each state synchronization results in a session-based global transaction to be created.
 
Whenever `context` or `semantics` are mutated, they will invoke the context `@strategies` defined in the parent `Gooey.Outline`.

### Examples

```
mutation ({ entity, instance, rootInstance }, data)
```

Where `entity` is a special object containing:

 - `rels`
 - `context`

and `instance` represents the canonical normalized state of the entity instance and may be safely modfified in mutations.

Any changes made to entity properties that are relationships/links to other entities will be delegated to those related entity instances via their internal `update` mutation.

An entity's `context` may also be safely modified in mutations. Changes to this context will be delegated to all entity instances of that type.
 - Actually, that might be dumb. We might want to introduce a special action-like construct for delegating events to entity contexts. We can call it `broadcast` or something.

Mutations may not trigger other mutations (actions are for grouping together and orchestrating mutations).

## Dataflow

```
 ---> = 1:1
 -->> = 1:M
 l -> r = Callback
```

### Synchronizations

```
Sync(Rel, Entity, Id, Instance)
  EntitySources = SourcesOf(Entity)
  EntitySyncStrategy = SyncStrategyOf(Entity)
  ChildEntityInstances = NormalizedChildEntityInstances(Entity, Instance)

  # TODO: Probably iterate through each Store here
  for Source in EntitySources
    State = CreateOrMutateState(EntitySyncStrategy, Source, Rel, Entity, Id, Instance)

    CreateTransaction(State)

  for (ChildEntity, ChildId, ChildInstance) in ChildEntityInstances
    Sync(Rel, ChildId, ChildEntity, ChildInstance)
```

### Mutations

### Actions

```
Action(Topic, Rel, Entity, Data) --->
  EntityInstances = FindEntities(Entity, Rel)

  Dispatch(Entity, EntityInstances, ParentEntity?) -->>
    for (EntityID, EntityInstance) in EntityInstances
      Sync(Rel, Entity, EntityID, EntityInstance)

    # EntitySources = SourcesOf(Entity)

    # SyncEntityContext(Entity, 

    SyncSubscribers(Topic, Rel, EntityInstances) -->>
      # ChildEntityInstances = DenormalizeEntity(Data)

      # QUESTION: Is this where we should also delegate `create` and `update` actions to child entity instances?
      # CreateImplicitSemanticRelationships(Rel, Entity, EntityInstance, ChildEntityInstances)

      Recurse(Dispatch, [ChildEntity, ChildEntityInstances, ChildEntity => ChildEntity.Parent])
```

### Broadcasts

## Problems

 - Right now these ideas are locked-in to JavaScript because they expect or allow JavaScript functions to be provided. In order to achieve greatest portability, we might want to find a way to replace them with Unicode strings (via JSON Pointer or something).
   * Perhaps we could allow a nice JS-exclusive API (or more generically define an API that can be implemented in any language) that allows users to either override or provide advanced functionality to their configurations in run-time.
 - `rel` feels a bit ambiguous when it comes to how `action`s should be implicitly cascaded to related entities/instances
   * If we change the `selected` User, we want to clear out `all` of the Orders
 - Details need to be sorted out on how to handle collections vs. items, particularly around their `@resources`.
