# lambda-api
**PLEASE NOTE:** This project is still in beta and should be used with caution in production.

[![Build Status](https://travis-ci.org/jeremydaly/lambda-api.svg?branch=master)](https://travis-ci.org/jeremydaly/lambda-api)
[![npm](https://img.shields.io/npm/v/lambda-api.svg)](https://www.npmjs.com/package/lambda-api)
[![npm](https://img.shields.io/npm/l/lambda-api.svg)](https://www.npmjs.com/package/lambda-api)

### Lightweight Node.js API for AWS Lambda

Lambda API is a lightweight Node.js API router for use with AWS API Gateway and AWS Lambda using Lambda Proxy integration. This closely mirrors (and is based on Express.js) but is significantly stripped down to maximize performance with Lambda's stateless, single run executions. The API uses Bluebird promises to serialize asynchronous execution.

## Lambda Proxy integration
Lambda Proxy Integration is an option in API Gateway that allows the details of an API request to be passed as the `event` parameter of a Lambda function. A typical API Gateway request event with Lambda Proxy Integration enabled looks like this:

```javascript
{
  "resource": "/v1/posts",
  "path": "/v1/posts",
  "httpMethod": "GET",
  "headers": {
    "Authorization": "Bearer ...",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
    "Accept-Encoding": "gzip, deflate",
    "Accept-Language": "en-us",
    "cache-control": "max-age=0",
    "CloudFront-Forwarded-Proto": "https",
    "CloudFront-Is-Desktop-Viewer": "true",
    "CloudFront-Is-Mobile-Viewer": "false",
    "CloudFront-Is-SmartTV-Viewer": "false",
    "CloudFront-Is-Tablet-Viewer": "false",
    "CloudFront-Viewer-Country": "US",
    "Cookie": "...",
    "Host": "...",
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_3) ...",
    "Via": "2.0 ... (CloudFront)",
    "X-Amz-Cf-Id": "...",
    "X-Amzn-Trace-Id": "...",
    "X-Forwarded-For": "xxx.xxx.xxx.xxx",
    "X-Forwarded-Port": "443",
    "X-Forwarded-Proto": "https"
  },
  "queryStringParameters": {
    "qs1": "q1"
  },
  "stageVariables": null,
  "requestContext": {
    "accountId": "...",
    "resourceId": "...",
    "stage": "prod",
    "requestId": "...",
    "identity": {
      "cognitoIdentityPoolId": null,
      "accountId": null,
      "cognitoIdentityId": null,
      "caller": null,
      "apiKey": null,
      "sourceIp": "xxx.xxx.xxx.xxx",
      "accessKey": null,
      "cognitoAuthenticationType": null,
      "cognitoAuthenticationProvider": null,
      "userArn": null,
      "userAgent": "...",
      "user": null
    },
    "resourcePath": "/v1/posts",
    "httpMethod": "GET",
    "apiId": "..."
  },
  "body": null,
  "isBase64Encoded": false
}
```

The API automatically parses this information to create a normalized `REQUEST` object. The request can then be routed using the APIs methods.

## Simple Example

```javascript
const API = require('lambda-api') // API library

// Init API instance
const api = new API({ version: 'v1.0', base: 'v1' });

api.get('/test', function(req,res) {
  res.status(200).json({ status: 'ok' })
})

module.exports.handler = (event, context, callback) => {

  // Run the request
  api.run(event,context,callback);

} // end handler
```

## Configuration

Include the `lambda-api` module into your Lambda handler script and initialize an instance. You can initialize the API with an optional `version` which can be accessed via the `REQUEST` object and a `base` path. The base path can be used to route multiple versions to different instances.

```javascript
const API = require('lambda-api') // API library

// Init API instance with optional version and base path
const api = new API({ version: 'v1.0', base: 'v1' });
```

## Routes and HTTP Methods

Routes are defined by using convenience methods or the `METHOD` method. There are currently five convenience route methods: `get()`, `post()`, `put()`, `delete()` and `options()`. Convenience route methods require two parameters, a *route* and a function that accepts two arguments. A *route* is simply a path such as `/users`. The second parameter must be a function that accepts a `REQUEST` and a `RESPONSE` argument. These arguments can be named whatever you like, but convention dictates `req` and `res`. Examples using convenience route methods:

```javascript
api.get('/users', function(req,res) {
  // do something
})

api.post('/users', function(req,res) {
  // do something
})

api.delete('/users', function(req,res) {
  // do something
})
```
Additional methods are support by calling the `METHOD` method with three arguments. The first argument is the HTTP method, a *route*, and a function that accepts a `REQUEST` and a `RESPONSE` argument.

```javascript
api.METHOD('patch','/users', function(req,res) {
  // do something
})
```

## REQUEST

The `REQUEST` object contains a parsed and normalized request from API Gateway. It contains the following values by default:

- `version`: The version set at initialization
- `params`: Dynamic path parameters parsed from the path (see [path parameters](#path-parameters))
- `method`: The HTTP method of the request
- `path`: The path passed in by the request
- `query`: Querystring parameters parsed into an object
- `headers`: An object containing the request headers
- `body`: The body of the request.
 - If the `Content-Type` header is `application/json`, it will attempt to parse the request using `JSON.parse()`
 - If the `Content-Type` header is `application/x-www-form-urlencoded`, it will attempt to parse a URL encoded string using `querystring`
 - Otherwise it will be plain text.
- `route`: The matched route of the request

The request object can be used to pass additional information through the processing chain. For example, if you are using a piece of authentication middleware, you can add additional keys to the `REQUEST` object with information about the user. See [middleware](#middleware) for more information.

## RESPONSE

The `RESPONSE` object is used to send a response back to the API Gateway. The `RESPONSE` object contains several methods to manipulate responses. All methods are chainable unless they trigger a response.

### status
The `status` method allows you to set the status code that is returned to API Gateway. By default this will be set to `200` for normal requests or `500` on a thrown error. Additional built-in errors such as `404 Not Found` and `405 Method Not Allowed` may also be returned. The `status()` method accepts a single integer argument.

```javascript
api.get('/users', function(req,res) {
  res.status(401).error('Not Authorized')
})
```

### header
The `header` method allows for you to set additional headers to return to the client. By default, just the `Content-Type` header is sent with `application/json` as the value. Headers can be added or overwritten by calling the `header()` method with two string arguments. The first is the name of the header and then second is the value.

```javascript
api.get('/users', function(req,res) {
  res.header('Content-Type','text/html').send('<div>This is HTML</div>')
})
```

### send
The `send` methods triggers the API to return data to the API Gateway. The `send` method accepts one parameter and sends the contents through as is, e.g. as an object, string, integer, etc. AWS Gateway expects a string, so the data should be converted accordingly.

### json
There is a `json` convenience method for the `send` method that will set the headers to `application\json` as well as perform `JSON.stringify()` on the contents passed to it.

```javascript
api.get('/users', function(req,res) {
  res.json({ message: 'This will be converted automatically' })
})
```

### html
There is also an `html` convenience method for the `send` method that will set the headers to `text/html` and pass through the contents.

```javascript
api.get('/users', function(req,res) {
  res.html('<div>This is HTML</div>')
})
```

### error
An error can be triggered by calling the `error` method. This will cause the API to stop execution and return the message to the client. Custom error handling can be accomplished using the [Error Handling](#error-handling) feature.

```javascript
api.get('/users', function(req,res) {
  res.error('This is an error')
})
```

## Path Parameters
Path parameters are extracted from the path sent in by API Gateway. Although API Gateway supports path parameters, the API doesn't use these values but insteads extracts them from the actual path. This gives you more flexibility with the API Gateway configuration. Path parameters are defined in routes using a colon `:` as a prefix.

```javascript
api.get('/users/:userId', function(req,res) {
  res.send('User ID: ' + req.params.userId)
})
```

Path parameters act as wildcards that capture the value into the `params` object. The example above would match `/users/123` and `/users/test`. The system always looks for static paths first, so if you defined paths for `/users/test` and `/users/:userId`, exact path matches would take precedence. Path parameters only match the part of the path they are defined on. E.g. `/users/456/test` would not match `/users/:userId`. You would either need to define `/users/:userId/test` as its own path, or create another path with an additional path parameter, e.g. `/users/:userId/:anotherParam`.

A path can contain as many parameters as you want. E.g. `/users/:param1/:param2/:param3`.

## Wildcard Routes
Wildcard routes are supported for methods that match an existing route. E.g. `options` on an existing `get` route. As of now, the best use case is for the OPTIONS method to provide CORS headers. Wildcards only work in the base path. `/users/*`, for example, is not supported. For additional wildcard support, use [Path Parameters](#path-parameters) instead.

```javascript
api.options('/*', function(req,res) {
  // Do something
  res.status(200).send({});
})
```

## Middleware
The API supports middleware to preprocess requests before they execute their matching routes. Middleware is defined using the `use` method and require a function with three parameters for the `REQUEST`, `RESPONSE`, and `next` callback. For example:

```javascript
api.use(function(req,res,next) {
  // do something
  next()
})
```

Middleware can be used to authenticate requests, log API calls, etc. The `REQUEST` and `RESPONSE` objects behave as they do within routes, allowing you to manipulate either object. In the case of authentication, for example, you could verify a request and update the `REQUEST` with an `authorized` flag and continue execution. Or if the request couldn't be authorized, you could respond with an error directly from the middleware. For example:

```javascript
// Auth User
api.use(function(req,res,next) {
  if (req.headers.Authorization === 'some value') {
    req.authorized = true
    next() // continue execution
  } else {
    res.status(401).error('Not Authorized')
  }
})
```

The `next()` callback tells the system to continue executing. If this is not called then the system will hang and eventually timeout unless another request ending call such as `error` is called. You can define as many middleware functions as you want. They will execute serially and synchronously in the order in which they are defined.

## Error Handling
The API has simple built-in error handling that will log the error using `console.log`. These will be available via CloudWatch Logs. By default, errors will trigger a JSON response with the error message. If you would like to define additional error handling, you can define them using the `use` method similar to middleware. Error handling middleware must be defined as a function with **four** arguments instead of three like normal middleware. An additional `error` parameter must be added as the first parameter. This will contain the error object generated.

```javascript
api.use(function(err,req,res,next) {
  // do something with the error
  next()
})
```

The `next()` callback will cause the script to continue executing and eventually call the standard error handling function. You can short-circuit the default handler by calling a request ending method such as `send`, `html`, or `json`.

## Promises
The API uses Bluebird promises to manage asynchronous script execution. Additional methods such as `async / await` or simple callbacks should be supported. The API will wait for a request ending call before returning data back to the client. Middleware will wait for the `next()` callback before proceeding to the next step.

## CORS Support
CORS can be implemented using the [wildcard routes](#wildcard-routes) feature. A typical implementation would be as follows:

```javascript
api.options('/*', function(req,res) {
  // Add CORS headers
  res.header('Access-Control-Allow-Origin', '*');
  res.header('Access-Control-Allow-Methods', 'GET, PUT, POST, DELETE, OPTIONS');
  res.header('Access-Control-Allow-Headers', 'Content-Type, Authorization, Content-Length, X-Requested-With');
  res.status(200).send({});
})
```

Conditional route support could be added via middleware or with conditional logic within the `OPTIONS` route.

## Configuring Routes in API Gateway
Routes must be configured in API Gateway in order to support routing to the Lambda function. The easiest way to support all of your routes without recreating them is to use [API Gateway's Proxy Integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-set-up-simple-proxy.html#api-gateway-proxy-resource?icmpid=docs_apigateway_console).

Simply create one `{proxy+}` route that uses the `ANY` method and all requests will be routed to your Lambda function and processed by the `lambda-api` module.
