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

## "As a user, I want to browse and shop for products in a store"

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
A: We may need to introduce a JSON micro-format that allows APIs to provide all of the entity instances related to a provided entity instance (by their unique identifiers).
   We could also say that a `product` may contain a link to its parent `store`, which would allow the consumer to avoid making an extra API call.

## "As a user, I want to add items to a shopping cart for each store that I'm shopping"

## "As a user, I want to remove items from my shopping cart(s)"

## "As a user, I want to place orders based on the items in my cart(s)"

## "As a user, I want to create lists of products"

## "As a user, I want to browse all of my current and past orders"

## "As a user, whenever a store stops selling a product, I need that product removed from any cart and lists that may contain it"
