# gooey

> :honey_pot: State blueprints for consumers of Hypermedia-driven RESTful APIs

---

Gooey is an experimental specification that aims to provide a standardized solution to the common state problems found in modern web applications by incorporating application-level semantics.

This specification particuarly focuses on issues related to entity relationships, variable entity state represenations and the coordination of multiple sources of data.

## Status

This currently serves as a living document for building and refining concepts around the Gooey specification.

This experimental specification is still being developed and fully welcomes contributions, ideas and criticism from the Open Source community.

## Problem

Managing the client-side states of modern RESTful Web APIs has become increasingly difficult as web applications have become remarkably robust and complex.

RESTful APIs, especially public facing ones, typically offer many different locations and methods of accessing the same data, and the reasons for this are usually legitimate.

For instance, it's not uncommon to have both local (denormalized) and external (normalized) representations of entities:

**Normalized**

In this setup the URLs must be followed to access nested entity states from an API

```js
// GET http://api.madhax.io/v1/user

{
  "user": "some-user",
  "name": "Michael Scott",
  "cart": {
    "url": "http://api.madhax.io/v1/cart"
  }
}
```

**Denormalized**

In this scnario the nested entity `cart` is denormalized and flattened into its parent entity (`user`).

Note the `flat` query parameter in the URL, a simple convention indicating that entities should be denormalized (shallow):

```js
// GET http://api.madhax.io/v1/user?flat

{
  "user": "some-user",
  "name": "Michael Scott",
  "cart" : {
    "status": "active",
    "items": [
      { "id": 383, "url": "http://api.madhax.io/v1/item/383" },
      { "id": 814, "url": "http://api.madhax.io/v1/item/814" }
    ]
  }
}
```

> **Note**
> The above examples are NOT using semantic URLs and thus do NOT adhere to HATEOAS properly, part of the "Uniform Interface" constraint in REST.
>
> This is just an example of a very common yet flawed approach to supporting both.

Although very useful, this design, which we will call **variable entity state representation**, leads to a number of complications for managing entity states in the client:

- Generally controlling how and when entity states are loaded can be cumbersome. Sometimes you want to load a large tree of entities pre-emptively, and other times you want to lazily load individual entities or sub-entities on demand.
- Modern web applications typically allow the states of entities to come from multiple sources (e.g. nested in a parent object vs. fetched from an API, memory vs. Web Worker, etc.).
- Synchronizing the states between related entities often requires an additional layer of coordination/delegation (e.g. publish/subscribe, long polling, etc.).
- Ensuring that the states of parent entities are ready and loaded before their children is typically non-trivial, especially in complex views with many entity instances.
- There are essentially no well-established patterns for managing a tree of resource entity states that must be synchronized with a RESTful API.
- Using query tools such as GraphQL exacerbates this problem since data can come from any number of sources in any number of forms.

In short, designing and implementing elegant state management solutions can be enormously difficult. Because the use of standards, conventions and semantics in this problem has been historically minimal to non-existent, sharing repeatable solutions or even encapsulating common behavior (especially across frameworks) has proven to be a clumsy and seemingly intractable task.

## Solution

Standards and application semantics save the day.

Gooey is a JSON-based specification for defining semantic blueprints of RESTful API resource entity states.

It is specifically designed to provide semantic solutions the state problems that are commonly found in Progressive Web Applications (PWAs) and Single Page Applications (SPAs) backed by RESTful Web APIs.

Gooey solves the variable state represenation issue by:

1. Formally describing the **relationships** between localized and remote API resource entity states.
2. Outlining how these states and their relationships should be managed, opening up interesting opportunities for highly generalized state management libraries and state debugging tools.
3. Defining the potential origins/sources of a state, such as local session memory, the browser URL, a Restful Web API resource, a Service Worker, or IndexedDB.
4. Describing a procedure for resolving URI templates using active entity states (`/v1/orders/{$ref.orders.one.uuid}`)
5. Ensuring states are compatible with the HTML5 History API

### Design

- Complete interopability with any JSON-based Hypermedia specification (JSON Hyper-Schema, JSON L-D, HAL, Siren, etc.)
- Differentiate between state classes/schemas and state instances
- Allow freedom of implementation beyond the core semantics constructs, especially `rels`
- Targeted at the most common state management problems, such as normalized vs denormalized representations and multiple sources of data
- States can be initialized from one or more of the provided sources
- Support referencing of elements using JSON Pointers in order to keep definitions DRY
- Complementary and minimally invasive. You don't always have to use it

### Flows

#### Instantiation

1. Determine the canonical source of the current entity's state, starting at the `root` entity (specified with `"root": true`, but this should also be implicit as multiple roots can theoritically exist - at least, why not)
2. Access the state's data through the source tied to the root entity
3. Broadcast a state update to all non-canonical `sources`, ensuring the entity state has been synchronized
4. Repeat this process for all sub-entities until all leaf entities have been processed

#### Updates

1. Determine the canonical source of the relevant entity's state
2. Acquire the latest state from the canonical source
3. Create a UUID and associate it with the proposed state change
4. Calculate the difference between the previous state and the proposed state change
5. Update client's internal representations (in each `store`) of the acquired state and delegate the update to any sub-entity `store`s
 - Allows the use of a callback to intercept and modify the state commit
 - Behavior should be avoidable under user-defined condition
6. Delegate the update to any sub-entity, `sources` (only non-canonical)
 - Allows user to determine whether or not this behavior is necessary (and modify it) using a callback
7. Upon successful entity state delegation, the differences between the previous and latest states (on a per-entity basis) will be appended to a linked-list in the specified `history` store
8. Repeat process for any sub-entities and their states until all leaf entities have been processed

### Example

```js
{
  "@spec": "gooey",
  "@version": "1.0",
  "@schema": {
    "type": "hyper-schema",
    "url": "http://api.madhax.io/schemas/index.json"
  },
  "@sources": { // FIXME: this is overly verbose. can flatten this quite a bit.
    "memory": [
      { "type": "ram" },
      { "type": "session" },
      { "type": "local" }
    ],
    "offline": {
      "type": "worker",
      "value": "worker/offline.js"
    },
    "api": {
      "type": "url",
      "value": "http://{s}.madhax.io/v1/",
      "root": true
      "shards": [
        'api1',
        'api2',
        'api3',
        'api4'
      ]
    }
  },
  "@entities": {
    "auth": {
      "@sources": [
        { "type": "memory.local" },
        { "type": "api", "rel": "one", "url": "/auth", "method": "POST" }
      ]
    },
    "user": {
      "@sources": [
        { "type": "memory.ram", "rels": ["one", "all"] },
        { "type": "api", "rel": "one", "url": "/user" }, // TODO: support `"link": "self"` syntax (requires association with JSON Hyper-Schema or the use of a semantic definition framework like ALPS)
        { "type": "api", "rel": "all", "url": "/users" }
      ],
      "@entities": { // TODO: determine if `entities` can actually be recursive. if not, rename to `properties` or something
        "cart": {
          "@ref": "#/entities/cart"
        },
        "orders": {
          "@ref": "#/entities/orders"
        }
      },
      "@root": true
    },
    "cart": {
      "@sources": [
        { "type": "memory.ram", "rels": ["one", "all"] },
        { "type": "api": "rel": "one", "url": "/cart" },
        { "type": "api", "rel": "all", "url": "/carts" }
      ]
    },
    "orders": {
      "@sources": [
        { "type": "memory.ram", "rels": ["one", "all"] },
        { "type": "api": "rel": "one": "url": "/order" },
        { "type": "api", "rel": "all", "url": "/orders" }
      ],
      "@entities": {
        "items": {
          "@ref": "#/entities/products"
        }
      }
    },
    "products": {
      "@sources": [
        { "type": "memory.ram", "rels": ["one", "all", "page"] },
        { "type": "api", "rel": "one", "url": "/product/{uuid}" },
        { "type": "api", "rel": "all", "url": "/products" }
      ]
    }
  },
  "@routes": [
    {
      "rel": "login",
      "path": "/login",
      "use": ["auth", "user"]
    },
    {
      "rel": "user",
      "path": "/me",
      "use": ["auth", "user"]
    },
    {
      "rel": "cart",
      "path": "/cart",
      "use": ["auth", "user", "cart"],
      "keep": true
    },
    {
      "rel": "orders",
      "path": "/orders",
      "use": ["auth", "user", "orders"]
    },
    {
      "rel": "order",
      "path": "/order/:id",
      "params": {
        "id": {
          "rel": "one",
          "entity": "order",
          "data": "#/uuid"
        }
      }
    }
  ],
  "@history": {
    "type": "memory.ram",
    "enabled": true // default, if the @history property is defined
  }
}
```

You only have to define `@entities` that have variation. In other words, only those entities with states that can stem from multiple sources or have multiple formats.

Any unmentioned properties of an entitiy are essentially ignored since their state management is trivial - just use whatever the current value is and assume the format will always be the same.

Primitive, static and non-relational values never simply need to be synchronized with other entities, unlike `user.cart` and `cart`.

## Documentation

### `rel`

Defines a set of application semantics for describing entity states.

Required when used in `@source` definitions.

- `one`
- `all`
- `page`
- `stream`

### `@schema`

Optional. Defines a JSON Hyper-Schema to use as a base reference for resolving `link` entities in `@sources`. Can also be used to enable automatic validation of entity instances.

### `@sources`

Required. Allows user to define the potential sources for an entity's state

TODO: Consider renaming this `@origins`

- `memory`
  * `ram`
  * `local`
  * `session`
- `cookie`
- `offline`
  * `worker`
- `db`
  * `indexed`
  * `web-sql`
- `stream`
- `push`
- `api`
  * `url`
   - `method`
  * `link`
  * `shards`
  * `version`
  * `root`
- `root` (TODO: remember what this is lol)

### `@entities`

Required. Defines the set of semantic entities that a web application may work with.

Generally focused on the front-end, but `@entities` should ideally match the domain entities found in the back-end.

### `@ref`

Optional. Specifies a JSON Pointer that references another entity's state..

Prevents the need for duplication and keeps your definitions DRY.

### `@classes`

Optional. Defines a collection of semantic tags to associate with an entity and its instances.

### `@routes`

Optional. Defines the potential URL routes that a web application supports.

TODO: support nested routes

- `rel`
- `path`
- `use`
- `params`

### `@history`

Optional. Defines where to store the state of your entities that will be used by the HTML History API.

The default store location should be `memory.ram`. Enabled by default.

## Questions

- How should updates to entities be described, such as `PUT` and `PATCH`? Do they even need to be defined?
- How do we define how to send updates to Service Workers? Do we just call some method that's guaranteed to exist such as `push`?

## Concerns

- Would this solution box people in to overly opinionated design? I think not since I keep running into the same problems with similar yet different solutions, but perhaps others can convince me otherwise.
- There is no need to save entities in too many sources. For instance, it's often adequate to only need `api` and `memory`. Perhaps an `offline` source as well.

## Future

- [ ] Establish lexicon
- [ ] Talk about entity schemas vs. entity instances
- [ ] Define implementationn API
- [ ] Establish state transaction standards (e.g. `dispatch`, `commit`)
- [ ] Establish core state transactions (e.g. `fetch`, `create`, `update`, `delete`, `reset`)
- [ ] Support wildcard state transactions for both entity schemas and entity instances
- [ ] Write JSON Schema definiion
- [ ] Consider how something like ALPS (a hypermedia profiling spec) could be used
- [ ] Establish a tighter convention or pattern around the use of the `@` symbol
- [ ] Establish constructs for expressing how and when entities should be refreshed (perhaps this should be done at the source level)
