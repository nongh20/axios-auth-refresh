![Package version](https://img.shields.io/npm/v/axios-auth-refresh?label=version)
![Package size](https://img.shields.io/bundlephobia/min/axios-auth-refresh)
![Package downloads](https://img.shields.io/npm/dm/axios-auth-refresh)
![Package types definitions](https://img.shields.io/npm/types/axios-auth-refresh)

# axios-auth-refresh
Library that helps you implement automatic refresh of authorization
via axios [interceptors](https://github.com/axios/axios#interceptors).
You can easily intercept the original request when it fails, refresh the authorization and continue with the original request,
without any user interaction.

What happens when the request fails dur to authorization is all up to you.
You can either run a refresh call for a new authorization token or run a custom logic. 

The plugin stalls additional requests that have come in while waiting for a new authorization token
and resolves them when a new token is available.

## Installation

Using [npm](https://www.npmjs.com/get-npm) or [yarn](https://yarnpkg.com/en/docs/install):

```bash
npm install axios-auth-refresh --save
# or
yarn add axios-auth-refresh
```

## Syntax

```typescript
createAuthRefreshInterceptor(
    axios: AxiosStatic,
    refreshAuthLogic: (failedRequest: any) => Promise<any>,
    options?: AxiosAuthRefreshOptions = {}
): number;
```

#### Parameters
- `axios` - an instance of Axios
- `refreshAuthLogic` - a Function used for refreshing authorization (**must return a promise**).
Accepts exactly one parameter, which is the `failedRequest` returned by the original call.
- `options` - object with settings for interceptor (See [available options](#available-options))

#### Returns
Interceptor `id` in case you want to reject it manually.

## Usage

In order to activate the interceptors, you need to import a function from `axios-auth-refresh`
which is *exported by default* and call it with the **axios instance** you want the interceptors for, 
as well as the **refresh authorization function** where you need to write the logic for refreshing the authorization.

The interceptors will then be bound onto the axios instance and the specified logic will be ran whenever a [401 (Unauthorized)](https://httpstatuses.com/401) status code 
is returned from a server (or any other status code you provide in options). All the new requests created while the refreshAuthLogic has been processing will be bound onto the 
Promise returned from the refreshAuthLogic function. This means that the requests will be resolved when a new access token has been fetched or when the refreshing logic faleid.

```javascript
import axios from 'axios';
import createAuthRefreshInterceptor from 'axios-auth-refresh';

// Function that will be called to refresh authorization
const refreshAuthLogic = failedRequest => axios.post('https://www.example.com/auth/token/refresh').then(tokenRefreshResponse => {
    localStorage.setItem('token', tokenRefreshResponse.data.token);
    failedRequest.response.config.headers['Authorization'] = 'Bearer ' + tokenRefreshResponse.data.token;
    return Promise.resolve();
});

// Instantiate the interceptor (you can chain it as it returns the axios instance)
createAuthRefreshInterceptor(axios, refreshAuthLogic);

// Make a call. If it returns a 401 error, the refreshAuthLogic will be run, 
// and the request retried with the new token
axios.get('https://www.example.com/restricted/area')
    .then(/* ... */)
    .catch(/* ... */);
```

#### Skipping the interceptor (BETA)
There's a possibility to skip the logic of the interceptor for specific calls.
To do this, you need to pass the `skipAuthRefresh` option to the request config for each request you don't want to intercept.
```javascript
axios.get('https://www.example.com/', { skipAuthRefresh: true });
```

If you're using typescript you can import the custom request config interface from `axios-auth-refresh`.
```typescript
import { AxiosAuthRefreshRequestConfig } from 'axios-auth-refresh';
```

#### Request interceptor
Since this plugin automatically stalls additional requests while refreshing the token,
it is a good idea to **wrap your request logic in a function**, 
to make sure the stalled requests are using the newly fetched data (like token).

Example of sending the tokens:
```javascript
// Obtain the fresh token each time the function is called
function getAccessToken(){
    return localStorage.getItem('token');
}

// Use interceptor to inject the token to requests
axios.interceptors.request.use(request => {
    request.headers['Authorization'] = `Bearer ${getAccessToken()}`;
    return request;
});
```

## Available options

#### Status codes to intercept 

You can specify multiple status codes that you want the interceptor to run for.

```javascript
{
    statusCodes: [ 401 ] // default
}
```

#### Unpause the instance while refreshing

While your refresh logic is ran, the instance is marked as "to-be-skipped"
in order to prevent the "interceptors loop" when refreshing causes one of the statuses specified
in `options.statusCodes`. If that's behavior is not wanted, you can set the `skipWhileRefreshing` option to false,
but keep in mind that you need to implement skipping the requests by yourself using `skipAuthRefresh` flag
in request's configuration

```javascript
{
    skipWhileRefreshing: false // default: true
}
```

#### Want to help me?
Check out [contribution guide](CONTRIBUTING.md) or my [patreon page!](https://www.patreon.com/dawidzbinski)
