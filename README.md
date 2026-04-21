# fastapi-cbx

Code is disciplined. Reality is natural selection.
One scenario, one route.

A precise complement to the FastAPI ecosystem, unifying layered routing patterns and strict resource lifecycle management into a clean, production‑ready solution.

Originally proposed and contributed to FastAPI as PR #15392.

![CI](https://github.com/HeHongyeFY/fastapi-cbx/actions/workflows/ci.yml/badge.svg)
![Codecov](https://codecov.io/github/HeHongyeFY/fastapi-cbx/graph/badge.svg)
---

## Overview
fastapi-cbx introduces three **orthogonal, scenario‑driven routing patterns** — FR, CBV, and CBR — each designed for a clear, distinct category of business logic. Built with non‑invasive composition, it fully aligns with FastAPI’s native decorator style and enforces strict **resource lifecycle placement rules**: heavyweight global resources initialized in `__init__`, lightweight request‑scoped resources managed via `Depends`.

## Features
- Adds two scenario-driven class-based routing patterns to native FastAPI
- Ultra-lightweight: ~110 lines of clean code
- 100% test coverage
- No monkey patching, no metaprogramming, no hidden magic
- Fully backward compatible, no breaking changes
- Zero runtime overhead
- Native typing & dependency injection support
- Predictable, transparent behavior
- Minimal API surface, easy to maintain

## Installation
```bash
pip install fastapi-cbx
```

## Quick Start (30 Seconds)
fastapi-cbx provides CBR (Class-Based Route), an original, intuitive, high-performance class-based routing utility for FastAPI.

### Concept-focused, zero business logic

#### Install
```bash
pip install fastapi-cbx fastapi uvicorn
```

#### Minimal Example (main.py)
```python
import logging
from typing import Dict
from fastapi import FastAPI, APIRouter, Response, status
from cbx import cbr
from pydantic import BaseModel


@cbr(router=APIRouter(prefix="/cbr"))
class MyCBR:

    logger = logging.getLogger(__qualname__)

    class CBRModel(BaseModel):
        key: str = "CBR"
        value: str = "Class-based route for complex business logic with multiple endpoints and method-level dependencies"

    def __init__(self, **kwargs: Dict[str, str]):
        self.heavies = {
            "name": "fastapi-cbx",
            "description": "Minimal class-based routing extension for FastAPI"
        }
        self.heavies.update(kwargs)

    @cbr.get("/welcome", summary="Welcome to fastapi-cbx")
    @staticmethod
    async def welcome(response: Response) -> str:  # Also supports sync methods
        response.status_code = status.HTTP_200_OK
        response.set_cookie("token", "fastapi-cbx")
        return "Welcome to fastapi-cbx"

    @cbr.get("/heavies", summary="Get heavies by key")  # Also supports sync & @classmethod
    async def get_heavies(self, key: str) -> CBRModel:
        self.logger.info(f"GET {key}")
        return self.CBRModel(key=key, value=self.heavies.get(key, "One scenario, one route"))


app = FastAPI()
MyCBR(version="1.0.0")
app.include_router(MyCBR.router)
```

#### Run
```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

#### API Document
http://localhost:8000/docs


## Hierarchical Routing Patterns

### FR(Function Route): Stateless functional route for simple standalone endpoints.
```python
@app.get("/cbx")
async def func():
    return {"message": "Hello CBX"}
```

### CBV (Class-Based View):Class-based view for CRUD operations with singleton global dependency injection.
```python
import logging
import asyncio
import time
from typing import Dict
from fastapi import FastAPI, APIRouter, Response, status, Cookie, Depends, HTTPException
from cbx import cbv
from pydantic import BaseModel


@cbv(router=APIRouter(prefix='/cbv'))
class MyCBV:

    logger = logging.getLogger(__qualname__)

    class CBVModel(BaseModel):
        key: str = "cbv"
        value: str = "Class-based view for CRUD operations with singleton global dependency injection"

    def __init__(self, **kwargs: Dict[str, str]):
        self.heavies = {
            "name": "fastapi-cbx",
            "description": "Minimal class-based routing extension for FastAPI",
            "requires-python": ">=3.8",
        }
        self.heavies.update(kwargs)

    @staticmethod
    async def head(response: Response) -> None:
        await asyncio.sleep(1)
        response.status_code = status.HTTP_200_OK
        response.set_cookie("token", "fastapi-cbx")

    def get(self, key: str) -> CBVModel:
        self.logger.info(f"GET {key}")
        time.sleep(1)
        return self.CBVModel(key=key, value=self.heavies.get(key, "One scenario, one route"))

    @staticmethod
    def session(token: str = Cookie(default="", alias="token")) -> str:
        if token != "fastapi-cbx":
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")
        return token

    def post(self, body: CBVModel, token: str = Depends(session)) -> None:
        self.logger.info(f"POST {body.key} {body.value} {token}")
        time.sleep(1)
        self.heavies[body.key] = body.value

    @classmethod
    def put(cls, body: CBVModel, token: str = Depends(session)) -> None:
        cls.logger.info(f"PUT {body.key} {body.value} {token}")
        time.sleep(1)
        cls.CBVModel.model_fields["key"].default = body.key
        cls.CBVModel.model_fields["value"].default = body.value


app = FastAPI()
MyCBV(version="1.0.0")
app.include_router(MyCBV.router)
```

### CBR (Class-Based Route): Class-based route for complex business logic with multiple endpoints and method-level dependencies.
```python
import logging
import asyncio
import time
from typing import Dict
from fastapi import FastAPI, APIRouter, Response, status, Cookie, Depends, HTTPException
from cbx import cbr
from pydantic import BaseModel


@cbr(router=APIRouter(prefix="/cbr"))
class MyCBR:

    logger = logging.getLogger(__qualname__)

    class CBRModel(BaseModel):
        key: str = "CBR"
        value: str = "Class-based route for complex business logic with multiple endpoints and method-level dependencies"

    def __init__(self, **kwargs: Dict[str, str]):
        self.heavies = {
            "name": "fastapi-cbx",
            "description": "Minimal class-based routing extension for FastAPI"
        }
        self.heavies.update(kwargs)

    @cbr.get("/welcome", summary="Welcome to fastapi-cbx")
    @staticmethod
    async def welcome(response: Response) -> str:
        response.status_code = status.HTTP_200_OK
        response.set_cookie("token", "fastapi-cbx")
        return "Welcome to fastapi-cbx"

    @cbr.get("/heavies", summary="Get heavies by key")
    async def get_heavies(self, key: str) -> CBRModel:
        self.logger.info(f"GET {key}")
        return self.CBRModel(key=key, value=self.heavies.get(key, "One scenario, one route"))

    @staticmethod
    def session(token: str = Cookie(default="", alias="token")) -> str:
        if token != "fastapi-cbx":
            raise HTTPException(
                status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")
        return token

    @cbr.post("/heavies", summary="Set heavies")
    async def set_heavies(self, body: CBRModel, token: str = Depends(session)) -> None:
        self.logger.info(f"POST {body.key} {body.value} {token}")
        await asyncio.sleep(1)
        self.heavies[body.key] = body.value

    @cbr.get("/default", summary="Get default heavies")
    @classmethod
    async def get_default(cls) -> CBRModel:
        cls.logger.info(f"GET default")
        return cls.CBRModel()

    @cbr.get("/sync", summary="Sync Support")
    @classmethod
    def sync(cls) -> CBRModel:
        time.sleep(1)
        return cls.CBRModel(key="sync", value="ok")


app = FastAPI()
MyCBR(version="1.0.0")
app.include_router(MyCBR.router)
```

## Resource Lifecycle Placement Rules

To build clean, performant, and maintainable Python web services (especially for ML / LLM / inference services), follow this strict but simple rule:

---

### Heavyweight Global Resources

**Use case:**
- Database connections & ORM engines
- Shared clients (Redis, object storage, third-party SDKs)
- LLM / transformer models
- Inference engines (vLLM, TensorRT-LLM)
- Any expensive-to-initialize component

**Where to initialize:**  
Class `__init__` method

**Why:**
- Initialized exactly once at application startup
- Reused across all requests
- No redundant memory or CPU overhead
- Avoids global variables
- Natural domain encapsulation

**Example:**
```python
@cbr(router=APIRouter(prefix="/cbr"))
class MyCBR:

    def __init__(self, **kwargs: Dict[str, str]):
        self.heavies = {
            "name": "fastapi-cbx",
            "description": "Minimal class-based routing extension for FastAPI"
        }
        self.heavies.update(kwargs)
```

### Lightweight Request-Scoped Resources

**Use case:**
- Current logged-in user
- Request session
- Auth / token validation
- Request ID / tracing context
- Per-request temporary state

**Where to initialize:**  
FastAPI `Depends`

**Why:**
- Created once per request
- Automatically isolated between requests
- Clean dependency injection
- Low mental overhead

**Example:**
```python
@cbr(router=APIRouter(prefix="/cbr"))
class MyCBR:
    ...

    @cbr.post("/heavies", summary="Set heavies")
    async def set_heavies(self, body: CBRModel, token: str = Depends(session)) -> None:
        self.logger.info(f"POST {body.key} {body.value} {token}")
        await asyncio.sleep(1)
        self.heavies[body.key] = body.value
```

## Notes & Warnings

1. **Focus on Routing Only**  
   This project focuses solely on routing extension for FastAPI. It does not handle business logic, data validation (beyond FastAPI's native capabilities), or other non-routing related functionalities.

2. **Resource Contention & Concurrency Safety**  
    The handling of contention over shared resources (e.g., database connections, SDK clients and other global shared instances) shall be undertaken by developers. fastapi-cbx does not embed built-in safeguards or isolation controls for concurrent access, whether in synchronous or asynchronous execution scenarios.

3. **Avoid `__annotations__['cbx_router']` Conflict**  
   The library uses the `__annotations__['cbx_router']` attribute internally to bind routers. Do not manually modify or use this attribute in your code to avoid conflicts and unexpected behavior.

4. **No Inheritance Support**  
   As a lightweight encapsulation for REST APIs, fastapi-cbx does not consider or support class inheritance scenarios for CBV/CBR. Each route class should be independent and self-contained.

5. **Class Initialization Requirement**  
   Classes decorated with @cbv(router=APIRouter(prefix='/cbv')) or @cbr(router=APIRouter(prefix="/cbr")) must be initialized before including the router.

6. **Support & Feedback**  
   Feel free to open issues or submit PRs on GitHub. If you find this library useful, please give it a star to support the project.
