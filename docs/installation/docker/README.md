### Install OpenProject with Docker

[Docker][docker] is a way to distribute self-contained applications easily. We
provide a Docker image for the Community Edition that you can very easily
install and upgrade on your servers. However, contrary to the manual or
package-based installation, your machine needs to have the Docker Engine
installed first, which usually requires a recent operating system. Please see
the [Docker Engine installation page][docker-install] if you don't have Docker
installed.

Also, please note that the Docker image is quite new and might not support all
the options that the package-based or manual installation provides.

[docker]: https://www.docker.com/
[docker-install]: https://docs.docker.com/engine/installation/

### Quick Start

The fastest way to get an OpenProject instance up and running is to run the
following command:

    docker run -it -p 8080:80 -e SECRET_KEY_BASE=secret openproject/community:latest

This assumes you want to run the latest stable version. Please see the docker tags listing for the tags we provide:
https://hub.docker.com/r/openproject/community/tags

In general, we provide the following tags:

 - `latest`: Points to the latest stable version released
 - `X`: The latest stable version for the given major release. e.g., (8) for 8.x.x release, (9) for 9.x.x -
        You will never upgrade between major versions automatically with this
 - `X.Y`: The latest stable version for the given major.minor release. e.g, (8.1) for 8.1.x releases
 - `X.Y.Z`: The pinned version for one specific release version e.g., 9.0.3

This will take a bit of time the first time you launch it, but after a few
minutes you should see a success message indicating the default administration
password (login: `admin`, password: `admin`).

You can then launch a browser and access your new OpenProject installation at
<http://localhost:8080>. Easy!

To stop the container, simply hit CTRL-C.

Note that the above command will not daemonize the container and will display
the logs to your terminal, which helps with debugging if anything goes wrong.
For normal usage you probably want to start it in the background, which can be
achieved with the `-d` flag:

    docker run -d -p 8080:80 -e SECRET_KEY_BASE=secret openproject/community:latest

### Recommended usage

The one-liner above is great to get started quickly, but if you want to run
OpenProject in production you will likely want to ensure that your data is not
lost if you restart the container.

To achieve this, we recommend that you create a directory on your host system
where the Docker Engine is installed (for instance: `/var/lib/openproject`)
where all this data will be stored.

You can use the following commands to create the local directories where the
data will be stored across container restarts, and start the container with
those directories mounted:

    sudo mkdir -p /var/lib/openproject/{pgdata,static}

    docker run -d -p 8080:80 --name openproject -e SECRET_KEY_BASE=secret \
      -v /var/lib/openproject/pgdata:/var/openproject/pgdata \
      -v /var/lib/openproject/static:/var/openproject/assets \
      openproject/community:latest

Since we named the container, you can now stop it by running:

    docker stop openproject

And start it again:

    docker start openproject

If you want to destroy the container, run the following commands

    docker stop openproject && docker rm openproject

### Configuration

OpenProject is usually configured through a YAML file, but with the Docker
image you need to pass all configuration through environment variables. You can
overwrite any of the values usually found in the standard YAML file by using
environment variables as explained in the [CONFIGURATION][configuration-doc]
documentation.

Environment variables can be either passed directly on the command-line to the
Docker Engine, or via an environment file:

    docker run -d -e KEY1=VALUE1 -e KEY2=VALUE2 ...
    docker run -d --env-file path/to/file ...

[configuration-doc]: https://github.com/opf/openproject/blob/dev/docs/configuration/configuration.md

### SMTP configuration

By default, the docker container will try to send emails via the local
`postfix` daemon. However emails sent this way are more than likely to fail or
end up in the spam inbox of your users. We recommend using an external SMTP
server to send your emails.

A good choice is [SendGrid](https://www.sendgrid.com/), which offers a free plan
with up to 12000 emails per month. Just sign up on the website, and once your
account is provisioned, generate a new API key and copy it somewhere (it looks
like `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`).
You can also just use your SendGrid username and password, but this is less
secure.

You can then configure OpenProject with the following additonal environment
variables (with SendGrid, the `SMTP_USER_NAME` is always `apikey`. Just replace
`SMTP_PASSWORD` with the API key you've generated and you should be good to
go):

    docker run -d \
        -e EMAIL_DELIVERY_METHOD=smtp \
        -e SMTP_ADDRESS=smtp.sendgrid.net \
        -e SMTP_PORT=587 \
        -e SMTP_DOMAIN=my.domain.com \
        -e SMTP_AUTHENTICATION=login \
        -e SMTP_ENABLE_STARTTLS_AUTO=true \
        -e SMTP_USER_NAME="apikey" \
        -e SMTP_PASSWORD="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" \
        ...

You can adjust those settings for other SMTP providers, such as GMail,
Mandrill, etc. Please refer to the documentation of the corresponding provider
to see what values should be used.

### FAQ

* Can I use SSL?

The current Docker image does not support SSL by default. Usually you would
already have an existing Apache or NginX server on your host, with SSL
configured, which you could use to set up a simple ProxyPass rule to direct
traffic to the container.

If you really want to enable SSL from within the container, you could try
mounting a custom apache2 directory when you launch the container with `-v
my/apache2/conf:/etc/apache2`. This would entirely replace the configuration
we're using.


* Can I use an external (MySQL or PostgreSQL) database?

Yes. You can simply pass a custom `DATABASE_URL` environment variable on the
command-line, which could point to an external database. You can even choose to
use MySQL instead of PostgreSQL if you wish. Here is how you would do it:

    docker run -d ... -e DATABASE_URL=mysql2://user:pass@host:port/dbname openproject/community:8

The container will make sure that the database gets the migrations and demo
data as well.

* I don't want the all-in-one installation. Can I still use the image to launch a specific process?

Yes, you can do so by passing a command when you launch the container. By default the container will run `./docker/supervisord`, but you can override this with `./docker/web`, `./docker/worker`, `./docker/cron` to launch the individual services separately (e.g. in a docker-compose file). Please note that in this configuration you will have to setup the external services (postgres, memcached, email sending) by yourself.

Example:

      docker run -d -e DATABASE_URL=xxx ... openproject/community:latest ./docker/web
      docker run -d -e DATABASE_URL=xxx ... openproject/community:latest ./docker/worker
