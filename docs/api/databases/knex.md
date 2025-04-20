---
outline: deep
---

# SQL Databases

<Badges>

[![npm version](https://img.shields.io/npm/v/@feathersjs/knex.svg?style=flat-square)](https://www.npmjs.com/package/@feathersjs/knex)
[![Changelog](https://img.shields.io/badge/changelog-.md-blue.svg?style=flat-square)](https://github.com/feathersjs/feathers/blob/dove/packages/knex/CHANGELOG.md)

</Badges>

Support for SQL databases like PostgreSQL, MySQL, MariaDB, SQLite or MSSQL is provided in Feathers via the `@feathersjs/knex` database adapter which uses [KnexJS](https://knexjs.org/). Knex is a fast and flexible query builder for SQL and supports many databases without the overhead of a full blown ORM like Sequelize. It still provides an intuitive syntax and more advanced tooling like migration support.

```bash
$ npm install --save @feathersjs/knex
```

<BlockQuote>

The Knex adapter implements the [common database adapter API](./common) and [querying syntax](./querying).

</BlockQuote>

## API

### KnexService(options)

`new KnexService(options)` returns a new service instance initialized with the given options. The following example extends the `KnexService` and then uses the `sqliteClient` (or relevant client for your SQL database type) from the app configuration and provides it to the `Model` option, which is passed to the new `MessagesService`.

```ts
import type { Params } from '@feathersjs/feathers'
import { KnexService } from '@feathersjs/knex'
import type { KnexAdapterParams, KnexAdapterOptions } from '@feathersjs/knex'

import type { Application } from '../../declarations'
import type { Messages, MessagesData, MessagesQuery } from './messages.schema'

export interface MessagesParams extends KnexAdapterParams<MessagesQuery> {}

export class MessagesService<ServiceParams extends Params = MessagesParams> extends KnexService<
  Messages,
  MessagesData,
  ServiceParams
> {}

export const messages = (app: Application) => {
  const options: KnexAdapterOptions = {
    paginate: app.get('paginate'),
    Model: app.get('sqliteClient'),
    name: 'messages'
  }
  app.use('messages', new MessagesService(options))
}
```

### Options

The Knex specific adapter options are:

- `Model {Knex}` (**required**) - The KnexJS database instance
- `name {string}` (**required**) - The name of the table
- `schema {string}` (_optional_) - The name of the schema table prefix (example: `schema.table`)

The [common API options](./common.md#options) are:

- `id {string}` (_optional_, default: `'id'`) - The name of the id field property. By design, Knex will always add an `id` property.
- `paginate {Object}` (_optional_) - A [pagination object](#pagination) containing a `default` and `max` page size
- `multi {string[]|boolean}` (_optional_, default: `false`) - Allow `create` with arrays and `patch` and `remove` with id `null` to change multiple items. Can be `true` for all methods or an array of allowed methods (e.g. `[ 'remove', 'create' ]`)

There are additionally several legacy options in the [common API options](./common.md#options)

### getModel([params])

`service.getModel([params])` returns the [Knex](https://knexjs.org/guide/query-builder.html) client for this table.

### db(params)

`service.db([params])` returns the Knex database instance for a request. This will include the `schema` table prefix and use a transaction if passed in `params`.

### createQuery(params)

`service.createQuery(params)` returns a query builder for a service request, including all conditions matching the query syntax. This method can be overriden to e.g. [include associations](#associations) or used in a hook customize the query and then passing it to the service call as [params.knex](#paramsknex).

```ts
app.service('messages').hooks({
  before: {
    find: [
      async (context: HookContext) => {
        const query = context.service.createQuery(context.params)

        // do something with query here
        query.orderBy('name', 'desc')

        context.params.knex = query
      }
    ]
  }
})
```

### params.knex

When making a [service method](https://docs.feathersjs.com/api/services.html) call, `params` can contain an `knex` property which allows to modify the options used to run the KnexJS query. See [createQuery](#createqueryparams) for an example.

## Querying

In addition to the [common querying mechanism](./querying.md), this adapter also supports the following operators. Note that these operators need to be added for each query-able property to the [TypeBox query schema](../schema/typebox.md#query-schemas) or [JSON query schema](../schema/schema.md#querysyntax) like this:

```ts
const messageQuerySchema = Type.Intersect(
  [
    // This will additionally allow querying for `{ name: { $ilike: 'Dav%' } }`
    querySyntax(messageQueryProperties, {
      name: {
        $ilike: Type.String()
      }
    }),
    // Add additional query properties here
    Type.Object({})
  ],
  { additionalProperties: false }
)
```

### $like

Find all records where the value matches the given string pattern. The following query retrieves all messages that start with `Hello`:

```ts
app.service('messages').find({
  query: {
    text: {
      $like: 'Hello%'
    }
  }
})
```

Through the REST API:

```
/messages?text[$like]=Hello%
```

### $notlike

The opposite of `$like`; resulting in an SQL condition similar to this: `WHERE some_field NOT LIKE 'X'`

```ts
app.service('messages').find({
  query: {
    text: {
      $notlike: '%bar'
    }
  }
})
```

Through the REST API:

```
/messages?text[$notlike]=%bar
```

### $ilike

For PostgreSQL only, the keywork $ilike can be used instead of $like to make the match case insensitive. The following query retrieves all messages that start with `hello` (case insensitive):

```ts
app.service('messages').find({
  query: {
    text: {
      $ilike: 'hello%'
    }
  }
})
```

Through the REST API:

```
/messages?text[$ilike]=hello%
```

## Search

Basic search can be implemented with the [query operators](#querying).

## Associations

While [resolvers](../schema/resolvers.md) offer a reasonably performant way to fetch associated entities, it is also possible to join tables to populate and query related data. This can be done by overriding the [createQuery](#createqueryparams) method and using the [Knex join methods](https://knexjs.org/guide/query-builder.html#join) to join the tables of related services.

### Querying

Considering a table like this:

```ts
await db.schema.createTable('todos', (table) => {
  table.increments('id')
  table.string('text')
  table.bigInteger('personId').references('id').inTable('people').notNullable()
  return table
})
```

To query based on properties from the `people` table, join the tables you need in `createQuery` like this:

```ts
class TodoService<ServiceParams = KnexAdapterParams<TodoQuery>> extends KnexService<Todo> {
  createQuery(params: KnexAdapterParams<AdapterQuery>) {
    const query = super.createQuery(params)

    query.join('people as person', 'todos.personId', 'person.id')

    return query
  }
}
```

This will alias the table name from `people` to `person` (since our Todo only has a single person) and then allow to query all related properties as dot separated properties like `person.name`, including the [Feathers query syntax](./querying.md):

```ts
// Find the Todos for all Daves older than 100
app.service('todos').find({
  query: {
    'person.name': 'Dave',
    'person.age': { $gt: 100 }
  }
})
```

Note that in most applications, the query-able properties have to explicitly be added to the [TypeBox query schema](../schema/typebox.md#query-schemas) or [JSON query schema](../schema/schema.md#querysyntax). Support for the query syntax for a single property can be added with the `queryProperty` helper:

```ts
import { queryProperty } from '@feathersjs/typebox'

export const todoQueryProperties = Type.Pick(userSchema, ['text'])
export const todoQuerySchema = Type.Intersect(
  [
    querySyntax(userQueryProperties),
    // Add additional query properties here
    Type.Object(
      {
        // Only query the name for strings
        'person.name': Type.String(),
        // Support the query syntax for the age
        'person.age': queryProperty(Type.Number())
      },
      { additionalProperties: false }
    )
  ],
  { additionalProperties: false }
)
```

### Populating

Related properties from the joined table can be added as aliased properties with [query.select](https://knexjs.org/guide/query-builder.html#select):

```ts
class TodoService<ServiceParams = KnexAdapterParams<TodoQuery>> extends KnexService<Todo> {
  createQuery(params: KnexAdapterParams<AdapterQuery>) {
    const query = super.createQuery(params)

    query
      .join('people as person', 'todos.personId', 'person.id')
      // This will add a `personName` property
      .select('person.name as personName')
      // This will add a `person.age' property
      .select('person.age')

    return query
  }
}
```

<BlockQuote type="warning" label="important">

Since SQL does not have a concept of nested objects, joined properties will be dot separated strings, **not nested objects**. Conversion can be done by e.g. using Lodash `_.set` in a [resolver converter](../schema/resolvers.md#options).

</BlockQuote>

This works well for individual properties, however if you require the complete (and safe) representation of the entire related data, use a [resolver](../schema/resolvers.md) instead.

## Transactions

The Knex adapter includes transaction support that helps ensure data integrity across multiple service operations. Transactions allow you to group a series of database operations so that either all operations succeed or all operations are rolled back if any fail, maintaining your database in a consistent state.

### Understanding Transaction Hooks

The adapter provides three hooks for working with transactions:

- `transaction.start()` - Initializes a new transaction at the beginning of a request
- `transaction.end()` - Commits the transaction after all operations complete successfully
- `transaction.rollback()` - Rolls back the transaction if any operation fails

These hooks can be applied application-wide or at the service level:

```ts
import { transaction } from '@feathersjs/knex'

// Applied at the service level
export const order = (app: Application) => {
  app.use('orders', new OrderService(getOptions(app)), {
    methods: ['find', 'get', 'create', 'patch', 'remove'],
    events: []
  })
  
  app.service('orders').hooks({
    around: {
      all: []
    },
    before: {
      all: [transaction.start()],
      find: [],
      get: [],
      create: [],
      patch: [],
      remove: []
    },
    after: {
      all: [transaction.end()]
    },
    error: {
      all: [transaction.rollback()]
    }
  })
}
```

When using the `around` hook instead of the `before`/`after`/`error` hooks pattern:

```ts
import { transaction } from '@feathersjs/knex'

export const order = (app: Application) => {
  app.use('orders', new OrderService(getOptions(app)), {
    methods: ['find', 'get', 'create', 'patch', 'remove'],
    events: []
  })
  
  app.service('orders').hooks({
    around: {
      all: [
        async (context, next) => {
          // Start transaction
          await transaction.start()(context);
          
          try {
            // Wait for the next hook or service method
            const result = await next();
            
            // End transaction on success
            await transaction.end()(context);
            
            return result;
          } catch (error) {
            // Rollback transaction on error
            await transaction.rollback()(context);
            throw error;
          }
        }
      ]
    }
  })
}
```

### How Transactions Work

At the start of any request where the transaction hooks are applied, a new transaction will be created. All changes made during the request to services using the Knex adapter will use this transaction. If the request is successful, the changes will be committed at the end. If an error occurs, the changes will be rolled back—none of the `creates`, `patches`, `updates`, or `deletes` will be saved to the database.

The transaction object is stored in `params.transaction` for each request. This is important for nested service calls, as we'll see in the next section.

### Nested Service Calls with Transactions

When calling another Knex service within a hook, you must pass the `context.params.transaction` to share the transaction between services:

```ts
async createOrder(data, params) {
  // This service method already has transaction.start() applied through hooks
  
  // Create an order record
  const order = await super.create(data, params);
  
  // Create a shipping order record in another service, passing the transaction
  // This ensures both operations are part of the same transaction
  const shippingOrder = await this.app.service('shipping-orders').create({
    orderId: order.id,
    address: data.shippingAddress
  }, {
    ...params, // Pass the transaction from the parent service
  });
  
  return {
    ...order,
    shipping: shippingOrder
  };
}
```

### Real-world Example: Order and ShippingOrder Services

Let's look at a comprehensive example demonstrating how to use transactions to maintain atomicity when creating related records across multiple services.

First, we'll define our schemas:

```ts
// schemas/order.schema.ts
import { Type, getValidator, querySyntax } from '@feathersjs/typebox'

// Order schema
export const orderSchema = Type.Object(
  {
    id: Type.Number(),
    customerId: Type.Number(),
    total: Type.Number(),
    status: Type.String(),
    createdAt: Type.String({ format: 'date-time' })
  },
  { $id: 'Order', additionalProperties: false }
)

export const orderQueryProperties = Type.Pick(orderSchema, ['id', 'customerId', 'status'])
export const orderQuerySchema = Type.Intersect(
  [
    querySyntax(orderQueryProperties),
    Type.Object({}, { additionalProperties: false })
  ],
  { additionalProperties: false }
)

// Order data for creating and updating
export const orderDataSchema = Type.Pick(orderSchema, ['customerId', 'total', 'status'])

// Schema for creating new orders
export const orderExternalDataSchema = Type.Intersect([
  orderDataSchema,
  Type.Object({
    shippingAddress: Type.Object({
      street: Type.String(),
      city: Type.String(),
      state: Type.String(),
      zipCode: Type.String()
    })
  })
])

// Order service types
export type Order = Static<typeof orderSchema>
export type OrderData = Static<typeof orderDataSchema>
export type OrderExternalData = Static<typeof orderExternalDataSchema>
export type OrderQuery = Static<typeof orderQuerySchema>
```

```ts
// schemas/shipping-order.schema.ts
import { Type, getValidator, querySyntax } from '@feathersjs/typebox'

// ShippingOrder schema
export const shippingOrderSchema = Type.Object(
  {
    id: Type.Number(),
    orderId: Type.Number(),
    street: Type.String(),
    city: Type.String(),
    state: Type.String(),
    zipCode: Type.String(),
    status: Type.String(),
    createdAt: Type.String({ format: 'date-time' })
  },
  { $id: 'ShippingOrder', additionalProperties: false }
)

export const shippingOrderQueryProperties = Type.Pick(shippingOrderSchema, ['id', 'orderId', 'status'])
export const shippingOrderQuerySchema = Type.Intersect(
  [
    querySyntax(shippingOrderQueryProperties),
    Type.Object({}, { additionalProperties: false })
  ],
  { additionalProperties: false }
)

// ShippingOrder data for creating and updating
export const shippingOrderDataSchema = Type.Pick(shippingOrderSchema, [
  'orderId', 'street', 'city', 'state', 'zipCode', 'status'
])

// ShippingOrder service types
export type ShippingOrder = Static<typeof shippingOrderSchema>
export type ShippingOrderData = Static<typeof shippingOrderDataSchema>
export type ShippingOrderQuery = Static<typeof shippingOrderQuerySchema>
```

Next, let's implement our services:

```ts
// services/order.service.ts
import { KnexService } from '@feathersjs/knex'
import type { KnexAdapterParams, KnexAdapterOptions } from '@feathersjs/knex'
import type { Application } from '../../declarations'
import type { Order, OrderData, OrderQuery, OrderExternalData } from './order.schema'
import { transaction } from '@feathersjs/knex'

export interface OrderParams extends KnexAdapterParams<OrderQuery> {}

export class OrderService<ServiceParams extends Params = OrderParams> extends KnexService<
  Order,
  OrderData,
  ServiceParams,
  OrderExternalData
> {
  constructor(options: KnexAdapterOptions, app: Application) {
    super(options)
    this.app = app
  }

  async create(data: OrderExternalData, params?: ServiceParams): Promise<Order> {
    const { shippingAddress, ...orderData } = data
    
    // Create the order
    const order = await super.create(orderData, params)
    
    try {
      // Create shipping order using the same transaction
      const shippingOrder = await this.app.service('shipping-orders').create({
        orderId: order.id,
        ...shippingAddress,
        status: 'pending'
      }, {
        ...params // Pass the transaction to the shipping-orders service
      })
      
      return order
    } catch (error) {
      // The transaction rollback will happen automatically if an error occurs
      // because of the error hook we've set up
      throw error
    }
  }
}

export const getOptions = (app: Application): KnexAdapterOptions => {
  return {
    paginate: app.get('paginate'),
    Model: app.get('sqliteClient'),
    name: 'orders'
  }
}

export const order = (app: Application) => {
  app.use('orders', new OrderService(getOptions(app), app), {
    methods: ['find', 'get', 'create', 'patch', 'remove'],
    events: []
  })
  
  app.service('orders').hooks({
    before: {
      all: [transaction.start()],
      find: [],
      get: [],
      create: [],
      patch: [],
      remove: []
    },
    after: {
      all: [transaction.end()]
    },
    error: {
      all: [transaction.rollback()]
    }
  })
}
```

```ts
// services/shipping-order.service.ts
import { KnexService } from '@feathersjs/knex'
import type { KnexAdapterParams, KnexAdapterOptions } from '@feathersjs/knex'
import type { Application } from '../../declarations'
import type { ShippingOrder, ShippingOrderData, ShippingOrderQuery } from './shipping-order.schema'
import { transaction } from '@feathersjs/knex'

export interface ShippingOrderParams extends KnexAdapterParams<ShippingOrderQuery> {}

export class ShippingOrderService<ServiceParams extends Params = ShippingOrderParams> extends KnexService<
  ShippingOrder,
  ShippingOrderData,
  ServiceParams
> {}

export const getOptions = (app: Application): KnexAdapterOptions => {
  return {
    paginate: app.get('paginate'),
    Model: app.get('sqliteClient'),
    name: 'shipping-orders'
  }
}

export const shippingOrder = (app: Application) => {
  app.use('shipping-orders', new ShippingOrderService(getOptions(app)), {
    methods: ['find', 'get', 'create', 'patch', 'remove'],
    events: []
  })
  
  app.service('shipping-orders').hooks({
    before: {
      all: [transaction.start()],
      find: [],
      get: [],
      create: [],
      patch: [],
      remove: []
    },
    after: {
      all: [transaction.end()]
    },
    error: {
      all: [transaction.rollback()]
    }
  })
}
```

### Transaction Completion and Event Publishing

Sometimes it's important to know when a transaction has been completed (committed or rolled back). For example, you might want to wait for the transaction to complete before sending out any realtime events. This can be done by awaiting the `transaction.committed` promise:

```ts
app.service('orders').publish(async (data, context) => {
  const { transaction } = context.params

  if (transaction) {
    const success = await transaction.committed

    // Only publish the event if the transaction was successfully committed
    if (!success) {
      return []
    }
  }

  return app.channel(`customers/${data.customerId}`)
})
```

This works with nested service calls and nested transactions. If a service calls `transaction.start()` and passes the transaction param to a nested service call, which also calls `transaction.start()` in its own hooks, they will share the topmost `committed` promise that will resolve once all of the transactions have successfully committed.

### Error Handling within Transactions

Proper error handling is crucial when working with transactions to ensure they are rolled back when needed:

```ts
try {
  // Start with service that has transaction hooks
  const order = await app.service('orders').create({
    customerId: 101,
    total: 99.99,
    status: 'new',
    shippingAddress: {
      street: '123 Main St',
      city: 'Anytown',
      state: 'CA',
      zipCode: '12345'
    }
  });
  
  // If successful, both order and shipping order were created
  console.log('Order created:', order);
} catch (error) {
  // If an error occurred at any point, both the order and shipping order
  // creations were rolled back - nothing was saved to the database
  console.error('Error creating order:', error);
  
  // Additional error handling - you can check for specific error types
  if (error.name === 'Validation') {
    // Handle validation errors
  } else if (error.name === 'NotFound') {
    // Handle not found errors
  } else {
    // Handle other types of errors
  }
}
```

When an error occurs within a transaction:

1. The error is thrown up to the closest error handler
2. The `transaction.rollback()` hook catches the error and rolls back the transaction
3. All database changes made within the transaction are undone
4. The original error continues to propagate up, allowing you to handle it

### Application-wide Transaction Configuration

You can also configure transactions application-wide, which is useful when many services need to use transactions:

```ts
// app.ts
import { feathers } from '@feathersjs/feathers'
import { transaction } from '@feathersjs/knex'

// Configure Feathers app
const app = feathers()

// Set up application-wide transaction hooks
app.hooks({
  before: {
    all: [transaction.start()]
  },
  after: {
    all: [transaction.end()]
  },
  error: {
    all: [transaction.rollback()]
  }
})

// Register services
app.configure(order)
app.configure(shippingOrder)
```

With this configuration, any service method call will automatically be wrapped in a transaction, and all nested service calls will share the same transaction when the params are passed properly.

## Error handling

The adapter only throws [Feathers Errors](https://docs.feathersjs.com/api/errors.html) with the message to not leak sensitive information to a client. On the server, the original error can be retrieved through a secure symbol via `import { ERROR } from '@feathersjs/knex'`

```ts
import { ERROR } from 'feathers-knex'

try {
  await knexService.doSomething()
} catch (error: any) {
  // error is a FeathersError with just the message
  // Safely retrieve the Knex error
  const knexError = error[ERROR]
}
```

## Migrations

In a generated application, migrations are already set up. See the [CLI guide](../../guides/cli/knexfile.md) and the [KnexJS migrations documentation](https://knexjs.org/guide/migrations.html) for more information.
