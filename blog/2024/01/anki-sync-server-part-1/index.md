---
layout: blog
title: Building and Deploying Anki Sync Server Part 1
date: 2024-01-06T19:39:23-06:00
lastMod: 2024-01-06T19:39:23-06:00
categories: self-hosted
tags:
  - docker
  - anki
  - containers
  - kubernetes
  - self-hosted
  - python
  - github
  - docker-compose
  - makefile
description: First part of a 3 part write up on deploying an Anki Sync Server
disableComments: true
draft: false
---
## Learning Japanese
I'm really trying to learn how to Read and Speak Japanese this year for a future trip to Japan. Part of this goal is to learn to read Hiragana and Katakana and I was having trouble finding a learning method that would help with this in a way I wanted. I wanted a flash card system for learning the characters, but specially to learn both Hiragana and Katakana at the same time. So instead of learning one writing system's characters and sound, I'd learn both the character system at the same time since they're pretty much the same, plus I got it in my head that it'd probably be easier to learn it this way. Now there's tons of applications/programs/etc to actually learn this. Tons of practice writing charts and what not. However I don't specifically want to learn how to write (I know, I know, insane 🤯) I mean, when was the last time I actually hand wrote something? It's all type type type in today's world. For my use cases, I really only need to understand speech, be able to speak back and then be able to read signs, menus, dialogue, etc. So I want a flash card system that can accommodate this, most learning apps probably aren't built this way.

For my requirements, I need the following in an application/system
- Runs on Linux/Android
- Can create my own flashcards that contains images **and** sound so that I can learn the character and the sound for said character
- Regularly maintained

So in comes [Anki](https://apps.ankiweb.net/) a flash card application that runs on multiple platforms (Windows, MacOS, Linux, iOS, and Android) *and* can sync between devices as well. From what I understand, Anki is extremely customizable and easy to get started with. And if youtube is any indication, if it's good enough for Medical Students, it's good enough for my basic language learning needs.

## AnkiWeb
You can use AnkiWeb to sync your Flashcards between devices (e.g. desktop and smartphone), however while I have no intentions of doing anything malicious or illegal, it's very likely that me scrapping the internet for graphics and sound is not exactly within [Acceptable Content](https://ankiweb.net/account/terms#Acceptable%20Content) usage for a Free AnkiWeb sync account. But there is a Self-Hosted option for a [Anki Self-Hosted Sync Server](https://docs.ankiweb.net/sync-server.html) so that's perfect, I can just spin up a docker container and have a local sync server that I can point my devices to.

## The Problem
Doing some research there doesn't appear to be a regularly maintained Docker Image in English from what I can tell. Now I did find one from [AnkiCommunity Github](https://github.com/ankicommunity/anki-sync-server)  however that doesn't look to be actively maintained. I would really prefer not to use out of date software, so how hard could it be to build a docker image for the Anki Sync Server?

## Challenge Accepted
Let's go over the requirements for hosting an Anki Sync Server in my Homelab

1. Running the Anki Sync Server in a Docker Container
2. Deploy the Container Image in my Kubernetes Cluster
3. Automate the Build process for the Docker Image so that I can build new images easily in the future
4. Push said Docker Image to both Github Container Registry **and** Docker Hub Registry
5. Automate Release Version Tagging for the docker images pushed

## RTFM
So let's start with the Sync Server [Documentation](https://docs.ankiweb.net/sync-server.html) Reading this tells me that their Sync Server is part of their Anki application and I can install the application in Linux or install via Python Module. I decided to go with the Python module as I can easily install specific versions via `pip`  so with that in mind, I can build a `Dockerfile` based on the `python:3.9.18` base image.

From the doc, we can see that running the Sync Server really only requires 
1. First User Credentials (requires the environment variable `SYNC_USER1=username:password` additional users can be added via `SYNC_USER2` and so on)
2. Web Port (defaults to `SYNC_PORT=8080`)
3. Sync Directory (where the sync data is stored, this defaults to `/.syncserver` and can be overriden via environment variable `SYNC_BASE`)

In case you just want to check out the repository here you go [Anki SyncServer Repository](https://github.com/sinicide/anki-syncserver)

Now below is what I came up with for the `Dockerfile`
```Dockerfile
FROM python:3.9.18

ENV SYNC_HOST "0.0.0.0"
ENV SYNC_PORT "8080"
ENV SYNC_BASE "/data"

WORKDIR /

COPY requirements.txt .

RUN python -m pip install --upgrade pip && \
    python -m pip install -r ./requirements.txt

EXPOSE ${SYNC_PORT}

CMD ["python", "-m", "anki.syncserver"]
```

This is super basic, we're setting up the necessary Environment Variable (omitting `SYNC_USER1` for now). We're passing a `requirements.txt` which will be used by `pip` to install our Anki Python Module and specific version

```
anki == 23.12.1
```
At the time of this writing the latest version of Anki is `23.12.1` so that's what I'm going with.

Now I wrote a `Makefile` to build the docker image locally for testing just to make sure it all works as expected.
```makefile
APP_VERSION ?= "23.12.1"
APP_NAME ?= "anki-syncserver"

build:
	docker build --no-cache -t ${APP_NAME}:${APP_VERSION} .
```

This only has one option `build` and will run the `docker build` command to create a docker image called `anki-syncserver:23.12.1`

So I can just run a simple `make build` command to get that going.

## Testing
While I can run a `docker run` command to deploy my container, I'd rather test using a `docker-compose` yaml file.

```docker-compose
version: '3'
services:
  anki-sync:
    image: anki-syncserver:23.12.1
    container_name: anki-sync
    environment:
      SYNC_USER1: "username:password"
    ports:
      - 8080:8080
    volumes:
      - ./data:/data
```

Here we're putting it all together, I'm running a single service specifying our local docker image and version. Exposing the web port `8080`. Mounting a local directory for the Data volume. By default I'm mapping the Sync Directory to `/data` in the docker image, but this could also be overwritten by specifying the `SYNC_BASE` environment variable.
Finally I'm setting our 1 and only user, `SYNC_USER1=username:password` which is by no means secure in any shape of form.

Next a simple `docker-compose up` will show that our application is running successfully as it's listening on port `8080`
```
anki-sync  | 2024-01-07T05:13:58.351480Z  INFO listening addr=0.0.0.0:8080
```

## Next Time ~~on Dragon Ball Z~~
And that's it! We now have a simple docker image, next time we'll cover automating the build process because I really don't want to manually push the docker image to the container registries.

つづく