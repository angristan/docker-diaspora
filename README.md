# Diaspora*

[![pipeline status](https://gitlab.com/angristan/docker-diaspora/badges/master/pipeline.svg)](https://gitlab.com/angristan/docker-diaspora/pipelines) ![https://hub.docker.com/r/angristan/diaspora/](https://img.shields.io/microbadger/image-size/angristan/diaspora.svg?maxAge=3600&style=flat-square) ![https://hub.docker.com/r/angristan/diaspora/](https://img.shields.io/microbadger/layers/angristan/diaspora.svg?maxAge=3600&style=flat-square) ![https://hub.docker.com/r/angristan/diaspora/](https://img.shields.io/docker/pulls/angristan/diaspora.svg?maxAge=3600&style=flat-square) ![https://hub.docker.com/r/angristan/diaspora/](https://img.shields.io/docker/stars/angristan/diaspora.svg?maxAge=3600&style=flat-square)

![Diaspora Logo](https://i.imgur.com/J50tnoC.png)

[Diaspora](https://diasporafoundation.org/) is a nonprofit, user-owned, distributed social network that is based upon the free Diaspora software. Diaspora consists of a group of independently owned nodes (called pods) which interoperate to form the network.

This image is automatically built by [GitLab CI](https://gitlab.com/angristan/docker-diaspora/pipelines) and pushed to the [Docker Hub](https://hub.docker.com/r/angristan/diaspora/).

**I won't update this image anymore. Feel free to open PRs or to fork the repo.**

## Features

- Based on the official [ruby:2.4-slim-stretch](https://hub.docker.com/_/ruby/) image
- Running the latest stable version of [diaspora/diaspora](https://github.com/diaspora/diaspora)
- Ran as an unprivileged user (see `UID` and `GID`)

### Build-time variables

- **`DIASPORA_VER`**: Diaspora version (`0.7.9.0`)
- **`GID`**: group id *(default: `942`)*
- **`UID`**: user id *(default: `942`)*

### Volumes

- **`/diaspora/public`**: location of the assets and user uploads

## Usage

The image will not work *as-is*, it requires a command to start and other services.

### Configuration files

**Before** doing anything, copy the [database.yml.exemple](https://github.com/diaspora/diaspora/blob/develop/config/database.yml.example) to your current folder and add the correct values (which should be the postgres ENVIRONMENT viariables below).

Do the same with [diaspora.yml.example](https://github.com/diaspora/diaspora/blob/develop/config/diaspora.yml.example) and **read it** completely. Diaspora won't work with the defaults.

FYI you will need to modify these at least:

- environment.url
- environment.certificate_authorities: `/etc/ssl/certs/ca-certificates.crt`
- environment.redis: `redis://redis` (if you follow the docker-compose below)
- server.listen: `0.0.0.0:3000`
- server.rails_environment: `production`

### Docker Compose

Here is an exemple `docker-compose.yml`:

```yaml
version: '2.3'

services:
  postgres:
    container_name: diaspora_postgres
    image: postgres:9.6-alpine
    restart: always
    volumes:
      - ./postgres:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=diaspora
      - POSTGRES_PASSWORD=diaspora
      - POSTGRES_DB=diaspora_production

  redis:
    container_name: diaspora_redis
    image: redis:4.0-alpine
    restart: always
    volumes:
      - ./redis:/data

  unicorn:
    container_name: diaspora_unicorn
    image: angristan/diaspora:0.7.9
    restart: always
    command: bin/bundle exec unicorn -c config/unicorn.rb -E production
    volumes:
      - ./data:/diaspora/public/
      - ./diaspora.yml:/diaspora/config/diaspora.yml
      - ./database.yml:/diaspora/config/database.yml
    depends_on:
      - postgres
      - redis

  sidekiq:
    container_name: diaspora_sidekiq
    image: angristan/diaspora:0.7.9
    restart: always
    command: bin/bundle exec sidekiq
    volumes:
      - ./data:/diaspora/public/
      - ./diaspora.yml:/diaspora/config/diaspora.yml
      - ./database.yml:/diaspora/config/database.yml
    depends_on:
      - postgres
      - redis

  nginx:
    container_name: diaspora_nginx
    image: nginx:stable
    restart: always
    volumes:
      - ./nginx-vhost.conf:/etc/nginx/conf.d/default.conf:ro
      - ./data:/var/www/html
    ports:
      - 127.0.0.1:80:80
```

We need a Nginx container to server the uploads and assets, as Unicorn doesn't do it.

Here is an example Nginx vhost:

```nginx
server {
    listen 80;

    root /var/www/html;

    client_max_body_size 5M;
    client_body_buffer_size 256K;

    try_files $uri @diaspora;

    location /assets/ {
        expires max;
        add_header Cache-Control public;
    }

    location @diaspora {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_pass http://unicorn:3000;
    }
}
```

I assume you're using another container for HTTPS, but feel free to use this as a base if it's not the case.

### Installation

When running the instance for the first time, run this command to setup the database:

```sh
docker-compose run --rm unicorn bin/rake db:create db:migrate
```

Then compile the assets:

```sh
docker-compose run --rm unicorn bin/rake assets:precompile
```

You can now lauch your pod!

```sh
docker-compose up -d
```

### Update

Modify the versions in your `docker-compose.yml`, then pull the new images:

```sh
docker-compose pull
```

Update the database:

```sh
docker-compose run --rm unicorn bin/rake db:migrate
```

Then compile the assets:

```sh
docker-compose run --rm unicorn bin/rake assets:precompile
```

Recreate containers with new images:

```sh
docker-compose up -d
```
