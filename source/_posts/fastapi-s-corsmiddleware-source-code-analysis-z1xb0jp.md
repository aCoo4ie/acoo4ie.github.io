---
title: FastAPI的CORSMiddleware源码分析
date: '2024-06-24 10:24:38'
updated: '2024-06-24 15:50:21'
tags:
  - FastAPI
categories:
  - 源码分析
permalink: /post/fastapi-s-corsmiddleware-source-code-analysis-z1xb0jp.html
comments: true
toc: true
---

# FastAPI的CORSMiddleware源码分析

```python
# starlette/middleware/cors.py

import functools
import re
import typing

from starlette.datastructures import Headers, MutableHeaders
from starlette.responses import PlainTextResponse, Response
from starlette.types import ASGIApp, Message, Receive, Scope, Send

ALL_METHODS = ("DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT")
SAFELISTED_HEADERS = {"Accept", "Accept-Language", "Content-Language", "Content-Type"}


class CORSMiddleware:
    def __init__(
        self,
        app: ASGIApp,
        allow_origins: typing.Sequence[str] = (),
        allow_methods: typing.Sequence[str] = ("GET",),
        allow_headers: typing.Sequence[str] = (),
        allow_credentials: bool = False,
        allow_origin_regex: typing.Optional[str] = None,
        expose_headers: typing.Sequence[str] = (),
        max_age: int = 600,
    ) -> None:
        if "*" in allow_methods:
            allow_methods = ALL_METHODS

        compiled_allow_origin_regex = None
        if allow_origin_regex is not None:
            compiled_allow_origin_regex = re.compile(allow_origin_regex)

        allow_all_origins = "*" in allow_origins
        allow_all_headers = "*" in allow_headers
        preflight_explicit_allow_origin = not allow_all_origins or allow_credentials

        simple_headers = {}
        if allow_all_origins:
            simple_headers["Access-Control-Allow-Origin"] = "*"
        if allow_credentials:
            simple_headers["Access-Control-Allow-Credentials"] = "true"
        if expose_headers:
            simple_headers["Access-Control-Expose-Headers"] = ", ".join(expose_headers)

        preflight_headers = {}
        if preflight_explicit_allow_origin:
            # The origin value will be set in preflight_response() if it is allowed.
            preflight_headers["Vary"] = "Origin"
        else:
            preflight_headers["Access-Control-Allow-Origin"] = "*"
        preflight_headers.update(
            {
                "Access-Control-Allow-Methods": ", ".join(allow_methods),
                "Access-Control-Max-Age": str(max_age),
            }
        )
        allow_headers = sorted(SAFELISTED_HEADERS | set(allow_headers))
        if allow_headers and not allow_all_headers:
            preflight_headers["Access-Control-Allow-Headers"] = ", ".join(allow_headers)
        if allow_credentials:
            preflight_headers["Access-Control-Allow-Credentials"] = "true"

        self.app = app
        self.allow_origins = allow_origins
        self.allow_methods = allow_methods
        self.allow_headers = [h.lower() for h in allow_headers]
        self.allow_all_origins = allow_all_origins
        self.allow_all_headers = allow_all_headers
        self.preflight_explicit_allow_origin = preflight_explicit_allow_origin
        self.allow_origin_regex = compiled_allow_origin_regex
        self.simple_headers = simple_headers
        self.preflight_headers = preflight_headers

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http":  # pragma: no cover
            await self.app(scope, receive, send)
            return

        method = scope["method"]
        headers = Headers(scope=scope)
        origin = headers.get("origin")

        if origin is None:
            await self.app(scope, receive, send)
            return

        if method == "OPTIONS" and "access-control-request-method" in headers:
            response = self.preflight_response(request_headers=headers)
            await response(scope, receive, send)
            return

        await self.simple_response(scope, receive, send, request_headers=headers)

    def is_allowed_origin(self, origin: str) -> bool:
        if self.allow_all_origins:
            return True

        if self.allow_origin_regex is not None and self.allow_origin_regex.fullmatch(
            origin
        ):
            return True

        return origin in self.allow_origins

    def preflight_response(self, request_headers: Headers) -> Response:
        requested_origin = request_headers["origin"]
        requested_method = request_headers["access-control-request-method"]
        requested_headers = request_headers.get("access-control-request-headers")

        headers = dict(self.preflight_headers)
        failures = []

        if self.is_allowed_origin(origin=requested_origin):
            if self.preflight_explicit_allow_origin:
                # The "else" case is already accounted for in self.preflight_headers
                # and the value would be "*".
                headers["Access-Control-Allow-Origin"] = requested_origin
        else:
            failures.append("origin")

        if requested_method not in self.allow_methods:
            failures.append("method")

        # If we allow all headers, then we have to mirror back any requested
        # headers in the response.
        if self.allow_all_headers and requested_headers is not None:
            headers["Access-Control-Allow-Headers"] = requested_headers
        elif requested_headers is not None:
            for header in [h.lower() for h in requested_headers.split(",")]:
                if header.strip() not in self.allow_headers:
                    failures.append("headers")
                    break

        # We don't strictly need to use 400 responses here, since its up to
        # the browser to enforce the CORS policy, but its more informative
        # if we do.
        if failures:
            failure_text = "Disallowed CORS " + ", ".join(failures)
            return PlainTextResponse(failure_text, status_code=400, headers=headers)

        return PlainTextResponse("OK", status_code=200, headers=headers)

    async def simple_response(
        self, scope: Scope, receive: Receive, send: Send, request_headers: Headers
    ) -> None:
        send = functools.partial(self.send, send=send, request_headers=request_headers)
        await self.app(scope, receive, send)

    async def send(
        self, message: Message, send: Send, request_headers: Headers
    ) -> None:
        if message["type"] != "http.response.start":
            await send(message)
            return

        message.setdefault("headers", [])
        headers = MutableHeaders(scope=message)
        headers.update(self.simple_headers)
        origin = request_headers["Origin"]
        has_cookie = "cookie" in request_headers

        # If request includes any cookie headers, then we must respond
        # with the specific origin instead of '*'.
        if self.allow_all_origins and has_cookie:
            self.allow_explicit_origin(headers, origin)

        # If we only allow specific origins, then we have to mirror back
        # the Origin header in the response.
        elif not self.allow_all_origins and self.is_allowed_origin(origin=origin):
            self.allow_explicit_origin(headers, origin)

        await send(message)

    @staticmethod
    def allow_explicit_origin(headers: MutableHeaders, origin: str) -> None:
        headers["Access-Control-Allow-Origin"] = origin
        headers.add_vary_header("Origin")
```

CORS 是一种浏览器安全特性，用于防止一个网站上的恶意脚本访问另一个网站上的敏感信息。在现代 Web 应用开发中，正确处理 CORS 是很重要的。

### 1. 常量定义

```Python
ALL_METHODS = ("DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT")
SAFELISTED_HEADERS = {"Accept", "Accept-Language", "Content-Language", "Content-Type"}
```

* ​`ALL_METHODS`​：定义了 HTTP 的所有请求方法。
* ​`SAFELISTED_HEADERS`​：定义了安全的、默认被允许的请求头字段。

### 2. CORSMiddleware 类定义

```Python
class CORSMiddleware:
    def __init__(
        self,
        app: ASGIApp,
        allow_origins: typing.Sequence[str] = (),
        allow_methods: typing.Sequence[str] = ("GET",),
        allow_headers: typing.Sequence[str] = (),
        allow_credentials: bool = False,
        allow_origin_regex: typing.Optional[str] = None,
        expose_headers: typing.Sequence[str] = (),
        max_age: int = 600,
    ) -> None:
```

定义了中间件的初始化方法，接受的参数包括：

* ​`app`​：ASGI 应用实例。
* ​`allow_origins`​：允许的来源（Origin）。
* ​`allow_methods`​：允许的方法。
* ​`allow_headers`​：允许的请求头。
* ​`allow_credentials`​：是否允许凭据。
* ​`allow_origin_regex`​：允许的来源正则表达式。
* ​`expose_headers`​：暴露的响应头。
* ​`max_age`​：预检请求结果的缓存时间。

#### 2.1 为什么allow_header是必须的？

​`allow_headers`​ 参数定义了在跨域请求中允许使用的 HTTP 请求头。对于 `OPTIONS`​ 预检请求，浏览器会发送 `Access-Control-Request-Headers`​，表示实际请求中会使用的自定义请求头。服务器必须明确允许这些请求头，才能使实际请求成功。

* **安全性**：限制允许的请求头可以防止客户端发送潜在危险的信息，如认证令牌、敏感数据等，只有预先批准的头部字段才能发送。
* **兼容性**：某些跨域请求使用了自定义头部，这些头部可能是为了携带认证信息、内容类型或其他控制信息。指定哪些头部是允许的，确保了这些请求能够顺利通过。

#### 2.2 为什么allow_credentials是必须的？

​`allow_credentials`​ 参数决定了是否允许跨域请求携带凭据（例如 Cookies、授权头部等）。当应用需要用户认证信息时，这个参数特别重要。

* **安全性**：

  * 默认情况下，跨域请求不会携带凭据。
  * 允许凭据必须搭配具体的 `Access-Control-Allow-Origin`​ 头部（不能是 `*`​），从而避免信息泄露到所有来源网站。
* **功能性**：在某些身份验证机制中，如基于 Cookies 的会话管理，必须允许跨域请求带上 Cookies，以便于验证用户。

在 CORS 中，浏览器会根据 `allow_headers`​ 和 `allow_credentials`​ 的响应头来决定是否允许实际跨域请求：

1. **预检请求** (`OPTIONS`​ 方法):

    * 浏览器使用 `Access-Control-Request-Headers`​ 列出实际请求将使用的头部。
    * 服务器需要响应 `Access-Control-Allow-Headers`​ 指定允许的头部。
    * 如果 `Access-Control-Allow-Headers`​ 不包含预检请求的头部，浏览器将禁止该跨域请求。
2. **实际请求**:

    * 如果指定了 `allow_credentials`​ (即 `Access-Control-Allow-Credentials: true`​)，浏览器会携带凭据如 Cookies。
    * 服务器响应时，如需处理带凭据的请求，必须返回具体的 `Access-Control-Allow-Origin`​ 头部，而不能是 `*`​。

简单理解就是，当一个网站要向另一个网站发送信息时，需要提前问一下："我能用这些信息吗？" 这个过程叫“预检请求”（相当于问可以不可以）。如果服务器说 "可以"，那么浏览器就会允许发送这个信息。如果服务器说 "不可以"，那么浏览器就会阻止发送。并且访问一个需要登录的网站时，网站需要知道你是不是已经登录，就必须带上这个“登录凭据”（相当于身份证明）。但是，为了安全，带登录凭据的请求必须很小心，不能随便允许所有网站都带这些信息。

1. **预检请求**：

    * 浏览器：服务器，可以带上这些信息吗？（例如：一些特殊的消息头）
    * 服务器：可以带上这些信息噢。
2. **实际请求**：

    * 浏览器：那我就带着这些信息（比如说登录信息）发请求了。
    * 服务器：好的，我允许你这样做。

#### 2.4 为什么expose_headers是必须的？

暴露的消息头exose_headers是指当浏览器收到服务器的响应时，可以读取哪些消息头。**默认情况下，浏览器只能访问一些基本的消息头，比如** **​`Content-Type`​**​ **。**

有时候，服务器会在响应中添加一些自定义的消息头，比如：

* ​`X-Custom-Header`​：表示服务器为此请求进行了一些特殊处理。
* ​`Link`​：提供有关资源的进一步信息。

如果浏览器不能访问这些自定义消息头，那么这些信息对前端开发者来说就没有用。

下面的代码告诉服务器，它可以允许浏览器读取哪些自定义的消息头：

```Python
if expose_headers:
    simple_headers["Access-Control-Expose-Headers"] = ", ".join(expose_headers)
```

简单理解就是，当你在写一封信，信封的内容是一般的人都能看到的，但信的边角上附了一些小纸条，这些纸条上有一些额外的信息。我们需要明确告诉收信的人："嘿，这些小纸条的信息也是你可以读的"。

#### 2.5 为什么max_age是必须的？

最大存活时间`max_age`​ 是一个时间值，用来告诉浏览器在多长时间内不需要重复进行预检请求。

1. **减少预检请求**

    每当浏览器向一个不同的网站发送跨域请求时，它会先发送一个预检请求（OPTIONS 请求），问服务器是否允许该请求。这是为了安全。

    如果没有 `max_age`​，浏览器每次发送请求前都要进行这一步。这会造成很多额外的通信，增加了延迟和服务器负担。
2. **提高性能**

    ​`max_age`​ 告诉浏览器在指定时间内，不需要重复进行预检请求。比如 `max_age`​ 设置为 600 秒（10 分钟），那么在这 10 分钟内，浏览器只要有相同类型的请求，就不需要再进行预检。这大大提升了性能，减少了延迟。

下面的代码就实现了设置浏览器在多长时间内可以缓存预检请求的结果：

```Python
preflight_headers["Access-Control-Max-Age"] = str(max_age)
```

简单理解就是，想象你在进游乐园时，保安要检查你的包。如果没有 `max_age`​，每次你进园子前都要检查一次。但如果有了 `max_age`​，它就像一张快速通行证，在一定时间内（比如 1 小时）你就不用每次都检查，让你可以更快地进出。

### 3. 参数处理

```Python
if "*" in allow_methods:
    allow_methods = ALL_METHODS

compiled_allow_origin_regex = None
if allow_origin_regex is not None:
    compiled_allow_origin_regex = re.compile(allow_origin_regex)

allow_all_origins = "*" in allow_origins
allow_all_headers = "*" in allow_headers
	preflight_explicit_allow_origin = not allow_all_origins or allow_credentials
```

* 将 `"*"`​ 替换成所有的方法。
* 编译 `allow_origin_regex`​ 成正则表达式。
* 判断是否允许所有来源和头部。
* 确定是否需要显式允许 `Origin`​（因为带凭据的请求不能使用 `"*"`​）。

### 4. 生成默认的预检响应头和简单响应头

```Python
simple_headers = {}
if allow_all_origins:
    simple_headers["Access-Control-Allow-Origin"] = "*"
if allow_credentials:
    simple_headers["Access-Control-Allow-Credentials"] = "true"
if expose_headers:
    simple_headers["Access-Control-Expose-Headers"] = ", ".join(expose_headers)

preflight_headers = {}
if preflight_explicit_allow_origin:
    preflight_headers["Vary"] = "Origin"
else:
    preflight_headers["Access-Control-Allow-Origin"] = "*"
preflight_headers.update(
    {
        "Access-Control-Allow-Methods": ", ".join(allow_methods),
        "Access-Control-Max-Age": str(max_age),
    }
)
allow_headers = sorted(SAFELISTED_HEADERS | set(allow_headers))
if allow_headers and not allow_all_headers:
    preflight_headers["Access-Control-Allow-Headers"] = ", ".join(allow_headers)
if allow_credentials:
    preflight_headers["Access-Control-Allow-Credentials"] = "true"

self.app = app
self.allow_origins = allow_origins
self.allow_methods = allow_methods
self.allow_headers = [h.lower() for h in allow_headers]
self.allow_all_origins = allow_all_origins
self.allow_all_headers = allow_all_headers
self.preflight_explicit_allow_origin = preflight_explicit_allow_origin
self.allow_origin_regex = compiled_allow_origin_regex
self.simple_headers = simple_headers
self.preflight_headers = preflight_headers
```

* ​`simple_headers`​：用于简单请求（即不是预检请求）的默认响应头。
* ​`preflight_headers`​：用于预检请求的默认响应头。
* 初始化类属性。

### 5. 请求处理方法

```Python
async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
    if scope["type"] != "http":
        await self.app(scope, receive, send)
        return

    method = scope["method"]
    headers = Headers(scope=scope)
    origin = headers.get("origin")

    if origin is None:
        await self.app(scope, receive, send)
        return

    if method == "OPTIONS" and "access-control-request-method" in headers:
        response = self.preflight_response(request_headers=headers)
        await response(scope, receive, send)
        return

    await self.simple_response(scope, receive, send, request_headers=headers)
```

* 判断请求类型是否为 HTTP。
* 获取请求头中的 `Origin`​ 字段，若没有 `Origin`​ 字段则放行请求。如果没有 `Origin`​ 字段，通常意味着这是一个同源请求或是由服务器端发起的请求，这类请求通常不涉及跨域问题。因此，默认可以放行这些请求。
* 先处理 `OPTIONS`​ 请求（预检请求）。
* 再处理简单请求。

### 6. 检查是否允许 `Origin`​

```Python
def is_allowed_origin(self, origin: str) -> bool:
    if self.allow_all_origins:
        return True

    if self.allow_origin_regex is not None and self.allow_origin_regex.fullmatch(
        origin
    ):
        return True

    return origin in self.allow_origins
```

* 首先检查是否允许所有 `Origin`​。
* 然后检查 `Origin`​ 是否匹配正则表达式。
* 最后检查 `Origin`​ 是否在允许列表中。

### 7. 生成预检响应

```Python
def preflight_response(self, request_headers: Headers) -> Response:
    requested_origin = request_headers["origin"]
    requested_method = request_headers["access-control-request-method"]
    requested_headers = request_headers.get("access-control-request-headers")

    headers = dict(self.preflight_headers)
    failures = []

    if self.is_allowed_origin(origin=requested_origin):
        if self.preflight_explicit_allow_origin:
            headers["Access-Control-Allow-Origin"] = requested_origin
    else:
        failures.append("origin")

    if requested_method not in self.allow_methods:
        failures.append("method")

    if self.allow_all_headers and requested_headers is not None:
        headers["Access-Control-Allow-Headers"] = requested_headers
    elif requested_headers is not None:
        for header in [h.lower() for h in requested_headers.split(",")]:
            if header.strip() not in self.allow_headers:
                failures.append("headers")
                break

    if failures:
        failure_text = "Disallowed CORS " + ", ".join(failures)
        return PlainTextResponse(failure_text, status_code=400, headers=headers)

    return PlainTextResponse("OK", status_code=200, headers=headers)
```

* 根据预检请求生成预检响应，包括检查 `origin`​、请求方法和请求头的合法性。如果有不允许的项，返回带有错误信息的响应。

### 8. 生成简单请求的响应

```Python
async def simple_response(
    self, scope: Scope, receive: Receive, send: Send, request_headers: Headers
) -> None:
    send = functools.partial(self.send, send=send, request_headers=request_headers)
    await self.app(scope, receive, send)

async def send(
    self, message: Message, send: Send, request_headers: Headers
) -> None:
    if message["type"] != "http.response.start":
        await send(message)
        return

    message.setdefault("headers", [])
    headers = MutableHeaders(scope=message)
    headers.update(self.simple_headers)
    origin = request_headers["Origin"]
    has_cookie = "cookie" in request_headers

    if self.allow_all_origins and has_cookie:
        self.allow_explicit_origin(headers, origin)

    elif not self.allow_all_origins and self.is_allowed_origin(origin=origin):
        self.allow_explicit_origin(headers, origin)

    await send(message)

@staticmethod
def allow_explicit_origin(headers: MutableHeaders, origin: str) -> None:
    headers["Access-Control-Allow-Origin"] = origin
    headers.add_vary_header("Origin")
```

* ​`simple_response`​：处理简单请求，在传递给主要 ASGI 应用之前用 `send`​ 方法包装响应。
* ​`send`​：用于发送响应消息，并设置相应的 CORS 响应头。
* ​`allow_explicit_origin`​：如果请求包含 Cookie 或只允许特定来源，则设置响应头中的 `Origin`​。

#### 8.1 为什么allow_explicit_origin必须要用staticmethod修饰？

在Python中，使用`@staticmethod`​装饰器可以将方法定义为静态方法。静态方法不依赖于类实例进行调用，可以直接通过类名来访问。将`allow_explicit_origin`​定义为静态方法的主要原因在于其逻辑与实例状态无关，且能提高方法的可访问性，使代码更简洁。

1. 保持无状态

    ​`allow_explicit_origin`​ 的功能可能不涉及更改类实例的状态或访问实例变量。把它定义为静态方法，可以表明此方法不依赖于特定实例的状态。
2. 逻辑独立于类实例

    如果`allow_explicit_origin`​ 只是根据传入的参数来执行逻辑，而不需要访问或修改类的实例属性，那么将其定义为静态方法是合适的。

##### 示例代码

下面是一个简单的例子，展示如何将`allow_explicit_origin`​定义为静态方法，以及如何使用它：

```Python
class CorsHandler:
    allowed_origins = ["https://example.com", "http://example.com"]
  
    @staticmethod
    def allow_explicit_origin(origin):
        if origin in CorsHandler.allowed_origins:
            return True
        return False

# 可以直接通过类名调用静态方法，无需实例化
origin = "https://example.com"
is_allowed = CorsHandler.allow_explicit_origin(origin)
print(is_allowed)  # 输出：True
```

##### 对比类的实例方法

如果定义为实例方法，调用时需要实例化类。而且实例方法通常需要访问类实例的属性或方法。

```Python
class CorsHandler:
    def __init__(self):
        self.allowed_origins = ["https://example.com", "http://example.com"]
  
    def allow_explicit_origin(self, origin):
        if origin in self.allowed_origins:
            return True
        return False

# 需要实例化类才能调用方法
handler = CorsHandler()
is_allowed = handler.allow_explicit_origin("https://example.com")
print(is_allowed)  # 输出：True
```
