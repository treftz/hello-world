
# CHOP CODA Deployment Guide

A guide for configuring the CODA API and CODA UI applications in production-like
environments, in this document we will discuss:

1. Components & Dependencies
1. CODA API
   1. Background
   1. Environment Variables
   1. Database Migrations
   1. Initial Role & User
1. CODA UI
   1. Background
   1. Proxy Configuration


## Components & Dependencies

The entire application suite is comprised of two major components:

* An backend API application ("CODA API")
* An frontend UI application ("CODA UI")

Those components have a set of dependencies:

* A PostgreSQL (v12+) database ("DB")
* A connection to CHOP Active Directory ("LDAP/AD")
* A connection to CHOP Data Warehouse ("CDW")

The dependencies lay out as follows:

* CODA API depends on DB, LDAP/AD, CDW
* CODA UI depends on CODA API

Details of the configurations used _in development_ can be gleaned from the set
of Docker Compose files within this repository:

* `docker-compose.base.yml` outlines systems
* `docker-compose.dev.yml` lays in additional local developer convenience
* `docker-compose.prod.yml` suggests production configuration options to services

It is assumed in production-like environments that the major dependencies are
external systems (from the app) that already exist. The CODA application will
be made aware of these via configuration.



## CODA API

### Background

The CODA API is an HTTP web application written in Python and leveraging the
FastAPI framework. It is designed to run using a Python ASGI app server. The 
production ASGI server Gunicorn will spin up multiple uvicorn workers to process
requests by the application.

The `Dockerfile` includes a `CMD` that will launch the Gunicorn system via a
start script in the `/scripts/` directory.

References:

* [Python](https://www.python.org/) v3.7 – Programming language
* [FastAPI](https://fastapi.tiangolo.com/) – API/Application framework
* [Gunicorn](https://gunicorn.org/) - Production server and process host
* [uvicorn](https://www.uvicorn.org/) - ASGI App runner


### Environment Variables

The application is configured by the presence of environment variables set into
its environment.

In local development, this is achieved by specifying an `env_file` directive within
the `docker-compose.yml` declaration for the service. This file can be any
shell file that sets variables. Examples are provided in the repository, (e.g., 
`dev.env.example`), but changes to env file will be required per-environment.

The environment variables required include:

**`DATABASE_URL`** – Connection string to application database
  * Example: `postgresql://coda_db_user:coda_db_password@db/coda_dev`
  * Format is user:password@hostname/database_name
  * This connection string should match the connection details for the dependency DB

**`LDAP_URL`** – Connection string and LDAP host for Active Directory
  * Example: `ldap://path.to.activedirectory.ldap.server`
  * Format is an LDAP protocol URI to the server, can include port
  * On enterprise Active Directory, this may need to be an LDAP-compatibility port

**`LDAP_ADMIN_DN`** – LDAP base tree for admin lookups
  * Example: `CN=admin,DC=chop,DC=edu`
  * Format is an LDAP search base that contains the admin/search user

**`LDAP_ADMIN_PASSWORD`** – Admin-level password for base LDAP searches
  * Password string for the search/root/admin user
  * This account is used to search for authenticating user records, before the credentials
    from those users are used to verify

**`LDAP_SEARCH_DN`** – LDAP base search tree for CODA userspace
  * Example: `OU=CODA Users,DC=chop,DC=edu`
  * Expected value is search base specific to CODA users
  * This base will be searched for users by `uid` when they attempt logins

**`SECRET_KEY`** – Keybase string used to seed generated auth tokens
  * This can be any string, but should be independent per environment
  * Changing this will invalidate any tokens already issued for that instance
  * The application will fail to start with `SECRET_KEY` is not defined




### Database Migrations

Database structure can be manipulated by Alembic migrations specified within the
application code. Migrations can be run from an application context that has 
access to the database (e.g., the server itself or within the running container).

Migrating to the most recent state of the database is handled with the command:

```sh
> alembic upgrade head
```

References:
* [Alembic](https://alembic.sqlalchemy.org/) – Migration management tool


### Initial Role & User

An "Admin" role should be created for CODA users within the database.

```sql
INSERT INTO roles ("name", "admin", "created_at") VALUES ('Admin', 't', NOW())
```

A first user will have to be seeded into the database. This user must have a
`username` that matches an Active Directory `uid` of an admin user, and they should
be assigned to the admin role created, above:

```sql
INSERT INTO users
    ( username, first_name, last_name, role_id, created_at )
VALUES
    ('jsmith', 'John', 'Smith', 1, NOW())
```


## CODA UI

### Background

The CODA UI is a client pplication written in TypeScript and leveraging the Angular
framework. It presents an interface for the CODA administrative and user workflow. It
communicates with the CODA API via service classes.

In production-like environments, the `ng build` tool should be used to export the
compiled source of the project. This source should be served from a webserver or
proxy.

References:
* [Angular](https://angular.io/guide/setup-local) – Application framework
* [TypeScript](https://www.typescriptlang.org/) - Programming language


### Proxy Configuration

The CODA UI application will require a properly-configured proxy.config.json file,
which specifies where to proxy API requests made from the frontend.

There is a single rewrite rule for the `/api/*` paths that must have a value for
the `target` property which points to the API server. Example:

```json
{
    "/api/*": {
        "target": "http://path.to.coda.api.vm",
        "secure": false,
        "pathRewrite": {
            "^/api" : ""
        }
    }
}
```

Other portions of the file can remain the same. This file allows the frontend to make
requests against `/api/anything` and have them forwarded to the proper API server.
