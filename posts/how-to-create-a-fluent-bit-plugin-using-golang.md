---
title: "How to create a Fluent Bit MySQL output plugin using golang"
date: 2023-03-29
draft: true
---

# How to create a Fluent Bit MySQL output plugin using golang

## Introduction

Some people while migrating from Fluentd to Fluent Bit might have missing some of the plugins that they were using and although Fluent Bit comes with a range of built-in plugins, sometimes we might need to create a custom plugin to meet ours specific requirements. In this tutorial, we'll show you how to create a custom MySQL output plugin for Fluent Bit using the [Calyptia Plugin library](https://github.com/calyptia/plugin). 

## Prerequisites

- [Golang](https://golang.org) installed
- [Docker](https://docker.com) installed

We will be using Docker to run Fluent Bit and MySQL without the need to install it on your machine.

## Hands-on

So lets get started and create ou new plugin. First, we need to create a new directory and initialize a new go module.

```bash
mkdir fluent-bit-mysql-output-plugin
cd fluent-bit-mysql-output-plugin
go mod init github.com/gabrielbussolo/fluent-bit-mysql-output-plugin
```

I will be creating a new file called `plugin.go` and import the `github.com/calyptia/plugin` package from there.

```go
package main

import (
    "github.com/calyptia/plugin"
)
```

There few things we need to be aware on this file. First we need to implement the `plugin.OutputPlugin` interface that consists in 2 methods:

```go
// OutputPlugin interface to represent an output fluent-bit plugin.
type OutputPlugin interface {
	Init(ctx context.Context, fbit *Fluentbit) error
	Flush(ctx context.Context, ch <-chan Message) error
}
```

The `Init(ctx context.Context, fbit *Fluentbit)` will be called by fluent bit to start the plugin, so its here were we should create what we need for ou plugin to work, like get the config keys from the `fluent-bit.conf`, instanciate the MySQL, log, etc. The `Flush(ctx context.Context, ch <-chan Message)` will be called by fluent bit to flush the messages that are coming from the input plugin, it's were our logic will work.

We also two other functions to be created they are `init()` and `main()`. The `init()` will be called by the go runtime and it will register our plugin on fluent bit and the `main()` it's a requiement for the go runtime to create a binary so basically will be empty.

Following those requirements we can create our plugin like this:

```go
package main

import (
    "context"
    "github.com/calyptia/plugin"
)

func init() {
}

type MySQL struct {
}

func (m *MySQL) Init(ctx context.Context, fbit *plugin.Fluentbit) error {
    return nil
}

func (m *MySQL) Flush(ctx context.Context, ch <-chan plugin.Message) error {
    return nil
}

func main() {
}
``` 

Let's start to fill the gaps. First, we need to register our plugin on fluent bit, so we can do that by calling the `plugin.RegisterOutput` function on the `init()` function.

```go
func init() {
    plugin.RegisterOutput("mysql", "mysql output plugin written in go", &MySQL{})
}
```

Now our plugin will be registered by fluent bit. Next, we need to implement the `Init(ctx context.Context, fbit *Fluentbit)` method. This method will be called by fluent bit to start our plugin, so we can get the config keys from the `fluent-bit.conf` and create the MySQL connection. But before that, let's just see something working, lets log some fluent bit configs.

```go
func (m *MySQL) Init(ctx context.Context, fbit *plugin.Fluentbit) error {
    fbit.Logger.Info("Address: %s", fbit.Conf.String("Address"))
    fbit.Logger.Info("User: %s", fbit.Conf.String("User"))
}
```

Let's try it out. We need two config files when we are creating our own plugins, the usual `fluent-bit.conf` and another file that i will call `plugins.conf` that will be used to register our plugin on fluent bit. So let's create those files.

`plungins.conf`:
```conf
[PLUGINS]
    Path /fluent-bit/etc/mysql-output-plugin.so
``` 

`fluent-bit.conf`:
```conf
[SERVICE]
    Flush           1
    Log_Level       info
    plugins_file    /fluent-bit/etc/plugins.conf

[OUTPUT]
    Name mysql
    Address localhost:3306
    User root
```

To prevent architecture limitation as im using a mac to create this tutorial, i will be using a `Dockerfile` to compile and run our Fluent Bit. 

`Dockerfile`:
```Dockerfile
FROM golang:1.20 AS builder

# cmetrics needs to be installed in the builder image
ARG CMETRICS_VERSION=0.5.8
ENV CMETRICS_VERSION=${CMETRICS_VERSION}
ARG CMETRICS_RELEASE=v0.5.8
ENV CMETRICS_RELEASE=${CMETRICS_RELEASE}

ARG PACKAGEARCH=amd64
ENV PACKAGEARCH=${PACKAGEARCH}

WORKDIR /plugin

COPY ./go.mod ./
COPY ./go.sum ./

RUN go mod download
RUN go mod verify

COPY . .

ADD https://github.com/fluent/cmetrics/releases/download/${CMETRICS_RELEASE}/cmetrics_${CMETRICS_VERSION}_${PACKAGEARCH}-headers.deb external/
ADD https://github.com/fluent/cmetrics/releases/download/${CMETRICS_RELEASE}/cmetrics_${CMETRICS_VERSION}_${PACKAGEARCH}.deb external/
RUN dpkg -i external/*.deb

RUN go build -trimpath -buildmode=c-shared -o mysql-output-plugin.so ./plugin.go

FROM fluent/fluent-bit

COPY --from=builder /plugin/mysql-output-plugin.so /fluent-bit/etc/
COPY ./fluent-bit.conf /fluent-bit/etc/
COPY ./plugins.conf /fluent-bit/etc/

ENTRYPOINT [ "/fluent-bit/bin/fluent-bit" ]
CMD [ "/fluent-bit/bin/fluent-bit", "-c", "/fluent-bit/etc/fluent-bit.conf" ]
```

It's pretty straightforward, we are using the `golang:1.20` image to build our plugin and the `fluent/fluent-bit` to run it. We are also installing the `cmetrics` dependency that is required by the `plugin` package.

Now we can build and run our Fluent Bit.

```bash
# build the image
docker build --progress=plain --platform linux/amd64  -t fluent-bit-mysql .
# run the container
docker run -it --rm --name fluent-bit-mysql fluent-bit-mysql
```

With this we should see something like:
```Bash

```