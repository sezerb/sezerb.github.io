---
layout: post
title: Running a Docker container non-root
tags: [docker, security]
---

If one writes a Dockerfile, builds a container image and runs it; the container will run with root privileges by default.
It is not recommended from security point of view and hence the mitigation tactics will be scope of this post.
There are three design tactis to prevent running containers with non-root privileges:
1. **USER** instruction of Docker
2. **entrypoint.sh** for root privilege requrired actions
3. Dropping root **capabilites**

## 1. USER Instruction of Docker
Docker's *USER* instruction provides a functionality to run a Docker container with non-root privileges.

#### Dockerfile:
```
FROM openjdk:11.0.15-jre-slim

WORKDIR /opt/testApp
COPY ./target/testApp.jar /opt/testApp/app.jar

USER testUser
CMD ["java","-jar","app.jar"]
```
When this container runs, the application will run with the privileges of *testUser*.
Please note that this is just an examle Dockerfile and testUser must exist in the container.

## 2. entrypoint.sh for root Privilege Required Actions
In some cases like changing ownership of a directory, one may need root privileges during container startup.
Using an entrypoint script is a design tactic where doing root privileged actions in running container is allowed first and privileges are dropped later on.
Finally the application runs with non-root user privileges with the help of **gosu**.

A note from gosu github repo: "*The core use case for gosu is to step down from root to a non-privileged user during container startup (specifically in the ENTRYPOINT, usually)*".

#### entrypoint.sh:
```bash
#!/bin/bash

set -e
EXEC_USER=testUser

# Do some root privileged stuff
chown ${EXEC_USER}:${EXEC_USER} -R /opt/testApp

# Run the main container command.
exec gosu ${EXEC_USER} "$@"
```

#### Dockerfile:
```
FROM openjdk:11.0.15-jre-slim

RUN apt-get update && apt-get install gosu;

WORKDIR /opt/testApp
COPY ./target/testApp.jar /opt/testApp/app.jar
COPY ./entrypoint.sh /opt/testApp/entrypoint.sh

ENTRYPOINT ["sh", "entrypoint.sh"]
CMD ["java","-jar","app.jar"]
```

## 3. Dropping Root Capabilities

For linux containers we can still run them as root, however we can drop the capabilities that a root user can do.
This is a linux kernel feature which breaks down the privileges of the root user into distinct units referred to as *capabilities*.
For instance *CHOWN* capability allows making arbitrary changes to file UIDs and GIDSs and *NET_RAW* capability allows using RAW and PACKET sockets; binding to any address for transparent proxying.

In the example given below, the container is run with dropping all root capabilities and only adding *NET_RAW* capability.
```bash
docker run -it --cap-drop=all --cap-add=net_raw debian /bin/sh
```
Please also note that this future can also be used for adding some root capabilities to non-root users granularly.



