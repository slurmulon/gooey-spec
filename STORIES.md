# Domain

                User
              /      \
            /          \
        Stores        Lists
       /      \      /
      /        \    /
     /          \  /
    Cart      Products
     |       /
     |     /
     |   /
    Items

                __User__
               /        \
              /          \
             /            \
            /              \
         Stores           Lists
        /      \          /
       /        \        /
      /          \      /
     /            \    /
   Cart  Orders  Products
     \      |      /
      \     |     /
       \    |    /
        \   |   /
          Items

# Stories

## A. "As a user, I want to browse and shop for products in a store"

### Scenario 1: User visits a store's page

1. Parse the unique identifier of the route's primary entity (i.e. `store`) from the URL (i.e. `/store/:id`)
2. Fetch all parent entities, starting with the roots (e.g. `user`)
3. Fetch the primary entity (i.e. `store`) based on its unique identifier
4. Fetch all child entities (e.g. `products`, `cart`, `items`)

### Scenario 2: User visit's a product's page

1. Parse the unique identifier of the route's primary entity (i.e. `store`) from the URL (i.e. `/store/:id`)
2. Fetch all parent entities, starting with the roots (e.g. `user`, `store`)
3. Fetch the primary entity (i.e. `product`) based on its unique identifier
4. Fetch all child entities (e.g. `cart`, `items`)

Q: How do we identify the `product`'s, parent `store`, assuming it isn't already part of the URL?
A: We may need to introduce a JSON micro-format that allows APIs to provide all of the entity instances related to another entity instance (by their unique identifiers).
   We could also say that a `product` may contain a link to its parent `store`, which would allow the consumer to avoid making an extra API call.

### Scenario 3: User switches from one store's page to another

TODO

 - Allow user to view the carts across all of their stores regardless of which store they are currently shopping in
 - Really only need to update the list of products that the user is allowed to browse throug (we might need to allow `@cascade` to support both an entity and an action)
   * We only want to update the list of products such that it that doesn't affect any products being used in the user's cart, items, orders or lists!
     - We might want to add a property to our actions that allows us to limit the scope of the action to entities instances who have no semenatic relationships established, other than to the root/invoking entity instance.

## B. "As a user, I want to add items to a shopping cart for each store that I'm shopping"

## C. "As a user, I want to remove items from my shopping cart(s)"

## D. "As a user, I want to place orders based on the items in my cart(s)"

## E. "As a user, I want to create lists of products"

## F. "As a user, I want to browse all of my current and past orders"

## G. "As a user, whenever a store stops selling a product, I need that product removed from any cart and lists that may contain it"

### Scenario 1: User is viewing the page for a product as it is being deleted

1. Client receives push event from the API that refreshes its list of available `products`
2. The primary entity instance is purged from all remaining resources (e.g. `memory`, `storage.local`, etc.)
3. All of the entity instances related to the primary entity instance are recursively updated so that they no longer reference the primary entity instance (i.e. `lists, items -> [cart, orders]`)
 - Another way of handling this is to simply "refresh" all of the related entity instances. It's more crude and requires an extra network call, but it minimizes the potential for data drift.

Q: How do we know if the server or client should be responsible for deleting the related entities?
A: The server should definitely be responsible for cascading the deletions. Perhaps it's fine to lock-in to this behavior.
   One way we could provide more flexibility here is to allow users to specify the `@resource` scope of their actions.

## H. "As a user, I need to be able to securely logout of my shopping session"

1. Recursively reset the states of all dependent entity instances, but ONLY in the client!
 - We could introduce a semantic value around `@resources` that specifies whether it's local or remote, which would allow users to simply say `@resources: 'local'` in their actions.
