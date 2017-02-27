# Koa2 文档翻译
# Index

## 安装
Koa需要Node4.0或以上版本（部分）支持ES2015特性。
*Koa2的安装参考其他笔记。*

### Async函数（通过Babel支持）

要在Koa中使用`async`，建议使用[babel's require hook](http://babeljs.io/docs/usage/require/)进行配置。
```js
require('babel-core/register');
// require the rest of the app that needs to be transpiled after the hook
const app = require('./app');
```
至少需要安装 [transform-async-to-generator](http://babeljs.io/docs/plugins/transform-async-to-generator/)和[transform-async-to-module-method](http://babeljs.io/docs/plugins/transform-async-to-module-method/) 中的一个插件，才能解析和转换async函数。
例如：在`.babekrc`文件中插入以下代码：
```json
{
  "plugins": ["transform-async-to-generator"]
}
```
你也可以使用[stage-3 preset](http://babeljs.io/docs/plugins/preset-stage-3/)。

## 应用

Koa应用是一种对象，它包含了一系列中间件函数，这些中间件像栈一样按照顺序依次执行。Koa和其他中间件系统（比如Ruby's Rack, Connect等等）很想相似。关键的不同是Koa提供了一种使用底层中间件来开发高级“sugar”的设计方法。这提升了兼容性和鲁棒性（稳健性），并且使得编写中间件变得更加有趣。

在这些中间件中，有负责内容协商（content-negotation）、缓存控制（cache freshness）、反向代理（proxy support）与重定向（redirection）等功能的常用中间件。尽管提供了如此多的方法，但是这些中间件并没有绑定在内核中，所以Koa依然十分小巧、轻量。

下面是一个简单的hello world 应用：
```js
const Koa = require('koa');
const app = new Koa();

app.use(ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```

## 级联

Koa中间件以一种非常传统的方式级联起来，你可能会非常熟悉这种写法(在类似的工具中使用过这种形式)，而这在以前使用Node的回调时非常麻烦。现在，使用异步函数（`async function`），我们可以写出“真正”的中间件。Connect实现中间件的方法是在一系列函数中传递控制权直到某个函数返回或程序结束。而Koa引用“顺流而下”（downstream）的形式，将控制权交给下一个中间件，最后（没有下一个中间件时）再将控制权逐级向上交回。

下面的这个示例程序响应结果为“Hello World”。请求发出后，请求先通过`x-response-time`中间件和`logging`中间件，然后继续将控制移交给响应的中间件。当一个中间件发出`next()`,当前函数将挂起，然后将控制传递给下一个中间件，直到后面没有需要执行的中间件，栈将释放，然后每个中间件继续运行，回溯地执行它的“上游”中间件。

```js
const Koa = require('koa');
const app = new Koa();

// x-response-time

app.use(async function (ctx, next) {
  const start = new Date();
  await next();
  const ms = new Date() - start;
  ctx.set('X-Response-Time', `${ms}ms`);
});

// logger

app.use(async function (ctx, next) {
  const start = new Date();
  await next();
  const ms = new Date() - start;
  console.log(`${ctx.method} ${ctx.url} - ${ms}`);
});

// response

app.use(ctx => {
  ctx.body = 'Hello World';
});

app.listen(3000);
```

## 设置

程序的设置是`app`实例的一些属性。现在支持以下属性：

-  `app.name`，可选。程序的名字。
-  `app.env`，默认设置为NODE_ENV或"development"。
-  `app.proxy`，值为"true"时，`proxy header`的参数将被加入信任列表。
-  `app.subdomainOffset`，忽略的`.subdomains`列

### app.listen(...)

Koa 程序不是一个程序对应一个HTTP server。一个或者多个Koa程序可以一起组成一个使用同一个HTTP sever的大型程序。

给`Server#listen()`传入指定的参数，从而创建一个HTTP服务器。这些参数可以在[nodejs](http://nodejs.org/api/http.html#http_server_listen_port_hostname_backlog_callback)中查到。下面这个例子在3000端口创建了一个没有任何功能的的Koa程序：

```js
const Koa = require('koa');
const app = new Koa();
app.listen(3000);
```

`app.listen(...)`方法实际上是这样实现的：

```js
const http = require('http');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000);
```

这意味着你可以在多个端口上启动一个 app，或者在一个程序中同时支持 HTTP 和 HTTPS：

```js
const http = require('http');
const Koa = require('koa');
const app = new Koa();
http.createServer(app.callback()).listen(3000);
http.createServer(app.callback()).listen(3001);
```

### app.callback()

返回一个回调函数来处理请求。你也可以使用这个回调函数把你的Koa程序放到一个Connect或者Express程序里。

##  app.use(function)

把中间件添加到这个程序中。查阅[Middleware](myWiki#middleware)节以了解更信息。

##  app.keys=

设置签名的cookie密钥。

这些参数被传递给KetGrip。你也可以传入你自己的`KeyGrip`实例：

```js
app.keys = ['im a newer secret', 'i like turtle'];
app.keys = new KeyGrip(['im a newer secret', 'i like turtle'], 'sha256');
```

签署cookie时，只有设置了`{ signed: true }`的密钥才可以使用：

```js
ctx.cookies.set('name', 'tobi', { signed: true });
```

###  app.context

 `app.context`是`ctx`的原型。你可以编辑`app.context`来给`ctx`添加属性。这种方法对于向`ctx`添加属性或方法是非常有效的，`ctx`贯穿整个应用，在给`ctx`添加许多依赖时，这样做能够提供更高的性能（因为没有中间件）和/或者易用性（`requre()`较少）。可以将其看做一种[反面模式](https://en.wikipedia.org/wiki/Anti-pattern)。
 例如，向你的数据库中添加一个引用：
 
 ```js
 app.context.db = db();

app.use(async (ctx) => {
  console.log(ctx.db);
});
```

注意：
- `ctx`的许多属性是通过getters、setttings和`Object.defineProperty()`定义的。如果想要修改这些属性（不推荐），必须使用` app.context`的`Object.defineProperty()`方法。参见：https://github.com/koajs/koa/issues/652。
- 挂载的app使用父级的`ctx`和设置。所以，这些加载的app其实就是中间件组。

## 错误处理

默认情况下程序输出所有错误到`stderr`中，除非`app.listen`的值为`true`。当`err.status`为`404`，或者`err.expose`为`true`时，默认的错误处理器不会输出错误。如果要自定义错误处理逻辑，如集中式日志记录，可以手动添加一个错误事件监听器：

```js
app.on('error', err =>
  log.error('server error', err)
);
```

如果请求-响应循环中发生了错误，程序无法对客户端作出响应，那么这个错误的`Context`实例也会传递进参数列表：

```js
app.on('error', (err, ctx) =>
  log.error('server error', err, ctx)
);
```

当发生错误且无法响应客户端请求时，或者说没有数据写进socket，Koa会返500 "Internal Server Error"。在任何一种情况下，应用级别的错误都会被记录下来。

# Context

Koa Context(上下文)将node的`request`和`response`封装为一个对象，为编写Web应用和API提供了许多实用的方法。这些操作在HTTP服务器开发中很常用，所以他们被添加到如此级别而不是更高级别的框架中，这将要求中间件重新实现这些普通的功能。

一个request创建一个`Context`，它在中间件中引用为接收器，或者是`ctx`，就如下面这段代码：

```js
app.use(async (ctx, next) => {
  ctx; // is the Context
  ctx.request; // is a koa Request
  ctx.response; // is a koa Response
});
```

为了方便使用，context的许多存取器和方法都委托给了`ctx.request`和`ctx.response`，他们是完全等同的。例如，`ctx.type`和`ctx.length`委托给了`response`，`ctx.path`和`ctx.method`委托给了`request`。

## API

Context的方法和存取器。

### ctx.req
node的`request`对象。

### ctx.res
node的`response`对象。

**不推荐**绕过Koa的response处理。避免使用下列`Node`属性：
* `res.statusCode`
* `res.writeHead()`
* `res.write()`
* `res.end()`

### ctx.request
Koa的`request`对象。

### ctx.response
Koa的`response`对象。

### ctx.state
koa推荐的命名空间：用以向中间件和前端视图中传递信息。

```js
ctx.state.user = await User.find(id);
```

###  ctx.app

应用实例。

### ctx.cookies.get(name, [options])

获取cookie--`name`。
- `options`：
  * `signed`  [boolean]：返回签名的cookie。
koa使用[cookies](https://github.com/jed/cookies)模块--参数将被传进此模块。

### ctx.cookies.set(name, value, [options])

设置名为`name`的cookie，其值为`value`.
- `options`:
  * `maxAge` ：从现在`Date.now()`开始的存活时间(milliseconds)
  * `signed` [boolean] ：签署cookie值
  * `expires` [date] ：cookie存活期限
  * `path` [string] ：cookie路径，默认为`/'`
  * `domain` [string] ：cookie域
  * `secure` [string] ：安全的cookie
  * `httpOnly` [boolean] ：是否可从服务器获取cookie，默认为ture
  * `overWrite` [boolean] ：是否覆盖先前设置的同名cookie，默认为false。如果设为true，在设置cookie时，同一请求设置的任何同名cookie(无论路径或域名是否相同)，都将过滤掉Set-Cookie头。

koa使用[cookies](https://github.com/jed/cookies)模块提供支持。

### ctx.throw([msg], [status], [properties])

辅助方法。抛出一个含有`.status`属性（默认为`500`）的错误，使Koa能够做出正确的响应。可以使用下列组合：

```js
ctx.throw(403);
ctx.throw('name required', 400);
ctx.throw(400, 'name required');
ctx.throw('something exploded');
```

例如，`ctx.throw('name required', 400)`等同于：

```js
const err = new Error('name required');
err.status = 400;
throw err;
```

注意：这些错误是用户级别的，标记了`err.expose`，这表示这些信息仅适用于客户端向服务器响应，它们并不适用于显示错误信息的场合，因为你不会想要展示错误细节。

你可以传入一个糅合了当前错误的`properties`对象。这样包装回传给上游请求的错误，可以提高机器可读性。

```js
ctx.throw(401, 'access_denied', { user: user });
ctx.throw('access_denied', { user: user });
```

koa使用中间件[`http-errors`](https://github.com/jshttp/http-errors)创建错误。

### ctx.assert(value, [status], [msg], [properties])

辅助方法。当`!value`时，抛出一个类似于`.throw()`的错误。和node的[assert()](https://nodejs.org/api/assert.html)方法相似。

```js
ctx.assert(ctx.state.user, 401, 'User not found. Please login!');
```

koa使用了[http-assert](https://github.com/jshttp/http-assert)。

### ctx.respond

当你想要改写原生的`res`对象，自己处理响应逻辑时，你可以设置`ctx.respond = false;`来绕过Koa内置的响应处理器。

但是Koa并不推荐这种做法。这可能会破坏Koa中间件和Koa本身的功能。使用这一属性将被视作hack，这一属性只是想给那些想要使用传统的`fn(req, res)`函数和Koa内部的中间件提供便利。

## Request 别名

下列存取器和同名的[Request](request.md)方法相同：

  - `ctx.header`
  - `ctx.headers`
  - `ctx.method`
  - `ctx.method=`
  - `ctx.url`
  - `ctx.url=`
  - `ctx.originalUrl`
  - `ctx.origin`
  - `ctx.href`
  - `ctx.path`
  - `ctx.path=`
  - `ctx.query`
  - `ctx.query=`
  - `ctx.querystring`
  - `ctx.querystring=`
  - `ctx.host`
  - `ctx.hostname`
  - `ctx.fresh`
  - `ctx.stale`
  - `ctx.socket`
  - `ctx.protocol`
  - `ctx.secure`
  - `ctx.ip`
  - `ctx.ips`
  - `ctx.subdomains`
  - `ctx.is()`
  - `ctx.accepts()`
  - `ctx.acceptsEncodings()`
  - `ctx.acceptsCharsets()`
  - `ctx.acceptsLanguages()`
  - `ctx.get()`

## Response aliases

  下列存取器和同名的 [Response](response.md) 方法相同:

  - `ctx.body`
  - `ctx.body=`
  - `ctx.status`
  - `ctx.status=`
  - `ctx.message`
  - `ctx.message=`
  - `ctx.length=`
  - `ctx.length`
  - `ctx.type=`
  - `ctx.type`
  - `ctx.headerSent`
  - `ctx.redirect()`
  - `ctx.attachment()`
  - `ctx.set()`
  - `ctx.append()`
  - `ctx.remove()`
  - `ctx.lastModified=`
  - `ctx.etag=`

  # Request

  Koa Request对象是对Node的顶层request对象的抽象，它提供了额外的功能，这些功能对于日常的HTTP开发十分有用。

  ## API

  ### request.header
  Request 头对象。

  ### request.headers
  Request 头对象。等同于`request.header`。

  ### request.method
  Request 方法。 

  ### request.method=
  设置请求头，方便实现类似`methodOverride()`的中间件。 

  ### reques.length
  返回请求内容的长度，返回值为数字；若请求内容不存在返回`undefined`。 

  ### request.url
  返回请求的URL。

  ### request.url=
  设置要请求的的URL，用于覆盖请求地址。

  ### request.originalUrl
  获取请求的初试URL。

  ### request.origin
  获取请求的URL，包含`protocol`(协议)和`host`(主机)。 

  ```js
  ctx.request.origin
  //返回 http://example.com
  ```

  ### request.href
  获取请求的完整URL，包括`protocol`(协议)、`host`(主机)和`url`。

  ```js
  ctx.request.href
  //返回 http://example.com/foo/bar?q=1
  ```

  ### request.path
  获取请求的路径。

  ### request.path=
  设置请求路径，如果存在查询字符串则保留。

  ### request.querystring
  获取原始查询字符串，去除了头部的`?`。

  ### request.querystring=
  设置原始查询字符串，不包含`?`。

  ### request.search
  返回包含`?`的原始查询字符串。

  ### request.search=
  设置查询字符串，包含`?`。

  ### request.host
  返回主机（主机名：端口）。当`app.proxy`= **true**时支持`X-Forwarded-Host`，否则返回当前使用的`Host`。

  ### request.hostname
  返回主机名。当`app.proxy`= **true**时支持`X-Forwarded-Host`，否则返回当前使用的`Host`。

  ### request.type
  返回请求的`Content-Type`，不包括类似"charset"的参数。

  ```js
  const ct = ctx.request.type
  // 返回 "image/png"
  ```

  ### request.charset
  返回请求使用的字符集参数，当参数不存在时返回`undefined`。

  ```js
  ctx.request.charset
  // 返回 "utf-8"
  ```

  ### request.query
  获取解析的查询字符串。没有查询字符串则返回一个空对象。注意：这个获取器**不支持**嵌套解析。

  例如，当 url 包含查询字符串 "color=blue&size=small" 时，返回如下：

  ```js
  {
  color: 'blue',
  size: 'small'
  }
  ```

  ### request.query=
  给指定的对象设置查询字符串。注意：这个设定方法不支持嵌套对象。

  ```js
  ctx.query = { next: '/login' }
  ```

  ### request.fresh
  检查请求的缓存是否是“新”的，或者说缓存内容是否变化过。这一方法用作`If-None-Match`和`ETag`, 或者`If-Modified-Since` 和 `Last-Modified`之间的缓存协商。它应该在设置了一个或多个此类响应头时引用。
  
  ```js
  // freshness check requires status 20x or 304
  // 检查是否为新需要 20x 或 304 状态
  ctx.status = 200;
  ctx.set('ETag', '123');

  // cache is ok 缓存检查正常
  if (ctx.fresh) {
    ctx.status = 304;
    return;
  }

  // cache is stale  缓存为最新
  // fetch new data  获取新数据
  ctx.body = yield db.find('something');
  ```

  ### request.stale
  与`request.fresh`相反。

  ### request.protocol
  返回请求使用的协议："https" 或者 "http"。当`app.proxy`设置为**true**时支持`X-Forwarded-Proto`。

  ### request.secure
  `ctx.protocol == "https"`的简写，检查一个请求是否通过TLS发出。

  ### request.ip
  请求远程地址。当`app.proxy`设置为**true**时支持`X-Forwarded-Proto`。

  ### request.ips
  当`X-Forwarded-For`存在且`app.proxy`启用时，返回一个由这些ip组成的array，其顺序从上层流到下层流。否则返回一个空队列。

  ### request.subdomains
  返回子域队列。

  子域是app的主机主域名前用点分隔的部分。app的域名默认为主机名的最后两部分。它可以通过`app.subdomainOffset`更改。
  例如，如果域名为"tobi.ferrets.example.com"，如果没有设置`app.subdomainOffset`，`ctx.subdomains`的返回结果为`["ferrets", "tobi"]`。如果`app.subdomainOffset`的值为3，`ctx.subdomains` 的返回结果为`["tobi"]`。

  ### request.is(types...)
  检查下一请求是否含有 "Content-Type" 头字段，以及是否含有任何给定的mime类型。如果没有request body，返回`null`。如果没有包含内容类型参数，或者匹配失败，则返回`false`。否则，返回匹配的内容类型。

  ```js
  // With Content-Type: text/html; charset=utf-8
  ctx.is('html'); // => 'html'
  ctx.is('text/html'); // => 'text/html'
  ctx.is('text/*', 'text/html'); // => 'text/html'

  // When Content-Type is application/json
  ctx.is('json', 'urlencoded'); // => 'json'
  ctx.is('application/json'); // => 'application/json'
  ctx.is('html', 'application/*'); // => 'application/json'

  ctx.is('html'); // => false
  ```

  如果你想要确保只把图像发送给路由，你可以这样处理：

  ```js
  if (ctx.is('image/*')) {
    // process
  } else {
    ctx.throw(415, 'images only!');
  }
  ```

  ## 内容协商

  Koa的`request`对象拥有许多有用的内容协商组件（由[accepts](http://github.com/expressjs/accepts)和[negotiator](https://github.com/federomero/negotiator)支持）。这些组件是：
  - `request.accepts(types)`
  - `request.acceptsEncodings(types)`
  - `request.acceptsCharsets(charsets)`
  - `request.acceptsLanguages(langs)`

  如果没有提供类型参数，返回所有可接受的类型。
  如果指定了复合类型，返回最佳的匹配。如果都不匹配，则返回`false`，你必须向客户端发送一个`406 "Not Acceptable"`的响应。
  如果丢失了accept header，返回第一个类型。因此，类型参数的顺序十分重要。

  ### request.accepts(types)
  检查请求对象是否为可接受的类型，如果是则返回最佳匹配类型，否则返回`false`。参数`type`的值可以为一个或多个mime类型字符串（如"application/json"），也可以是拓展名（如"json"），或者是类似`["json", "html", "text/plain"]`的列表。

  ```js
  // Accept: text/html
  ctx.accepts('html');
  // => "html"

  // Accept: text/*, application/json
  ctx.accepts('html');
  // => "html"
  ctx.accepts('text/html');
  // => "text/html"
  ctx.accepts('json', 'text');
  // => "json"
  ctx.accepts('application/json');
  // => "application/json"

  // Accept: text/*, application/json
  ctx.accepts('image/png');
  ctx.accepts('png');
  // => false

  // Accept: text/*;q=.5, application/json
  ctx.accepts(['html', 'json']);
  ctx.accepts('html', 'json');
  // => "json"

  // No Accept header
  ctx.accepts('html', 'json');
  // => "html"
  ctx.accepts('json', 'html');
  // => "json"
  ```

  可以多次调用`ctx.accepts()`，也可以使用switch方法：

  ```js
  switch (ctx.accepts('json', 'html', 'text')) {
  case 'json': break;
  case 'html': break;
  case 'text': break;
  default: ctx.throw(406, 'json, html, or text only');
  }
  ```

  ### request.acceptsEncodings(encodings)
  检查客户端是否接受给定的编码方式`encodings`，接受时返回最佳匹配否则返回`false`。注意：编码参数列表必须包含`identity`。

  ```js
  // Accept-Encoding: gzip
  ctx.acceptsEncodings('gzip', 'deflate', 'identity');
  // => "gzip"

  ctx.acceptsEncodings(['gzip', 'deflate', 'identity']);
  // => "gzip"
  ```

  如果没有给定参数，返回所有的可接受编码类型数组。

  ```js
  // Accept-Encoding: gzip, deflate
  ctx.acceptsEncodings();
  // => ["gzip", "deflate", "identity"]
  ```

  注意：如果客户端显式地发送`dentity;q=0`，那么`identity`编码（代表没有编码）可能不被接受。尽管这种情形很少见，最好在返回`false`时处理这种情况。

  ### request.acceptsCharsets(charsets)
  检查字符集`charsets`是否可接受，如果可以接受则返回最佳匹配的结果，否则返回`false`。

  ```js
  // Accept-Charset: utf-8, iso-8859-1;q=0.2, utf-7;q=0.5
  ctx.acceptsCharsets('utf-8', 'utf-7');
  // => "utf-8"

  ctx.acceptsCharsets(['utf-7', 'utf-8']);
  // => "utf-8"
  ```

  当没有指定参数时返回所有可接受的字符集数组。

  ```js
  // Accept-Charset: utf-8, iso-8859-1;q=0.2, utf-7;q=0.5
  ctx.acceptsCharsets();
  // => ["utf-8", "utf-7", "iso-8859-1"]
  ```

  ### request.acceptsLanguages(langs)
  检查语言`langs`是否可接受，如果可以接受则返回最佳匹配的结果，否则返回`false`。

  ```js
  // Accept-Language: en;q=0.8, es, pt
  ctx.acceptsLanguages('es', 'en');
  // => "es"

  ctx.acceptsLanguages(['en', 'es']);
  // => "es"
  ```
  当没有指定参数时返回所有可接受的语言数组。

  ```js
  // Accept-Language: en;q=0.8, es, pt
  ctx.acceptsLanguages();
  // => ["es", "pt", "en"]
  ```

  ### request.idempotent
  检查请求是否为幂等的。

  ### request.socket
  返回请求的socket。

  ### request.get(field)
  返回请求的header。

# Response
Koa的`response`对象是对node的response对象的抽象封装。

## API
*** 

### response.header
响应头对象。

### Response.headers
响应头对象。等同于`response.header`。

### response.socket
响应socket（套接字）。

### response.status
获取响应状态。`response.status`默认情况下没有设置。（这一点不同于`res.statusCode`，其默认设置为200）

### respons.status=
使用数字代码设置响应状态。

  - 100 "continue" 继续
  - 101 "switching protocols" 切换协议
  - 102 "processing" 处理中
  - 200 "ok" 
  - 201 "created" 已创建
  - 202 "accepted" 接受
  - 203 "non-authoritative information" 
  - 204 "no content"
  - 205 "reset content"
  - 206 "partial content"
  - 207 "multi-status"
  - 300 "multiple choices"
  - 301 "moved permanently"
  - 302 "moved temporarily"
  - 303 "see other"
  - 304 "not modified"
  - 305 "use proxy"
  - 307 "temporary redirect"
  - 400 "bad request"
  - 401 "unauthorized"
  - 402 "payment required"
  - 403 "forbidden"
  - 404 "not found"
  - 405 "method not allowed"
  - 406 "not acceptable"
  - 407 "proxy authentication required"
  - 408 "request time-out"
  - 409 "conflict"
  - 410 "gone"
  - 411 "length required"
  - 412 "precondition failed"
  - 413 "request entity too large"
  - 414 "request-uri too large"
  - 415 "unsupported media type"
  - 416 "requested range not satisfiable"
  - 417 "expectation failed"
  - 418 "i'm a teapot"
  - 422 "unprocessable entity"
  - 423 "locked"
  - 424 "failed dependency"
  - 425 "unordered collection"
  - 426 "upgrade required"
  - 428 "precondition required"
  - 429 "too many requests"
  - 431 "request header fields too large"
  - 500 "internal server error"
  - 501 "not implemented"
  - 502 "bad gateway"
  - 503 "service unavailable"
  - 504 "gateway time-out"
  - 505 "http version not supported"
  - 506 "variant also negotiates"
  - 507 "insufficient storage"
  - 509 "bandwidth limit exceeded"
  - 510 "not extended"
  - 511 "network authentication required"

__注意__:不必为记忆这些代码担心，如果你输入错误的代码，程序会抛出错误并显示此列表以便你进行更正。

### response.message
获取响应状态信息。默认情况下，`response.message`与`response.status`关联。

### response.message=
设置响应状态信息为给定的值。

### response.length=
设置响应的Content-Length为指定值。

### response.length
返回响应的Content-Length。当Content-Length存在时返回其值，否则通过`ctx.body`推测，或者返回`undefined`。

### response.body
获取response body。

### response.body=
设置response body，其内容可为以下类型：
- `string` 
- `Buffer` 
- `Stream`  管道流
- `Object`  序列化的字符串
- `null`  无内容

如果没有设置`response.status`，Koa会自动设置状态为200或204。

#### String
Content-Type将默认设置为 text/html 或者 text/plain，默认字符集是 utf-8，Content-Length 也将一并设置。

#### Buffer
Content-Type 将默认设置为 application/octet-stream，Content-Length 也将一并设置。

#### Stream
Content-Type 将默认设置为 application/octet-stream。  
无论何时，当流被设置为响应的主体时，`.onerror`都会被自动添加，来监听`error`事件，捕获错误。另外，无论什么时候，只要请求关闭（哪怕是提前的），流都会被销毁。如果你不需要这两个特性，不要直接将流设置为主体。例如，在代理中把一个HTTP流设置为主体，可能会导致底层连接的销毁。  
移步：[https://github.com/koajs/koa/pull/612](https://github.com/koajs/koa/pull/612) 查看更多信息。

下面是一个例子，该例中的流错误处理没有自动销毁流：

```js
const PassThrough = require('stream').PassThrough;

app.use(function * (next) {
  ctx.body = someHTTPStream.on('error', ctx.onerror).pipe(PassThrough());
});
```

#### Object
Content-Type 将默认设置为 application/json 注意：默认的json返回会添加空格，如果你希望压缩json返回中的空格，可以这样配置：`app.jsonSpaces = 0`。

### response.get(field)
获取响应的头信息中指定的`field`（大小写敏感）属性值。

```js
const etag = ctx.get('ETag');
```

### response.set(field, value)
设置响应头的`field`属性值为 `value`。

```js
ctx.set('Cache-Control', 'no-cache');
```

### response.append(field, value)
添加额外的头属性`field`，值为`value`。

```js
ctx.append('Link', '<http://127.0.0.1/>');
```

### response.set(fields)
使用对象来为响应头设置多项信息。

```js
ctx.set({
  'Etag': '1234',
  'Last-Modified': date
});
```

### response.remove(field)
移除头信息中的字段`field`。

### response.type
获取响应的`Content-Type`，去掉诸如 "charset" 的参数。

```js
const ct = ctx.type;
// => "image/png"
```

### response.type=
使用mime字符串或文件扩展名设置响应的`Content-Type`。

```js
ctx.type = 'text/plain; charset=utf-8';
ctx.type = 'image/png';
ctx.type = '.png';
ctx.type = 'png';
```

**注意**： 适当时，系统会为你选择一个字符集`charset`，例如header为`response.type = 'html'`时，字符集默认为“utf-8”。但是当明确定义类型全称时(如 `response.type = 'text/html'`)，不会指定字符集。

### response.is(types...)
和 `ctx.request.is()` 非常相似。检查响应的类型是否为参数中提供的类型。这个方法在创建操作响应之类的中间件时特别有用。

例如，下面这个中间件能够缩小除流外的所有HTML响应。

```js
const minify = require('html-minifier');

app.use(function * minifyHTML(next) {
  yield next;

  if (!ctx.response.is('html')) return;

  let body = ctx.body;
  if (!body || body.pipe) return;

  if (Buffer.isBuffer(body)) body = body.toString();
  ctx.body = minify(body);
});
```

### response.redirect(url, [alt])
返回`302`并跳转到指定的`url`。

关键字“back” 用来提供“refferrer”的功能(返回上一页)。当referrer不存在时，使用`alt` 或“/”代替。

```js
ctx.redirect('back');
ctx.redirect('back', '/index.html');
ctx.redirect('/login');
ctx.redirect('http://google.com');
```

若要修改默认状态302为其它值，只需在这个调用前或调用后附加状态码即可。若奥修改主题，将其附加在调用后：

```js
ctx.status = 301;
ctx.redirect('/cart');
ctx.body = 'Redirecting to shopping cart';
```

### response.attachment([filename])
设置 `Content-Disposition` 为"attachment"，向客户端发送下载指示。可以指定下载的文件名为`filename`。

### response.headerSent
检查响应头是否已经发送。在查看客户端是否接收到错误通知时十分有用。

### response.lastModified
如果头信息`Last-Modified`存在，将其作为日期`Date`返回。

### response.lastModified=
设置`Last-Modified` 属性为合适的UTC（协调世界时）字符串。也可以将其设置为`Date`或日期字符串。

```js
ctx.response.lastModified = new Date();
```

### response.etag=
设置响应的ETag并以`"`包围。注意：没有对应的`response.etag`获取器。

```js
ctx.response.etag = crypto.createHash('md5').update(ctx.body).digest('hex');
```

### response.vary(field)
在`field`字段的不同。

### response.flushHeaders()
刷新（立即/强制发送）所有设置的头信息headers，并开始主体body。