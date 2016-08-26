# Nginx

[![](https://images.microbadger.com/badges/image/slickframework/nginx.svg)](https://microbadger.com/images/slickframework/nginx "Get your own image badge on microbadger.com")

This is a **very flexible** nginx image, with support for several types of applications.

Based on [helderco's work](https://github.com/helderco/docker-nginx).

## Virtual host templates

Thanks to the provided [entrypoint.sh](https://github.com/helderco/docker-nginx/blob/master/entrypoint.sh) and [Jinja](http://jinja.pocoo.org), we're able to have a virtual host configurable at runtime using environment variables. This is a great way to reuse the same image for many different purposes.

To get a list of the available types of apps included:

    docker run -t --rm helder/nginx ls -1 /etc/nginx/templates/vhost


## Basic usage

Run a static website (default):

    docker run -d -v "$PWD:/usr/src/app" -p 80:80 helder/nginx web_static

If you don't provide `/usr/src/app`, Nginx's welcome message is show:

    docker run -d -p 80:80 helder/nginx nginx_welcome

If you want your files in a different path:

    docker run -d -v "$PWD:/web" -e CONF_ROOT=/web -p 80:80 helder/nginx web_static

### How to chose an app

Anything listed in the `templates/vhost` folder of nginx is an app (you can add your own).
To define which you want, set the environment variable `CONF_APP`.

## Bypass template system

When an app is chosen, the rendered template will be written to `/etc/nginx/sites-enabled/default.conf`.
If the file exists, nothing is done so if you don't want to use the template system,
you can bypass it completely by adding your own file.

    FROM helder/nginx
    COPY mysite.conf /etc/nginx/sites-enabled/default.conf

## How to work with the templates

Thanks to Jinja, templates are extensible. Here's an overview of the hierarchy:

    .
    └── default
        └── php
            └── slick

All of these are valid values for `CONF_APP`.

### Configurable with environment variables

Any environment variable starting with `CONF_` will be provided to the template, without
the prefix and lowercase.

E.g., `CONF_MY_SETTING` would be `my_setting` in the template.

#### Default variables

Variables found in a template, are also available for every template that extends it.

* **`CONF_APP`**

    *Default: `default`*

    Type of app. See list of files in `/etc/nginx/templats/vhost/` for possible values.

    **Example:** `CONF_APP=php`

* **`CONF_PROJECT`**

    *Default: `$CONF_APP`*

    Not really used anywhere but can be used in your own templates and will affect de value
    of `CONF_SOCKET`.

* **`CONF_SOCKET`**

    *Default: `$CONF_PROJECT`*

    If you want a unix socket in the upstream, you can override it's name. Leave the `.sock`
    extension out because it will be added by default by `CONF_UPSTREAM`.

    **Example:** `CONF_SOCKET=php-fpm`

* **`CONF_UPSTREAM`**

    *Default: `unix:/var/run/${CONF_SOCKET}.sock`*

    The upstream socket is shared from a volume (e.g. php) by default. If that's not
    the case, override this setting.

    **Example:** `CONF_UPSTREAM=php:9000`

* **`CONF_ROOT`**

    *Default: `/usr/src/app`*

    Location for the `root` directive in Nginx. No trailing slash because the actual value of the
    `root` directive will be `${CONF_ROOT}/${CONF_PUBLIC}`.

    **Example:** `CONF_ROOT=/var/www`

* **`CONF_PUBLIC`**

    *Default: empty*

    Value to append to `CONF_ROOT` for the nginx's `root` directive. No leading slash, see
    `CONF_ROOT` description. Useful in a project with a subfolder as public facing (e.g. symfony).

    **Example:** `CONF_PUBLIC=web`

* **`CONF_SERVER_NAME`**

    *Default: `_`*

    If this is the only vhost in this container running nginx, there's no sense in setting your
    `name_server`. But if you need it somehow...

* **`CONF_ACCESS_LOG`**

    *Default: empty*

    The default is to send access logs to *stdout*, but you can use this variable to send your
    app's logs to another file or shut them off entirely.

    **Example:** `CONF_ACCESS_LOG=off`


## Add your own template

Build a new image with your own jinja template.

    FROM helder/nginx
    COPY myapp.jinja /etc/nginx/templates/vhost/myapp

## Full example with docker compose

PHP 7.0 project with public facing folder in `webroot`:

    version: '2'

    services:
      web:
        image: slickframework/nginx
        ports:
          - 80
        volumes_from:
          - php
        environment:
          - CONF_APP=php
          - CONF_PUBLIC=webroot
          - CONF_UPSTREAM=php:9000

      php:
        image: slickframework/php:7.0
        links:
          - db
        volumes:
          - ./:/usr/src/app
        working_dir: /usr/src/app

      db:
        image: mysql:5.7
        ports:
          - 3306

