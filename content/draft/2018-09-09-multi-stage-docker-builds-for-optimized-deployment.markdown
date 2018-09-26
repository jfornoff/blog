---
categories:
- docker
date: "2018-09-09T18:07:16Z"
draft: true
title: Multi-stage Docker builds for optimized deployment
---

When trying to deploy an application, there is a number of desirable goals for the artifact:
* it should be as small as possible
* it should not contain any sensitive data (e.g., source code)
* the build process should be reproducible
* minimal dependencies of the build process

Today, I'll show you how to achieve all these goals using [**Docker multi-stage builds**](https://docs.docker.com/develop/develop-images/multistage-build/). Docker provides its own artifacts known as **Images**, which define a file system, environment and commands to run when a **Container** is started from it.
This post assumes knowledge of `docker build` and basic Dockerfile syntax.

# The problem with Docker build

Standard Dockerfiles specify a base image in the `FROM` statement.
To ensure a high level of reproducibility, building the application inside the Docker build is very common.
However, this requires special build tools and additional dependencies, which bloat the image.

Furthermore, there are very minimal Linux images such as `alpine` that provide a very small footprint in terms of size.

This points into a direction where separation between the **building** an artifact and defining its **runtime** environment would be useful. Docker multi-stage builds make this possible.

# Example application

For the purpose of demonstration, we will use a an example from one of my side projects.
It is an Elixir application where the release process looks as follows:
* Setting up the build environment with dependencies
* Building an Erlang release using Elixir's build tool `mix`
* Running the release when the container starts

{{< highlight Docker linenos >}}
# Build
FROM elixir:1.7.3-alpine as buildstep

RUN apk update && apk add git

ENV MIX_ENV prod

RUN mix local.hex --force
RUN mix local.rebar --force

RUN mkdir /app
WORKDIR /app

COPY . .
RUN mix deps.get

RUN mix release

# Release
FROM alpine:3.7

ENV REPLACE_OS_VARS true

RUN mkdir /app
RUN apk update && apk add bash openssl

COPY --from=buildstep /app/_build/prod/rel/standupbot/ /app

CMD /app/bin/standupbot foreground
{{< / highlight >}}




