# Implementation of a set of separate FastAPI-based microservices in one project

> This is an example of the organization of the python module code, a demonstration of the solution concept, "torn" from the working project without guarantees of operability "out of the box", further development and support.

## Key Features

- The project is focused on simultaneous work with separate services and a shared library within a single git repository
- In general, the library is installed on each server in the system cluster, and the context of the role performed by each point of the system is determined by the running FastAPI application implementing a separate set of microservice APIs.
- Missing in this example, but important components of the solution architecture:
- Integration bus for messaging based on RabbitMQ
- Caching and data exchange between instances of Redis-based services
- The entire set of microservices and shared code is compiled in one Python module, with the following structure:
- `nms` - wrapper module for individual services and shared code module
- `nms.common` - shared code module
- `nms.auth` - user authorization and user management service
- `nms.cfg` - a service that implements configuration management of a managed system
- `nms.inbox' - integration service with an external system (integration API)
- `nms.{...}` - every new service

> Example: launching a user authorization service
> ```shell
> # Development server
> uvicorn nms.auth.main:app --host 0.0.0.0 --port 4000 --workers 1 --log-config loggers.json
>
> # Production server
> gunicorn --bind 0.0.0.0:$PORT --workers=4 nms.auth.main:app -k uvicorn.workers.UvicornWorker
> ```

## Comments on implementation
The configuration of the service can be set:
- using environment variables (in priority)
- in the local .env file
```shell
TZ=Europe/Moscow
ENVIRONMENT=dev
TESTING=True
DEBUG=True
DB_NMS_DSN=postgres://nms:nms@nms-db:5432/nms
DB_DWH_DSN=postgres://dwh:dwh@nms-db:5432/dwh
JWT_SECRET_KEY=P4xCg7Dhk+Db
```

Services can work simultaneously with different databases, access to which is determined by the basic type of repository used.

Repositories for working with the database: `nms.common.db.repositories`

Depending on the repository type, a database connection pool is created with the corresponding DSN: `DB_NMS_DSN`, `DB_DWH_DSN`, etc.
```python
# types of DB repositories
class BaseNmsRepository(BaseRepository):
_dsn_type = DSNType.NMS

class BaseDwhRepository(BaseRepository):
_dsn_type = DSNType.DWH
```

A connection pool is created when `get_db_repository()` is called (see `nms/common/api/dependencies/database.py `)

The project does not use ORM, all SQL queries are implemented through the [aiosql] library (https://pypi.org/project/aiosql /)


### Initializing the environment with [Poetry](https://python-poetry.org/docs /)

```shell
poetry install
```

### Deployment in Docker. Database preparation and initialization

1. You must first create a docker volume:
```shell
docker volume create nms-db-data
```
2. Run the database for the first time. At the first start of the database (an empty nms-db-data section), the script will be executed `db/create-db-v3.sh ` , which creates the necessary bases.
```shell
docker-compose up -d nms-db
```
3. After the successful start of the database and initialization, it is necessary to apply migrations for all databases. The [aiosqlembic] package is used as a migration tool for the database in this project (https://pypi.org/project/aiosqlembic /):
```shell
cd ./db
aiosqlembic -m ./auth/migrations asyncpg postgres://auth:auth@localhost:5432/auth up
aiosqlembic -m ./nms/migrations asyncpg postgres://nms:nms@localhost:5432/nms up
...
```
