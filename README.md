# @requence/consumer

This package connects a JavaScript runtime (e.g. Node.js / Bun) to the operator bus. It manages the retrieval and responses of messages.

## Usage

```typescript
import createConsumer from '@requence/consumer'

createConsumer((ctx) => {
  return 'this is my response'
})
```

Every consumer instance needs a `url` parameter to connect to the operator and a `version`. In the basic example, those parameters get automatically retrieved from environment variables `REQUENCE_URL` and `VERSION`.
To be more explicit about those configurations, you can pass an object as the first parameter:

```typescript
createConsumer(
  {
    url: 'your operator connection url string',
    version: 'your version in format major.minor.patch',
  },
  (ctx) => {
    return 'this is my response'
  },
)
```

## Additional configuration

By default, the consumer retrieves one message from the operator, processes it and passes the response back to the operator. If your service is capable of processing multiple messages in parallel, you can define a higher `prefetch`.

```typescript
createConsumer(
  {
    prefetch: 10, // this will process max. 10 messages at once when available
  },
  async (ctx) => {
    return 'this is my response'
  },
)
```

## Unsubscribing service

Should you ever need to unsubscribe your service from the operator programmatically, `createConsumer` returns a promise with an unsubscribe function.

```typescript
const unsubscribe = await createConsumer(...)

// later
unsubscribe()
```

To resubscribe, you have to call `createConsumer` again

## Processing messages

The message handler callback provides one argument: the message context.
The context provides helper methods to access previous operator results and also methods to abort the processing early.

### context api data retrieval

```typescript
ctx.getInput()
```

The input that was defined when the task started

```typescript
ctx.getMeta()
```

The additional meta information added to the task

```typescript
ctx.getConfiguration(): unknown
```

The optional configuration of the current service

```typescript
ctx.getServiceMeta(serviceIdentifier): {
  executionDate: Date | null, // null when the service was not yet executed
  id: string, // service id used for internal routing
  alias?: string, // service alias (see service Identifier)
  name: string,
  configuration?: unknown,
  version: string
}
```

The meta parameters of a service, usually not needed

```typescript
ctx.getServiceData(serviceIdentifier): unknown
```

The response of a previously executed service

```typescript
ctx.getLastServiceData(serviceIdentifier): unknown
```

Same as `ctx.getServiceData` but only returns the last data when a service is used multiple times

```typescript
ctx.getServiceError(serviceIdentifier): string | null
```

The error message of a previously executed service or null when said service was not yet executed or did process without error.

```typescript
ctx.getLastServiceError(serviceIdentifier): string | null
```

Same as `ctx.getServiceError` but only returns the last error when a service is used multiple times

```typescript
ctx.getResults(): Array<{
  executionDate: Date | null, // null when the service was not yet executed
  id: string, // service id used for internal routing
  alias?: string, // service alias (see service Identifier)
  name: string,
  configuration?: unknown,
  version: string
  error?: string
  data?: unknown | null
}>
```

get results of all configured services in this task. When a service did run prior to the current service, `executionDate` and `error` or `data` will be available.

```typescript
ctx.getTenantName(): String
```

The name of the tenant that initiated the task

### context api processing control

```typescript
ctx.retry(delay?: number): never
```

Instructs the operator to retry the service after an optional delay in milliseconds (minimum is 100ms). No code gets executed after this line.
**Currently, it is your responsibility to prevent infinite loops.**

```typescript
ctx.abort(reason?: string): never
```

Instructs the operator to abort the processing of this service immediately.
If the service is not configured with a fail over, the complete task will fail.

### service identifier

Most context methods require a service identifier as parameter. This identifier can either be a service name or a service alias. The latter is useful for situations where a service is used multiple times in a task and needs the data from one specific execution.

## Full example

Pseudo implementation of a database service

```typescript
import createConsumer from '@requence/consumer'
import db from './database.js' // pseudo code
createConsumer(
  {
    url: 'amqp://username:password@host',
    version: '1.2.3',
    prefetch: 2, // we can process 2 messages in parallel
  },
  async (ctx) => {
    const ocrData = ctx.getServiceData('ocr')

    if (!ocrData) {
      ctx.abort('Ocr service is mandatory for this AI service')
    }

    if (!db.isConnected) {
      // lets wait 2s for the db to recover
      ctx.retry(2000)
    }

    const response = await db.getDataBasedOnOcr(ocrData)

    return response
  },
)
```
