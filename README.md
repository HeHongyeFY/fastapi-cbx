# fastapi-cbx

Code is disciplined. Reality is natural selection.
One scenario, one route.

---

## Overview
fastapi-cbx is a minimal, class-based routing extension for FastAPI.
It defines three orthogonal, scenario-driven routing patterns — FR, CBV, and CBR —
each responsible for a clear, distinct category of business logic.
Built around non-invasive composition, 
and stays fully aligned with FastAPI's native decorator style.

## Features
- Complements native FastAPI routing with two scenario-driven class-based patterns
- Ultra-lightweight: ~110 lines of clean code
- 100% test coverage
- No monkey patching, no metaprogramming, no hidden magic
- Fully backward compatible, no breaking changes
- Zero runtime overhead
- Native typing & dependency support
- Predictable, transparent behavior
- Minimal API surface, easy to maintain


## Installation
```bash
pip install fastapi-cbx
```

## Hierarchical Routing Patterns

### FR(Function Route): Stateless functional route for simple standalone endpoints.

```python
@router.get("/")
def index():
    return {"route": "function"}
```

### CBV (Class-Based View):Class-based view for CRUD operations with singleton global dependency injection.

```python
@cbv(router=router)
class UserCBV:

    def __init__(self, db: FakeDB = db):
        self.db = db

    def get(self, user_id: int, session: FakeSession = Depends(get_session)):
        user = self.db.get_user(user_id)
        if not user:
            raise HTTPException(status_code=404, detail="User not found")
        
        return {
            "user": user,
            "operated_by": session.uid
        }

    async def post(self, user_id: int, name: str):
        if self.db.get_user(user_id):
            raise HTTPException(status_code=400, detail="User already exists")
        
        return {
            "user_id": user_id,
            "name": name,
            "message": "User created successfully"
        }

    async def put(self, user_id: int, name: str):
        if not self.db.update_user(user_id, name):
            raise HTTPException(status_code=404, detail="User not found, update failed")


    def delete(self, user_id: int):
        if not self.db.delete_user(user_id):
            raise HTTPException(status_code=404, detail="User not found, delete failed")
```

### CBR (Class-Based Route): Class-based route for complex business logic with multiple endpoints and method-level dependencies.

```python
@cbr(router=router)
class OrderCBR:

    _order_prefix = "ORDER-"

    def __init__(self, db: FakeDB = db):
        self.db = db


    @cbr.get("/info")
    def info(self, order_id: int, session: FakeSession = Depends(get_session)):
        return {
            "order_id": f"{self._order_prefix}{order_id}",
            "current_user_id": session.uid,
            "db": self.db
        }


    @cbr.post("/create")
    async def create(self, order_id: int, name: str):
        return {
            "order_id": f"{self._order_prefix}{order_id}",
            "user": name,
            "db_tag": str(self.db)
        }

    @cbr.post("/batch")
    @classmethod
    def batch_create(cls, total: int):
        return {
            "method": "classmethod",
            "order_prefix": cls._order_prefix,
            "batch_total": total
        }


    @cbr.get("/validate")
    @staticmethod
    def validate_order(order_id: int):
        return {
            "method": "staticmethod",
            "is_valid": order_id > 0
        }
```

## Example

```python
import uvicorn
from fastapi import FastAPI, APIRouter, Depends, HTTPException
from cbx import cbv, cbr

# ==============================================
# Shared mock resources
# ==============================================
class FakeDB:
    def __init__(self):
        self.users = {1: "Alice", 2: "Bob"}
    def get_user(self, user_id: int):
        return self.users.get(user_id)

    def update_user(self, user_id: int, name: str):
        if user_id not in self.users:
            return False
        self.users[user_id] = name
        return True

    def delete_user(self, user_id: int):
        if user_id not in self.users:
            return False
        del self.users[user_id]
        return True    

db = FakeDB()


class FakeSession:
    def __init__(self):
        self.uid = 1001


def get_session():
    return FakeSession()


# ==============================================
# Application & Routers (All with prefix)
# ==============================================
app = FastAPI(title="fastapi-cbx")


# One scenario, one route

# ==============================================
# 1. FR (Function Route)
# ==============================================
router = APIRouter(prefix="/fr")

@router.get("/")
def index():
    return {"route": "function"}

app.include_router(router)

# ==============================================
# 2. CBV (Class-Based View)
# ==============================================
router = APIRouter(prefix="/user")
@cbv(router=router)
class UserCBV:

    def __init__(self, db: FakeDB = db):
        self.db = db

    def get(self, user_id: int, session: FakeSession = Depends(get_session)):
        user = self.db.get_user(user_id)
        if not user:
            raise HTTPException(status_code=404, detail="User not found")
        
        return {
            "user": user,
            "operated_by": session.uid
        }

    async def post(self, user_id: int, name: str):
        if self.db.get_user(user_id):
            raise HTTPException(status_code=400, detail="User already exists")
        
        return {
            "user_id": user_id,
            "name": name,
            "message": "User created successfully"
        }

    async def put(self, user_id: int, name: str):
        if not self.db.update_user(user_id, name):
            raise HTTPException(status_code=404, detail="User not found, update failed")


    def delete(self, user_id: int):
        if not self.db.delete_user(user_id):
            raise HTTPException(status_code=404, detail="User not found, delete failed")


UserCBV(db)

app.include_router(router)

# ==============================================
# 3. CBR (Class-Based Route)
# ==============================================
router = APIRouter(prefix="/order")
@cbr(router=router)
class OrderCBR:

    _order_prefix = "ORDER-"

    def __init__(self, db: FakeDB = db):
        self.db = db


    @cbr.get("/info")
    def info(self, order_id: int, session: FakeSession = Depends(get_session)):
        return {
            "order_id": f"{self._order_prefix}{order_id}",
            "current_user_id": session.uid,
            "db": self.db
        }


    @cbr.post("/create")
    async def create(self, order_id: int, name: str):
        return {
            "order_id": f"{self._order_prefix}{order_id}",
            "user": name,
            "db_tag": str(self.db)
        }

    @cbr.post("/batch")
    @classmethod
    def batch_create(cls, total: int):
        return {
            "method": "classmethod",
            "order_prefix": cls._order_prefix,
            "batch_total": total
        }


    @cbr.get("/validate")
    @staticmethod
    def validate_order(order_id: int):
        return {
            "method": "staticmethod",
            "is_valid": order_id > 0
        }

OrderCBR(db)

app.include_router(router)

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```


## Recommended Pattern

### Resource Lifecycle Placement Rules

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
# Heavy global resource (initialized once)
class FakeDB:
    def __init__(self):
        self.data = {}

db = FakeDB()

@cbr(router=router)
class OrderAPI:
    # Inject into constructor
    def __init__(self, db: FakeDB = db):
        self.db = db
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
class FakeSession:
    def __init__(self):
        self.user_id = 1

def get_session():
    return FakeSession()

@cbr.get("/{order_id}")
def get_order(self, order_id: int, session: FakeSession = Depends(get_session)):
    return {"user": session.user_id, "order_id": order_id}
```


## Notes & Warnings

1. **Focus on Routing Only**  
   This project focuses solely on routing extension for FastAPI. It does not handle business logic, data validation (beyond FastAPI's native capabilities), or other non-routing related functionalities.

2. **Resource Competition & Thread Safety**  
   The responsibility of handling resource competition and ensuring thread safety (e.g., for shared global resources like DB connections, SDK clients) lies with the user. fastapi-cbx does not provide additional thread safety mechanisms.

3. **Avoid `__annotations__['cbx_router']` Conflict**  
   The library uses the `__annotations__['cbx_router']` attribute internally to bind routers. Do not manually modify or use this attribute in your code to avoid conflicts and unexpected behavior.

4. **No Inheritance Support**  
   As a lightweight encapsulation for REST APIs, fastapi-cbx does not consider or support class inheritance scenarios for CBV/CBR. Each route class should be independent and self-contained.

5. **Class Initialization Requirement**  
   Classes decorated with `@cbv(router=router)` or `@cbr(router=router)` must be initialized (e.g., `UserCBV(db)`, `OrderCBR(db)`) after definition. Failure to do so will result in routing not being registered correctly.

6. **Support & Feedback**  
   Feel free to open issues or submit PRs on GitHub. If you find this library useful, please give it a star to support the project.
