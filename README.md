# OAuth 2 Endpoints for Restify

本报为[Restify][]框架提供一个*非常简单*的OAuth 2.0 终端. 特别是, 它只实现了[客户端凭证][cc]和[资源所有者密码认证][ropc] 流量.

## 你会得到什么

If you provide Restify–OAuth2 with the appropriate hooks, it will:

* Set up a [token endpoint][], which returns [access token responses][token-endpoint-success] or
  [correctly-formatted error responses][token-endpoint-error].
* For all other resources, when an access token is [sent via the `Authorization` header][send-token], it will validate
  it:
  * If the token fails validation, it will send [an appropriate 400 or 401 error response][token-usage-error], with a
    [`WWW-Authenticate`][www-authenticate] header and a [`Link`][web-linking] [`rel="oauth2-token"`][oauth2-token-rel]
    header pointing to the token endpoint.
  * Otherwise, it will set either `req.clientId` or `req.username` (depending on which flow you are using) to a value
    determined by looking up the access token.
* If no access token is sent, it simply sets `req.clientId`/`req.username` to `null`:
  * You can check for this whenever there is a resource you want to protect.
  * If the user tries to access a protected resource, you can use Restify–OAuth2's `res.sendUnauthorized()` to send
    appropriate 401 errors with helpful `WWW-Authenticate` and `Link` headers.

## 使用和配置

To use Restify–OAuth2, you'll need to pass it your server plus some options, including the hooks discussed below.
Restify–OAuth2 also depends on the built-in `authorizationParser` and `bodyParser` plugins, the latter with `mapParams`
set to `false`. So in short, it looks like this:

```js
var restify = require("restify");
var restifyOAuth2 = require("restify-oauth2");

var server = restify.createServer({ name: "My cool server", version: "1.0.0" });
server.use(restify.authorizationParser());
server.use(restify.bodyParser({ mapParams: false }));

restifyOAuth2.cc(server, options);
// or
restifyOAuth2.ropc(server, options);
```

不幸的是, Restify–OAuth2不能成为一个简单的Restify插件. 它需要为令牌终端安装一个路由,而插件只是在每次请求时运行，不修改服务器的路由表.

## 选项

The options you pass to Restify–OAuth2 depend heavily on which of the two flows you are choosing. There are some
options common to both flows, but the `options.hooks` hash will vary depending on the flow. Once you provide the
appropriate hooks, you get an OAuth 2 implementation for free.

### 客户端认证挂钩

The idea behind this very simple OAuth 2 flow is that your API clients identify themselves with client IDs and secrets,
and if those values authenticate, you grant them an access token they can use for further requests. The advantage of
this over simply requiring basic access authentication headers on every request is that now you can set those tokens to
expire, or revoke them if they fall in to the wrong hands.

To install Restify–OAuth2's client credentials flow into your infrastructure, you will need to provide it with the
following hooks in the `options.hooks` hash. You can see some [CC钩子实例][example CC hooks] in the demo application.

#### `grantClientToken(clientId, clientSecret, cb)`

Checks that the API client is authorized to use your API, and has the correct secret. It should call back with a new
token for that client if so, or `false` if the credentials are incorrect. It can also call back with an error if there
was some internal server error while validating the credentials.

#### `authenticateToken(token, cb)`

Checks that a token is valid, i.e. that it was granted in the past by `grantClientToken`. It should call back with the
client ID for that token if so, or `false` if the token is invalid. It can also call back with an error if there
was some internal server error while looking up the token.

### 资源所有者密码认证挂钩

The idea behind this OAuth 2 flow is that your API clients will prompt the user for their username and password, and
send those to your API in exchange for an access token. This has some advantages over simply sending the user's
credentials to the server directly. For example, it obviates the need for the client to store the credentials, and
allows expiration and revocation of tokens. However, it does imply that you trust your API clients, since they will
have at least one-time access to the user's credentials.

To install Restify–OAuth2's resource owner password credentials flow into your infrastructure, you will need to
provide it with the following hooks in the `options.hooks` hash. You can see some [ROPC钩子实例][example ROPC hooks] in the demo
application.

#### `validateClient(clientId, clientSecret, cb)`

Checks that the API client is authorized to use your API, and has the correct secret. It should call back with `true`
or `false` depending on the result of the check. It can also call back with an error if there was some internal server
error while doing the check.

#### `grantUserToken(username, password, cb)`

Checks that the API client is authenticating on behalf of a real user with correct credentials. It should call back
with a new token for that user if so, or `false` if the credentials are incorrect. It can also call back with an error
if there was some internal server error while validating the credentials.

#### `authenticateToken(token, cb)`

检测令牌是否有效, i.e. that it was granted in the past by `grantUserToken`. It should call back with the
username for that token if so, or `false` if the token is invalid. It can also call back with an error if there
was some internal server error while looking up the token.

### 其他操作

The `hooks` hash is the only required option, but the following are also available for tweaking:

* `tokenEndpoint`: the location at which the token endpoint should be created. 默认是`"/token"`.
* `wwwAuthenticateRealm`: the value of the "Realm" challenge in the `WWW-Authenticate` header. Defaults to
  `"Who goes there?"`.
* `tokenExpirationTime`: the value returned for the `expires_in` component of the response from the token endpoint.
  Note that this is *only* the value reported; you are responsible for keeping track of token expiration yourself and
  calling back with `false` from `authenticateToken` when the token expires. Defaults to `Infinity`.

## 看起来像什么？

OK, let's try something a bit more concrete. If you check out the [example servers][] used in the integration tests,
you'll see our setup. Here we'll walk you through the more complicated resource owner password credentials example,
but the idea for the client credentials example is very similar.

## /

最初的资源，在人们进入服务器。

* If a valid token is supplied in the `Authorization` header, `req.username` is truthy, and the app responds with
  links to `/public` and `/secret`.
* If no token is supplied, the app responds with links to `/token` and `/public`.
* If an invalid token is supplied, Restify–OAuth2 intercepts the request before it gets to the application, and sends
  an appropriate 400 or 401 error.

## /token

令牌终端, 完全由Restify-OAuth2管理。 为给定的client ID/client secret/username/password组合生成令牌.

The client validation and token-generation logic is provided by the application, but none of the ceremony necessary for
OAuth 2 conformance, error handling, etc. is present in the application code: Restify–OAuth2 takes care of all of that.

## /public

任何人能访问的全局资源.

* If a valid token is supplied in the Authorization header, `req.username` contains the username, and the app uses
  that to send a personalized response.
* If no token is supplied, `req.username` is `null`. The app still sends a response, just without personalizing.
* If an invalid token is supplied, Restify–OAuth2 intercepts the request before it gets to the application, and sends
  an appropriate 400 or 401 error.

## /secret

认证用户访问的加密资源.

* If a valid token is supplied in the Authorization header, `req.username` is truthy, and the app sends the secret
  data.
* If no token is supplied, `req.username` is `null`, so the application uses `res.sendUnauthorized()` to send a nice
  401 error with `WWW-Authenticate` and `Link` headers.
* If an invalid token is supplied, Restify–OAuth2 intercepts the request before it gets to the application, and sends
  an appropriate 400 or 401 error.

[Restify]: http://mcavage.github.com/node-restify/
[cc]: http://tools.ietf.org/html/rfc6749#section-1.3.4
[ropc]: http://tools.ietf.org/html/rfc6749#section-1.3.3
[token endpoint]: http://tools.ietf.org/html/rfc6749#section-3.2
[token-endpoint-success]: http://tools.ietf.org/html/rfc6749#section-5.1
[token-endpoint-error]: http://tools.ietf.org/html/rfc6749#section-5.2
[send-token]: http://tools.ietf.org/html/rfc6750#section-2.1
[token-usage-error]: http://tools.ietf.org/html/rfc6750#section-3.1
[oauth2-token-rel]: http://tools.ietf.org/html/draft-wmills-oauth-lrdd-07#section-3.2
[web-linking]: http://tools.ietf.org/html/rfc5988
[www-authenticate]: http://tools.ietf.org/html/rfc2617#section-3.2.1
[example ROPC hooks]: https://github.com/domenic/restify-oauth2/blob/master/examples/ropc/hooks.js
[example CC hooks]: https://github.com/domenic/restify-oauth2/blob/master/examples/cc/hooks.js
[example servers]: https://github.com/domenic/restify-oauth2/tree/master/examples
