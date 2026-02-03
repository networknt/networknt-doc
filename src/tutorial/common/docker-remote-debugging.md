# Docker Remote Debugging

Light-4j applications are standalone Java applications without any JEE container and can be debugged inside [IntelliJ](/tutorial/common/debug/idea) or [Eclipse](/tutorial/common/debug/eclipse) directly. Most of the time, we are going to debug the application before dockerizing it to ensure it is functioning. Sometimes, an application works when starting with `java -jar xxx.jar` but when running with a docker container, it stops working. Most cases, this is due to the docker network issue.

In case this happens, it would be great if we can debug the application running inside the Docker container remotely from your favorite IDE. In this tutorial, we are going to walk through the steps with IntelliJ IDEA. For developers who are using Eclipse, the process should be very similar.

## Create Dockerfile-Debug

First, we need to create a Dockerfile with debug agent in the java command line.

Here is the original Dockerfile for light-router.

```dockerfile
FROM openjdk:11.0.3-slim
ADD /target/light-router.jar server.jar
CMD ["/bin/sh","-c","java -Dlight-4j-config-dir=/config -Dlogback.configurationFile=/config/logback.xml -jar /server.jar"]
```

Let's create a `Dockerfile-Debug` with the following modifications. If necessary, we are going to create this Dockerfile-Debug from light-codegen for all generated projects.

```dockerfile
FROM openjdk:11.0.3-slim
ADD /target/light-router.jar server.jar
CMD ["/bin/sh","-c","java -agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=*:5005 -Dlight-4j-config-dir=/config -Dlogback.configurationFile=/config/logback.xml -jar /server.jar"]
```

## Build Debug Image

Once the Dockerfile-Debug is created, let's build an alternative image with it locally without publishing it to Docker Hub.

```bash
docker build -t networknt/light-router:latest -f ./docker/Dockerfile-Debug .
```

## Prepare Environment

Before running the docker-compose, we need to make sure that Consul docker-compose is running (if your service depends on it).

```bash
cd ~/networknt/light-docker
docker-compose -f docker-compose-consul.yml up -d
```

Now, the local image for the light-router is the debug image. Let's take a look at the docker-compose file that started two instances of light-codegen and light-router. This file can be found in the [light-config-test](https://github.com/networknt/light-config-test/tree/master/light-codegen) along with all configurations.

```yaml
version: "2"

services:
  2_0_x:
    image: networknt/codegen-server-1.0.0
    volumes:
      - ./2_0_x/config:/config
      - ./2_0_x/service:/service
    ports:
      - 8440:8440
    environment:
      - STATUS_HOST_IP=${DOCKER_HOST_IP}
    network_mode: host    

  1_6_x:
    image: networknt/codegen-server-1.0.0
    volumes:
      - ./1_6_x/config:/config
      - ./1_6_x/service:/service
    ports:
      - 8439:8439
    environment:
      - STATUS_HOST_IP=${DOCKER_HOST_IP}
    network_mode: host    

  light-router:
    image:  networknt/light-router:latest
    environment:
      - STATUS_HOST_IP=${DOCKER_HOST_IP}
    network_mode: host    
    ports:
      - 443:8443
    volumes:
      - ./router/config:/config
      - ./codegen-web/build:/codegen-web/build
```

## Start Debugging

Before starting the docker-compose, we might need to run `docker-compose rm` to remove the reference to the old image.

```bash
cd ~/networknt/light-config-test/light-codegen
docker-compose up
```

You will notice that the `light-router` application within the container is not up and running but outputs the following line:

```
light-router_1  | Listening for transport dt_socket at address: 5005
```

This line indicates that the application inside the container is waiting for IDE debug connection on port number 5005. Let's open the light-router project in IDEA. To set up the remote debug, follow the steps below.

1.  Click `Run` tab and `Debug Configuration` menu.
2.  Click `+` to add a remote application called `remote`.
3.  Setup the parameters as below:

    ![idea-remote-debug](/images/idea-remote-debug.png)

4.  Click `Apply` and `OK` to close the popup window.
5.  Click `Run` tab and `Debug` menu and select `remote` in the popup dropdown to start the debug session.

You can set the breakpoints in the initialization code if you want to debug the logic during the server startup. Once the debug session is started, the light-router application inside the container will be started and running as usual. The rest of the debug is the same as you are debugging a standalone application.
