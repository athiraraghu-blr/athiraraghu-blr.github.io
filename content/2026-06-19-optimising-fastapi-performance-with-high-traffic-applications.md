Title: Optimizing FastAPI Performance for High-Traffic Applications
Date: 2026-06-19
Category: Article
Tags: GenAI, LLM, machine-learning, deep-learning, announcement
Slug: fastpai-performance-high-traffic-application


FastAPI is fast by default, but scaling requires optimized async code, caching, database tuning, and efficient infrastructure. These practices help handle high traffic smoothly.

# *Embrace Async All the Way Down* 

FastAPI is built on Starlette and runs on an async event loop (via uvicorn or hypercorn). The single biggest mistake developers make is mixing blocking I/O into async route handlers.

python:
❌ Blocks the event loop — kills concurrency

@app.get("/users/{user_id}")

async def get_user(user_id: int):

    user = db.execute("SELECT * FROM users WHERE id = ?", 
    
    user_id)  # sync!
    
    return user

✅ Non-blocking — lets other requests run while waiting on I/O

@app.get("/users/{user_id}")

async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):

    result = await db.execute(select(User).where(User.id == user_id))

    return result.scalar_one()

Use async-native libraries everywhere:

**Database**: asyncpg, databases, SQLAlchemy 2.x with async sessions

**HTTP clients**: httpx (async), aiohttp

**Redis**: aioredis or redis-py with async support

**File I/O**: aiofiles

If you must call a blocking function (e.g., a CPU-heavy computation or a legacy sync library), offload it to a thread pool using asyncio.run_in_executor or FastAPI’s built-in run_in_threadpool:

python:

from fastapi.concurrency import run_in_threadpool

@app.get("/report")

async def generate_report():

    result = await run_in_threadpool(generate_heavy_report)

    return result


# *Use Multiple Workers and the Right Server*

A single uvicorn process uses one CPU core. For a production system, you need to scale across cores.

**Option A: Gunicorn with Uvicorn workers**

bash

gunicorn app.main:app \

  --workers 4 \

  --worker-class uvicorn.workers.UvicornWorker \

  --bind 0.0.0.0:8000

A rule of thumb for worker count is (2 × CPU cores) + 1, but profile your application — I/O-bound apps can often handle more.

**Option B: Uvicorn with multiple processes (uvicorn >= 0.20)**

bash

uvicorn app.main:app --workers 4 --host 0.0.0.0 --port 8000

**Option C: Hypercorn** for HTTP/2 support, which multiplexes multiple streams over a single TCP connection — especially beneficial for API clients making many parallel requests.


# *Connection Pooling for Databases*

Every database query that opens and closes a connection pays a significant TCP + authentication overhead. A connection pool keeps connections alive and reuses them.

python:

from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

from sqlalchemy.orm import sessionmaker

engine = create_async_engine(

    "postgresql+asyncpg://user:pass@localhost/db",

    pool_size=20,          # Connections kept alive

    max_overflow=10,       # Extra connections under peak load

    pool_timeout=30,       # Wait time before raising an error

    pool_pre_ping=True,    # Verify connection before use

)

AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

Tune pool_size based on your database server’s max_connections limit and the number of application workers. A common mistake is having workers × pool_size exceed the database’s connection limit.


# *Caching: The Single Biggest Throughput Multiplier* 

If the same data is requested repeatedly, compute it once and cache the result. This is often the most impactful optimization available.

**In-Memory Caching with cachetools**

For data that fits in memory and changes infrequently:

python:

from cachetools import TTLCache

from cachetools.func import ttl_cache

cache = TTLCache(maxsize=1000, ttl=300)  # 1000 items, 5-minute TTL

@ttl_cache(maxsize=500, ttl=60)

def get_country_list():

    return db.query(Country).all()

**Redis for Distributed Caching** 

When you have multiple workers or machines, use Redis so all instances share the same cache:

python:

import redis.asyncio as redis

import json

redis_client = redis.from_url("redis://localhost:6379", decode_responses=True)

@app.get("/products/{product_id}")

async def get_product(product_id: int):

    cache_key = f"product:{product_id}"

    cached = await redis_client.get(cache_key)

    if cached:

        return json.loads(cached)

    product = await fetch_product_from_db(product_id)

    await redis_client.setex(cache_key, 300, json.dumps(product))

    return product

 **HTTP-Level Caching**

For public, read-heavy endpoints, add Cache-Control headers so CDNs and browsers cache the response upstream:

python:

from fastapi import Response

@app.get("/public/catalog")

async def get_catalog(response: Response):

    response.headers["Cache-Control"] = "public, max-age=3600"

    return await fetch_catalog()


# *Optimize Serialization*

Pydantic v2 (the default in FastAPI >= 0.100) is dramatically faster than v1 — if you haven’t upgraded, do so. Beyond that:

**Use response_model wisely.** FastAPI validates and serializes the full response object. For large datasets, this validation overhead matters.

python:

#Only serialize the fields you actually need

@app.get("/users", response_model=list[UserSummary])  # Not UserFull

async def list_users():

    ...

**Use orjson for faster JSON encoding:**

python:

from fastapi.responses import ORJSONResponse

app = FastAPI(default_response_class=ORJSONResponse)

orjson is typically 2–3× faster than the standard json library and natively handles datetime, UUID, and numpy types.


# *Background Tasks for Non-Critical Work*

Don’t make the user wait for things that don’t need to happen before sending a response — like sending emails, logging analytics events, or triggering downstream webhooks.

python:

from fastapi import BackgroundTasks

@app.post("/orders")

async def create_order(order: OrderCreate, background_tasks: 
 BackgroundTasks):

    new_order = await db_create_order(order)

    background_tasks.add_task(send_confirmation_email, new_order.id)

    background_tasks.add_task(notify_warehouse, new_order.id)

    return {"order_id": new_order.id}

For heavier workloads, move to a dedicated task queue like Celery (with Redis or RabbitMQ as a broker) or ARQ (async-native, Redis-backed).


# *Rate Limiting and Request Throttling*

Under high traffic, a small number of clients can monopolize resources. Rate limiting protects your infrastructure and ensures fair access.

python:

from slowapi import Limiter, _rate_limit_exceeded_handler

from slowapi.util import get_remote_address

from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)

app.state.limiter = limiter

app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/search")

@limiter.limit("30/minute")

async def search(request: Request, q: str):

    return await perform_search(q)

For more sophisticated rate limiting (per user, per tier, sliding windows), implement it at the API gateway layer (NGINX, Kong, AWS API Gateway) so it intercepts traffic before it reaches your application.


# *Efficient Dependency Injection*

FastAPI’s Depends system is powerful but can become a bottleneck if you’re creating expensive objects (like database sessions or HTTP client connections) on every request. Use yield-based dependencies with proper lifecycle management:

python:

from contextlib import asynccontextmanager

@asynccontextmanager

async def lifespan(app: FastAPI):

    # Startup: create shared resources

    app.state.http_client = httpx.AsyncClient()

    app.state.redis = await redis.from_url("redis://localhost")

    yield

    # Shutdown: clean up

    await app.state.http_client.aclose()
    
    await app.state.redis.close()

app = FastAPI(lifespan=lifespan)

Avoid re-creating clients inside request handlers. A single httpx.AsyncClient shared across requests is far more efficient than creating a new one per request.


# *Database Query Optimization*

The application layer rarely is the true bottleneck — the database usually is. Profile your queries before optimizing application code.

**Avoid N+1 queries** by using joinedload or selectinload:

python:

❌ N+1: 1 query for orders + N queries for users

orders = await db.execute(select(Order))

for order in orders:

    print(order.user.name)  # Triggers another query each time

✅ Single query with eager loading

orders = await db.execute(

    select(Order).options(joinedload(Order.user))

)

**Use database indexes** on columns that appear in WHERE, ORDER BY, and JOIN clauses. A missing index on a high-traffic query is almost always the single most impactful fix available.

**Paginate large result sets** rather than fetching everything:

python:

@app.get("/events")

async def list_events(page: int = 1, size: int = 50):

    offset = (page - 1) * size

    result = await db.execute(select(Event).offset(offset).limit(size))

    return result.scalars().all()


# *Observability: Measure Before You Optimize*

Premature optimization without measurement leads to effort in the wrong places. Instrument your application before tuning.

**Prometheus metrics** via prometheus-fastapi-instrumentator:

python:

from prometheus_fastapi_instrumentator import Instrumentator

Instrumentator().instrument(app).expose(app)

This exposes request duration histograms, response codes, and throughput — exactly what you need to spot slow endpoints.

**Structured logging** with structlog or Python’s logging module in JSON format makes it easy to correlate requests across services and detect patterns in errors.

**Distributed tracing** with OpenTelemetry lets you trace a request across FastAPI, database calls, Redis, and downstream services — invaluable for diagnosing latency in complex systems.

**Putting It All Together**

There’s no single silver bullet for high-traffic FastAPI performance. The optimizations that matter most depend on your specific bottleneck — and the only way to know your bottleneck is to measure.

**That said, here’s a general priority order for most applications:**

1. Make all I/O non-blocking (async libraries, no sync calls in async handlers)

2. Add connection pooling for databases and external services

3. Cache aggressively at the right layer (in-memory → Redis → CDN)

4. Use multiple workers to exploit all CPU cores

5. Switch to ORJSONResponse and Pydantic v2 for faster serialization

6. Move non-critical work to background tasks or a task queue

7. Add rate limiting and throttling at the gateway

8. Profile and optimize database queries — indexes, eager loading, paginatio

9. Instrument with metrics and tracing to validate improvements

FastAPI gives you an excellent starting point. These practices take you the rest of the way — to an application that handles serious traffic with room to grow.