---
sidebar_position: 2
---

# Quickstart

To jump right in you can start by creating a new project using the composer create-project command:

```bash
$ composer create-project forestcitylabs/framework-standard-install fcl-framework
```

This will create a new project based on this repo in the `fcl-framework` folder on your machine.

You can then create a new `.env` file by copying the existing `.env.dist` file and immediately start a development environment using [podman](https://podman.io/) or [docker](https://www.docker.com/) compose.

```bash
$ cp .env.dist .env
$ docker compose up -d
```

Once your site is built and running it will be available at [http://localhost:8080](http://localhost:8080)!

From here it is up to you to start building your site! A good place to start is the [GraphQL](GraphQL/index.md) documentation or the [routing](Routing.md) documentation.
