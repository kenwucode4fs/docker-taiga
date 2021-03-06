# What is Taiga?

Taiga is a project management platform for startups and agile developers & designers who want a simple, beautiful tool that makes work truly enjoyable.

> [taiga.io](https://taiga.io)

# How to use this image

There is an example project available at [benhutchins/docker-taiga-example](https://github.com/benhutchins/docker-taiga-example) that provides base configuration files available for you to modify and allows you to easily install plugins. I recommend you clone this repo and modify the files, then use it's provided scripts to get started quickly.

    git clone https://github.com/benhutchins/docker-taiga-example.git mytaiga && cd mytaiga
    vi taiga-conf/local.py # configuration for taiga-back
    vi taiga-conf/conf.json # configuration for taiga-front
    vi docker-compose.yml # update environmental variables
    docker-compose up

Or to use this container directly, run:

    docker run -itd \
      --link taiga-postgres:postgres \
      -p 80:80 \
      -e TAIGA_HOSTNAME=taiga.mycompany.net \
      -v ./media:/usr/src/taiga-back/media \
      benhutchins/taiga

See `Summarize` below for a complete example. Partial explanation of arguments:

- `--link` is used to link the database container. See `Configure Database` below for more details.

Once your container is running, use the default administrator account to login: username is `admin`, and the password is `123123`.

If you're having trouble connecting, make sure you've configured your `TAIGA_HOSTNAME`. It will default to `localhost`, which almost certainly is not what you want to use.

## Extra configuration options

Use the following environmental variables to generate a `local.py` for [taiga-back](https://github.com/taigaio/taiga-back).

  - `-e TAIGA_HOSTNAME=` (**required** set this to the server host like `taiga.mycompany.com`)
  - `-e TAIGA_SSL=True` (see `Enabling HTTPS` below)
  - `-e TAIGA_SECRET_KEY` (set this to a random string to configure the `SECRET_KEY` value for taiga-back; defaults to an insecure random string)
  - `-e TAIGA_SKIP_DB_CHECK` (set to skip the database check that attempts to automatically setup initial database)
  - `-e TAIGA_ENABLE_EMAIL=True` (see `Configuring SMTP` below)

*Note*: Database variables are also required, see `Using Database server` below. These are required even when using a container for your database.

## Configure Database

The above example uses `--link` to connect Taiga with a running [postgres](https://registry.hub.docker.com/_/postgres/) container. This is probably not the best idea for use in production, keeping data in docker containers can be dangerous.

### Using Docker container

If you want to run your database within a docker container, simply start your database server before starting your Taiga container. Here is a simple example pulled from [postgres](https://registry.hub.docker.com/_/postgres/)'s guide.

    docker run --name taiga-postgres -e POSTGRES_PASSWORD=mypassword -d postgres

### Using Database server

Use the following, **required**, environment variables for connecting to another database server:

 - `-e TAIGA_DB_NAME=...` (defaults to `postgres`)
 - `-e TAIGA_DB_HOST=...` (defaults to the address of a linked `postgres` container)
 - `-e TAIGA_DB_USER=...` (defaults to `postgres)`)
 - `-e TAIGA_DB_PASSWORD=...` (defaults to the password of the linked `postgres` container)

Taiga will attempt to connect to the DB server for up to TAIGA_DB_CONNECT_TIMEOUT seconds before exiting.

If the `TAIGA_DB_NAME` specified does not already exist on the provided PostgreSQL server, it will be automatically created the the Taiga's installation scripts will run to generate the required tables and default demo data.

An example `docker run` command using an external database:

    docker run \
      --name mytaiga \
      -e TAIGA_DB_HOST=10.0.0.1 \
      -e TAIGA_DB_USER=taiga \
      -e TAIGA_DB_PASSWORD=mypassword \
      -itd \
      benhutchins/taiga

## Taiga Events

Taiga has an optional dependency, [taiga-events](https://github.com/taigaio/taiga-events). This adds additional usability to Taiga. To support this, there is an optional docker dependency available called [docker-taiga-events](https://github.com/benhutchins/docker-taiga-events). It has a few dependencies of its own, so this is how you run it:

```bash
# Setup RabbitMQ and Redis services
docker run --name taiga-redis -d redis:3
docker run --name taiga-rabbit -d --hostname taiga rabbitmq:3

# Start up a celery worker
docker run --name taiga-celery -d --link taiga-rabbit:rabbit celery

# Now start the taiga-events server
docker run --name taiga-events -d --link taiga-rabbit:rabbit benhutchins/taiga-events
```

Then append the following arguments to your `docker run` command running your `benhutchins/taiga` container:

    --link taiga-rabbit:rabbit
    --link taiga-redis:redis
    --link taiga-events:events

See the example below in `Summarize` section for an example `docker run` command.

## Enabling HTTPS

If you want to enable support for HTTPS, you'll need to specify all of these
additional arguments to your `docker run` command.

  - `-e TAIGA_SSL=True`
  - `-v $(pwd)/ssl.crt:/etc/nginx/ssl/ssl.crt:ro`
  - `-v $(pwd)/ssl.key:/etc/nginx/ssl/ssl.key:ro`

Or create a folder called `ssl`, place your `ssl.crt` and `ssl.key` inside
this directory and then mount it with:

  - `-e TAIGA_SSL=True`
  - `-v $(pwd)/ssl/:/etc/nginx/ssl/:ro`

### HTTPS behind reverse proxies

If you have a reveser proxy that is already handling https set `TAIGA_SSL_BY_REVERSE_PROXY`:

- `-e TAIGA_SSL_BY_REVERSE_PROXY=True`

The value of `TAIGA_SSL` will then be ignored and taiga will not handle https,
it will however set all links to https.

## Configuring SMTP

If you want to use an SMTP server for emails, you'll need to specify all of
these additional arguments to your `docker run` command.

- `-e TAIGA_ENABLE_EMAIL=True`
- `-e TAIGA_EMAIL_FROM=no-reply@taiga.mycompany.net`
- `-e TAIGA_EMAIL_USE_TLS=True` (only if you want to use tls)
- `-e TAIGA_EMAIL_HOST=smtp.google.com`
- `-e TAIGA_EMAIL_PORT=587`
- `-e TAIGA_EMAIL_USER=me@gmail.com`
- `-e TAIGA_EMAIL_PASS=super-secure-pass phrase thing!`

**Note:** This can also be configured directly inside your own config file, look
at [this example](https://github.com/benhutchins/docker-taiga-example/tree/master/simple)'s
setup where it specifies a `taiga-conf/local.py`. You can then configure this
and other [settings available in Taiga](https://github.com/taigaio/taiga-back/blob/master/settings/local.py.example).

## Volumes

Uploads to Taiga go to the media folder, located by default at `/usr/src/taiga-back/media`.

Use `-v /my/own/media:/usr/src/taiga-back/media` as part of your docker run command to ensure uploads are not lost easily.

## Summarize

To sum it all up, if you want to run Taiga without using docker-compose, run this:

    docker run --name taiga-postgres -d -e POSTGRES_PASSWORD=password postgres
    docker run --name taiga-redis -d redis:3
    docker run --name taiga-rabbit -d --hostname taiga rabbitmq:3
    docker run --name taiga-celery -d --link taiga-rabbit:rabbit celery
    docker run --name taiga-events -d --link taiga-rabbit:rabbit benhutchins/taiga-events

    docker run -itd \
      --name taiga \
      --link taiga-postgres:postgres \
      --link taiga-redis:redis \
      --link taiga-rabbit:rabbit \
      --link taiga-events:events \
      -p 80:80 \
      -e TAIGA_HOSTNAME=$(docker-machine ip default) \
      -v ./media:/usr/src/taiga-back/media \
      benhutchins/taiga

Again, you can avoid all this by using [benhutchins/docker-taiga-example](https://github.com/benhutchins/docker-taiga-example) and then just run `docker-compose up`.

# Using this git repo

If you want to get the latest and greatest, you can clone this repo, then update
Taiga to the latest by running:

```bash
git clone https://github.com/benhutchins/docker-taiga.git && cd docker-taiga/
git submodule update --init --remote
docker-compose up -d # this will build and then start taiga locally
```
