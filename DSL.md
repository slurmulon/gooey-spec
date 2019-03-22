This is a placeholder for ideas around an experimental DSL for supplementing Web Components with entity state contracts

# Syntax

## Example

```
<template></template>

<script></script>

<state>
// Option 1
sync {
  'products' = {
    entity: 'product'
    scope: parent
    rel: all
    origin: *
  }
}

// Option 2
// <entity>/<rel> as '<namespace>'
sync {
  product/all as 'products' {
    scope: parent
    origin: *
  }
}

// Option 3
// ~ is a query operator, could make this a first-class citizen!
// `shape` specifies which form/shape the data should be mapped to. Perhaps `mask` is better?
//   - http://nmotw.in/json-mask/
sync {
  'products' with ~{
    entity: 'product'
    scope: parent
    shape: 'json-schema:product-only-ids.json'
    query: 'graph-ql:products-only-ids'
    rel: all
    origin: *
  }
}

actions {
  // bind product {
  from @product {
    buy
    prev
    next
  }

  // bind cart {
  from @cart {
    add
  }

  def custom {
    // pub `glueMethod`
    // yield 'fn:custom'
    yield // just yields to the method or function with the same namespace in the context
  }

  // def yield custom
  // quote custom
}

sub {
  'api/*/error' yield 'error.api'
}
</state>
```

## Routes

```
pipes:
  auth:
    meta:
      protected: true
    require:
      @user/current
      @auth/current
    error:
      pub 'auth:fail'
      go '/login'

routes:
  /login
  /products
    require:
      @product/all
        scope: parent
    component: 'view/products'
    routes:
      /:uuid
        meta:
          details: true
        require:
          @product/selected
        component: 'view/product'
  /cart
    pipe: auth
    require:
      @user/current
      @cart/current
        scope: parent
    component: 'view/cart'
    before:
      yield 'cart/prices/check-and-alert-changes'
    after:
      ...

```

## Loaders

I've conceptualized this as `origins` already, but the following is a great start for more detailed inspiration

https://github.com/facebook/dataloader

## Glue

Consider a `glue` keyword that allows users to say "this part needs to be handled by dynamic glue logic outside of here (usually in JS) identified with this name".

Components, entities or schemas that do not implement their glue methods will throw an error.

How about we just call this a `yield` instead? Cool concept, allows you to just leave Gooey land whenever you want and return as needed :)
 - Support the basic scalar types, arrays and JSON objects (UTF-8). Simple.

```
actions {
  destroy {
    glue `destroy`
  }
}
```
