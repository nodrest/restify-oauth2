# Restify的OAuth 2终端

本包为[Restify][]框架提供一个*非常简单*的OAuth 2.0 终端. 特别是, 它只实现了[客户端凭证][cc]和[资源所有者密码认证][ropc] 流量.

## 你会得到什么

如果你提供Restify–OAuth2,并使用合适的钩子, 它将:

* 设置一个[令牌这终端][token endpoint], 将返回[访问令牌响应][token-endpoint-success]或者[正确格式错误回应][token-endpoint-error].
* 对于所有其他资源, 当一个访问令牌是[通过`Authorization`头发送][send-token], 它会验证它:
  * 如果令牌验证失败, 它将发送[恰当的400或401错误响应][token-usage-error], 并带有[`WWW-Authenticate`][www-authenticate]头和[`Link`][web-linking] [`rel="oauth2-token"`][oauth2-token-rel]
    头指向令牌终端.
  * 否则, 它将设置要么`req.clientId`要么`req.username` (依赖下面你所用的)为通过查找访问令牌确定的值.
* 如果没有访问令牌发送, 简单设置`req.clientId`/`req.username`为`null`:
  * 您可以检查这个只要有你想要保护的资源。
  * 如果用户试图访问受保护的资源，你可以使用Restify–OAuth2的`res.sendUnauthorized()`来发送合适401错误通过有帮助的 `WWW-Authenticate`和`Link`头.

## 使用和配置

使用Restify–OAuth2, 你需要传递给你的服务插件一些选项, 包括下面提及的钩子.Restify–OAuth2同时依赖内建的`authorizationParser` 和 `bodyParser` 插件, 后者 的`mapParams`参数设为`false`. 简言之, 如下:

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

传递给Restify–OAuth2的参数严重依赖你选择的两个流中的一个。一些选项在所有流中通用, 但是`options.hooks`哈希将取决于流变化. 一旦您提供相应的钩子, 你会免费得到一个OAuth 2实现.

### 客户端认证钩子

非常简单的OAuth 2流下面的想法是你的API客户端通过客户端IDs和客户段谜钥认证自身，并且如果这些值认证了，你赋予他们一个访问令牌,将来在请求中可以使用。比在每次请求时简单的基于访问认证头请求的好处是现在你可一设置这些令牌过期或者在它们落在了坏人的手中撤销他们如果。

安装Restify–OAuth2的客户端证书流到你的架构, 你需要提供它带以下钩子在`options.hooks`哈希里. 你可以在演示演示应用程序里查看一些[CC钩子实例][example CC hooks].

#### `grantClientToken(clientId, clientSecret, cb)`

检测API客户端是否认证可用，并有正确的密钥。如果这样它应该为那个客户端回调一个新的令牌，或者回调'false'如果证书是错误的。如果当验证凭据的时候发生内部服务错误它也可以回调一个错误。

#### `authenticateToken(token, cb)`

检测令牌是否有效，通过上一步`grantClientToken`赋予的。如果那样它应该为那个令牌回调带客户ID，或者'false'如果令牌失效。当查看令牌的时候如果发生服务器内部错误它也可以回调错误。

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
