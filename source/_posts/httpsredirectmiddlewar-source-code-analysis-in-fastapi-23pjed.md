---
title: FastAPI中HTTPSRedirectMiddlewar源码分析
date: '2024-06-25 14:17:27'
updated: '2024-06-25 15:57:08'
tags:
  - FastAPI
  - 源码
permalink: /post/httpsredirectmiddlewar-source-code-analysis-in-fastapi-23pjed.html
comments: true
toc: true
---

# FastAPI中HTTPSRedirectMiddlewar源码分析

```python
# starlette/middleware/httpsredirect.py

from starlette.datastructures import URL
from starlette.responses import RedirectResponse
from starlette.types import ASGIApp, Receive, Scope, Send


class HTTPSRedirectMiddleware:
    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] in ("http", "websocket") and scope["scheme"] in ("http", "ws"):
            url = URL(scope=scope)
            redirect_scheme = {"http": "https", "ws": "wss"}[url.scheme]
            netloc = url.hostname if url.port in (80, 443) else url.netloc
            url = url.replace(scheme=redirect_scheme, netloc=netloc)
            response = RedirectResponse(url, status_code=307)
            await response(scope, receive, send)
        else:
            await self.app(scope, receive, send)

```

这段代码展示了一个`HTTPSRedirectMiddleware`​中间件类，用于在`Starlette`​框架（一个Python的ASGI框架）中实现HTTP到HTTPS的重定向。它确保所有的HTTP或WebSocket（ws）请求被重定向到HTTPS或安全WebSocket（wss）。以下是详细解释：

# 导入模块

```Python
from starlette.datastructures import URL
from starlette.responses import RedirectResponse
from starlette.types import ASGIApp, Receive, Scope, Send
```

* ​**​`URL`​**​：用于处理和操作URL。
* ​**​`RedirectResponse`​**​：用于生成重定向响应。
* ​**​`ASGIApp`​**​、**​`Receive`​**​、**​`Scope`​**​、**​`Send`​**​：ASGI应用程序类型及相关请求处理类型的定义。

# 中间件类定义

```Python
class HTTPSRedirectMiddleware:
    def __init__(self, app: ASGIApp) -> None:
        self.app = app
```

* ​ **​`__init__方法`​**​：将应用程序实例保存到`self.app`​。中间件会包装这个ASGI应用，拦截所有传入请求。

# 中间件调用方法

```Python
async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
```

* 该方法作为调用入口，通过`scope`​、`receive`​和`send`​处理ASGI请求生命周期中的各个步骤。

## 为什么`__call__`​是必须的？

在ASGI（Asynchronous Server Gateway Interface）规范中，`__call__`​ 方法是一个核心组成部分，它定义了ASGI应用程序如何接受和处理请求。

### 1. `__call__`​ 方法的必要性

#### a. ASGI 规范要求

ASGI应用程序必须实现一个异步的`__call__`​方法，这是ASGI应用如何处理协议级别事件的核心机制。这个方法必须接受`scope`​、`receive`​ 和 `send`​ 这三个参数。这是ASGI与WSGI（Web Server Gateway Interface）的主要区别之一：ASGI是为异步处理设计的。

#### b. 接口一致性

为了确保与ASGI服务器兼容，任何ASGI应用或中间件必须实现这个接口，即定义`__call__`​方法。这使得ASGI服务器能够调用应用程序来处理请求。

### 2. `__call__`​ 的参数

#### a. `scope`​

​`scope`​ 参数包含了请求的上下文信息。这些信息包括但不限于：

* 请求类型（`http`​、`websocket`​ 等）
* 协议（`http`​、`https`​ 等）
* 路径、查询参数、头部信息等。

这是ASGI应用程序处理请求的基础信息。

#### b. `receive`​

​`receive`​ 是一个异步可调用对象，它用于接收客户端发送的消息。对于HTTP请求，它用于接收请求体；对于WebSocket，它用于接收消息，类似于读取操作。

#### c. `send`​

​`send`​ 也是一个异步可调用对象，用于发送消息或响应给客户端。对于HTTP请求，它用于发送HTTP响应；对于WebSocket，它用于发送消息，类似于写入操作。

### 具体交互流程

1. **请求到达**:

    * ASGI服务器接收到请求。
    * 调用中间件或最外层ASGI应用的`__call__`​方法。
2. **处理中间件逻辑**:

    * 中间件检查`scope`​，确定请求类型和协议。
    * 根据逻辑决定是否需要重定向或直接处理。
3. **重定向逻辑**:

    * 创建重定向URL和响应。
    * 使用`send`​发送重定向响应。
4. **传递请求**:

    * 如不需要重定向，直接调用下一个中间件或ASGI应用的`__call__`​方法，传递`scope`​、`receive`​和`send`​。

这种架构设计使得ASGI中间件和应用能够高效地进行异步处理，灵活应对不同类型的请求。通过标准化接口和参数，ASGI框架成功地为Python异步web开发提供了一个强大的基础设施。

## 请求类型和协议检查

```Python
if scope["type"] in ("http", "websocket") and scope["scheme"] in ("http", "ws"):
```

* ​**​`scope["type"]`​** ​：确保请求类型是HTTP或WebSocket。
* ​**​`scope["scheme"]`​** ​：确保请求协议是`http`​或`ws`​。

#### URL对象创建和重定向方案确定

```Python
url = URL(scope=scope)
redirect_scheme = {"http": "https", "ws": "wss"}[url.scheme]
```

* ​**​`URL(scope=scope)`​** ​：通过`scope`​创建一个URL对象。
* ​**​`redirect_scheme`​**​：根据当前URL的 scheme（http或ws），确定重定向的目标方案（https或wss）。上面的代码是先url.scheme中取出内容，有可能是http，或是ws，然后再从字典{"http":"https", "ws":"wss"}中取出对应键的值，若是http，则是https；若是ws，则是wss。然后再赋值给redirect_scheme，最后完成的效果就是将http://www.examples.com转化为https://www.example.com；或是将ws://127.0.0.1:12345转化为wss://127.0.0.1:12345。

#### 主机名和端口检查

```Python
netloc = url.hostname if url.port in (80, 443) else url.netloc
```

* ​**​`netloc`​**​：确保在常用端口（80或443）的情况下仅使用主机名，否则包含端口在内的网络位置（hostname:port）。

#### URL替换和重定向响应创建

```Python
url = url.replace(scheme=redirect_scheme, netloc=netloc)
response = RedirectResponse(url, status_code=307)
```

* ​**​`url.replace(scheme=redirect_scheme, netloc=netloc)`​** ​：替换URL的方案和网络位置。
* ​**​`RedirectResponse(url, status_code=307)`​** ​：创建一个307临时重定向响应对象，该状态码表示请求应使用相同的方法重复。

##### 为什么使用307状态码？

HTTP 307 是一个临时重定向，这意味着客户端应继续使用原始 URL 进行将来的请求，直到服务器发送具有不同状态代码（例如 200 OK）的最终响应。因为想要将客户端重定向到不同的 URL，但想要保持原始请求方法（GET、POST 等）不变时，这非常有用。

#### 发送重定向响应

```Python
await response(scope, receive, send)
```

* ​**​`await response(scope, receive, send)`​** ​：使用传入的`scope`​、`receive`​和`send`​调用生成的重定向响应，发送给客户端。

#### 传递非HTTP/非WebSocket请求

```Python
else:
    await self.app(scope, receive, send)
```

* 对于非HTTP或非WebSocket请求，直接传递到下一个中间件或最终的ASGI应用。

### 总结

这个中间件类用于将所有的HTTP和WebSocket请求重定向到其对应的HTTPS和WSS版本，确保通信安全性。其核心逻辑是在请求的协议检查和URL的修改上，然后通过生成适当的重定向响应来实现这一目标。

‍
