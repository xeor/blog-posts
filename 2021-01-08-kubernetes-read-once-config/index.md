---
title: Kubernetes pod - read-once config
date: 2021-01-08
summary: Make sure the pod can only read a configuration only one time
tags: ['kubernetes', 'security']
---

## Table of Contents

## Introduction

Some times you need to stores sensitive information in a configuration, your app would need it to start.
But what if you could populate the data needed, then remove it once the application has started...
Thats what this post is all about. My quest to store as little sensitive data as possible in my home-assistant setup.

As a bonus, but not mandatory, we will also encrypt the configuration at rest. That is in case the configuration file itself is stored on an untrusted medium, like on a shared nfs storage..

To see an implementation of all this, take a look at https://github.com/xeor/container-hass. It contains most of what you need, except the kubernetes parts. I will go through that as well in this post, so it's better to read this post :)

Also note that this implememtation is for `alpine` using `s6-overlay`, but it should work on other distroes as well with little modifications, most of the logic is the same. To keep it simple the alpine and s6-overlay setup is out of scope of this post.

### Prerequisites

* Knowledge about containers, alpine and some s6-overlay
* An application that doesnt care if it's config is deleted after it is started.
* An application/container that doesnt need `SYS_PTRACE` (usually they don't), but if you do, this trick wont be that helpful, since someone could spy on the application and steal the secret config from it direcly instead.

## Actual content

### The container

There is little you would need to do to the Containerfile. Whats important is to include gnupg and some scripts.

:::tip
If you don't care about the encryption-part, only about the `read-once` part. Take what you need, there is a lot of things that can be left out..
:::

```dockerfile
FROM alpine

RUN apk add gnupg

# Copy everything from the directory `root`, and merge it into the container
COPY root /
```

Inside the root directory, we put some helper scripts..

#### /usr/bin/secret_encrypt

This is a simple helper-script to encrypt a file `/config/secrets.yaml` into the file `/config/secrets.yaml.enc`. This will be helpfull to test and bootstrap your setup, and in case you want to change the data in the encrypted file.

```shell{codeTitle: "In root/usr/bin/secret_encrypt"}{file=./secret_encrypt}
```

#### /usr/bin/secret_decrypt

This file is a helper, and is used for the decryption when the container starts.
It should be called at container start to make sure the decrypted version of the config is ready when the appliation reads it.

```shell{codeTitle: "In root/usr/bin/secret_decrypt"}{file=./secret_decrypt}
```

#### /usr/bin/secret_delete_waiter

This file might need to be modified to your need. This version is tailored to work with the intofy utility that comes with busybox.
See `/etc/cont-init.d/00-secrets-handler` below for how it is used. It's single goal is work with `inotifyd` to delete a file once an event (read) has happened.

```shell{codeTitle: "In root/usr/bin/secret_delete_waiter"}{file=./secret_delete_waiter}
```

#### /etc/cont-init.d/00-secrets-handler

Placing this script inside `cont-init.d` when using `s6-overlay` makes it start before every service.
It's goal is simple,

1. Decrypt the secret
2. Put a watcher in the background that will delete the config-file once the application is started.

```shell{codeTitle: "In root/etc/cont-init.d/00-secrets-handler"}{file=./00-secrets-handler}
```