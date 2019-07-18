# Nginx - Development Image (Forked by VictxrLarixs)

> Don't use this for production!

This is a tiny development docker image for [Nginx](https://www.nginx.com/). Used for my
local dev environment.

`docker pull vnought/nginx-dev`

Either use a Dockerfile to customize per-project, docker-compose, or use this one-liner
from your project:

```
docker run -d -v "$(pwd):/www" \
              -v "$(pwd)/conf/vhost.conf:/etc/nginx/conf.d/vhost.conf" \
              vnought/nginx-dev`
```

App runtimes should be in linked containers (See
[docker-phpfpm-dev](https://github.com/hlissner/docker-phpfpm-dev))


## Dockerfile

```
FROM vnought/nginx-dev
COPY conf/vhost.conf /etc/nginx/conf.d/vhost.conf
```

Then build+start it:

```
docker build -t my-project .
docker run -v $(pwd):/www my-project
```

## docker-compose

```yaml
web:
  image: vnought/nginx-dev
  ports:
    - "8080:80"
  volumes:
    - .:/www
    - ./conf/vhost.conf:/etc/nginx/conf.d/vhost.conf
  links:
    - php

php:
  image: vnought/phpfpm-dev
  volumes:
    - .:/www
```
