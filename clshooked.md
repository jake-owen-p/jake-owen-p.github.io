# Simple Request Specific Logging for Node.js

If you want to skip to the code, you can here [Logger in Action](#Logger-in-action). You can plug the logger in to any web app thats invoked with a callback, usually through middleware. 

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

So how do we ensure that when we want to log, we retrieve the correct `requestId` for that request? Due to the nature of Node.js, it turns from asking `How can we access properties globally for a specific request?` to `How can we keep track of a request's asynchronous operations?`

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

[Detailed cls-hooked explanation](https://habr.com/en/post/442392/)

### cls-hooked

[cls-hooked](https://www.npmjs.com/package/cls-hooked) keeps track of asynchronous operations by creating a map of `asyncId:context`, where `asyncId` is always unique and `context` contains details about the request, where you can store your request level data.

Lets take the same scenario from earlier:

1) `requestOne` starts, we add the `requestId` to the `context`, asign an id to the request and its added to the map. (I'm adding the request name in the table for clarity)

| request | asyncId | context |
|---------|---------|---------|
| requestOne |   22    |{ ...ctx }| 

2) Now our asynchronous db operation starts. async hooks' `init` is called with `init(asyncId, type, triggerAsyncId)`. 
    - `triggerAsyncId` is the asyncId of the operation that called it (in this case 22)
    - `asyncId` is the id of the new resource

It checks whether the `triggerAsyncId` exists in the map, and if it does it will add a new entry to the map with the new `asyncId` and the same context.

| request | asyncId | context |
|---------|---------|---------|
| requestOne |   22    |{ ...ctx }| 
| requestOne |   49    |{ ...ctx }| 

3) `requestTwo` enters your application, requestId is added to a new context object, an id is assigned to the request and its added to the map.

| request | asyncId | context |
|---------|---------|---------|
| requestOne |   22    |{ ...ctx }| 
| requestOne |   49    |{ ...ctx }| 
| requestTwo |   72    |{ ...ctx }| 

4) `requestOne` now goes to log that it has finished its db operation, it knows its current `asyncId` is `49`, so it will get the context of `49` from the map, which contains the `requestId` for `requestOne` and not `requestTwo`. Once this db operation is finished, async hooks `destroy(asyncId)` is called as `destroy(49)` where it will then be removed from the map.

| request | asyncId | context |
|---------|---------|---------|
| requestOne |   22    |{ ...ctx }| 
| requestTwo |   72    |{ ...ctx }| 

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

Then when you want to access the request level data, you can simply use `useNamespace('app')`:

```ts
function logInfo(message: string): void {
    console.log(message, {
        requestId: cls.getNamespace('app').get('requestId')
    });
}
```

I know that feels like a lot of information, but its implementaton ends up being fairly easy to reason with.

## Logger in Action

```ts
import { getNamespace, createNamespace, Namespace } from 'cls-hooked';

let clsNamespace: Namespace | undefined = getNamespace('app');
export default class logger {
    static setDefaultsAndCreateNamespace(requestId: string, next: () => void ) {
        if(!clsNamespace) clsNamespace = createNamespace('app');
        clsNamespace.run(() => {
            (clsNamespace as Namespace).set('requestId', requestId);
            next();
        });
    }

    static info(message: string, additionalProps: object = {}){
        console.info({
            name: message,
            requestId: (getNamespace('app') as Namespace).get('requestId'),
            ...additionalProps
        });
    }

    static error(message: string, error: Error, additionalProps: object = {}) {
        console.error({
            name: message,
            message: error.message,
            stackTrace: error.stack,
            requestId: (getNamespace('app') as Namespace).get('requestId')
        });
    }
}
```

### Middy
Below is a [Middy](https://github.com/middyjs/middy) middleware for AWS Lambda that utilizes the above logger. It checks if the request has a `requestId` header, and if so it sets the requestId to the header value, else generate a uuid.

```ts
import middy from '@middy/core';
import { APIGatewayEvent, APIGatewayProxyResult } from 'aws-lambda';
import { v4 as uuid } from 'uuid';
import logger from './logger';

export function requestIdResolver(): middy.MiddlewareObject<APIGatewayEvent, APIGatewayProxyResult> {
    return {
        before: (handler : middy.HandlerLambda<APIGatewayEvent, APIGatewayProxyResult>, next) => {
            const requestIdKey = findKey(handler.event.headers, 'requestId')
            handler.event.headers.requestIdKey = requestIdKey
                ? handler.event.headers[requestIdKey]
                : uuid();
            logger.setDefaultsAndCreateNamespace(handler.event.headers.requestId, next);
        }
    };
}
export function findKey(object: object, key: string) {
    return Object.keys(object).find(k => k.toLowerCase() === key.toLowerCase());
}
```

For my own use cases, I chose a very simple wrapper around console log. However, `cls-hooked` can also be plugged in to popular logging libraries. Here is an [alternative using winston](https://danoctavian.com/2019/04/13/thinking-coroutines-nodejs-part2/)

## Conclusion

Async hooks offers powerful utilties to track the lifecycle of asynchronous operations. `cls-hooked` leverages those in a way that offers request level storage for Node.js, which is much more powerful that the example i've given above. For example, you can use this persistant storage to store information you access sporodically accross the application like an accountId instead of passing it down every function.

However, there two downsides that have to be noted. 
1) As of Node.js v15, async hooks is still under the experimental umbrella. Meaning there can make non-backward compatible changes or removals in any release, use it at your own risk.
2) Using this can have an affect on performance, as `bmeurer` shows with his [Async hooks performance impact](https://github.com/bmeurer/async-hooks-performance-impact). The level of performance impact will depend on your application, framework and use case. 

