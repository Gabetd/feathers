---
outline: deep
---

# Resolvers

Resolvers dynamically resolve individual properties based on a context, in a Feathers application usually the [hook context](../hooks.md#hook-context).

This provide a flexible way to do things like:

- Populating associations
- Returning computed properties
- Securing queries and e.g. limiting requests for a user
- Setting context (e.g. logged in user or organization) specific default values
- Removing protected properties for external requests
- Add read- and write permissions on the property level
- Hashing passwords and validating dynamic password policies

You can create a resolver for any data type and resolvers can also be used outside of Feathers.

## Example

Here is an example for a standalone resolver using a custom context:

```ts
import { resolve } from '@feathersjs/schema'

type User = {
  id: number
  name: string
}

type Message = {
  id: number
  userId: number
  likes: number
  text: string
  user: User
}

class MyContext {
  async getUser(id) {
    return {
      id,
      name: 'David'
    }
  }

  async getLikes(messageId) {
    return 10
  }
}

const messageResolver = resolve<Message, MyContext>({
  likes: async (value, message, context) => {
    return context.getLikes(message.id)
  },
  user: async (value, message, context) => {
    return context.getUser(message.userId)
  }
})

const resolvedMessage = await messageResolver.resolve(
  {
    id: 1,
    userId: 23,
    text: 'Hello!'
  },
  new MyContext()
)
```

## Property resolvers

Property resolvers are a map of property names to resolver functions. A resolver function is an `async` function that resolves a property on a data object. If it returns `undefined` the property will not be included. It gets passed the following parameters:

- `value` - The current value which can also be `undefined`
- `data` - The initial data object
- `context` - The context for this resolver
- `status` - Additional status information like current property resolver path, the properties that should be resolved or a reference to the initial context.

```ts
const userResolver = resolve<User, MyContext>({
  isDrinkingAge: async (value, user, context) => {
    const drinkingAge = await context.getDrinkingAge(user.country)

    return user.age >= drinkingAge
  },
  fullName: async (value, user, context) => {
    return `${user.firstName} ${user.lastName}`
  }
})
```

<BlockQuote type="danger">

Property resolver functions should only return a value and not have side effects. This means a property resolver **should not** do things like create new data or modify the `data` or `context` object. [Hooks](../hooks.md) should be used for side effects.

</BlockQuote>

## Virtual property resolvers

Virtual resolvers are property resolvers that do not use the `value` but instead always return a value of their own. The parameters are (`(data, context, status)`). The above example can be written like this:

```ts
import { resolve, virtual } from '@feathersjs/schema'

const userResolver = resolve<User, MyContext>({
  isDrinkingAge: virtual(async (user, context) => {
    const drinkingAge = await context.getDrinkingAge(user.country)

    return user.age >= drinkingAge
  }),
  fullName: virtual(async (user, context) => {
    return `${user.firstName} ${user.lastName}`
  })
})
```

<BlockQuote type="warning" label="Important">

Virtual resolvers should always be used when combined with a [database adapter](../databases/adapters.md) in order to make valid [$select queries](../databases/querying.md#select). Otherwise queries could try to select fields that do not exist in the database which will throw an error.

</BlockQuote>

## Options

A resolver takes the following options as the second parameter:

- `converter` (optional): A `async (data, context) => {}` function that can return a completely new representation of the data. A `converter` runs before `properties` resolvers.

```ts
const userResolver = resolve<User, MyContext>(
  {
    isDrinkingAge: async (value, user, context) => {
      const drinkingAge = await context.getDrinkingAge(user.country)

      return user.age >= drinkingAge
    },
    fullName: async (value, user, context) => {
      return `${user.firstName} ${user.lastName}`
    }
  },
  {
    // Convert the raw data into a new structure before running property resolvers
    converter: async (rawData, context) => {
      return {
        firstName: rawData.data.first_name,
        lastName: rawData.data.last_name
      }
    }
  }
)
```

## Hooks

In a Feathers application, resolvers are used through [hooks](../hooks.md) to convert service method query, data and responses. The context for these resolvers is always the [hook context](../hooks.md#hook-context).

### resolveData

Data resolvers use the `schemaHooks.resolveData(...resolvers)` hook and convert the `data` from a `create`, `update` or `patch` [service method](../services.md) or a [custom method](../services.md#custom-methods). This can be used to validate against the schema and e.g. hash a password before storing it in the database or to remove properties the user is not allowed to write. It is possible to pass multiple resolvers which will run in the order they are passed, using the previous data. `schemaHooks.resolveData` can be used as an `around` and `before` hook.

```ts
import { schemaHooks, resolve } from '@feathersjs/schema'
import { Type } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'
import type { HookContext } from '../declarations'

const messageSchema = Type.Object(
  {
    id: Type.Number(),
    text: Type.String(),
    createdAt: Type.Number(),
    updatedAt: Type.Number(),
    userId: Type.Number()
  },
  { $id: 'Message', additionalProperties: false }
)

type Message = Static<typeof messageSchema>

// Pick the data for creating a new message
const messageDataSchema = Type.Pick(messageSchema, ['text'])
type MessageData = Static<typeof messageDataSchema>

// Resolver that automatically set `userId` and `createdAt`
const messageDataResolver = resolve<Message, HookContext>({
  userId: async (value, message, context) => {
    // Associate the currently authenticated user
    return context.params?.user.id
  },
  createdAt: async () => {
    // Return the current date
    return Date.now()
  }
})

// Resolver that automatically sets `updatedAt`
const messagePatchResolver = resolve<Message, HookContext>({
  updatedAt: async () => {
    // Return the current date
    return Date.now()
  }
})

app.service('users').hooks({
  before: {
    create: [schemaHooks.resolveData(messageDataResolver)],
    patch: [schemaHooks.resolveData(messagePatchResolver)]
  }
})
```

Note that as an `all` hook `resolveData` will run for any method that has `data`, including [custom methods](../services.md#custom-methods). If you want to validate custom methods differently the hook should be registered on each service method it is used:

```ts
app.service('users').hooks({
  before: {
    create: [schemaHooks.resolveData(messageDataResolver)],
    update: [schemaHooks.resolveData(messageDataResolver)],
    patch: [schemaHooks.resolveData(messageDataResolver)],
    customMethod: [schemaHooks.resolveData(customMethodDataResolver)]
  }
})
```

### resolveResult

Result resolvers use the `schemaHooks.resolveResult(...resolvers)` hook and resolve the data that is returned by a service call ([context.result](../hooks.md#context-result) in a hook). This can be used to populate associations or add other computed properties etc. It is possible to pass multiple resolvers which will run in the order they are passed, using the previous data. `schemaHooks.resolveResult` can be used as an `around` and `after` hook.

<BlockQuote type="warning" label="Important">

Use as an `around` hook is recommended since this will ensure that database adapters will be able to handle [$select queries](../databases/querying.md#select) properly when using [virtual properties](#virtual-property-resolvers).

</BlockQuote>

```ts
import { schemaHooks, resolve, virtual } from '@feathersjs/schema'
import { Type } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'
import type { HookContext } from '../declarations'

const userSchema = Type.Object(
  {
    id: Type.Number(),
    email: Type.String(),
    password: Type.String(),
    avatar: Type.Optional(Type.String())
  },
  { $id: 'User', additionalProperties: false }
)
type User = Static<typeof userSchema>

const messageSchema = Type.Object(
  {
    id: Type.Number(),
    text: Type.String(),
    createdAt: Type.Number(),
    userId: Type.Number(),
    user: Type.Ref(userSchema)
  },
  { $id: 'Message', additionalProperties: false }
)

type Message = Static<typeof messageSchema>

export const messageResolver = resolve<Message, HookContext>({
  user: virtual(async (message, context) => {
    // Populate the user associated via `userId`
    return context.app.service('users').get(message.userId)
  })
})

app.service('messages').hooks({
  around: {
    all: [schemaHooks.resolveResult(messageResolver)]
  }
})
```

### resolveExternal

External (or dispatch) resolver use the `schemaHooks.resolveDispatch(...resolvers)` hook to return a safe version of the data that will be sent to external clients. It is possible to pass multiple resolvers which will run in the order they are passed, using the previous data. Returning `undefined` for a property resolver will exclude the property which can be used to hide sensitive data like the user password. This includes nested associations and real-time events. `schemaHooks.resolveExternal` can be used as an `around` or `after` hook.

```ts
import { schemaHooks, resolve } from '@feathersjs/schema'
import { Type } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'
import type { HookContext } from '../declarations'

const userSchema = Type.Object(
  {
    id: Type.Number(),
    email: Type.String(),
    password: Type.String(),
    avatar: Type.Optional(Type.String())
  },
  { $id: 'User', additionalProperties: false }
)
type User = Static<typeof userSchema>

export const userExternalResolver = resolve<User, HookContext>({
  // Always hide the password for external responses
  password: async () => undefined
})

// Dispatch should be resolved on every method
app.service('users').hooks({
  around: {
    all: [schemaHooks.resolveExternal(userExternalResolver)]
  }
})
```

<BlockQuote type="warning" label="important">

In order to get the safe data from resolved associations **all services** involved need the `schemaHooks.resolveExternal` hook registered even if it does not need a resolver (`schemaHooks.resolveExternal()`).

`schemaHooks.resolveExternal` should be registered first when used as an `around` hook or last when used as an `after` hook so that it gets the final result data.

</BlockQuote>

### resolveQuery

Query resolvers use the `schemaHooks.resolveQuery(...resolvers)` hook to modify `params.query`. This is often used to set default values or limit the query so a user can only request data they are allowed to see. It is possible to pass multiple resolvers which will run in the order they are passed, using the previous data. `schemaHooks.resolveQuery` can be used as an `around` or `before` hook.

```ts
import { schemaHooks, resolve } from '@feathersjs/schema'
import { Type } from '@feathersjs/typebox'
import type { Static } from '@feathersjs/typebox'
import type { HookContext } from '../declarations'

const userSchema = Type.Object(
  {
    id: Type.Number(),
    email: Type.String(),
    password: Type.String(),
    avatar: Type.Optional(Type.String())
  },
  { $id: 'User', additionalProperties: false }
)
type User = Static<typeof userSchema>

export const userQueryProperties = Type.Pick(userSchema, ['id', 'email'])
export const userQuerySchema = querySyntax(userQueryProperties)
export type UserQuery = Static<typeof userQuerySchema>

export const userQueryResolver = resolve<UserQuery, HookContext>({
  // If there is an authenticated user, they can only see their own data
  id: async (value, query, context) => {
    if (context.params.user) {
      return context.params.user.id
    }

    return value
  }
})

// The query can be resolved on every method
app.service('users').hooks({
  before: {
    all: [resolveQuery(userQueryResolver)]
  }
})
```
