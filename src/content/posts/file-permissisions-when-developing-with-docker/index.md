---
id: 'd3f00f30-81ea-45fd-b4d6-bf905e7847c5'
title: 'File permissions when developing with Docker'
description: "A brief guide on how you can set up your Docker development environment to work with your host's file permissions."
date: '2020-08-10 00:02:30'
tags:
  - Docker
  - Laravel
  - PHP
---

File permissions can be a little hard to get right when working with Docker due to how the host
machine and containers are mapped to one another. During development, it can be aggravating to
encounter the following issues:

- The host cannot read/write files created by the container
- The container cannot read/write files belonging to the host

These problems have cropped up for me in different ways, but deploying Laravel apps to a Docker
container have been consistently frustrating due to it needing write access to files and directories.
I'll be using this as my main example for the rest of this post.

## The problem

In a typical Laravel application, logs and cache files are created and stored in the `storage`
directory. These file operations are performed by the PHP process (either through FPM or Apache).
This will usually be ran as the `www-data` system user.

Usually when we develop a PHP application with Docker, we'll want to mount the application directory as a volume. Unfortunately this means that `storage` ends up being owned by the host user. This is usually seen as user `1000` within the container, but can vary depending on your OS. Within the container, these two users have no concept of one another, let alone have permission to change each other's files.

Here are some solutions to get around this.

## Add container user to host user group

Thinking about this purely in terms of the Linux permission model, it should be obvious we can
simply add the `www-data` user to our host user's group and add group write permissions to our
`storage` directory.

We can do this by adding something like this to our `Dockerfile`:

```dockerfile
FROM php:7.3-apache

# Add `www-data` to group `appuser`
RUN addgroup --gid 1000 appuser; \
  adduser --uid 1000 --gid 1000 --disabled-password appuser; \
  adduser www-data appuser;
```

Within the container, user `1000` can only be referenced by user ID (UID) and does not have an associated group ID (GID). We need to create an actual user in the container and assign it with the host user's UID and GID. In our example, we've created a user called `appuser`, and a corresponding `appuser` group. Both have a UID and GID matching the host user/group.

It's worth noting again that the UID and GID may differ depending on your OS and user set up. You can typically check this by running the following commands on your host:

```bash
echo $UID
echo $GID
```

This should allow our app to write into `storage`, but we'll need to be sure to check that the
directory has group write access too:

```dockerfile
COPY ./src /var/www/html

# Add group write access to `storage`
RUN chmod -R 760 /var/www/html/storage
```

## Map container user to host

One problem with the previous solution is that new files written by the container's service user
will also be owned by that user. This means that the host user won't have write access and you'll
need to use `sudo` to modify them outside of the container.

This can be a bit of a pain if you need to change these files often. A workaround for this is to map
the container's system user directly to the host:

```dockerfile
FROM php:7.3-apache

# Set www-data to have UID 1000
RUN usermod -u 1000 www-data;
```

This sets `www-data` to have a UID of 1000, which corresponds to our host user. Now whenever `www-data`
creates files within the container, they are also owned by the host user too.

## Change the container user

By default many Docker containers will start as `root`, leaving consumers to decide if they want to
start a container as a different user. From a security perspective, starting containers as `root`
can introduce the risk of privilege escalation, so it's usually a good practice to explicitly set it
to a non-root user.

In our case, it's desirable to change the user just so that we can correctly set user file
permissions. We can start the container as our host user by running the following:

```bash
docker run --user 1000:1000 your-container
```

Or setting it in our `docker-compose.yml`:

```yaml
services:
  app:
    user: '1000:1000'
```

This approach is more flexible than the previous solutions as we don't need to explicitly set the
user inside of the `Dockerfile` at build time. This allows us to more easily change the user at run
time e.g. with environment variables.

Unfortunately this solution doesn't quite work in our Apache-based PHP example as it needs to start
as `root` to run properly. Thankfully we can override this by setting the user with the
`APACHE_RUN_USER` and `APACHE_RUN_GROUP` environment variables
(see [documentation](https://github.com/docker-library/docs/tree/master/php#running-as-an-arbitrary-user)).
We can set these in `docker-compose.yml`:

```yaml
services:
  app:
    environment:
      APACHE_RUN_USER: '#1000'
      APACHE_RUN_GROUP: '#1000'
```

## Avoid writing files to shared volumes

Although it's possible to wrangle file permissions to work correctly, I would say that it's actually
better to not share files between the host/container at all. Containers try to provide isolated,
disposable environments and mounting shared volumes are counter-intuitive to this. As we've seen,
we need to leak host details like UIDs and GIDs through to the container to make everything work
correctly.

Ideally we should adjust our application so that it:

- Only has read access to volume(s) belonging to the host
- Writes logs to `stdout`
- Writes cache/temporary files to an ephemeral location in the container
- Uses non-shared volumes for writing files that need durability (databases, uploaded files, etc)
