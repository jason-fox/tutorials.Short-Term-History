![FIWARE Banner](https://fiware.github.io/tutorials.Short-Term-History/img/fiware.png)

This tutorial is an introduction to [FIWARE STH-Comet](https://fiware-sth-comet.readthedocs.io/) - a generic enabler which is used to retrieve trend data from a Mongo-DB database. The tutorial activates the IoT sensors connected in the [previous tutorial](https://github.com/Fiware/tutorials.IoT-Agent) and persists measurements from those sensors into a database and retrieves time-based aggregations of that data.

The tutorial uses [cUrl](https://ec.haxx.se/) commands throughout, but is also available as [Postman documentation](http://fiware.github.io/tutorials.Short-Term-History/)

[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/4824d3171f823935dcab)


# Contents

- [Querying Time Series Data](#querying-time-series-data)
  * [Analyzing time series data](#analyzing-time-series-data)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
  * [Docker and Docker Compose](#docker-and-docker-compose)
  * [Cygwin for Windows](#cygwin-for-windows)
- [Start Up](#start-up)
- [*minimal* mode (STH-Comet only)](#minimal-mode-sth-comet-only)
  * [Database Server Configuration](#database-server-configuration)
  * [STH-Comet Configuration](#sth-comet-configuration)
  * [*minimal* mode - Start up](#minimal-mode---start-up)
    + [Checking the STH-Comet Service Health](#checking-the-sth-comet-service-health)
    + [Generating Context Data](#generating-context-data)
  * [*minimal* mode - Subscribing STH-Comet to Context Changes](#minimal-mode---subscribing-sth-comet-to-context-changes)
    + [STH-Comet - Aggregate Motion Sensor Count Events](#sth-comet---aggregate-motion-sensor-count-events)
    + [STH-Comet - Sample Lamp Luminosity](#sth-comet---sample-lamp-luminosity)
- [Time Series Data Queries](#time-series-data-queries)
- [*formal* mode (Cygnus + STH-Comet)](#formal-mode-cygnus--sth-comet)
  * [Database Server Configuration](#database-server-configuration-1)
  * [STH-Comet Configuration](#sth-comet-configuration-1)
  * [Cygnus Configuration](#cygnus-configuration)
  * [*formal* mode - Start up](#formal-mode---start-up)
    + [Checking the STH-Comet Service Health](#checking-the-sth-comet-service-health-1)
    + [Checking the Cygnus Service Health](#checking-the-cygnus-service-health)
    + [Generating Context Data](#generating-context-data-1)
  * [*formal* mode - Subscribing Cygnus to Context Changes](#formal-mode---subscribing-cygnus-to-context-changes)
    + [Cygnus - Aggregate Motion Sensor Count Events](#cygnus---aggregate-motion-sensor-count-events)
    + [Cygnus - Sample Lamp Luminosity](#cygnus---sample-lamp-luminosity)
  * [*formal* mode - Time Series Data Queries](#formal-mode---time-series-data-queries)
- [Next Steps](#next-steps)


# Querying Time Series Data

> "The *"moment"* has no yesterday or tomorrow. It is not the result of thought and therefore has no time."
>
> — Bruce Lee


Within the FIWARE platform, historical context data can be persisted to a database using a combination of the **Orion 
Context Broker** and the **Cygnus** generic enabler. This results in a series of data points being written to the
database of your choice. Each time-stamped data point represents the state of context entities at a given moment in time.
The individual data points are relatively meaningless on their own, it is only through combining a series data points
that meaningful statistics such as maxima, minima and trends can be observed.

The creation and analysis of trend data is a common requirement of context-driven systems - therefore the FIWARE platform 
offers a generic enabler ([STH-Comet](https://fiware-sth-comet.readthedocs.io/)) specifically to deal with the issue of persisting and interpreting time series data. **STH-Comet** itself can be used in two modes:

* In *minimal* mode, **STH-Comet** is responsible for both data collection and interpreting the data when requested
* In *formal* mode, the collection of data is delegated to **Cygnus**, **STH-Comet** merely reads from an existing database.

Of the two modes of operation, the *formal* mode is more flexible, but *minimal* mode is simpler and easier to set-up. The key differences between the two are summarized in the table below:


|                                                         | *minimal* mode                | *formal* mode              |
|---------------------------------------------------------|-------------------------------|----------------------------|
| Is the system easy to set-up  properly?                 | Only one configuration supported - Easy to set up                | Highly configurable - Complex to set up          |
| Which component is responsible for a data persistance?  | **STH-Comet**                 | **Cygnus**                 |
| What is the role of **STH-Comet**?                      | Reading and writing data      | Data Read only             |
| What is the role of **Cygnus**?                         | Not Used                      | Data Write only            |
| Where is the data aggregated?                           | Mongo-DB database connected to **STH-Comet** only| Mongo-DB database connected to both **Cygnus** and **STH-Comet** |
| Can the system be configured to use other databases?    | No                            | Yes                        |
| Does the solution scale easily?                         | Does not scale easily - use for simple systems | Scales easily - use for complex systems |
| Can the system cope with high rates of throughput?      | No - use where throughput is low | Yes - use where throughput is high |


## Analyzing time series data

The appropriate use of time series data analysis will depend on your use case and the reliability of the data measurements you receive. Time series data analysis can be used to answer questions such as:

* What was the maximum measurement of a device within a given time period?
* What was the average measurement of a device within a given time period?
* What was the sum of the measurements sent by a device within a given time period?

It can also be used to reduce the signficance of each individual data point to exclude outliers by smoothing.



#### Device Monitor

For the purpose of this tutorial, a series of dummy IoT devices have been created, which will be attached to the context broker. Details of the architecture and protocol used can be found in the [IoT Sensors tutorial](https://github.com/Fiware/tutorials.IoT-Sensors).
The state of each device can be seen on the UltraLight device monitor web-page found at: `http://localhost:3000/device/monitor`

![FIWARE Monitor](https://fiware.github.io/tutorials.Short-Term-History/img/device-monitor.png)



# Architecture

This application builds on the components and dummy IoT devices created in 
[previous tutorials](https://github.com/Fiware/tutorials.IoT-Agent/). It will use three or four FIWARE components depending on the configuration of the system:
the [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/), the
[IoT Agent for Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/),
[STH-Comet](http://fiware-cygnus.readthedocs.io/en/latest/) and
[Cygnus](http://fiware-cygnus.readthedocs.io/en/latest/). 

Therefore the overall architecture will consist of the following elements:

* Four **FIWARE Generic Enablers**:
  * The FIWARE [Orion Context Broker](https://fiware-orion.readthedocs.io/en/latest/) which will receive requests using [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
  * The FIWARE [IoT Agent for Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/) which will receive northbound measurements from the dummy IoT devices in [Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual) format and convert them to [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) requests for the context broker to alter the state of the context entities
  * FIWARE [STH-Comet](http://fiware-sth-comet.readthedocs.io/) will:
    + interpret time-based data queries
    + subscribe to context changes and persist them into a **Mongo-DB** database  (*minimal* mode only)
   * FIWARE [Cygnus](http://fiware-cygnus.readthedocs.io/en/latest/) where it will subscribe to context changes and persist them into a **Mongo-DB** database (*formal* mode only) 

> :information_source: **Note:** **Cygnus** will only be used if **STH-Comet** is configured in *formal* mode.

* A [MongoDB](https://www.mongodb.com/) database:
  * Used by the **Orion Context Broker** to hold context data information such as data entities, subscriptions and registrations
  * Used by the **IoT Agent** to hold device information such as device URLs and Keys
  * Used as a data sink to hold time-based historical context data
     + In *minimal* mode - this is read and populated by **STH-Comet** 
     + In *formal* mode - this is populated by **Cygnus**  and read by **STH-Comet** 
* Three **Context Providers**:
  * The **Stock Management Frontend** is not used in this tutorial. It does the following:
    + Display store information and allow users to interact with the dummy IoT devices
    + Show which products can be bought at each store
    + Allow users to "buy" products and reduce the stock count.
  * A webserver acting as set of [dummy IoT devices](https://github.com/Fiware/tutorials.IoT-Sensors) using the [Ultralight 2.0](http://fiware-iotagent-ul.readthedocs.io/en/latest/usermanual/index.html#user-programmers-manual) protocol running over HTTP.
  * The **Context Provider NGSI** proxy is not used in this tutorial. It does the following:
    + receive requests using [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2)
    + makes requests to publicly available data sources using their own APIs in a proprietary format 
    + returns context data back to the Orion Context Broker in [NGSI](https://fiware.github.io/specifications/OpenAPI/ngsiv2) format.




Since all interactions between the elements are initiated by HTTP requests, the entities can be containerized and run from exposed ports. 

The specific architecture of both the *minimal* and *formal* configurations is discussed below.



# Prerequisites

## Docker and Docker Compose 

To keep things simple all components will be run using [Docker](https://www.docker.com). **Docker** is a container technology which allows to different components isolated into their respective environments. 

* To install Docker on Windows follow the instructions [here](https://docs.docker.com/docker-for-windows/)
* To install Docker on Mac follow the instructions [here](https://docs.docker.com/docker-for-mac/)
* To install Docker on Linux follow the instructions [here](https://docs.docker.com/install/)

**Docker Compose** is a tool for defining and running multi-container Docker applications. A  series of [YAML files](https://raw.githubusercontent.com/Fiware/tutorials.Historic-Context/master/cygnus) are used configure the required
services for the application. This means all container services can be brought up in a single command. Docker Compose is installed by default as part of Docker for Windows and  Docker for Mac, however Linux users will need to follow the instructions found [here](https://docs.docker.com/compose/install/)

## Cygwin for Windows

We will start up our services using a simple Bash script. Windows users should download [cygwin](http://www.cygwin.com/) to provide a command line functionality similar to a Linux distribution on Windows.




# Start Up

Before you start you should ensure that you have obtained or built the necessary Docker images locally. Please run

```console
./services create
``` 

>**Note** The `context-provider` image has not yet been pushed to Docker hub.
> Failing to build the Docker sources before proceeding will result in the following error:
>
>```
>Pulling context-provider (fiware/cp-web-app:latest)...
>ERROR: The image for the service you're trying to recreate has been removed.
>```


Thereafter, all services can be initialized from the command line by running the [services](https://github.com/Fiware/tutorials.Historic-Context/blob/master/services) Bash script provided within the repository:

```console
./services <command>
``` 

Where `<command>` will vary depending upon the mode we wish to activate.
This command will also import seed data from the previous tutorials and provision the dummy IoT sensors on startup.

>:information_source: **Note:** If you want to clean up and start over again you can do so with the following command:
>
>```console
>./services stop
>``` 
>


# *minimal* mode (STH-Comet only)

In the *minimal* configuration, **STH-Comet** is used to persisting historic context data and also used to make time-based queries.
All operations take place on the same port `8666`. The MongoDB instance listening on the standard
`27017` port is used to hold data the historic context data as well as holding data related to the **Orion Context Broker** and the **IoT Agent**.
The overall architecture can be seen below:

![](https://fiware.github.io/tutorials.Short-Term-History/img/sth-comet.png)

## Database Server Configuration

```yaml
  mongo-db:
    image: mongo:3.6
    hostname: mongo-db
    container_name: db-mongo
    ports:
        - "27017:27017"
    networks:
        - default
    command: --bind_ip_all --smallfiles
```

## STH-Comet Configuration 

```yaml
  sth-comet:
    image: fiware/sth-comet
    hostname: sth-comet
    container_name: fiware-sth-comet
    depends_on:
        - mongo-db
    networks:
        - default
    ports:
        - "8666:8666"
    environment:
        - STH_HOST=0.0.0.0
        - STH_PORT=8666
        - DB_PREFIX=sth_
        - DB_URI=mongo-db:27017
        - LOGOPS_LEVEL=DEBUG
```

The `sth-comet` container is listening on one port: 

* The Operations for port for STH-Comet - `8666` is where the service will be listening for notifications from the Orion context broker as well
  as time based query requests from cUrl or Postman

The `sth-comet` container is driven by environment variables as shown:

| Key        |Value            |Description|
|------------|-----------------|-----------|
|STH_HOST    |`0.0.0.0`        | The address where STH-Comet is hosted - within this container it means all IPv4 addresses on the local machine  |
|STH_PORT    |`8666`           | Operations Port that STH-Comet listens on, it is also used when subscribing to context data changes |
|DB_PREFIX   |`sth_`           | The prefix added to each database entity if none is provided |
|DB_URI      |`mongo-db:27017` | The  Mongo-DB server which STH-Comet will contact to persist historical context data |
|LOGOPS_LEVEL|`DEBUG`          | The logging level for STH-Comet |


## *minimal* mode - Start up

To start the system using the *minimal* configuration using **STH-Comet**  only, run the following command:

```console
./services sth-comet
``` 


### Checking the STH-Comet Service Health
 
Once STH-Comet is running, You can check the status by making an HTTP request to the exposed `STH_PORT` port. 
If the response is blank, this is usually because **STH-Comet** is not running or is listening on another port.


#### :one: Request:

```console
curl -X GET \
  'http://localhost:8666/version'
```

#### Response:

The response will look similar to the following:

```json
{
    "version": "2.3.0-next"
}
```

>**Troubleshooting:** What if the response is blank ?
>
> * To check that a docker container is running try
>
>```bash
>docker ps
>```
>
>You should see several containers running. If `sth-comet`  or `cygnus` is not running, you can restart the containers as necessary.

### Generating Context Data

For the purpose of this tutorial, we must be monitoring a system where the context is periodically being updated.
The dummy IoT Sensors can be used to do this. Open the device monitor page at `http://localhost:3000/device/monitor`
and unlock a **Smart Door** and switch on a **Smart Lamp**. This can be done by selecting an appropriate the command 
from the drop down list and pressing the `send` button. The stream of measurements coming from the devices can then
be seen on the same page:

![](https://fiware.github.io/tutorials.Short-Term-History/img/door-open.gif)


## *minimal* mode - Subscribing STH-Comet to Context Changes

Once a dynamic context system is up and running, under minimal mode,  **STH-Comet**  needs to be informed of changes in context.
Therefore we need to set up a subscription in the **Orion Context Broker** to notify **STH-Comet** of these changes. The details
of tht subscription will differ dependent upon the device being monitored and the sampling rate.

### STH-Comet - Aggregate Motion Sensor Count Events

The rate of change of the **Motion Sensor** is driven by events in the real-world. We need to receive
every event to be able to aggregate the results.

This is done by making a POST request to the `/v2/subscription` endpoint of the **Orion Context Broker**.

* The `fiware-service` and `fiware-servicepath` headers are used to filter the subscription to only listen to measurements from the attached IoT Sensors
* The `idPattern` in the request body ensures that **STH-Comet** will be informed of all **Motion Sensor** data changes.
* The notification `url` must match the configured `STH_PORT`
* The `attrsFormat=legacy` is required since **STH-Comet** currently only accepts notifications in the older NGSI v1 format.

#### :two: Request:

```console
curl -iX POST \
  'http://localhost:1026/v2/subscriptions/' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify STH-Comet of all Motion Sensor context changes",
  "subject": {
    "entities": [
      {
        "idPattern": "Motion.*"
      }
    ]
  },
  "notification": {
    "http": {
      "url": "http://sth-comet:8666/notify"
    },
    "attrsFormat": "legacy"
  }
}'
```

### STH-Comet - Sample Lamp Luminosity

The luminosity of the **Smart Lamp** is constantly changing, we only need to **sample** the values to be
able to work out relevant statistics such as minimum and maximum values and rates of change.

This is done by making a POST request to the `/v2/subscription` endpoint of the **Orion Context Broker**
and including the `throttling` attribute in the request body.

* The `fiware-service` and `fiware-servicepath` headers are used to filter the subscription to only listen to measurements from the attached IoT Sensors
* The `idPattern` in the request body ensures that **STH-Comet** will be informed of all **Smart Lamp** data changes only
* The notification `url` must match the configured `STH_PORT`
* The `attrsFormat=legacy` is required since **STH-Comet** currently only accepts notifications in the older NGSI v1 format.
* The `throttling` value defines the rate that changes are sampled.

#### :three: Request:

```console
curl -X POST \
  'http://localhost:1026/v2/subscriptions/' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify Cygnus to sample Lamp changes every five seconds",
  "subject": {
    "entities": [
      {
        "idPattern": "Lamp.*"
      }
    ]
  },
  "notification": {
    "http": {
      "url": "http://cygnus:5050/notify"
    },
    "attrsFormat": "legacy"
  },
  "throttling": 5
}'
```

# Time Series Data Queries

**STH-Comet**














# *formal* mode (Cygnus + STH-Comet)

The *formal* configuration is uses **Cygnus** to persist historic context data into a MongoDB database in the same manner as had been presented in the
[previous tutorial](https://github.com/Fiware/tutorials.Historic-Context). The existing MongoDB instance (listening on the standard
`27017` port) is used to hold data related to the **Orion Context Broker**, the **IoT Agent** and the historic
context data persisted by **Cygnus**. **STH-Comet** is also attached to the same database to read data from it. The overall architecture can be seen below:

![](https://fiware.github.io/tutorials.Short-Term-History/img/cygnus-sth-comet.png)

## Database Server Configuration

```yaml
  mongo-db:
    image: mongo:3.6
    hostname: mongo-db
    container_name: db-mongo
    ports:
        - "27017:27017"
    networks:
        - default
    command: --bind_ip_all --smallfiles
```

## STH-Comet Configuration 

```yaml
  sth-comet:
    image: fiware/sth-comet
    hostname: sth-comet
    container_name: fiware-sth-comet
    depends_on:
        - mongo-db
    networks:
        - default
    ports:
        - "8666:8666"
    environment:
        - STH_HOST=0.0.0.0
        - STH_PORT=8666
        - DB_PREFIX=sth_
        - DB_URI=mongo-db:27017
        - LOGOPS_LEVEL=DEBUG
```

## Cygnus Configuration 

```yaml
  cygnus:
    image: fiware/cygnus-ngsi:latest
    hostname: cygnus
    container_name: fiware-cygnus
    depends_on:
        - mongo-db
    networks:
        - default
    expose:
        - "5080"
    ports:
        - "5050:5050"
        - "5080:5080"
    environment:
        - "CYGNUS_MONGO_HOSTS=mongo-db:27017"
        - "CYGNUS_LOG_LEVEL=DEBUG"
        - "CYGNUS_SERVICE_PORT=5050"
        - "CYGNUS_API_PORT=5080"
```

The `sth-comet` container is listening on one port: 

* The Operations for port for STH-Comet - `8666` is where the service will be listening for time based query requests from cUrl or Postman

The `sth-comet` container is driven by environment variables as shown:

| Key        |Value            |Description|
|------------|-----------------|-----------|
|STH_HOST    |`0.0.0.0`        | The address where STH-Comet is hosted - within this container it means all IPv4 addresses on the local machine  |
|STH_PORT    |`8666`           | Operations Port that STH-Comet listens on |
|DB_PREFIX   |`sth_`           | The prefix added to each database entity if none is provided |
|DB_URI      |`mongo-db:27017` | The  Mongo-DB server which STH-Comet will contact to persist historical context data |
|LOGOPS_LEVEL|`DEBUG`          | The logging level for STH-Comet |

The `cygnus` container is listening on two ports: 

* The Subscription Port for Cygnus - `5050` is where the service will be listening for notifications from the Orion context broker
* The Management Port for Cygnus - `5080` is exposed purely for tutorial access - so that cUrl or Postman can make provisioning commands
  without being part of the same network.


The `cygnus` container is driven by environment variables as shown:

| Key                           |Value         |Description|
|-------------------------------|--------------|-----------|
|CYGNUS_MONGO_HOSTS         |`mongo-db:27017`  | Comma separated list of Mongo-DB servers which Cygnus will contact to persist historical context data |
|CYGNUS_LOG_LEVEL               |`DEBUG`       | The logging level for Cygnus |
|CYGNUS_SERVICE_PORT            |`5050`        | Notification Port that Cygnus listens when subscribing to context data changes|
|CYGNUS_API_PORT                |`5080`        | Port that Cygnus listens on for operational reasons |



## *formal* mode - Start up

To start the system using the *formal* configuration using **Cygnus**  and **STH-Comet**, run the following command:

```console
./services cygnus
``` 


### Checking the STH-Comet Service Health
 
Once **STH-Comet** is running, You can check the status by making an HTTP request to the exposed `STH_PORT` port. 
If the response is blank, this is usually because **STH-Comet** is not running or is listening on another port.


#### Request:

```console
curl -X GET \
  'http://localhost:8666/version'
```

#### Response:

The response will look similar to the following:

```json
{
    "version": "2.3.0-next"
}
```

### Checking the Cygnus Service Health
 
Once **Cygnus** is running, you can check the status by making an HTTP request to the exposed `CYGNUS_API_PORT` port. 
If the response is blank, this is usually because **Cygnus** is not running or is listening on another port.

#### Request:

```console
curl -X GET \
  'http://localhost:5080/v1/version'
```

#### Response:

The response will look similar to the following:

```json
{
    "success": "true",
    "version": "1.8.0_SNAPSHOT.ed50706880829e97fd4cf926df434f1ef4fac147"
}
```

>**Troubleshooting:** What if either response is blank ?
>
> * To check that a docker container is running try
>
>```bash
>docker ps
>```
>
>You should see several containers running. If `sth-comet`  or `cygnus` is not running, you can restart the containers as necessary.

### Generating Context Data

For the purpose of this tutorial, we must be monitoring a system where the context is periodically being updated.
The dummy IoT Sensors can be used to do this. Open the device monitor page at `http://localhost:3000/device/monitor`
and unlock a **Smart Door** and switch on a **Smart Lamp**. This can be done by selecting an appropriate the command 
from the drop down list and pressing the `send` button. The stream of measurements coming from the devices can then
be seen on the same page:

![](https://fiware.github.io/tutorials.Short-Term-History/img/door-open.gif)

## *formal* mode - Subscribing Cygnus to Context Changes

In *formal* mode, **Cygnus** is responsible for the persistance of historic context data.
Once a dynamic context system is up and running, we need to set up a subscription in the **Orion Context Broker**
to notify **Cygnus**  of changes in context - **STH-Comet** will only be used to read the persisted data.


### Cygnus - Aggregate Motion Sensor Count Events

The rate of change of the **Motion Sensor** is driven by events in the real-world. We need to receive
every event to be able to aggregate the results.

This is done by making a POST request to the `/v2/subscription` endpoint of the **Orion Context Broker**.

* The `fiware-service` and `fiware-servicepath` headers are used to filter the subscription to only listen to measurements from the attached IoT Sensors
* The `idPattern` in the request body ensures that **Cygnus**  will be informed of all **Motion Sensor** data changes.
* The notification `url` must match the configured `CYGNUS_API_PORT`
* The `attrsFormat=legacy` is required since **Cygnus** currently only accepts notifications in the older NGSI v1 format.

#### Request:

```console
curl -iX POST \
  'http://localhost:1026/v2/subscriptions/' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify Cygnus of all Motion Sensor context changes",
  "subject": {
    "entities": [
      {
        "idPattern": "Motion.*"
      }
    ]
  },
  "notification": {
    "http": {
      "url": "http://cygnus:5050/notify"
    },
    "attrsFormat": "legacy"
  }
}'
```

### Cygnus - Sample Lamp Luminosity

The luminosity of the **Smart Lamp** is constantly changing, we only need to **sample** the values to be
able to work out relevant statistics such as minimum and maximum values and rates of change.

This is done by making a POST request to the `/v2/subscription` endpoint of the **Orion Context Broker**
and including the `throttling` attribute in the request body.

* The `fiware-service` and `fiware-servicepath` headers are used to filter the subscription to only listen to measurements from the attached IoT Sensors
* The `idPattern` in the request body ensures that **Cygnus** will be informed of all **Smart Lamp** data changes only
* The notification `url` must match the configured `CYGNUS_API_PORT`
* The `attrsFormat=legacy` is required since **Cygnus** currently only accepts notifications in the older NGSI v1 format.
* The `throttling` value defines the rate that changes are sampled.

#### Request:

```console
curl -X POST \
  'http://localhost:1026/v2/subscriptions/' \
  -H 'Content-Type: application/json' \
  -H 'fiware-service: openiot' \
  -H 'fiware-servicepath: /' \
  -d '{
  "description": "Notify Cygnus to sample Lamp changes every five seconds",
  "subject": {
    "entities": [
      {
        "idPattern": "Lamp.*"
      }
    ]
  },
  "notification": {
    "http": {
      "url": "http://cygnus:5050/notify"
    },
    "attrsFormat": "legacy"
  },
  "throttling": 5
}'
```

## *formal* mode - Time Series Data Queries

When reading data from the database, there is no difference between *minimal* and *formal* mode, please refer to the previous
section of this tutorial to request time-series data from **STH-Comet**

# Next Steps

Want to learn how to add more complexity to your application by adding advanced features?
You can find out by reading the other tutorials in this series:

&nbsp; 101. [Getting Started](https://github.com/Fiware/tutorials.Getting-Started)<br/>
&nbsp; 102. [Entity Relationships](https://github.com/Fiware/tutorials.Entity-Relationships)<br/>
&nbsp; 103. [CRUD Operations](https://github.com/Fiware/tutorials.CRUD-Operations)<br/>
&nbsp; 104. [Context Providers](https://github.com/Fiware/tutorials.Context-Providers)<br/>
&nbsp; 105. [Altering the Context Programmatically](https://github.com/Fiware/tutorials.Accessing-Context)<br/> 
&nbsp; 106. [Subscribing to Changes in Context](https://github.com/Fiware/tutorials.Subscriptions)<br/>

&nbsp; 201. [Introduction to IoT Sensors](https://github.com/Fiware/tutorials.IoT-Sensors)<br/>
&nbsp; 202. [Provisioning an IoT Agent](https://github.com/Fiware/tutorials.IoT-Agent)<br/>

&nbsp; 301. [Persisting Context Data](https://github.com/Fiware/tutorials.Historic-Context)<br/>
&nbsp; 302. [Querying Time Series Data](https://github.com/Fiware/tutorials.Short-Term-History)<br/>
