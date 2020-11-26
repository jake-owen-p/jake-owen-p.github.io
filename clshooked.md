# Simple Request Specific Logging for Node.js

// kind of want aws lambda in the title

If you want to skip to the code without explanation, please skip to [#Logger in Action]

### Problem Statement
When developing an API it is good practice to have a requestId that is persisted throughout the service and to any dependencies. This way there is a singular identifier that ties a whole journey together.

To get the benefits out of the requestId, you'll want to use it each time you use your logger. In multithreaded languages, this is trivial, as you can leverage [Thread Local Storage (TLS)](https://docs.microsoft.com/en-us/cpp/parallel/thread-local-storage-tls?view=msvc-160#:~:text=Thread%20Local%20Storage%20(TLS)%20is,the%20TLS%20API%20(TlsAlloc)). When a request enters your application, you can set the requestId in the TLS and whenever you want to log, you can just retrieve it again. Unfotunately, we do not have this luxury in Node.

Let's say you have a simple logger implementation with functions `setRequestId` and `logInfo`.

```ts
let requestId: string;
function setRequestId(id: string): void {
    requstId = id;
}
function logInfo(message: string): void {
    console.log(message, {
        requestId
    });
}
```

1) `requestOne` enters your application and calls `setRequestId` 
2) `requestOne` then starting perfoming an asynchronous db operation. 
2) `requestTwo` enters your application and calls `setRequestId`
3) `requestOne` finishes its db operation and calls `logInfo('db.done')`

See the problem? By the time `requestOne` gets to logging that it has completed its operation in the database, `requestTwo` has started and overriden the `requestId` value. Request one has now logged a message with the wrong requestId and we lose one of the main benefits of requestId that was mentioned at the start `This way there is a singular identifier that ties a whole journey together.`.

So how do we ensure that when we want to log, we retrieve the correct `requestId` for that request? Due to the nature of Node.js, it turns from asking `How can access properties globally for a specific request?` to `How can we keep track of a request's asynchronous operations?`

## Async hooks & cls-hooked

### Async hooks 
[Async hooks](https://nodejs.org/api/async_hooks.html) was introduced in Node.js v8 and provides a simple api to access lifecycle events of asynchronous resources.

1) init - called when a async resource is initialized
2) before - called before the resource executes
3) after - called after the resource executes
4) destroy - called when the resource's execution has completed
5) promiseResolve - called when promise gets its `resolve` function called

To keep this article simple, i'll be explaining how cls-hooked leverages async hooks at an extremely high level. But if you wish to learn more here's some fantastic resources for async hooks and cls-hooked: 

[Async Hooks: A Journey To a Realm With Persistent Execution Context](https://www.youtube.com/watch?v=Sakn7GV6EOw&t=1037s&ab_channel=monday.Engineering)

[Exploring Node.js Async Hooks](https://blog.appsignal.com/2020/09/30/exploring-nodejs-async-hooks.html)

// add stuff for cls hooked

### cls-hooked

cls-hooked keeps track of asynchronous operations by creating a map of `asyncId:context`, where `asyncId` is always unique and `context` contains details about the request, where you can store your request level data.

Lets take the same scenario from earlier:

1) `requestOne` starts, we add the `requestId` to the `context`, asign an id to the request and its added to the map.

| asyncId | context |
|---------|---------|
|   22    |{ ...ctx }| 

2) Now our asynchronous db operation starts. async hooks' `init` is called with `init(asyncId, type, triggerAsyncId)`. 
    - `triggerAsyncId` is the asyncId of the operation that called it (in this case 22)
    - `asyncId` is the id of the new resource

    It checks whether the `triggerAsyncId` exists in the map, and if it does it will add a new entry to the map with the new `asyncId` and the same context.

| asyncId | context |
|---------|---------|
|   22    |{ ...ctx }| 
|   49    |{ ...ctx }| 

3) `requestTwo` enters your application, requestId is added to a new context object, an id is assigned to the request and its added to the map.

| asyncId | context |
|---------|---------|
|   22    |{ ...ctx }| 
|   49    |{ ...ctx }| 
|   72    |{ ...ctx }| 

4) `requestOne` now goes to log that it has finished its db operation, it knows its current `asyncId` is `49`, so it will get the context of `49` from the map, which contains the `requestId` for `requestOne` and not `requestTwo`. Once this db operation is finished, async hooks `destroy(asyncId)` is called as `destroy(49)` where it will then be removed from the map.


So how does this work in code?

cls-hooked first creates a namespace that is uses to keep track of asynchronous operations for the application.
```ts
const clsNamespace = createNamespace('app');
```

`.run` is called to start the tracking of a request. This create the new `context`, you can set the request level data you need, and then a callback is initiated. This will then track all sequentially operations for this request.
```ts
clsNamespace.run(() => {
    clsNamespace.set('requestId', requestId)
    next()
})
```

If you take this dummy example middleware example, you can see how you can start the request within cls using the next callback.
```ts
let clsNamespace = useNamespace('app');
if(!clsNamespace) {
    clsNamespace = createNamespace('app');
}
function customMiddleware(): MiddlewareObject {
    return {
        before: (request: Request, next) => {
            clsNamespace.run(() => {
                clsNamespace.set('requestId', requestId);
                next();
            })
        }
    };
}
```

Then when you want to access the request level data, you can simple use `useNamespace('app')`, e.g. in our logging function from earlier.
```ts
function logInfo(message: string): void {
    console.log(message, {
        requestId: cls.getNamespace('app').get('requestId')
    });
}
```


### Logger in Action

```ts
import { getNamespace, createNamespace, Namespace } from 'cls-hooked';

export default class logger {
    static setDefaultsAndCreateNamespace(traceId: string, next: ()=>void ) {
        if(!clsNamespace) clsNamespace = createNamespace('app');
        clsNamespace.run(() => {
            (clsNamespace as Namespace).set('traceId', traceId);
            next();
        });
    }

    static info(message: string, additionalProps: object = {}){
        console.info({
            name: message,
            traceId: (getNamespace('app') as Namespace).get('traceId'),
            ...additionalProps
        });
    }

    static error(message: string, error: Error, additionalProps: object = {}) {
        console.error({
            name: message,
            message: error.message,
            stackTrace: error.stack,
            traceId: (getNamespace('app') as Namespace).get('traceId')
        });
    }
}

```