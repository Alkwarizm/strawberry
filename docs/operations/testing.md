---
title: Testing
---

# Testing

The GraphiQL playground integrated with Strawberry available at
[http://localhost:8000/graphql](http://localhost:8000/graphql) (if you run the
schema with `strawberry server`) can be a good place to start testing your
queries and mutations. However, at some point, while you are developing your
application (or even before if you are practising TDD), you may want to create
some automated tests.

We can use the Strawberry `schema` object we defined in the
[Getting Started tutorial](../index.md#step-5-create-our-schema-and-run-it) to
run our first test:

```python
def test_query():
    query = """
        query TestQuery($title: String!) {
            books(title: $title) {
                title
                author
            }
        }
    """

    result = schema.execute_sync(
        query,
        variable_values={"title": "The Great Gatsby"},
    )

    assert result.errors is None
    assert result.data["books"] == [
        {
            "title": "The Great Gatsby",
            "author": "F. Scott Fitzgerald",
        }
    ]
```

This `test_query` example:

1. defines the query we will test against; it accepts one argument, `title`, as
   input
2. executes the query and assigns the result to a `result` variable
3. asserts that the result is what we are expecting: nothing in `errors` and our
   desired book in `data`

As you may have noticed, we explicitly defined the query variable `title`, and
we passed it separately with the `variable_values` argument, but we could have
directly hardcoded the `title` in the query string instead. We did this on
purpose because usually the query's arguments will be dynamic and, as we want to
test our application as close to production as possible, it wouldn't make much
sense to hardcode the variables in the query.

## Testing Async

Since Strawberry supports async, tests can also be written to be async:

```python
@pytest.mark.asyncio
async def test_query_async():
    ...

    resp = await schema.execute(query, variable_values={"title": "The Great Gatsby"})

    ...
```

## Testing Mutations

We can also write a test for our [`addBook` Mutation](../general/mutations.md)
example:

```python
@pytest.mark.asyncio
async def test_mutation():
    mutation = """
        mutation TestMutation($title: String!, $author: String!) {
            addBook(title: $title, author: $author) {
                title
            }
        }
    """

    resp = await schema.execute(
        mutation,
        variable_values={
            "title": "The Little Prince",
            "author": "Antoine de Saint-Exupéry",
        },
    )

    assert resp.errors is None
    assert resp.data["addBook"] == {
        "title": "The Little Prince",
    }
```

## Testing Subscriptions

And finally, a test for our [`count` Subscription](../general/subscriptions.md):

```python
import asyncio
import pytest
import strawberry


@strawberry.type
class Subscription:
    @strawberry.subscription
    async def count(self, target: int = 100) -> int:
        for i in range(target):
            yield i
            await asyncio.sleep(0.5)


@strawberry.type
class Query:
    @strawberry.field
    def hello() -> str:
        return "world"


schema = strawberry.Schema(query=Query, subscription=Subscription)


@pytest.mark.asyncio
async def test_subscription():
    query = """
    	subscription {
        	count(target: 3)
    	}
    """

    sub = await schema.subscribe(query)

    index = 0
    async for result in sub:
        assert not result.errors
        assert result.data == {"count": index}
        index += 1
```

As you can see testing Subscriptions is a bit more complicated because we want
to check the result of each individual result.

## Feature/Integration Tests
Unit tests focus on small, isolated portions of code, typically individual methods or functions. On the other hand, feature tests cover larger portions of your application, including interactions between objects or full HTTP requests to endpoints. Feature tests provide the most confidence that your system as a whole functions as intended.

In FastAPI applications, dependency injection is commonly used to manage dependencies. This practice enhances testing and maintainability. FastAPI's built-in support for overriding dependencies during testing makes it particularly well-suited for this.

Strawberry's context_getter option in GraphQLRouter allows you to inject custom context objects into your GraphQL resolvers. Here's how you can leverage these features to write effective tests.

We start by defining our application dependencies.
```python
# dependencies.py
from fastapi import Depends
from sqlalchemy.orm import Session
from src.infrastructure.database.core import get_db

async def get_user_service(session: Session = Depends(get_db)) -> UserService:
    return UserService(UserRepository(session))

# user_service.py
class UserService:
    def __init__(self, user_repository: UserRepository):
        self.user_repository = user_repository
```
Here, `get_user_service` depends on a database session and constructs a `UserService` instance.

A custom context object can inject dependencies into GraphQL resolvers.
```python
# context.py
class CustomContext:
    def __init__(self, user_service: UserService):
        self.user_service = user_service

async def custom_context_getter(user_service: UserService = Depends(get_user_service)) -> CustomContext:
    return CustomContext(
        user_service=user_service,
    )

```

Setting up the application
```python
# app.py
from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter
from contextlib import asynccontextmanager
from src.infrastructure.database.core import init_db

@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()
    yield

def create_application() -> FastAPI:
    app = FastAPI(lifespan=lifespan)
    app.include_router(GraphQLRouter(schema=schema, context_getter=custom_context_getter))
    return app

app = create_application()

```

Configuring test setup
```python
# conftest.py
import pytest
from fastapi.testclient import TestClient
from app import create_application

@pytest.fixture
def setup():
    app = create_application()
    # Override dependencies
    app.dependency_overrides[get_db] = get_db_override
    with TestClient(app) as client:
        yield client

```
`get_db_override` is a test-specific database session provider to isolate test data.

Writing a test case
```python
# test_user_resolver.py
import pytest

@pytest.mark.asyncio
async def test_get_all_users(setup):
    query = """
        query ($page: Int, $limit: Int) {
            users(page: $page, limit: $limit) {
                id
                name
            }
        }
    """
    response = await setup.post(
        "/graphql",
        json={"query": query, "variables": {"page": 1, "limit": 10}},
    )
    response = response.json()
    assert response["data"]["users"] == []

```
In this test:

1. We define the GraphQL query to retrieve users.
2. The test client sends a POST request to the /graphql endpoint.
3. Assertions verify the response.
