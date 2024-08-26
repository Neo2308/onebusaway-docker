# OneBusAway Docker Images

<a href="https://hub.docker.com/u/opentransitsoftwarefoundation"><img alt="Official Docker images" src="https://img.shields.io/badge/Docker_Hub-images-green?logo=docker"></a> <img alt="GitHub Actions Workflow Status" src="https://img.shields.io/github/actions/workflow/status/onebusaway/onebusaway-docker/test.yaml?branch=main">

This repository contains scripts and configuration for building version 2 of the
[OneBusAway Application Suite](https://github.com/OneBusAway/onebusaway-application-modules)
for use with [Docker](https://www.docker.com/).

## Running locally

To build bundles and run the webapp server with your own GTFS feed, use the [Docker Compose](https://docs.docker.com/compose/) services in this repository.

### Building bundles

To build a bundle, use the `oba_bundler` service:

```bash
GTFS_URL=https://www.soundtransit.org/GTFS-rail/40_gtfs.zip docker-compose up oba_bundler
```

This process will create all necessary bundle files and metadata, and all will be accessible in your local repo's `./bundle` directory.

When the GTFS_URL is unspecified, `oba_bundler` will download and use the GTFS data for Davis, CA's Unitrans service. This can be used with the `bin/validate.sh` script to verify that the stack is working correctly.

```bash
docker-compose up oba_bundler
```

### Running the OneBusAway server

Once you have a built OBA bundle inside `./bundle`, you can run the OBA server and make it accessible on your host machine with:

```bash
docker-compose up oba_app
```

You will then have three webapps available:

- API, hosted at `http://localhost:8080/onebusaway-api-webapp/api?key=TEST`
  - an example call could be to `http://localhost:8080/onebusaway-api-webapp/api/where/agencies-with-coverage.json?key=TEST`, which should show metadata about the agency you loaded
  - the test/demo API key is automatically handled in `oba/bootstrap.sh`, you can change it by setting the `TEST_API_KEY` environment variables in the `oba_app` service in `docker-compose.yml`

When done using this web server, you can use the shell-standard `^C` to exit out and turn it off. If issues persist across runs, you can try using `docker-compose down -v` and then `docker-compose up oba_app` to refresh the Docker containers and services.

### Inspecting the database

The MySQL database Docker Compose service should remain up after a call of `docker-compose up oba_app`. Otherwise, you can always invoke it using `docker-compose up oba_database`.

A database port is open to your host machine, so you can connect to it programmatically using `mysql`:

```bash
mysql -u oba_user -p -h localhost:3306
```

## Deployment

### Published Images

You can find the latest published Docker images on Docker Hub:

* [onebusaway-bundle-builder](https://hub.docker.com/r/opentransitsoftwarefoundation/onebusaway-bundle-builder) - This image is built from the `bundler` directory and contains the functionality needed to create a transit data bundle from a GTFS feed.
* [onebusaway-api-webapp](https://hub.docker.com/r/opentransitsoftwarefoundation/onebusaway-api-webapp) - This image is built from the `oba` directory and contains the functionality needed to run the OBA API webapp.

### Deployment Parameters

* Database
  * `JDBC_URL` - The JDBC connection URL for your MySQL database.
  * `JDBC_USER` - The username for your MySQL database.
  * `JDBC_PASSWORD` - The password for your MySQL database.
* GTFS (Optional, required only when using `oba_app` independently)
  * `GTFS_URL` - The URL to the GTFS feed you want to use.
* GTFS-RT Support (Optional)
  * `TZ` - The timezone for the server. Ensure that the server's timezone matches the timezone specified in your static GTFS `agency.txt` file. The timezone format is the IANA standard, and [a full list of timezones can be found on Wikipedia](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List).
  * `ALERTS_URL` - Service Alerts URL for GTFS-RT.
  * `TRIP_UPDATES_URL` - Trip Updates URL for GTFS-RT.
  * `VEHICLE_POSITIONS_URL` - Vehicle Positions URL for GTFS-RT.
  * `REFRESH_INTERVAL` - Refresh interval in seconds. Usually 10-30.
  * `AGENCY_ID` - Your GTFS-RT agency ID. Ostensibly the same as your GTFS agency ID.
  * Authentication (Optional)
    * Example: Specifying `FEED_API_KEY` = `X-API-KEY` and `FEED_API_VALUE` = `12345` will result in `X-API-KEY: 12345` being passed on every call to your GTFS-RT URLs.
    * `FEED_API_KEY` - If your GTFS-RT API requires you to pass an authentication header, you can represent the key portion of it by specifying this value.
    * `FEED_API_VALUE` - If your GTFS-RT API requires you to pass an authentication header, you can represent the value portion of it by specifying this value.



The `GTFS-RT` and `Google Map` related variables will be handled by the `oba/bootstrap.sh` script, which will set the config files for the OBA API webapp. If you want to use your own config files, you could set `USER_CONFIGURED=1` in the `oba_app` service in `docker-compose.yml` to skip `bootstrap.sh` and write your config file in the container.
```yaml
  oba_app:
    container_name: oba_app
    depends_on:
      - oba_database
    build:
      context: ./oba
    environment:
      # database configs are read from environment variables
      - JDBC_URL=jdbc:mysql://oba_database:3306/oba_database
      - JDBC_USER=oba_user
      - JDBC_PASSWORD=oba_password
      # change this to your GTFS url
      - GTFS_URL=https://unitrans.ucdavis.edu/media/gtfs/Unitrans_GTFS.zip
      # skip bootstrap.sh and use user-configured config files
      - USER_CONFIGURED=1
```

You will also need to create a transit data bundle from a GTFS Zip file. This needs more documentation, but this README does a decent job of outlining the process. The only tricky part is that you need to get it into your running container. Currently, we recommend building it locally, uploading the contents of the `./bundle` directory to S3 or another publicly accessible website, and then downloading it into your container. Obviously, this needs some improvement.

### Deploy to Render

[Render](https://www.render.com) is an easy-to-use Platform-as-a-Service (PaaS) provider. You can host OneBusAway on Render by either manually configuring it or by clicking the button below.

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/oneBusAway/onebusaway-docker/)

## Running in Kubernetes

### a) Using kubernetes resource yaml files (and locally built images):

#### Creating the docker images

1. Build the bundler image:

```bash
docker build ./bundler -t oba/bundler:test
```

2. Build the app image:

```bash
docker build ./oba -t oba/app:test
```

#### Creating the Kubernetes resources:

Apply the Kubernetes resources in oba.yaml

```bash
kubectl apply -f oba.yaml
```

The YAML file deploys the OneBusAway application and a MySQL database within a dedicated oba namespace in Kubernetes. It also sets up a secret for sensitive data and a ConfigMap for GTFS data URL, while exposing the database as a service for other pods to access.

You can portforward the oba app to your localhost using:

```bash
kubectl port-forward deploy/oba-app-deployment -n oba 8080:8080
```

#### Inspecting the database

You can portforward the service to your localhost using:

```bash
kubectl port-forward service/oba-database -n oba 3306:3306
```

Then you can connect to it programmatically using `mysql`:

```bash
mysql -u oba_user -p -h localhost:3306
```

### b) Using helm charts:

#### Installation

1. Create a namespace for the OneBusAway deployment:
    ```bash
    kubectl create namespace oba
    ```
2. Create a secret for sensitive data:
    ```bash
    kubectl create secret generic oba-secrets --from-literal=MYSQL_ROOT_PASSWORD=root_password --from-literal=JDBC_PASSWORD=oba_password -n oba
    ```
    _Note: Additional keys may be required if using Google Maps._
3. Create a file (eg: `oba-values.yaml`) containing your configurations:
    ```yaml
    gtfsURL: "https://unitrans.ucdavis.edu/media/gtfs/Unitrans_GTFS.zip"
    ```
4. Install the OneBusAway helm chart:
    ```bash
    helm install oba-server ./helm/one-bus-away -n oba -f oba-values.yaml
    ```

#### Using oba app on localhost

You can portforward the oba app to your localhost using:
```bash
kubectl port-forward deploy/oba-app-deployment -n oba 8080:8080
```
_Note: Sample apis can be found in `bin/validate.sh`_

#### Inspecting the database

1. You can portforward the service to your localhost using:
    ```bash
    kubectl port-forward service/oba-database -n oba 3306:3306
    ```
2. Then you can connect to it programmatically using `mysql`:
    ```bash
    mysql -u oba_user -p -h localhost:3306
    ```
   _Note: If the above command fails, try `mysql -u oba_user -p --host=127.0.0.1 --port=3306` instead._ 

## Using Google Maps

#### Prerequisites

- Have a valid Google Cloud Platform (GCP) account.
- Create a project on GCP and enable the Maps JavaScript API for it.
- Create an API key for your project. [See how to create an API key](https://cloud.google.com/docs/authentication/api-keys).
- If there is a rate limit error while using the test API key, it means you have reached the maximum usage limit for that key, you need to generate a new API key.

#### Configuring Google Maps

You'll need to set the following environment variables:

- `GOOGLE_MAPS_API_KEY`: Your Google Maps API key.
- `GOOGLE_MAPS_CLIENT_ID`: (Optional) Required if you are using the Google Maps Premium Plan.
- `GOOGLE_MAPS_CHANNEL_ID`: (Optional) A channel ID that helps identify the source of API requests for analytics and reporting purposes.

#### Docker Configuration

If using Docker Compose:

1.Modify the 'docker-compose.yml' file like this:

```yaml
services:
  oba-app:
    environment:
        - GOOGLE_MAPS_API_KEY=<YOUR_KEY_HERE>
        - GOOGLE_MAPS_CHANNEL_ID=<YOUR_CHANNEL_ID_HERE>
        - GOOGLE_MAPS_CLIENT_ID=<YOUR_CLIENT_ID_HERE>
```

2.Use the following command to start the oba-app service:

```bash
docker-compose up -d oba-app
```


If deployed in Kubernetes environment:

1.Use the kubectl set env command to set new environment variables,
make sure you replace deployment/oba-app with the actual name of your deployment:

```bash
kubectl set env deployment/oba-app GOOGLE_MAPS_API_KEY=<YOUR_KEY_HERE> \
    GOOGLE_MAPS_CHANNEL_ID=<YOUR_CHANNEL_ID_HERE> \
    GOOGLE_MAPS_CLIENT_ID=<YOUR_CLIENT_ID_HERE>
```

2.Use the following command to rebuild and start the oba app service:

```bash
kubectl rollout restart deployment/oba-app
```
