---
layout: blog
title: Migrating to Unifi Network Application
date: 2023-11-26T18:23:57-06:00
lastMod: 2023-11-26T18:23:57-06:00
categories: self-hosted
tags:
  - self-hosted
  - docker
  - portainer
  - unifi
  - mongodb
description: Linuxserver.io is deprecating the unifi-controller image in favor of unifi-network-application image on 2024-01-01, so it's now or never on switching.
disableComments: true
draft: false
---
## The Backstory
In my home network, I use Ubiquiti products for my Wireless Access Points. In particular I have Unifi AP AC Lite deployed in both my Apartment and in my Parent's home for Wifi. Recently I also bought a used Unifi Standard USW-24 Managed Switch, that I need to migrate to as I've been using a Dell PowerConnect 2824 Managed Switch for a several years now.

You might be asking why I use Unifi Wireless Access Points, when most consumer grade Router/Wifi combos already include Wifi. Well I also don't use Consumer Grade Routers or at least I haven't for the better part of a decade now.

The entry point into my Home Networking typically follows

```
Modem -> pfSense (Router) -> Switch -> Unifi AP (wifi)
```

While pfSense can handle Wifi as well, it was preferred back in the day to have that be handled by something else (not sure if this is still true or not), either a Consumer Grade Wifi/Router acting as a Wifi only Access Point or a Unifi AP.

The Unifi Wifi Access Points can be Powered over Ethernet, so if you have a PoE switch, you can power the Access Points directly with 1 cable instead of having a power dongle to go with the Ethernet, unfortunately I don't yet have a PoE switch, so I'm still using the power dongle.

With these Ubiquiti products, you'll need an Application to configure them. For Self-Hosted minded individuals, this is where the Unifi Controller Web Application comes in.

For years now I've been using the `unifi-controller` docker image by [linuxserver.io](https://github.com/linuxserver/docker-unifi-controller) This was/is a relatively painless setup. However in January 1st, 2024, linuxserver.io will be deprecating the `unifi-controller` image and instead we'll need to upgrade/migrate to [Unifi Network Application](https://github.com/linuxserver/docker-unifi-network-application).

Since the new year is fast approaching, I went ahead and made the necessary changes. So let's go over the migration requirements and steps to perform.

## Requirements
- In place upgrade does not work (e.g. can't just swap container image)
- Create a backup of existing unifi-controller
- Unifi Network Application requires Mongo DB
- Create docker-compose file for new deployment

In my existing deployment of `unifi-controller` I used a docker-compose file, so I'll need to replace that with a new one.

## Backing Up the Config
This is extremely straight forward, just go to the Unifi Controller `Settings > System > Backups` and download a copy of the backup file. Yay! now you have a `.unf` backup file, we'll need this to restore our Unifi Settings when we've deployed the Unifi Network Application since it won't have any configuration on start essentially being a brand new setup.

## Let's talk docker-compose
We need to create a new docker-compose file that containers the Unifi Network Application image in addition to the Mongo DB image, since that's now a requirement.

Below is what I'm using for my deployment.

```docker
version: '3'
services:
  unifi-db:
    image: mongo:4.4.25
    container_name: unifi-db
    networks:
      - traefik_proxy
    volumes:
      - "/data/containers/unifi-db/data:/data/db"
      - "/data/containers/unifi-db/initialize/init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro"
    restart: unless-stopped

  unifi-network-application:
    image: lscr.io/linuxserver/unifi-network-application:8.0.7
    container_name: unifi-network-application
    hostname: unifi-app
    depends_on:
      - unifi-db
    networks:
      - traefik_proxy
    ports:
      # Unifi STUN port
      - 3478:3478/udp
      # Required for AP Discovery
      - 10001:10001/udp
      # Required for controller to be discoverable on L2 Network
      - 1900:1900/udp
      # Required for device communication
      - 8080:8080
      # Remote syslog capture
      - 5514:5514/udp
    environment:
      # MongoDB configuration, evaluated on first run
      MONGO_USER: ${MONGO_DB_USERNAME}
      MONGO_PASS: ${MONGO_DB_PASSWORD}
      MONGO_HOST: "unifi-db"
      MONGO_PORT: "27017"
      MONGO_DBNAME: ${MONGO_DBNAME}
    restart: unless-stopped
    volumes:
      - "/data/containers/unifi-network-application:/config"
    labels:
      - "traefik.http.routers.unifi-network-application-http.entrypoints=web"
      - "traefik.http.routers.unifi-network-application-http.middlewares=redirect-to-https@file"
      - "traefik.http.routers.unifi-network-application-http.rule=Host(`unifi-app.sin.lan`)"
      - "traefik.http.routers.unifi-network-application-https.entrypoints=websecure"
      - "traefik.http.routers.unifi-network-application-https.rule=Host(`unifi-app.sin.lan`)"
      - "traefik.http.services.unifi-network-application-https.loadBalancer.server.port=8443"
      - "traefik.http.services.unifi-network-application-https.loadBalancer.server.scheme=https"
      - "traefik.http.routers.unifi-network-application-https.middlewares=unifi-network-application-headers"
      - "traefik.http.middlewares.unifi-network-application-headers.headers.customRequestHeaders.Authorization="
      - "traefik.http.middlewares.unifi-network-application-headers.headers.customRequestHeaders.X-Forwarded-Proto=https"
      - "traefik.http.serverstransports.unifi-network-application-st.insecureSkipVerify=true"
networks:
  traefik_proxy:
    external: true
```
## The Mongo Requirement
So starting off is the MongoDB deployment. Here, I'm following the guidance of [linuxserver.io](https://docs.linuxserver.io/images/docker-unifi-network-application/?h=unifi+network#setting-up-your-external-database) and configuring Mongo with version `4.4.25` as it's document that Unifi Network Application supporting only Mongo `3.6.x` to `4.4.x` versions. Here I'm also specifying an initialization file that will create user for the unifi database.
```docker
unifi-db:
  image: mongo:4.4.25
  container_name: unifi-db
  networks:
    - traefik_proxy
  volumes:
    - "/data/containers/unifi-db/data:/data/db"
    - "/data/containers/unifi-db/initialize/init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro"
  restart: unless-stopped
```
The `init-mongo.js` as referenced by linuxserver.io, this create our `unifi-user` to connect to the database named `unifi`
```javascript
db.getSiblingDB("unifi").createUser({user: "unifi-user", pwd: "unifi-password", roles: [{role: "dbOwner", db: "unifi"}]});
db.getSiblingDB("unifi_stat").createUser({user: "unifi-user", pwd: "unifi-password", roles: [{role: "dbOwner", db: "unifi_stat"}]});
```
In my setup, I have multiple docker-compose files that use a common Network so that containers can talk to each other and use a common Proxy Manager, otherwise each container deployed via docker-compose is potentially isolated to it's own network.

No ports are exposed here because we don't need to directly access anything ourselves.

## Unifi Network Application
```docker
unifi-network-application:
  image: lscr.io/linuxserver/unifi-network-application:8.0.7
  container_name: unifi-network-application
  hostname: unifi-app
  depends_on:
    - unifi-db
  networks:
    - traefik_proxy
  ports:
    # Unifi STUN port
    - 3478:3478/udp
    # Required for AP Discovery
    - 10001:10001/udp
    # Required for controller to be discoverable on L2 Network
    - 1900:1900/udp
    # Required for device communication
    - 8080:8080
    # Remote syslog capture
    - 5514:5514/udp
  environment:
    # MongoDB configuration, evaluated on first run
    MONGO_USER: ${MONGO_DB_USERNAME}
    MONGO_PASS: ${MONGO_DB_PASSWORD}
    MONGO_HOST: "unifi-db"
    MONGO_PORT: "27017"
    MONGO_DBNAME: ${MONGO_DBNAME}
```
At the time of this writing, version `8.0.7` is the latest. Here we're specifying a dependency on `unifi-db`

My original `unifi-controller` deployment exposes the same Ports that are necessary for device communication. However I'm not exposing Unifi Guest Portal ports nor mobile throughput test port and finally I'm not exposing the Unifi Web Admin Ports (http/https) since I'm using Traefik to Proxy the requests so that I don't have to type in a port value in the browser.

Finally new Docker Environment configurations are included to tell our Unifi Network Application how to connect to the Mongo Database.

## Bonus Traefik configuration
I think most of the Self-Hosted community seems to use Nginx Proxy Manager, but in case you're using [Traefik Proxy](https://traefik.io/traefik/) here's what I'm using, my Traefik Proxy looks at docker labels to configure the routing.
```docker
labels:
  - "traefik.http.routers.unifi-network-application-http.entrypoints=web"
  - "traefik.http.routers.unifi-network-application-http.middlewares=redirect-to-https@file"
  - "traefik.http.routers.unifi-network-application-http.rule=Host(`unifi-app.sin.lan`)"
  - "traefik.http.routers.unifi-network-application-https.entrypoints=websecure"
  - "traefik.http.routers.unifi-network-application-https.rule=Host(`unifi-app.sin.lan`)"
  - "traefik.http.services.unifi-network-application-https.loadBalancer.server.port=8443"
  - "traefik.http.services.unifi-network-application-https.loadBalancer.server.scheme=https"
  - "traefik.http.routers.unifi-network-application-https.middlewares=unifi-network-application-headers"
  - "traefik.http.middlewares.unifi-network-application-headers.headers.customRequestHeaders.Authorization="
  - "traefik.http.middlewares.unifi-network-application-headers.headers.customRequestHeaders.X-Forwarded-Proto=https"
  - "traefik.http.serverstransports.unifi-network-application-st.insecureSkipVerify=true"
```
The above specifies an `http` entrypoint, this is port `80`that does a redirect to `https` on port `443`

Next we're specifying the `loadBalancer` port, which specifies the Unifi Web Admin Port `8443`

```docker
- "traefik.http.services.unifi-network-application-https.loadBalancer.server.port=8443"
- "traefik.http.services.unifi-network-application-https.loadBalancer.server.scheme=https"
```
So essentially Web Request goes from `80` -> `443` -> `8443`

The Unifi Application handles TLS itself, so we'll include HTTP headers and ignore the TLS cert presented by the Unifi Application.
```docker
- "traefik.http.routers.unifi-network-application-https.middlewares=unifi-network-application-headers"
- "traefik.http.middlewares.unifi-network-application-headers.headers.customRequestHeaders.Authorization="
- "traefik.http.middlewares.unifi-network-application-headers.headers.customRequestHeaders.X-Forwarded-Proto=https"
- "traefik.http.serverstransports.unifi-network-application-st.insecureSkipVerify=true"
```

## Deploying Mongo and Unifi Network Application
A simple `docker-compose -f unifi.yaml up -d` will start everything.

Check logs to make sure there's no errors in Mongo
```
docker logs -f unifi-db
```
Finally check the new Unifi Network Application
```
docker logs -f unifi-network-application
```
If all went well we should see
```
*** Waiting for MONGO_HOST unifi-db to be reachable. ***

Generating 4,096 bit RSA key pair and self-signed certificate (SHA384withRSA) with a validity of 3,650 days

	for: CN=unifi

[custom-init] No custom files found, skipping...

[ls.io-init] done.
```

## Restoring the old configuration
Now we can navigate to our application in browser and click on `Restore Server from a Backup` then click on `Upload Backup File` and selecting our `.unf` file. Restoring should take a few minutes at most and we'll be redirect to the login page when it's done.
Logging in and we can verify that our previous Unifi AP devices are found and connected.

## Postmortem
While migrating to this new deployment is fairly painless, the need for Mongo is an additional overhead that we didn't need previously and as far as I can tell, Unifi Network Application is still generating backup `.unf` files in the same `autobackup` location, so in terms of backing and restoring in the future, we really only need the `.unf` files and mongo database can be discarded without issue, if this changes in the future, I'll update this post to note what we'll be losing without backing up Mongo.