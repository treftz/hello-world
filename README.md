# CHOP CODA

This repository contains the CHOP **Clinical Outcomes Data Archive**
project and its backend processes and APIs. This project contains code
responsible primarily for:

* Ingesting clinical data from the CHOP Data Warehouse
* Providing a go-between for workflow APIs and CODA data

The application also has a [front-end component](https://gitlab.arcweb.co/chop/coda-ui),
which provides the rich user interface for administration and workflow processes.

This repository also includes configuration and convenience tooling for
working with both the API and UI projects simultaneously. Read more about
how to achieve this in the appropriate sections, below.


## Technologies

* App Stack
  * [Python](https://www.python.org/) v3.7 – Programming language
  * [FastAPI](https://fastapi.tiangolo.com/) – API/Application framework
* Data Layer
  * [PostgreSQL](https://www.postgresql.org/) – Relational Database
  * [SQLAlchemy](https://sqlalchemy.org/) – Object Relational Mapping, DB tooling
  * [Alembic](https://alembic.sqlalchemy.org/) – Migration management tool
* Ops Stack
  * [Gunicorn](https://gunicorn.org/) - Production server and process host
  * [uvicorn](https://www.uvicorn.org/) - ASGI App runner
* Client Integrations
  * Netezza-based Data Warehouse
  * LDAP/Active Directory
  * OAuth2 Client-Server Authentication w/ JWT Bearer tokens


## Deployed Environments

The CODA applications will be deployed to CHOP-managed virtual machines
within an internal data center. The `Dockerfile` within this project will
dictate the dependencies and assets necessary to build and deploy the
product in that context.

A [Docker Compose](https://docs.docker.com/compose/) overlay configuration
file exists, which describes production-specific settings.


## Development Environment

It is recommended you pull code for this application suite into a folder structure
that provides a project "umbrella." For example:

* `~/git/arcweb/chop` – _CHOP Git root_
  * `coda-api` – _this project_
  * `coda-ui` – _front-end sister project_

This project contains code that manages data ingestion, a services API, and shared
resources. The folder structure is as follows:

* `/api` – FastAPI application
* `/ingest` – Python ingestion mechanism for Data Warehouse data
* `/alembic` – Database migrations
* `/scripts` – Convenience scripts and shorthands

At the developer's discretion, this project supports either:

* Host machine development
* Dockerized development

The preferred mechanism for local development is to follow a Dockerized approach,
but instructions below can get you up and running in either paradigm.

*NOTE: Developers that wish to fully integrate an IDE, such as VSCode, with
project-specific package recognition, linting support, and autocompletion may
have to work on their configuration to support such features. One option is to
explicity configure to support containers
([VSCode container doc](https://code.visualstudio.com/docs/remote/))
and the alternative is installing dependencies on your host machine, as well.*


### Host Machine Development

For developers that prefer to install dependencies and run services on their host
machines, follow these steps:

1. Install a Python version manager, such as [PyEnv](https://github.com/pyenv/pyenv) (optional)
1. Install Python 3.7
1. Install [Pipenv](https://pipenv.pypa.io/) for dependency management

You will require system libaries be installed to assist with compilation of some
native code and Python wheels. On Mac it is recommended you use [Homebrew](https://brew.sh/)
to install these packages. Up-to-date OS-level packages required can be gleaned
from the build steps within the `Dockerfile`. Installations necessary will include:

```sh
> brew install postgresql
```

Once prerequisites are in place, you can get up and running by installing project
dependencies and creating your virtual environment using Pipenv:

```
> pipenv install
```

Now, the shorthand scripts defined within the project's `Pipfile` will be available
to you. Using those scripts you can start and stop development servers, create
data migrations, run the test suite, and more. Examples:

```sh
# Step into a shell with environment sourced:
> pipenv shell

# Start a load server with hot code reload:
> pipenv run start-reload

# Generate a database migration:
> pipenv run alembic revision --autogenerate

# Run test suite
> pipenv run tests
```

See the `[scripts]` section of the `Pipfile` for complete details.


### Dockerized Development

Developing locally with containers can present some initial configuration
challenges, but ultimately allows the developer to run a version of the
applications that very closely matches production.

To get up and running, these prerequisites must be in place:

1. Install [Docker Desktop](https://www.docker.com/products/docker-desktop)
1. Configure project environment as described, below, under "Environment Variables"
1. Install [Docker-Sync](http://docker-sync.io/)

#### Docker-Sync
Docker-Sync is used to enable fast-syncing volumes of the project source code
between the host machine (where you will be editing code) and the running
container (where the software is running).

Typically, when building a Docker image, the code is copied in place at build
time and edits to it are not reflected in any running container. With Docker
Sync, volumes can be used to make hot code reloading a reality.

Docker-Sync is written in Ruby and requires a version of ruby be installed.
A `Gemfile` is provided in the project which makes installing Docker-Sync easy
if Bundler is available:

```sh
> bundle install
```

If additional details are required, read the
[Docker Sync Installation Guide](https://docker-sync.readthedocs.io/en/latest/getting-started/installation.html)

Configurations of volumes are defined in `docker-sync.yml`

#### Running the Services

Once the prequisites are in place, the entire suite of services associated
with the application can be started using Docker Compose:

```sh
# Bring up all services (API, UI, Database):
> docker-compose up

# Also (re)build the services when starting up:
> docker-compose up --build
```

To take full advantage of Docker Sync, the sync service must be up and
running, as well:

```sh
# Start sync servics and then compose environment:
> docker-sync start
> docker-compose up

# OR you can start sync AND compose environment simultaneously:
> docker-sync-stack start
```
Learn about the docker sync commands in detail:
[docker-sync commands guide](https://docker-sync.readthedocs.io/en/latest/getting-started/commands.html)


#### Postgres Database Initialization

The official Postgres Docker images support the initialization of users, roles,
and databases via two mehtods: **environment variables** for simple efforts and
**custom initialization scripts** for more detailed needs. This project uses
a combination of the two to achieve its needs:

* Environment Variables (defined in docker-compose.yml files):
  * `POSTGRES_USER` is set to make "coda_db_user" the default user
  * `POSTGRES_PASSWORD` is set to make "coda_db_password" that user's password
* Initialization Scripts:
  * `scripts/postgres-docker-init.sh` is mounted into the container to create both
  a dev and test database

When you first boot the services for this project, the initialization script will
run and create the necessary databases. However, if databases have already been
created, the initialization script will not run -- this is by design.

If you have previously run the project _prior to the introduction of the initialization
scripts_, you may have to clear your postgres Docker volume and restart. This will
result in loss of data, but will cause the initialization scripts to run, which will
create the dev and test databases correctly. If you need to do this, the volume can be
removed by running these commands:

```sh
# Stop all running services (if running):
> docker-compose down
# Remove CODA Postgres images form machine:
> docker volume rm coda_postgres
# Restart all services (will trigger DB initialization):
> docker-compose up
```

Once you've used the initialization script and have both databases, you will not have
to remove the docker volume on subsequent runs.


#### OpenLDAP Support Service Initialization

To simulate Active Directory for authentication, the OpenLDAP Docker image is
used within this project. A simulated base of users are included in an LDIF file,
which is mounted into the LDAP container. A corresponding data migration will insert
roles and users into the CODA database.

* LDIF records are defined within `/scripts/ldap_base.ldif` and mounted via docker-compose
* User records are defined within `/scripts/ldap_users.sql`, which is sourced and
run via an alembic data migration "ldap_users_and_roles"

**NOTE: Your application database must have users that match those in the LDAP configuration.**
Without users that match, you will not be able to authenticate. Users defined within
the `ldap_base.ldif` file and the `ldap_users.sql` file include John Smith (an admin)
and Sally Brown (a user). If your environment in some way falls out of sync with the
automated bootstrapping, above, users will have to be brought in sync between the LDAP
container and local development database. 

As a local development convenience, the [PHP LDAP Admin](http://phpldapadmin.sourceforge.net/wiki/index.php/Main_Page)
tool is included in the `docker-compose.yml` definitions. This tool, brought up on
[localhost port 6060](http://localhost:6060), allows for viewing and modifying the
LDAP configuration, organization, and user spaces used for authentication.


#### Executing Commands

In the docker context, the `pipenv` commands for your host are unnecessary, and
won't actually work. You can run them within the docker container or run the
appropriate commands directly, _but the commands must be run inside the container
to take effect._

In contrast to the host commands, above, we want to use `docker-compose run`
to run commands within the appropriate context. Here are examples:

```sh
# Get a shell in the container:
> docker-compose run api /bin/sh
# Once you are within the container, you can run any "host" commands you
# are familiar with inside that container, if that's your style.

# Alembic commands:
> docker-compose run api alembic current
> docker-compose run api alembic upgrade head
> docker-compose run api alembic revision --autogenerate
```

As you can see, you should be able to use `docker-compose run` to run any command
within the appropriate container. The container names match the name of the service
defined within the `docker-compse.*.yml` files.

```
# Run against the API:
> docker-compose run api <your command>

# Run against the UI:
> docker-compose run ui <your command>

# Run against the DB (psql terminal or other):
> docker-compose run db <your command>
```

The API container sets a working directory and `PYTHONPATH` environment variable
during its build, so it is unnecessary to specify these on each command run.


#### Running Tests in Container

You can run tests locally against your container using the same process as described
above using the `docker-compose run` approach:

```
> docker-compose run api pytest
> docker-compose run api pylint api/ ingest/ tests/
```

*PLEASE NOTE: By running the commands against the api container using the style above,
you are running tests against your development database and environment.*

A convenience script has been provided that uses the `docker-compose.test.yml` configuration
for the `api` service to ensure you are **sourcing your config from the test.env file**:

```
# Run tests with test environment sourced:
> ./scripts/tests.sh
```

Internally, this script is building a docker compose configuration from the base and test
files. This allows you to specify a different database target (e.g., "coda_test" instead of
"coda_dev") in your `test.env` environment file and ensure your tests are run there.




### Environment Variables

The application requires some environment variables be present when running
components, including the application server. You can bootstrap your local
environment with properly-fitted env files using the included bootstrap script:

```sh
> scripts/bootstrap.sh
./test.env.example -> ./test.env
./dev.env.example -> ./dev.env
```

The `dev.env` file, now present, includes environment variable definitions
necessary to run the project:

* `DATABASE_URL` – Connection string to application database
  * In Docker container, the host is `db` to match the service name
  * On host machine, the host is typically `localhost`
* `LDAP_URL` – Connection string and LDAP host for Active Directory
  * In Docker container, the host is `ldap` to match service name
  * On host machine, the host is typically `localhost`
* `LDAP_ADMIN_DN` – LDAP base tree for admin lookups
* `LDAP_ADMIN_PASSWORD` – Admin-level password for base LDAP searches
* `LDAP_SEARCH_DN` – LDAP base search tree for CODA userspace
* `SECRET_KEY` – Keybase string used to seed generated auth tokens
  * This can be any string, but should be independent per environment
  * Changing this will invalidate any tokens already issued for that instance
  * The application will fail to start with `SECRET_KEY` is not defined
* `PORT` – System port on which to run the API server
  * This defaults to `8000` for development, `80` for production

_NOTE: There is a committed `.env` file in this project, which is used to specify
environment for docker-compose. The dev and test files referenced here are identified
within the service configurations and are loaded during `docker-compose up`._


## Running the Applications

Once you have installed the necessary prerequisites and have your environment
properly configured, you should be able to bring the application and its dependencies
up by launching docker-compose:

```sh
> docker-compose up
```

On _the first run_, this will:
* Launch and bootstrap the Postgres database service (db)
* Launch and bootstrap the OpenLDAP service (ldap)

On _every run_, this will:
* Launch the CODA API application (api)
* Launch the CODA UI application (ui)
* Launch the dependent services (db, ldap)
* Launch the support services (pgadmin, ldap-admin)

_When necessary_, you may elect to:

* Run alembic data and schema migrations against your database

> On your first run, and when there are changes to data schema, you will want to run
> any pending data migrations. This is achieved by running an `alembic` upgrade command
> within the `api` container, like so:
>
> ```sh
> > docker-compose run api alembic upgrade head
> ```

* Bring up a python or shell terminal within the API context

> ```sh
> > docker-compose run api python
> > docker-compose run api /bin/sh
> ```


## Running Tests

Once you have your database and server running, run the test suite with
[PyTest](https://docs.pytest.org/):

For host machine developers, there is a helper script defined in the Pipfile:

```sh
# Host machine via Pipenv:
> pipenv run tests
```

For containerized developers, you can run tests locally against your container using
the same process as described above using the `docker-compose run` approach.

_**PLEASE NOTE**: If you were to simply run the test commands, you'd be running tests
against your development database and environment._

A convenience script has been provided that uses the `docker-compose.test.yml` configuration
for the `api` service to ensure you are **sourcing your config from the test.env file**:


```sh
# Convenient short-hand to that:
> ./scripts/tests.sh
```

Internally, this script is building a docker compose configuration from the base and test
files. This allows you to specify a different database target (e.g., "coda_test" instead of
"coda_dev") in your `test.env` environment file and ensure your tests are run there.

If coda_test does not exist, just run

```sh
> ./scripts/psql.sh
> create database coda_test;
```

Then, the `tests.sh` script should work.



## Ingest Application

Ingestion from the CHOP Data Warehouse to the CODA database is handled
by the **Ingestion Command-line (CLI) Script**. The ingestion script
leverages a shared configuration with the API to connect to the application
database, but brings in additional configuration to source data from the
CHOP Data Warehouse.

The script is a command-line tool that runs through batch processing tasks.
It is designed to be configured by a sysadmin to run on a regular cadence
(e.g., via a `cron` job or similar).

The Ingest Application requires the following in order to run:

* A connection to the CHOP Data Warehouse – specified in the environment
or on the command-line via the `--dw` option
* A connection to the CODA Application Database – specified in the environment
or on the command-line via the `--db` option
* Access to the API code – handled typically by defining the `PYTHONPATH`
env var to the project root

Failing these, the application will exit with a non-zero exit code.

### Ingest Features

The Ingest application is a command-line application leveraging the
Python third-party library [click_](https://click.palletsprojects.com/en/7.x/).


### Ingest Development

The environment you develop in locally should match the API environment.
Developers working with code in a locally-installed Python Virtual
Environment will run the command as follows:

```bash
# Via Pipfile script:
$> pipenv run ingest --help

## Within the PipEnv shell:
$> pipenv shell
(coda-api) $> ingest/coda-ingest --help
```

Developers choosing to work within a Docker context, must prepare:

1. Launch the **api** service via `docker-compose`. This will
automatically bring up the **db** service.
2. Launch the **dw** service via `docker-compose`. This is required
to pull source data.
3. Run the ingest script within the running **api** service container.

In practice, this will look something like this:

```bash
$> docker-compose up api
...

$> docker-compose up dw
...

$> docker-compose run api ingest/coda-ingest --help
```

