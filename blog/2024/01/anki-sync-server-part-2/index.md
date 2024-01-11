---
layout: blog
title: Building and Deploying Anki Sync Server Part 2
date: 2024-01-10T19:11:27-06:00
lastMod: 2024-01-10T19:11:27-06:00
categories: self-hosted
tags:
  - docker
  - anki
  - containers
  - kubernetes
  - self-hosted
  - python
  - github
  - workflows
  - semver
  - semantic-release
description: Second part of a 3 part write up on deploying an Anki Sync Server.
disableComments: true
draft: false
---
## Continuing...
Alright now that we have our `Dockerfile` fleshed out and we've verified that we can start it up without it crashing, let's move on and figure out how to automate all this so that we don't need to build locally and/or push a container image locally. Now I've only used Github Actions sparingly, this blog uses Github Actions to make a commit on the main repository whenever I update either the Theme or create a new blog post, this way I can keep my blog separated into standalone components should I one day want to move to a brand new theme or move to a different Static Site Generator. 

## Github Actions
[Github](https://docs.github.com/en/actions) provides a framework to help you build workflows/pipelines for your projects. These actions can be triggered based on specific activities such as a `Push` to the repository. In fact Github themselves provides an example yaml for [Publishing Docker Images](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images) which we'll use as a base for our own workflow.

The following is the workflow I'll be using

```yaml
name: Publish Docker image

on:
  workflow_dispatch:

jobs:
  push_to_registries:
    name: Push Docker image to multiple registries
    runs-on: ubuntu-latest
    permissions:
      packages: write
      contents: read
    steps:
      - name: Check out the repo
        uses: actions/checkout@v4
      
      - name: Log in to Docker Hub
        uses: docker/login-action@f4ef78c080cd8ba55a85445d5b36e214a81df20a
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Log in to the Container registry
        uses: docker/login-action@65b78e6e13532edd9afa3aa52ac7964289d1a9c1
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@9ec57ed1fcdbf14dcef7dfbe97b2010124a938b7
        with:
          images: |
            ${{ secrets.DOCKER_REPO }}
            ghcr.io/${{ github.repository }}
      
      - name: Build and push Docker images
        uses: docker/build-push-action@3b5e8027fcad23fda98b2e3ac259d8d67585f671
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

For now the trigger is a manual execution, meaning this won't trigger unless I do so from the Github Actions UI.

```
on:
  workflow_dispatch:
```

But let's break down what this workflow is doing. Here we have our job id `push_to_registries` and in Github Actions, jobs run parallel with each other by default, so you can have multiple jobs execute at the same time when triggered. You can also specify conditions for each job, so you can build out complex and robust workflows where 1 or more jobs are triggered. This can be useful for automating tests. However for our needs, we'll just stick to one job.

```
push_to_registries:
    name: Push Docker image to multiple registries
    runs-on: ubuntu-latest
    permissions:
      packages: write
      contents: read
```

Jobs contains a list of sequential steps which are executed when ran. Here we're doing the following.

1. Check out the repo
2. Log in to Docker Hub
3. Log in to the Container registry
4. Extract metadata (tags, labels) for Docker
5. Build and push Docker images

In this workflow we'll be pushing to both the Docker's Container Registry and Github Container Registry in one single action and we're doing this by using reusable actions, `uses: docker/build-push-action@3b5e8027fcad23fda98b2e3ac259d8d67585f671` What are reusable actions? Basically they're other workflows that perform a set of steps that we're now referencing. Think of it like a function call, you or others build out a function that does something specific and now you can reuse that function in other parts of your program.

Now an important consideration here is the Tagging, by default if you're just executing a workflow based on a given branch such as `main` you'll end up with an image  name + version like `sinicide/anki-syncserver:main` which is fine if I was never ever going to update this again, but really what I want is that nice `sinicide/anki-syncserver:v1.0.0` versioning.

## Semantic Versioning
So we need to take a detour here and figure out what the heck is Semantic Versioning and how can I get that nice Release Tagging. Luckily there is a handy website [semver.org](https://semver.org/) that goes over the whole version numbering. But really all we need to know is that the format is `MAJOR.MINOR.PATCH` and that's how the version number is formatted. The **Major** number generally dictates a big change, that is incompatible with the old Major number. The **Minor** number is for functionality changes that are backwards compatible. And finally the **Patch** number is generally for bug fixes. If you want to learn more, I highly recommend reading the link I referenced.

Doing some research online I found that a pretty popular Semantic Versioning automation is [semantic-release](https://semantic-release.gitbook.io/semantic-release/) which is an NPM package that uses [Angular's Commit Message Conventions](https://github.com/angular/angular/blob/master/CONTRIBUTING.md#-commit-message-format) meaning that as we work on our project and commit our changes to Git, we can add little prefixes which semantic-release's workflow will be able to read and determine if it needs to set the Major, Minor, and Patch version, this combined with Github's Actions gives us a very powerful automation workflow.

## Semantic-Release Setup
Now I wouldn't say that this is a typical application development since I'm just repackaging an existing program into a docker container so there's a lot in the Example workflow that I didn't need to make use of. So let's go over how I implemented semantic-release for this docker project.

### Configuration File
Semantic-release uses a configuration file, I initially started trying do this with a `.releaserc` file but found in my testing that I needed a `package.json` for some npm stuff, so I just opted to use a `package.json` solely instead of having both.

```json
{
    "name": "anki-syncserver",
    "private": true,
    "release": {
        "branches": ["main"]
    }
}
```

Here I'm setting `private=true` because we are not publishing this to npm, so we have no need of an NPM Token which would be necessary to publish to npm.
The important thing here is the `branches` which can take an array of string objects to determine which branches a release should be applied to. Since I'm only testing locally and publishing only final releases, I'm just gonna stick with the `main` branch in the Github repo.

### Semantic-Release Workflow
Now semantic-release provides a sample [Github workflow yaml example](https://semantic-release.gitbook.io/semantic-release/recipes/ci-configurations/github-actions), but we don't need a bunch of the configuration on there by default because again I'm not publishing this to npm.

So below is what I've settled on.

```yaml
name: Release
on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "lts/*"
      - name: Release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: npx semantic-release@22.0.12
```

Here I've stripped out `id-token` for permissions, gotten rid of `Install dependencies` and `Verify the integrity....` and finally removed the `NPM_TOKEN` as well. These would be used for publishing to npm, which I don't need.

Finally I'm also specifying a specific version of `semantic-release` npm package and bumping up the `actions/xxxx@v4` since it's generally good practice to have a consistent experience using the same version until you are ready to upgrade to a newer version. Otherwise you might eventually run into unexpected behaviors as it tries to use the latest.

## What's Next
So now we have a workflow that generates tags with Semantic Versioning in Github. These Tags can then be used for Publishing Docker containers to the Container Registries. For the moment Pushing Docker images is a manual trigger, at some point I'll work on getting that automated based on the releases, I just need to learn a bit more on Github Actions.

Next time we'll go over how I'm deploying the public containers in my Kubernetes setup.