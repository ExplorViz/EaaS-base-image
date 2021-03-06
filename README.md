# EaaS-base-image

Docker images to use as base when creating build artifact images for ExplorViz as a Service.

## What is this?

> [ExplorViz](https://www.explorviz.net/) is an open source research monitoring and visualization approach, which uses dynamic analysis techniques to provide a live trace visualization of large software landscapes. 

[ExplorViz as a Service](https://github.com/ExplorViz/EaaS-server) (EaaS) allows you to collect build artifacts and run them in ExplorViz instances on-demand.

When submitting builds to EaaS, they need to be wrapped inside a docker image that runs both the application and, if necessary, some load to create more interesting visualizations.

This repository keeps some Dockerfiles to create convenient docker image to use as base image for your build artifact images. It already includes a Kieker distribution in the AspectJ variant that is used to instrument your running application and send monitoring records to ExplorViz. *Using this base image is completely optional and not required*, all that matters in the end is that your image runs *something* that submits monitoring records to the ExplorViz analysis-service that runs in the same docker-compose network.

## Usage

In your `Dockerfile`, simply specify as the first line (see below for versions):

```
FROM explorviz/eaas-base:<version>
```

### Conventions

The base image will create a user `runner` and a directory `/opt/app` that is writable by said user. It is expected that you place your application here.
Remember to set `USER runner` before specifying the `CMD`. The CMD will likely be a shell script that runs your application with kieker and then starts a load generator, or the `java-with-kieker` command explained below.

For your convenience, some commands are provided inside the container:

- `set-kieker-property <property> <value>`: This can be used in your Dockerfile to modify the kieker monitoring configuration without overriding the entire file manually. Example: `RUN set-kieker-property kieker.monitoring.applicationName PetClinic`
- `java-with-kieker`: Like the `java` command, but already adds all of the arguments necessary to run your application with Kieker monitoring. Example (inside `run.sh`): `java-with-kieker -cp . -jar application.jar`. It is recommended to always specify `-cp .` to avoid some problems with Kieker not finding the configuration files. If you don't need to run separate load generation you can directly specify this command as `CMD` in your Dockerfile.

You need to create an AspectJ weaver configuration for Kieker that determines which and where monitoring probes are weaved into your application.
This file should be placed in `/opt/app/META-INF/aop.xml`. See [Kieker documentation](http://kieker-monitoring.net/documentation/), especially the [latest nightly user guide](http://kieker-monitoring.net/download/nightly-builds/).

**Note:** If you try to instrument an application server that is packaged as `war` or a `jar-of-jars`, e.g. like Spring Boot does, you need to unzip the war/jar and specify the classpath and main class manually, otherwise AspectJ will be unable to weave the monitoring probes into your application.
Example: `java-with-kieker -cp ".:BOOT-INF/classes/:BOOT-INF/lib/*" com.example.app.MyApplication`

See [EaaS-demo-application](https://github.com/ExplorViz/EaaS-demo-application) for a full example.

### Versions

The following versions are provided:

- `11-jre-alpine`: Based on `adoptopenjdk/openjdk11:alpine-jre`
- `11-jre-debian`: Based on `adoptopenjdk/openjdk11:debian-jre`
- `8-jre-alpine`: Based on `adoptopenjdk/openjdk8:alpine-jre`
- `8-jre-debian`: Based on `adoptopenjdk/openjdk8:debian-jre`

**Warning: Not on DockerHub yet!**

Right now, you have to build these images locally before you can use them in your own Dockerfiles. Simply clone this git repository, then run `build-all.sh`. (Run as root if you are not in the docker group.)

If possible, use the alpine variants because they are much more lightweight, e.g. OpenJDK 11: ~151MB on Alpine vs. ~288MB on Debian. Even though the image is not saved for every single build thanks to docker layer caching, network transfer between the build server and your EaaS server has to be considered as well.
