## Middy
Middy is a middleware framework for AWS lambdas.

### Middy Input Schemas
`@middy/validator` can be used to validate input requests. The middleware can be added to the middy framework using `.use(validator({inputSchema}))`

#### Example Input Schema
```javascript
var inputSchema =  {
    required: ['body'],
    properties: {
        body: {
            type: 'object',
            required: ['colour', 'brand'],
            properties: {
                colour: {type: 'string'},
                brand: {type: 'string'}
            }
        }
    }
}
```

### Custom Middy Middleware
Custom middleware can be created for Middy that can be added to the framework using `.use(myCustomMiddleware())`
```javascript
export function myCustomMiddleware(): any {
    return {
        before: async (handler : any) => {
            if(!handler.event.headers.hasOwnProperty("traceid")){
                handler.event.headers.traceid = uuid();
            }
            return;
        }
    }
}
```