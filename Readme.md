# vert.x micro-service workshop

In this workshop, you are going to develop a micro-service application called _vert2go_. This workshop covers:

* how to build micro-services with vert.x
* how you can use the event bus to bind your micro-services
* how to consume a REST API
* how to build an application using several persistence technologies
* how to provide a REST API, and build proxies
* how to deploy some micro-service to Openshift
* how to centralize your logs
* how to deploy a vert.x micro-service in a docker container
* how to monitor your application

The _vert2go_ application is a recommendation application where users can rates the place they like.

The slides for the talk are [online](http://pmlopes.github.io/sogeti-gurunight-2015/).

**Prerequisites:**

* Java 8 (JDK)
* Git
* Apache Maven
* Docker (or docker-machine)
* An IDE
* RoboMongo (optional)


## Using docker machine

Docker runs natively on Linux. Because the Docker daemon uses Linux-specific kernel features, you can’t run Docker 
natively in OS X or Windows. Instead, you must use docker-machine to create and attach to a virtual machine (VM). This
 machine is a Linux VM that hosts Docker for you on your Mac or Windows. If you are on Mac OS X or Windows, you can use 
 docker via docker machine (https://docs.docker.com/machine/).

1. Install docker and docker-machine from https://www.docker.com/docker-toolbox. Installation instructions are there: http://docs.docker.com/mac/step_one/
2. Once done run `docker run hello-world` to verify your installation 
  
The installation process will create a VM that has a minimal linux to run docker.

## Components

### Data Storage Service

This service shows how to connect to a mongodb database and genereate a API interface that will generate bindings for
the 2 languages used across the demo, groovy and javascript.

```
cd data-storage-service
mvn clean package
java -jar target/data-storage-service-1.0-SNAPSHOT-fat.jar --cluster --conf=src/conf/config.json
```

### Data Provisioning

This REST consumer will consume a REST service and populate the mongo collection directly it does not need to be part of
a cluster

```
cd data-provisioning
mvn clean package
java -jar target/data-provisioning-1.0-SNAPSHOT-fat.jar --conf=src/conf/config.json
```

### Data Storage Client

This is a simple client written in `Groovy` that show the usage of service proxies generated from the `data storage
service`. Since this client has no knowledge on mongo and uses a proxy it needs to be part of the cluster to find where
to delegate the the action from the proxy.

```
cd data-storage-client
mvn clean package
java -jar target/data-provisioning-1.0-SNAPSHOT-fat.jar --cluster
```

### Frontend

This is the web part of the application it will show the usage of the `sockjs` bridge and that generated proxies can be
consumed also from the browser. This part of the code show that a `Verticle` can be written in `javascript`.

```
cd frontend
mvn clean package
java -jar target/frontend-1.0-SNAPSHOT-fat.jar --cluster
```

### Map Render Service

This component show a part of the application that can be deployed on the cloud and we will consume its service using
REST. It works as the other components but since we are deploying it on [OpenShift](http://www.openshift.com) there are
a few extra steps.

For the first time we need to create an app:

```
rhc create-app map0render0service https://raw.githubusercontent.com/vert-x3/vertx-openshift-cartridge/master/metadata/manifest.yml
```

And then

```
cd frontend
mvn clean package

cd openshift
git add -A
git commit -m "deploy my application"
git push
```

### Map Render Proxy

We will consume the map service as a local proxy and add caching headers this we will deploy on `Docker`.

```
cd map-server-proxy
mvn clean install
mvn docker:build
mvn docker:start
```

If one refreshes the frontend javascript application the map should start showing on the right side of the screen.

### Recommendation Service

This shows again how to connect to `redis` and use `pub/sub` mode to pubish votes if one opens 2 browsers and clicks on
one restaurant and votes the counters should update in realtime on both web clients.

```
cd recommendation-service
mvn clean package
java -jar target/recommendation-service-1.0-SNAPSHOT-fat.jar --cluster --conf=src/conf/config.json
```

## Extras

### Centralized logging

replace the jul configuration content on `data-storage-service` with the example on `etc` and add the `GELF` dependency
to the `pom.xml`.

Start `graylog2` and enable the UDP transport.

Rebuild and restart the service

```
cd data-storage-service
mvn clean package
java -jar target/data-storage-service-1.0-SNAPSHOT-fat.jar --cluster --conf=src/conf/config.json
```

Using the client or the web browser load some data, the logs should now be collected in a `graylog` server.

### Metrics

#### Hawkular

Start the server see the scripts folder. Add a bridge to hawkular to the deployment

```
cd eventbus-to-hawkular-bridge
mvn clean package
java -jar target/eventbus-to-hawkular-bridge-1.0-SNAPSHOT-fat.jar --cluster --conf=src/conf/config.json
```

Running some queries to the data storage service should generate some metrics to be called.

#### Grafana

Start grafana (see scripts) Add a connection use `Influx 0.8` and connect to the hawkular server, then add a graph to
the dashboard using the name `vertx2go.metrics` and you should start seeing the average requests metrics in a visual
form.
