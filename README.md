<div align="center">

# GraiaX FastAPI

> :sparkles: Easy FastAPI Access for GraiaCommunity :sparkles:

[![codecov](https://codecov.io/gh/GraiaCommunity/graiax-fastapi/branch/master/graph/badge.svg?token=IU7kXPfTsV)](https://codecov.io/gh/GraiaCommunity/graiax-fastapi)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![Imports: isort](https://img.shields.io/badge/%20imports-isort-%231674b1?style=flat&labelColor=ef8336)](https://pycqa.github.io/isort/)
[![License](https://img.shields.io/github/license/GraiaCommunity/graiax-fastapi)](https://github.com/GraiaCommunity/graiax-fastapi/blob/master/LICENSE)
[![pdm-managed](https://img.shields.io/badge/pdm-managed-blueviolet)](https://pdm.fming.dev)
[![PyPI](https://img.shields.io/pypi/v/graiax-fastapi)](https://img.shields.io/pypi/v/graiax-fastapi)

</div>

你可以方便地使用 GraiaX FastAPI 配合 `graia.amnesia.builtins.asgi.UvicornASGIService`
轻松地在启动使用了 Graia Amnesia 的项目（如：Ariadne、Avilla）的同时启动一个
Uvicorn 服务器并在其中为 FastAPI 指定一个 entrypoint（入口点），且在 Launart
退出的时候自动关闭 Uvicorn。

## 安装

`pdm add graiax-fastapi` 或 `poetry add graiax-fastapi`。

> 我们强烈建议使用包管理器或虚拟环境

## 开始使用

以 Avilla 为例。

### 配合 Launart 使用

#### 机器人入口文件

如果你有使用 **Graia Saya** 作为模块管理工具，那么你可以使用 **FastAPIBehaviour**
以在 Saya 模块中更方便地使用 FastAPI。

FastAPI 本身并 **不自带** ASGI 服务器，因此你需要额外添加一个 **UvicornASGIService**。

```python
import pkgutil

from avilla.console.protocol import ConsoleProtocol
from avilla.core import Avilla
from creart import create
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from graia.amnesia.builtins.asgi import UvicornASGIService
from graia.broadcast import Broadcast
from graia.saya import Saya
from launart import Launart

broadcast = create(Broadcast)
saya = create(Saya)
launart = Launart()
avilla = Avilla(broadcast=broadcast, launch_manager=launart, message_cache_size=0)
fastapi = FastAPI()

avilla.apply_protocols(ConsoleProtocol())
saya.install_behaviours(FastAPIBehaviour(fastapi))
fastapi.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
launart.add_component(FastAPIService(fastapi))
launart.add_component(UvicornASGIService("127.0.0.1", 9000, {"": fastapi}))  # type:ignore

with saya.module_context():
    for module in pkgutil.iter_modules(["modules"]):
        if module.name[0] in ("#", ".", "_"):
            continue
        saya.require(f"modules.{module.name}")

launart.launch_blocking()
```

> [!NOTE]
> 需要留意的是，在把我们的 FastAPI 实例添加到 `UvicornASGIService` 中间件时，我们通过 `{"": fastapi}` 指定了一个**入口点**（enttrypoint）`""`，
> 这代表着我们此时传进去的 FastAPI 实例将占据 `http://127.0.0.1:9000/` 下所有入口，例如我们可以通过 `http://127.0.0.1:9000/docs` 访问我们的
> FastAPI 实例的 OpenAPI 文档。
>
> 假如我们使用 `{"/api": fastapi}` 指定 `/api` 为入口点，那么我们就需要通过 `http://127.0.0.1:9000/api/docs` 而不是
> `http://127.0.0.1:9000/docs` 来访问我们的 FastAPI 实例的 OpenAPI 文档。

#### Saya 模块中

```python
from graia.saya import Channel
from pydantic import BaseModel

from graiax.fastapi import RouteSchema, route

channel = Channel.current()


class ResponseModel(BaseModel):
    code: int
    message: str


# 方式一：像 FastAPI 那样直接使用装饰器
@route.get("/", response_model=ResponseModel)
async def root():
    return {"code": 200, "message": "Hello World!"}


# 方式二：当你先需要同一个路径有多种请求方式时你可以这样做
@route.route(["GET"], "/xxxxx")
async def xxxxx():
    return "xxxx"


# 方式三：上面那种方式实际上也可以这么写
@channel.use(RouteSchema("/xxx", methods=["GET", "POST"]))
async def xxx():
    return "xxx"


# Websocket
from fastapi import WebSocket
from starlette.websockets import WebSocketDisconnect
from websockets.exceptions import ConnectionClosedOK


@route.ws("/ws")
# 等价于 @channel.use(WebsocketRouteSchema("/ws"))
async def ws(websocket: WebSocket):
    await websocket.accept()
    while True:
        try:
            print(await websocket.receive_text())
        except (WebSocketDisconnect, ConnectionClosedOK, RuntimeError):
            break
```

#### 其他方式

假如你不想在 Saya 模块中为 FastAPI 添加路由，那么你可以选择以下几种方式：

<details>

##### 在机器人入口文件中直接添加

```python
...
fastapi = FastAPI()
...
fastapi.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@fastapi.get("/main")
async def main():
    return "main"

...
launart.add_component(FastAPIService(fastapi))
launart.add_component(UvicornASGIService("127.0.0.1", 9000, {"": fastapi}))  # type:ignore
...
```

##### 在 Avilla 启动成功后添加

```python
from fastapi.responses import PlainTextResponse
from graiax.fastapi.interface import FastAPIProvider

async def interface_test():
    return PlainTextResponse("I'm from interface!")


@listen(ApplicationLaunched)
async def function():
    launart = Launart.current()
    fastapi = launart.get_interface(FastAPIProvider)
    fastapi.add_api_route("/interface", fastapi.get("/interface")(interface_test))
```

</details>
