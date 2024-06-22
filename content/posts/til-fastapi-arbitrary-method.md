---
title: "TIL: FastAPI Supports Extended HTTP Methods"
date: 2024-05-09T08:00:00
draft: False
---

The other day, I was looking to emulate some webDAV functionality that required use of non-standard HTTP methods: `LOCK` and `UNLOCK`. While these are fully specified in <RFC 4918>[https://datatracker.ietf.org/doc/html/rfc4918#section-9.10], they are not the kind of thing you typically see explicitly supported by web frameworks out of the box. Neither FastAPI nor aiohttp show support in their documentation, so I decided to poke around in FastAPI to see if there was a way to get it to accept these two verbs.

There's nothing really special about any individual HTTP method since they're just a verb used in routing requests that, when used properly, help give the client a semantic understanding of the API. This means that there's no real functional difference between `GET /resource` and `PUT /resource` even though we semantically understand that `GET` will return the data representing `/resource` and `PUT` will update `/resource` with data that we specify. Functionally speaking, you could swap them and most servers wouldn't know the difference. According to <RFC 2616>[https://datatracker.ietf.org/doc/html/rfc2616#section-9], you could even create your own verb like `FANCYGET` and use it in your API:

    The set of common methods for HTTP/1.1 is defined below. Although
    this set can be expanded, additional methods cannot be assumed to
    share the same semantics for separately extended clients and servers.

This is a good example of something you *could* do but definitely *should not* do for many reasons, not least of which include interoperability, security, and general convention. This point just serves to illustrate that a REST framework that can handle the standard HTTP methods likely already has all the mechanics needed to handle extended methods like `LOCK` and `UNLOCK`.

The standard FastAPI method of creating a REST handler is to just use the `@app.<method>` decorator:

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/resource")
async def get_resource():
    return {"name": "myresource"}
```

Unfortunately, there's no easy `@app.lock` method to use in our case. But if we look under the hood of the `@app.get` decorator, we can find some hints about extending to other methods. The `FastAPI::get` decorators eventually calls `fastapi.routing.APIRouter::add_api_route` down the line, and there is a method by the same name on the `FastAPI` class that directly calls `APIRouter::add_api_route`. This python method takes the HTTP method as a string, so we can write a bit of code to try to use our `LOCK` method:

```python
async def lock_resource():
    ...

app.add_api_route("/resource", lock_resource, methods=["LOCK"])
```

If we run this code and place a request with something like curl, we'll see that the server accepts the request and returns a response:

```shell
curl -XLOCK http://localhost:8000/resource
```

It's very important to note that just because FastAPI supports any method, the ASGI server underneath and any proxies in between your server and the clients also have to support it. This poses a practical issue when trying to run your app since the proxy or ASGI server will likely reject the request before it even gets to your app. `LOCK` and `UNLOCK` happen to be supported by uvicorn (it appears to have support for the WebDAV extended methods) and many proxies have ways to enable WebDAV extensions. `FANCYGET`, on the other hand, is not likely to be supported.
